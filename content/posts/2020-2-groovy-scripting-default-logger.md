+++
title = "Obtaining clean exec startup of Groovy scripts with logging dependencies"
date = "2020-02-10"
tags = ["groovy"]
+++

I write a lot about leveraging Groovy scripts to create little bits of functionality that do a range of workloads; big and small. The nature of these workloads means they're classified closer to tooling than applications meaning I don't tend to care about proper logging. And, as such, it's irritating when startup isn't clean, like when additional lines are output.

Take the following example (saved as `jedisPoolExample`):

```groovy
#!/usr/bin/env groovy

@Grapes([
  @Grab(group='redis.clients', module='jedis', version='3.2.0')
])

import redis.clients.jedis.JedisPool

def jedisPool = new JedisPool()
```

In this example we resolve the [jedis](https://github.com/xetorthio/jedis) dependency, open a JedisPool, and exit. Pretty simple, right? Running this
script, however, shows that it violates the ideal that things startup clean:

```
./jedisPoolExample
SLF4J: Failed to load class "org.slf4j.impl.StaticLoggerBinder".
SLF4J: Defaulting to no-operation (NOP) logger implementation
SLF4J: See http://www.slf4j.org/codes.html#StaticLoggerBinder for further details.
```

So how do we fix this? Since [SLF4J](http://www.slf4j.org/manual.html) version 1.6.0, if no default logger implementation is set or resolvable in the
classpath, SLF4J will default to a no-operation mode, meaning no logs. Great! That's what I want for these basic scripts, maybe its what you want to?!
The real question, though, is how do we get rid of those three lines. The answer is simple, include the [slf4j-nop](https://mvnrepository.com/artifact/org.slf4j/slf4j-nop) implementation of the logger in the class path via a [Grape](https://docs.groovy-lang.org/latest/html/documentation/grape.html) directive. There's already one for Jedis...

``` groovy
#!/usr/bin/env groovy

@Grapes([
  @Grab(group='redis.clients', module='jedis', version='3.2.0'),
  @Grab(group='org.slf4j', module='slf4j-nop', version='1.7.30')
])

import redis.clients.jedis.JedisPool

def jedisPool = new JedisPool()
```

With this change, startup of our script is clean.
