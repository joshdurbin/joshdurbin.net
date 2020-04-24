+++
title = "Clean, easy concurrent inserts with Jedis and Groovy"
date = "2020-02-19"
tags = ["groovy", "concurrency"]
+++

Building on the example of a prior post, this covers creating a simple, succinct process for concurrently
issuing calls against Redis.

Our starting point is the following block of code where we open a pool against the default host/port of Redis,
which is 'localhost' and '6379':

```groovy
#!/usr/bin/env groovy

@Grapes([
  @Grab(group='redis.clients', module='jedis', version='3.2.0'),
  @Grab(group='org.slf4j', module='slf4j-nop', version='1.7.30')
])

import redis.clients.jedis.JedisPool

def jedisPool = new JedisPool()
```

As mentioned in the cited, prior post, this doesn't really do anything, though. So, to make it do things, we need
to grab a connection from the pool and operate on it...

```groovy
#!/usr/bin/env groovy

@Grapes([
  @Grab(group='redis.clients', module='jedis', version='3.2.0'),
  @Grab(group='org.slf4j', module='slf4j-nop', version='1.7.30')
])

import redis.clients.jedis.JedisPool

def jedisPool = new JedisPool()
def jedis = jedisPool.getResource()
jedis.set('thing', 'value')
```

While connected to the Redis instance via the cli I can see the key is set to our value and all is great in the world, yay...

```
127.0.0.1:6379> get thing
"value"
```

There are, however, a few issues with our code...

1. We aren't properly closing connections
2. We're using a pool, which is maintaining a set number of connections to Redis, but we aren't concurrently accessing or leveraging them in our code and thus the usage of the Pool is moot.

Addressing these is simple. First, objects that open long-standing connections/streams/etc... implement the [Closeable](https://docs.oracle.com/javase/7/docs/api/java/io/Closeable.html) interface and standard practice is to close these resources when you're done with them. Groovy has an opinionated way of dealing with this using [closures](http://groovy-lang.org/closures.html), specifically, the [`withCloseable`](https://docs.groovy-lang.org/latest/html/groovy-jdk/java/io/Closeable.html) closure. This works by invoking this closure on the `Closeable` resource and when the closure exits, the resource is automatically closed with no additional effort on your part.

```groovy
#!/usr/bin/env groovy

@Grapes([
  @Grab(group='redis.clients', module='jedis', version='3.2.0'),
  @Grab(group='org.slf4j', module='slf4j-nop', version='1.7.30')
])

import redis.clients.jedis.JedisPool

def jedisPool = new JedisPool()
jedisPool.getResource().withCloseable { jedis ->
  jedis.set('thing', 'value')
}
```

The second part here is how to use this pool concurrently, which is a bit more involved given we need to break into concurrency basics. But first, let's review the performance of setting 100,000 keys in Redis using just a single connection.

```groovy
#!/usr/bin/env groovy

@Grapes([
  @Grab(group='redis.clients', module='jedis', version='3.2.0'),
  @Grab(group='org.slf4j', module='slf4j-nop', version='1.7.30')
])

import redis.clients.jedis.JedisPool

def jedisPool = new JedisPool()
jedisPool.getResource().withCloseable { jedis ->
  100_000.times { number ->
    jedis.set("${number}", 'some val')
  }
}
```

Running this piece of code shows that the Redis process is minimally taxed -- is averaging about %9 CPU utilization on my 6-core 2019 MacBook Pro (which is 9-12 percentage usage of one core). Redis isn't breaking a sweat and neither is our app/script because, though we have a pool of connections to Redis, we are only using one, in one Thread. Setting 100,000 keys takes 1 min and 43 seconds; `8.37s user 1.91s system 9% cpu 1:43.11 total`.

So how do we change this? If you were building an actual application you might use an Executor for job/task submission and Thread management. In groovy, in these scripts specifically, it's usually easier to just spin up Threads yourself and manage their lifecycle using other Thread-safe objects, like concurrent queues and latches. We won't use latches and queues quite ye, though. To iterate on this, we'll first create our threads, which is super easy with Groovy's `Thread.start { }` closure, giving something that looks like this:

```groovy
#!/usr/bin/env groovy

@Grapes([
  @Grab(group='redis.clients', module='jedis', version='3.2.0'),
  @Grab(group='org.slf4j', module='slf4j-nop', version='1.7.30')
])

import redis.clients.jedis.JedisPool

def jedisPool = new JedisPool()

10.times { outerCounter ->

  Thread.start {

    jedisPool.getResource().withCloseable { jedis ->

      10_000.times { innerCounter ->
        jedis.set("${innerCounter}${outerCounter}", 'some val')
      }
    }   
  }
}
```

The run time for this block of code about 3x faster: `6.45s user 2.24s system 30% cpu 28.079 total`. What happens if we increase the thread "pool" 10x to a total of 100 threads?

```groovy
#!/usr/bin/env groovy

@Grapes([
  @Grab(group='redis.clients', module='jedis', version='3.2.0'),
  @Grab(group='org.slf4j', module='slf4j-nop', version='1.7.30')
])

import redis.clients.jedis.JedisPool

def jedisPool = new JedisPool()

100.times { outerCounter ->

  Thread.start {

    jedisPool.getResource().withCloseable { jedis ->

      1_000.times { innerCounter ->
        jedis.set("${innerCounter}${outerCounter}", 'some val')
      }
    }   
  }
}
```

We see a slight reduction time from 28 seconds to 22 seconds (`8.57s user 2.61s system 49% cpu 22.460 total`), but not as much as you'd expect. The reason for this, in this particular case, is because the connection pool for Jedis is set to its default value of `8` **and** each thread is holding onto that connection for a long period of time (the time it takes to set 1000 keys). A few improvements can be done here:

1. Set each thread to release the connection in between iterations of work. The work, though, in this case, is very lightweight and we probably won't see much benefit.
2. Increase the default size of the thread pool to something closer to if not equal to the number of threads using the connection pool (8 vs 100)
3. Batch set operations against Redis.

Let's go through these one-by-one.

#### 1 - non-greedy connection usage per thread

```
#!/usr/bin/env groovy

@Grapes([
  @Grab(group='redis.clients', module='jedis', version='3.2.0'),
  @Grab(group='org.slf4j', module='slf4j-nop', version='1.7.30')
])

import redis.clients.jedis.JedisPool

def jedisPool = new JedisPool()

100.times { outerCounter ->

  Thread.start {

    1_000.times { innerCounter ->
      jedisPool.getResource().withCloseable { jedis ->
        jedis.set("${innerCounter}${outerCounter}", 'some val')
      }
    }   
  }
}
```

Here, as expected, we don't see a big reduction in the time it takes to process and return: `11.56s user 3.79s system 69% cpu 22.174 total`, maybe a few hundred milliseconds. Notice that we've swapped the inner counter loop `times` with the call that obtains a jedis connection from the pool `jedisPool.getResource()`. Next, we'll iterate on this design and add a larger connection pool...

#### 2 - increasing the connection pool size

Doing this requires the additional use of an additional library, specifically, ``.

```groovy
#!/usr/bin/env groovy

@Grapes([
  @Grab(group='redis.clients', module='jedis', version='3.2.0'),
  @Grab(group='org.slf4j', module='slf4j-nop', version='1.7.30'),
  @Grab(group='org.apache.commons', module='commons-pool2', version='2.8.0')
])

import org.apache.commons.pool2.impl.GenericObjectPoolConfig
import redis.clients.jedis.JedisPool

def config = new GenericObjectPoolConfig()
config.setMaxTotal(100)
def jedisPool = new JedisPool(config)

100.times { outerCounter ->

  Thread.start {

    1_000.times { innerCounter ->
      jedisPool.getResource().withCloseable { jedis ->
        jedis.set("${innerCounter}${outerCounter}", 'some val')
      }
    }   
  }
}
```

Here we see our biggest improvement in performance yet... a little more than 2x faster than the previous run: `10.56s user 3.69s system 145% cpu 9.811 total`. This is because we're able to exert pressure against Redis, open and maintain connections, and more efficiently spend time communicating with it.

#### 3 - use of redis pipelines

Because our operations are mutually independent we can easily "batch" them and send them as a pipeline request and optimize
for the network overhead we incur when executing many commands against Redis.

```groovy
#!/usr/bin/env groovy

@Grapes([
  @Grab(group='redis.clients', module='jedis', version='3.2.0'),
  @Grab(group='org.slf4j', module='slf4j-nop', version='1.7.30'),
  @Grab(group='org.apache.commons', module='commons-pool2', version='2.8.0')
])

import redis.clients.jedis.JedisPool
import org.apache.commons.pool2.impl.GenericObjectPoolConfig

def config = new GenericObjectPoolConfig()
config.setMaxTotal(100)
def jedisPool = new JedisPool(config)

100.times { outerCounter ->

  Thread.start {

    10.times { innerCounter ->
      jedisPool.getResource().withCloseable { jedis ->

        def pipeline = jedis.pipelined()

        100.times { pipelinedCounter ->
          pipeline.set("${innerCounter}${outerCounter}${pipelinedCounter}", 'some val')
        }

        pipeline.sync()
      }
    }   
  }
}
```

In this example I wanted to retain the idea of efficient, non-greedy thread use by splitting the inner loop into two loops for a total of 3; an outer one for the threads, inner for a loop where we obtain a connection for each, and a pipelined loop which essentially is our "batch size" for operations against Redis.

Compared with our initial, single-threaded run, we're more than two orders of magnitude faster: `7.29s user 0.73s system 191% cpu 4.183 total` vs the `8.37s user 1.91s system 9% cpu 1:43.11 total`. Notice, too, the kernel scheduler for the time counting during the execution of both these commands shows far greater concurrent use of available CPU time from 9 percent to 191 percent.

### Separating production and consumption

We've done a lot of iterative change in this post and this will be the biggest change yet... Often, not always, but often, it makes sense to have the producing and consumption systems / threads different because, typically, the work is very different in nature; one might be CPU, another disk or network bound, etc...

We're going to alter this code a bit to achieve the same outcome as the others, but in a model that's more in-line with the [producer-consumder paradigm](https://en.wikipedia.org/wiki/Producer–consumer_problem).

To do so, we're going to do the following:

1. Create a concurrent queue for which we'll add entries to process
2. Create a smaller, dedicated set "pool" (but not a pool) of Threads to do the work
3. Loop 100,000 times on the main thread putting data into the queue
4. Use an AtomicBoolean to gate, signal to the threads that we're done producing work

```groovy
#!/usr/bin/env groovy

@Grapes([
  @Grab(group='redis.clients', module='jedis', version='3.2.0'),
  @Grab(group='org.slf4j', module='slf4j-nop', version='1.7.30'),
  @Grab(group='org.apache.commons', module='commons-pool2', version='2.8.0')
])

import java.util.concurrent.ConcurrentLinkedQueue
import java.util.concurrent.atomic.AtomicBoolean
import redis.clients.jedis.JedisPool
import org.apache.commons.pool2.impl.GenericObjectPoolConfig

def numThreads = 10
def pipelineSize = 200

def queue = new ConcurrentLinkedQueue()
def finished = new AtomicBoolean(false)

def config = new GenericObjectPoolConfig()
config.setMaxTotal(numThreads)
def jedisPool = new JedisPool(config)

numThreads.times {

  Thread.start {

    while (true) {

      def item = queue.poll()

      if (item) {

        jedisPool.getResource().withCloseable { jedis ->

          def pipeline = jedis.pipelined()
          def pipelineCount = 0

          while (item) {
            pipeline.set("${item}", 'some val')

            if (++pipelineCount < pipelineSize) {
              item = queue.poll()
            } else {
              item = null
            }  
          }

          pipeline.sync()
        }
      } else if (finished.get()) {
        break
      }
    }
  }
}

100_000.times { number ->
  queue.offer(number)
}

finished.set(true)
```

Reading through the code, it's clear the work is isolated, for the most part, to the Threads and we do
some basic optimizations in those Threads like:

1. Only request a connection from the pool if the queue has something for the Thread to work on
2. Once we have an item, keep local count and take up to the max batch / pipeline size, then commit that pipeline
3. If the queue doesn't have anything for the Thread to work on, check if the finished atomic boolean is set to true and if so, exit

With these modifications and much smaller pool of workers, `10` we're able to insert 100k entries, in 4 seconds: `7.02s user 0.41s system 184% cpu 4.024 total`.

I find concurrent programming enjoyable in Java (really, Java by proxy by way of Groovy) and Go (for another time) and hope you found this interesting. The [javadocs](https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/util/concurrent/package-summary.html) for java's `java.util.concurrent` package is a nice resource for learning about the various locks, semaphores, data structures with blocking/polling support, etc...

Cheers
