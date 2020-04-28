+++
title = "Building CLIs with Groovy's CLI Builder"
date = "2020-03-07"
tags = ["groovy", "cli"]
+++

Howdy, friends! You know what's better than heavy lifting, great performing (forgetting about startup expense), scripts and
command line tools on the JVM? You guessed it -- ones that provide well documented and user-friendly parameters and interfaces
for knob spinning and tweaking.

So, then, you've probably guessed this post is about Groovy's [`CliBuilder`](https://docs.groovy-lang.org/latest/html/gapi/groovy/cli/commons/CliBuilder.html). You. are. CORRECT!

The referenced Groovy/Java doc is actually pretty good, but I wanted to share some examples and pro tips... Let me introduce you to my `littleFriend`:

```groovy
./littleFriend -h
usage: littleFriend
Little Friend CLI
 -dt,--doubleThing <arg>   Some double. [defaults to '1.09']
 -h,--help                 Usage Information
 -nt,--numberThing <arg>   Some number. [defaults to '6200']
 -st,--stringThing <arg>   Some string. [defaults to 'How now little friend?']
```

`littleFriend` is a thing of wonder -- helping you and I understand `CliBuilder`. Here we have a little utility that takes 3 optional parameters/arguments:

1. `stringThing` (also `-st`)
2. `doubleThing` (also `-dt`)
3. `numberThing` (also `-nt`)

... and has another flag `-h` or `--help` which is parameter-less that renders the prior help menu. The code for this straight forward:

```groovy
#!/usr/bin/env groovy

def

def defaultStringThing = 'How now little friend?'
def defaultDoubleThing = '1.09'
def defaultNumberThing = "${1550 * 4}"

def cli = new CliBuilder(header: 'Little Friend CLI', usage:'littleFriend', width: -1)
cli.st(longOpt: 'stringThing', "Some string. [defaults to '${defaultStringThing}']", args: 1, defaultValue: defaultStringThing)
cli.dt(longOpt: 'doubleThing', "Some double. [defaults to '${defaultDoubleThing}']", args: 1, defaultValue: defaultDoubleThing)
cli.nt(longOpt: 'numberThing', "Some number. [defaults to '${defaultNumberThing}']", args: 1, defaultValue: defaultNumberThing)
cli.h(longOpt: 'help', 'Usage Information')

def cliOptions = cli.parse(args)

if (!cliOptions) {
  cli.usage()
  System.exit(-1)
}

if (cliOptions.help) {
  cli.usage()
  System.exit(0)
}
```

There are a few things to point out:

1. CLI options always have a short param ex (`cli.st`) and optionally have a long option `cli.st(longOpt: 'stringThing' ...`
2. Default values for parameterized options must **always** be strings, ex: `def defaultNumberThing = "${1550 * 4}"`
3. You must cast each value pulled from CliBuilder after parsing unless you've specified a type or converter
4. If you use a convert you cannot specify type
5. Validation, even printing the usage in a user-friendly format must be done manually -- that is the reason for the conditionals checking for parsing and the help menu. (It is also worth nothing the help CLI option could just as easily be any boolean flag for any other purpose.)

The following examples renders the values from the parsed options and their types...

```groovy
#!/usr/bin/env groovy

def defaultStringThing = 'How now little friend?'
def defaultDoubleThing = '1.09'
def defaultNumberThing = "${1550 * 4}"

def cli = new CliBuilder(header: 'Little Friend CLI', usage:'littleFriend', width: -1)
cli.st(longOpt: 'stringThing', "Some string. [defaults to '${defaultStringThing}']", args: 1, defaultValue: defaultStringThing)
cli.dt(longOpt: 'doubleThing', "Some double. [defaults to '${defaultDoubleThing}']", args: 1, defaultValue: defaultDoubleThing)
cli.nt(longOpt: 'numberThing', "Some number. [defaults to '${defaultNumberThing}']", args: 1, defaultValue: defaultNumberThing)
cli.h(longOpt: 'help', 'Usage Information')

def cliOptions = cli.parse(args)

if (!cliOptions) {
  cli.usage()
  System.exit(-1)
}

if (cliOptions.help) {
  cli.usage()
  System.exit(0)
}

println "stringThing is '${cliOptions.stringThing}' of type ${cliOptions.stringThing.getClass()}"
println "doubleThing is '${cliOptions.doubleThing}' of type ${cliOptions.doubleThing.getClass()}"
println "numberThing is '${cliOptions.numberThing}' of type ${cliOptions.numberThing.getClass()}"
```

...yields the following output:

```
./littleFriend  
stringThing is 'How now little friend?' of type class java.lang.String
doubleThing is '1.09' of type class java.lang.String
numberThing is '6200' of type class java.lang.String
```

Notice they are all of type String.

#### Making `littleFriend` Friendly

Types are important and you can just as easily check and cast things in your code after processing...

```groovy
#!/usr/bin/env groovy

def defaultStringThing = 'How now little friend?'
def defaultDoubleThing = '1.09'
def defaultNumberThing = "${1550 * 4}"

def cli = new CliBuilder(header: 'Little Friend CLI', usage:'littleFriend', width: -1)
cli.st(longOpt: 'stringThing', "Some string. [defaults to '${defaultStringThing}']", args: 1, defaultValue: defaultStringThing)
cli.dt(longOpt: 'doubleThing', "Some double. [defaults to '${defaultDoubleThing}']", args: 1, defaultValue: defaultDoubleThing)
cli.nt(longOpt: 'numberThing', "Some number. [defaults to '${defaultNumberThing}']", args: 1, defaultValue: defaultNumberThing)
cli.h(longOpt: 'help', 'Usage Information')

def cliOptions = cli.parse(args)

if (!cliOptions) {
  cli.usage()
  System.exit(-1)
}

if (cliOptions.help) {
  cli.usage()
  System.exit(0)
}

println "stringThing is '${cliOptions.stringThing}' of type ${cliOptions.stringThing.getClass()}"
println "doubleThing is '${cliOptions.doubleThing}' of type ${cliOptions.doubleThing.getClass()}"
println "numberThing is '${cliOptions.numberThing}' of type ${cliOptions.numberThing.getClass()}"

def doubleThing = cliOptions.doubleThing as Double

println "actual doubleThing is '${doubleThing}' of type ${doubleThing.getClass()}"
```

...yields the following output:

```
./littleFriend  
stringThing is 'How now little friend?' of type class java.lang.String
doubleThing is '1.09' of type class java.lang.String
numberThing is '6200' of type class java.lang.String
actual doubleThing is '1.09' of type class java.lang.Double
```

Notice the second to final non-empty line takes the String `doubleThing` and converts it to a double using Groovy's [`as`](http://groovy-lang.org/differences.html) keyword.

#### Making `littleFriend` Friendlier

A better way, though, which is less code and allows the parser to enforce correctness, etc... is to set the type on the
option declaration itself.

```groovy
#!/usr/bin/env groovy

def defaultStringThing = 'How now little friend?'
def defaultDoubleThing = '1.09'
def defaultNumberThing = "${1550 * 4}"

def cli = new CliBuilder(header: 'Little Friend CLI', usage:'littleFriend', width: -1)
cli.st(longOpt: 'stringThing', "Some string. [defaults to '${defaultStringThing}']", args: 1, defaultValue: defaultStringThing)
cli.dt(longOpt: 'doubleThing', "Some double. [defaults to '${defaultDoubleThing}']", args: 1, type: Double, defaultValue: defaultDoubleThing)
cli.nt(longOpt: 'numberThing', "Some number. [defaults to '${defaultNumberThing}']", args: 1, defaultValue: defaultNumberThing)
cli.h(longOpt: 'help', 'Usage Information')

def cliOptions = cli.parse(args)

if (!cliOptions) {
  cli.usage()
  System.exit(-1)
}

if (cliOptions.help) {
  cli.usage()
  System.exit(0)
}

println "stringThing is '${cliOptions.stringThing}' of type ${cliOptions.stringThing.getClass()}"
println "doubleThing is '${cliOptions.doubleThing}' of type ${cliOptions.doubleThing.getClass()}"
println "numberThing is '${cliOptions.numberThing}' of type ${cliOptions.numberThing.getClass()}"
```

...yields the following output:

```
./littleFriend
stringThing is 'How now little friend?' of type class java.lang.String
doubleThing is '1.09' of type class java.lang.Double
numberThing is '6200' of type class java.lang.String
```

Noting the line `cli.dt(longOpt: 'doubleThing', "Some double. [defaults to '${defaultDoubleThing}']", args: 1, defaultValue: defaultDoubleThing)` has been modified to `cli.dt(longOpt: 'doubleThing', "Some double. [defaults to '${defaultDoubleThing}']", args: 1, type: Double, defaultValue: defaultDoubleThing)`, where the `type` is declared.

#### Personalized Friendliness

When supplying converters, types cannot be used and casting isn't required. Here we have a class `Person` and a converted, as a closure, `personConverter`, that takes an input as `input` and returns a newly constructed `Person`.

```groovy
#!/usr/bin/env groovy

import groovy.transform.Canonical

def defaultStringThing = 'How now little friend?'
def defaultDoubleThing = '1.09'
def defaultNumberThing = "${1550 * 4}"

@Canonical class Person {
  def name
}

def personConverter = { input ->
  new Person(input)
}

def cli = new CliBuilder(header: 'Little Friend CLI', usage:'littleFriend', width: -1)
cli.st(longOpt: 'stringThing', "Some string. [defaults to '${defaultStringThing}']", args: 1, defaultValue: defaultStringThing)
cli.dt(longOpt: 'doubleThing', "Some double. [defaults to '${defaultDoubleThing}']", args: 1, type: Double, defaultValue: defaultDoubleThing)
cli.nt(longOpt: 'numberThing', "Some number. [defaults to '${defaultNumberThing}']", args: 1, defaultValue: defaultNumberThing)
cli.pt(longOpt: 'personThing', "Some person.", args: 1, convert: personConverter)
cli.h(longOpt: 'help', 'Usage Information')

def cliOptions = cli.parse(args)

if (!cliOptions) {
  cli.usage()
  System.exit(-1)
}

if (cliOptions.help) {
  cli.usage()
  System.exit(0)
}

println "stringThing is '${cliOptions.stringThing}' of type ${cliOptions.stringThing.getClass()}"
println "doubleThing is '${cliOptions.doubleThing}' of type ${cliOptions.doubleThing.getClass()}"
println "numberThing is '${cliOptions.numberThing}' of type ${cliOptions.numberThing.getClass()}"

if (cliOptions.personThing) {
  println "personThing is '${cliOptions.personThing}' of type ${cliOptions.personThing.getClass()}"
}
```

...yields the following output:

```
./littleFriend -pt hi
stringThing is 'How now little friend?' of type class java.lang.String
doubleThing is '1.09' of type class java.lang.Double
numberThing is '6200' of type class java.lang.String
personThing is 'Person(hi)' of type class Person
```

Cheers!
