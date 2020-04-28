+++
title = "Groovy/Java concurrency patterns with Jedis"
date = "2020-02-19"
tags = ["groovy", "concurrency", "redis"]
+++

Building on the example of a [prior post]({{< ref "/posts/2020-1-redis-graph-product-recommendation-part-1-data-loading.md" >}}), this covers creating a simple, succinct process for concurrently
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

There is nothing magical about this block of code -- it doesn't really do anything. To make it do things we need
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

Here we grab a connection a set the key `thing` to value `value`. Groundbreaking, right? We can verify our operation by connecting to the Redis instance via the CLI:

```
127.0.0.1:6379> get thing
"value"
```

There are a few issues with our code, though, even without necessarily, squarely considering concurrency:

1. We aren't properly closing connections
2. We're using a pool, which is maintaining a set number of connections to Redis, but we aren't concurrently accessing or leveraging them in our code and thus the usage of the Pool is moot.

Addressing these is simple. First, objects that open long-standing connections/streams/etc... implement the [Closeable](https://docs.oracle.com/javase/7/docs/api/java/io/Closeable.html) interface and standard practice is to close these resources when you're done with them. Groovy has a way of dealing with this using [closures](http://groovy-lang.org/closures.html), specifically, the [`withCloseable`](https://docs.groovy-lang.org/latest/html/groovy-jdk/java/io/Closeable.html) closure. This works by invoking this closure on the `Closeable` resource and when the closure exits, the resource is automatically closed with no additional effort on your part.

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

Before we break into the optimizations required of our application code to fully leverage the performance of Redis let's establish a baseline for how quickly we can perform certain operations. The following is an iteration of the former block
that creates 100,000 keys.

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

Running this piece of code shows that the application process is minimally taxed -- is averaging about %9 CPU utilization on my 6-core 2019 MacBook Pro (which is 9-12 percentage usage of one core). Redis isn't breaking a sweat, either. This is because our application code is using a single connection from the thread pool. The performance of such an approach means setting 100,000 keys takes 1 min and 43 seconds; `8.37s user 1.91s system 9% cpu 1:43.11 total`. The time output is captured by running the particular command, i.e. `groovy script.groovy`, as a child process of the `time` command, i.e. `time groovy script.groovy`. The output of this command is a bit different between BSD/Darin/MacOS and Linux, but effectively is broken into three buckets:

1. `system` time, or time spent executing kernel operations
2. `user` time, or time spent in user-land, which is typically your code
3. the final time is the ["wall clock"](https://en.wikipedia.org/wiki/Elapsed_real_time) time, meaning the time from invocation from exit -- this is the time you experience as a user

The fourth metric output is the total CPU over the scheduled time period for the process. In this case that's 9 percent, meaning that during the process consumed 9% of one cores available clock ticks for the given period.

So how do improve this -- one minute and 43 seconds seems like a long time? If you were building an actual application you might use an Executor for job/task submission and Thread management. In Groovy, in these scripts specifically, it is usually easier to just spin up Threads yourself and manage their lifecycle using other Thread-safe objects, like concurrent queues and latches. We won't use latches and queues quite ye, though. To iterate on this, we'll first create our threads, which is super easy with Groovy's `Thread.start { }` closure, giving something that looks like this:

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

In the former block we iterated by splitting the work up a bit; we start 10 threads and have each Thread run 10,000 loops. The run time for this block of code about 3x faster: `6.45s user 2.24s system 30% cpu 28.079 total`. What happens if we increase the thread "pool" 10x to a total of 100 threads?

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

    jedisPool.getResource().withCloseable { jedis ->

      1_000.times { innerCounter ->
        jedis.set("${innerCounter}${outerCounter}", 'some val')
      }
    }   
  }
}
```

Here we create 100 threads and reduce the iterations per thread to 1,000 (down from 10,000) and we observe a slight reduction time from 28 seconds to 22 seconds (`8.57s user 2.61s system 49% cpu 22.460 total`) -- not as much as you'd expect with the increase in threads. The reason for this, in this particular case, is because the connection pool for Jedis is set to its default value of `8` **and** each thread is holding onto that connection for a long period of time (the time it takes to set 1000 keys). A few improvements can be done here:

1. Make each thread non-greedy with the connection pool by configuring each to release the connection in between iterations of work. This is particularly useful when there's _more_ work being done in the threads. Given our threads are essentially string formatting and concatenating values, we shouldn't expect much of an increase.
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

Doing this requires the additional use of an additional library, specifically, [Apache Commons Pool](https://commons.apache.org/proper/commons-pool/).

```
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

The final optimization will be to maximize the efficiency of each call to Redis by leveraging [pipelineing](https://redis.io/topics/pipelining). Because our operations are mutually independent we can easily "batch" them and send them together resulting in fewer network operations against Redis.

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
