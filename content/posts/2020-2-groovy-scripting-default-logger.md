+++
title = "Obtaining clean exec startup of Groovy scripts with logging dependencies"
date = "2020-02-10"
tags = ["groovy", "sl4fj"]
+++

I write a lot about leveraging Groovy scripts to create tiny bits of powerful functionality that do a range of workloads; big and small. The nature of these workloads, though, as scripted-apps, means they're closer to tooling and thus don't usually require traditional, proper logging and metrics collection. And, given that collection is thrown to the side, it's irritating when startup isn't clean, like when additional lines are output.

Take the following example (saved as `jedisPoolExample`):

```groovy
#!/usr/bin/env groovy

@Grapes([
  @Grab(group='redis.clients', module='jedis', version='3.2.0')
])

import redis.clients.jedis.JedisPool

def jedisPool = new JedisPool()
```

In this example we resolve the [jedis](https://github.com/xetorthio/jedis) dependency, open a JedisPool, and exit. Pretty simple, right? For a script that does nothing you'd expect close to nothing output, but you'd be wrong... Because JedisPool uses logging, as most libraries/dependencies do, we see some statements from the SL4J sub-system itself. These statements tell you that its found no logging implementation on the classpath and is defaulting to a NOP logger.

```
./jedisPoolExample
SLF4J: Failed to load class "org.slf4j.impl.StaticLoggerBinder".
SLF4J: Defaulting to no-operation (NOP) logger implementation
SLF4J: See http://www.slf4j.org/codes.html#StaticLoggerBinder for further details.
```

So how do we fix this? [SLF4J](http://www.slf4j.org/manual.html) uses the NOP logger if non is configured/detected on the class path since version 1.6.0. If we don't care about logs and we don't want this pesky message removing it is as simple as providing the NOP logger as a dependency to the application.

```groovy
#!/usr/bin/env groovy

@Grapes([
  @Grab(group='redis.clients', module='jedis', version='3.2.0'),
  @Grab(group='org.slf4j', module='slf4j-nop', version='1.7.30')
])

import redis.clients.jedis.JedisPool

def jedisPool = new JedisPool()
```

With this change, startup of our script is clean.
