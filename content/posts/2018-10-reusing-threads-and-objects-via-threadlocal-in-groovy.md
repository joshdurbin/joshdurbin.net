+++
title = "Reusing threads and objects via ThreadLocal"
description = "Walkthrough of executor services, custom thread factories and threads with ThreadLocal references"
date = "2018-10-02"
tags = ["groovy", "concurrency"]
+++

Threads in Java are expensive -- they require allocation of large blocks of
memory, are tracked via descriptors which have to be created and maintained by the JVM,
and require numerous system calls to register the thread with the underlying operating
system. [Thread pool executors](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ThreadPoolExecutor.html) help
solve this problem by enabling developers
to create and maintain thread pools which re-use threads as the executor responds to
queued work. This thread re-use enables quicker turn around on submitted jobs to the
executor without resource overuse by the system. At the end of the day, however, the computational content of the job really shapes
the performance characteristics of the system. Some workloads maintain references to
thread safe data structures or thread safe objects. 

Other times workloads depend on objects that are not thread safe and or are expensive to construct.
Building on the [object pool pattern](https://en.wikipedia.org/wiki/Object_pool_pattern) 
we can extend threads by augmenting them with our expensive and, or non-current objects, and pass
these objects to the workloads as they step up for processing. In this scenario each thread
maintains its own instance of our expensive object and [ThreadLocal](https://docs.oracle.com/javase/8/docs/api/java/lang/ThreadLocal.html)
is what enables this passing, this reference, to happen. ThreadLocal variables act as sort of
global variables limited in scope to the Thread they execute in; but aren't passed 
around like other variables.

Consider the following diagram where we have many, many tasks, being submitted
to an executor backed by a blocking queue and a rejection policy set to block the caller.

![image](/img/2018-10-reusing-threads-and-objects-via-threadlocal-in-groovy/0.jpg)

The executor acts as an orchestrator, taking jobs, managing submitted jobs in a queue, and managing
all the threads or "workers" and their life cycles. Using extension and interface adherence we're
able to control some of this life cycle behavior. For example, thread construction can
be configured to use a custom factory that creates custom Threads. The custom threads can be written
with ThreadLocal variables initialized presumably in an expensive operation/computation that's done once per thread, not per [runnable](https://docs.oracle.com/javase/8/docs/api/java/lang/Runnable.html). In this case the ThreadLocal variable is static and scoped to that
single Thread and any runnable body of work at any given time.

The following code illustrates the aforementioned concepts:

```groovy
import java.util.concurrent.CountDownLatch
import java.util.concurrent.ThreadPoolExecutor
import java.util.concurrent.TimeUnit
import java.util.concurrent.LinkedBlockingQueue
import java.util.concurrent.ThreadFactory
import java.util.Random
import java.lang.ThreadLocal

def loops = 10

class ExpensiveToConstructObject {
  ExpensiveToConstructObject() {
    def sleepTime = new Random().nextInt(1_000) as Long
    sleep(sleepTime)
    println "${Thread.currentThread().name} construction complete in ${sleepTime} ms"
  }
  void doThing() {
    println "${Thread.currentThread().name} running doThing()"
  }
}

class ReusableThread extends Thread {
  static def expensiveObject = ThreadLocal.withInitial({
    new ExpensiveToConstructObject()         // construct the expensive object
  })
  ReusableThread(Runnable runnable) {
    super(runnable)
  }
  void run() {
    super.run()
  }
}

class ReusableThreadFactory implements ThreadFactory {
  Thread newThread(Runnable runnable) {
    println "New thread requested..."
    new ReusableThread(runnable)
  }
}

def executor = new ThreadPoolExecutor(
  1,                                          // starting number of threads
  2,                                          // max number of threads
  5,                                          // wait time value
  TimeUnit.SECONDS,                           // wait time unit
  new LinkedBlockingQueue(5),                 // queue implementation and size
  new ReusableThreadFactory(),                // uses our thread factory ^^^
  new ThreadPoolExecutor.CallerRunsPolicy())  // blocks the caller if the queue blocks

def latch = new CountDownLatch(loops)

println "submitting jobs..."

loops.times {
  executor.submit {
		latch.countDown()
    def expensiveObject = ReusableThread.expensiveObject.get()
    expensiveObject.doThing() // do thing with an expensive object
  }
}

latch.await()
executor.shutdown()
```

First defined is the class `ExpensiveToConstructObject` which, you guessed it, is expensive to construct.
In this case its variable cost is based on a random value from 0 to 1,000 milliseconds (a second). Next is
the custom `ReusableThread` object with a static reference to a `ThreadLocal` variable
of type `ExpensiveToConstructObject`, initialized, where we incur penalty. The penalty in
this case is explicit but it could be in terms of resources (connections, etc...). Then we have our custom `ReusableThreadFactory` which is responsible for
generating instance of `ReusableThread` when instructed to by our `ExecutorService`. Additionally,
the ExecutorService is configured to start with 1 thread, scale to 2, keep threads around up
until 5 seconds of idle time, backed by a [`LinkedBlockingQueue`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/LinkedBlockingQueue.html) of max size 5. 

The final bit is our loop simulating load, submitting jobs to the ExecutorService. The "job",
a `Runnable`, retrieves a handle on the ThreadLocal variable, obtains the wrapped object and
executes a method on that object. The latch exists to keep the script from terminating early.

Execution yields output similar to the following:   

```text
submitting job
New thread requested...
submitting job
submitting job
submitting job
submitting job
submitting job
submitting job
New thread requested...
submitting job
Thread-2 construction complete in 402 ms
Thread-2 running doThing()
Thread-2 running doThing()
Thread-2 running doThing()
Thread-2 running doThing()
Thread-2 running doThing()
Thread-2 running doThing()
main construction complete in 470 ms
main running doThing()
submitting job
submitting job
Thread-2 running doThing()
Thread-2 running doThing()
Thread-1 construction complete in 973 ms
Thread-1 running doThing()
```

We observe jobs submitted to the executor with the executor responding by instantiating new threads
for its pool, followed by more jobs, more pressure and more requests for thread instantiation.
At this point we also observe the behavior of the filled queue and `ThreadPoolExecutor.CallerRunsPolicy()` blocking
the submission of new jobs. Though there aren't many jobs submitted to this executor, we do observe thread
re-use and confirmed thread, variable passing behavior made possible by ThreadLocal.    

The following example renders the thread id, loop count, and pause time designated per thread.     

```groovy
@Grapes(@Grab(group='com.google.guava', module='guava', version='26.0-jre'))

import java.util.concurrent.atomic.LongAdder
import java.util.concurrent.ConcurrentHashMap
import java.util.concurrent.CountDownLatch
import java.util.concurrent.ThreadPoolExecutor
import java.util.concurrent.TimeUnit
import java.util.concurrent.LinkedBlockingQueue
import java.util.concurrent.ThreadFactory
import java.util.Random
import java.lang.ThreadLocal
import com.google.common.base.Stopwatch

class ReusableThread extends Thread {
  static def sleepDuration = ThreadLocal.withInitial({new Random().nextInt(1_000)})
  ReusableThread(Runnable runnable) {
    super(runnable)
  }
  void run() {
    super.run()
  }
}

class ReusableThreadFactory implements ThreadFactory {
  Thread newThread(Runnable runnable) {
    new ReusableThread(runnable)
  }
}

def executor = new ThreadPoolExecutor(
  5,                                          // starting number of threads
  25,                                         // max number of threads
  5,                                          // wait time value
  TimeUnit.SECONDS,                           // wait time unit
  new LinkedBlockingQueue(50),                // queue implementation and size
  new ReusableThreadFactory(),                // uses our thread factory ^^^
  new ThreadPoolExecutor.CallerRunsPolicy())  // blocks the caller if the queue blocks

def sleepTimeCounts = new ConcurrentHashMap()
def sleepTimeToThreadId = new ConcurrentHashMap()
def latch = new CountDownLatch(1_000)         // prevent progression to output until done
def stopwatch = Stopwatch.createStarted()     // overall timing

1_000.times {
  executor.submit {
    latch.countDown()
    def sleepDuration = ReusableThread.sleepDuration.get()
    sleepTimeCounts.computeIfAbsent(sleepDuration, { k -> new LongAdder() }).increment()
    sleepTimeToThreadId.computeIfAbsent(sleepDuration, { k -> Thread.currentThread().getId() })
    sleep(sleepDuration as Long)
  }
}

latch.await()
executor.shutdown()

def sleepTimeOutput = 'thread id %-5d completed %-5d times, sleeping %-5dms %n'

sleepTimeCounts.each { sleepTime, counter ->
  System.out.format(sleepTimeOutput, sleepTimeToThreadId.get(sleepTime), counter.intValue(), sleepTime)
}

println "----> ${sleepTimeCounts.values().sum()} threads finished in ${stopwatch.elapsed(TimeUnit.MILLISECONDS)} ms"
```

Results:

```text
thread id 29    completed 35    times, sleeping 193  ms 
thread id 28    completed 88    times, sleeping 75   ms 
thread id 30    completed 17    times, sleeping 396  ms 
thread id 36    completed 33    times, sleeping 205  ms 
thread id 21    completed 11    times, sleeping 655  ms 
thread id 34    completed 82    times, sleeping 80   ms 
thread id 32    completed 46    times, sleeping 146  ms 
thread id 33    completed 9     times, sleeping 791  ms 
thread id 31    completed 88    times, sleeping 154  ms 
thread id 19    completed 20    times, sleeping 346  ms 
thread id 17    completed 20    times, sleeping 347  ms 
thread id 26    completed 220   times, sleeping 28   ms 
thread id 18    completed 7     times, sleeping 991  ms 
thread id 24    completed 24    times, sleeping 287  ms 
thread id 35    completed 31    times, sleeping 223  ms 
thread id 1     completed 29    times, sleeping 225  ms 
thread id 25    completed 16    times, sleeping 422  ms 
thread id 15    completed 65    times, sleeping 102  ms 
thread id 13    completed 9     times, sleeping 814  ms 
thread id 23    completed 9     times, sleeping 752  ms 
thread id 22    completed 23    times, sleeping 304  ms 
thread id 14    completed 9     times, sleeping 755  ms 
thread id 16    completed 19    times, sleeping 373  ms 
thread id 20    completed 36    times, sleeping 186  ms 
thread id 37    completed 54    times, sleeping 124  ms 
----> 1000 threads finished in 6858 ms
```

