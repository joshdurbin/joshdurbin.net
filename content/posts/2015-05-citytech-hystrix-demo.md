+++
date = "2015-05-27T13:21:33-07:00"
title = "Intro to Hystrix"
tags = [ "citytechinc", "hystrix", "java" ]
+++

### What is Hystrix?

Hystrix is a latency and fault tolerance library designed by Netflix to
isolate, primarily, points of access remote systems, services, and 3rd party
libraries. Hystrix prevents cascading failures and enables resiliency in
complex distributed systems where failure is considered inevitable.

### What Hystrix provides

Hystrix aims to enable a system to handle unexpected, external situations,
allow for service availability scale back or service degradation in a systematic,
controlled manner.

### Applied Patterns

- Timeout
- [Circuit Breaker](http://microservices.io/patterns/reliability/circuit-breaker.html)
- Load Shedder
- Fallback

### Advanced Features

- Request Caching
- Request Collapsing
- Plugins
- Dashboard

### Basic Usage

```
public class CommandHelloWorld extends HystrixCommand<String> {

  private final String name;

  public CommandHelloWorld(String name) {
    super(HystrixCommandGroupKey.Factory.asKey("ExampleGroup"));
    this.name = name;
  }

  @Override
  protected String run() {
    // a real example would do work like a network call here
    return "Hello " + name + "!";
  }
}
```

### Sync, Blocking Usage

```
String s = new CommandHelloWorld("World").execute();

@Test
public void testSynchronous() {
  assertEquals("Hello World!", new CommandHelloWorld("World").execute());
  assertEquals("Hello Bob!", new CommandHelloWorld("Bob").execute());
}
```

### Async, Non-blocking Usage

```
Future<String> fs = new CommandHelloWorld("World").queue();

@Test
public void testAsynchronous1() throws Exception {
  assertEquals("Hello World!", new CommandHelloWorld("World").queue().get());
  assertEquals("Hello Bob!", new CommandHelloWorld("Bob").queue().get());
}
```

### RxJava, Reactive Streams Usage

```
Observable<String> ho = new CommandHelloWorld("World").observe();

ho.subscribe(new Action1<String>() {

  @Override
  public void call(String s) {
  }
});

@Test
public void testObservable() throws Exception {

  Observable<String> fWorld = new CommandHelloWorld("World").observe();

  fBob.subscribe(new Action1<String>() {

    @Override
    public void call(String v) {
      System.out.println("onNext: " + v);
    }
  });
}
```

### Basic Fallback Usage

```
public class CommandHelloWorld extends HystrixCommand<String> {

  private final String name;

  public CommandHelloWorld(String name) {
    super(HystrixCommandGroupKey.Factory.asKey("ExampleGroup"));
    this.name = name;
  }

  @Override
  protected String run() {
    // a real example would do work like a network call here
    return "Hello " + name + "!";
  }

  @Override
  protected String getFallback() {
    return "Hello Failure " + name + "!";
  }
}
```

### Error Propagation

```
public class CommandHelloWorld extends HystrixCommand<String> {

  private final String name;

  public CommandHelloWorld(String name) {
    super(HystrixCommandGroupKey.Factory.asKey("ExampleGroup"));
    this.name = name;
  }

  @Override
  protected String run() {
    throw new HystrixBadRequestException("I fail differently",
      new RuntimeException("I will always fail"));
  }

  @Override
  protected String getFallback() {
    return "Hello Failure " + name + "!";
  }
}
```

### Configuration Tweaks

- Command Execution Isolation Strategy
- Timeouts
- Circuit Breaker Thresholds
- Command Queue Sizes, Rejection Policies, etc...