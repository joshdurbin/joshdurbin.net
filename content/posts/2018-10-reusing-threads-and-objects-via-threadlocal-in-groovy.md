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
    new ExpensiveToConstructObject()
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

def executor = new ThreadPoolExecutor(1, 2, 5, TimeUnit.SECONDS, new LinkedBlockingQueue(5),
  new ReusableThreadFactory(), new ThreadPoolExecutor.CallerRunsPolicy())

def latch = new CountDownLatch(loops)

loops.times {
  println "submitting job"
  executor.submit {
    latch.countDown()
    def expensiveObject = ReusableThread.expensiveObject.get()
    expensiveObject.doThing()
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

The full source of this example is [here](https://github.com/joshdurbin/blog_post_groovy_scripts/blob/master/threadlocal_groovy_post-1.groovy),
with another example [here](https://github.com/joshdurbin/blog_post_groovy_scripts/blob/master/threadlocal_groovy_post-2.groovy).
   