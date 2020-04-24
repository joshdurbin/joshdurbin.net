+++
title = "Working around the static variable limitations in Groovy scripts"
date = "2020-01-11"
tags = ["groovy", "work-arounds"]
+++

I've written a bunch about and used Groovy for examples of various functionality. Still, to this day, coupled with Maven artifact resolution
using the Grape subsystem, Groovy is a powerful tool. I end up writing workers, complex scripts that do heavy multithreaded workloads, etc... Often
these units of work involve their own classes for some amount of structured data. I recently came across a problem where I was creating entities with
various keys and wanted to make sure changes to those keys were represented in the created/update/delete/report methods, so, I began using variables
to consistently refer to said keys. Example, from:

```groovy
def someService = ...

@Canonical
class Person {
  def name

  def insertStatement() {
    "person:${name}"
  }
}

def josh = new Person('josh')

someService.insert('person', josh)
```

...to:

```groovy
def someService = ...
def personKey = 'person'

@Canonical
class Person {
  def name

  def insertStatement() {
    "${personKey}:${name}"
  }
}

def josh = new Person('josh')

someService.insert(personKey, josh)
```

...which of course won't work because the `Person` class doesn't contain `personKey`, it's not scoped for the class. No worries,
I'll make it `static`. Oh wait, groovy scripts can't use `static` variable qualifiers. So, instead, leverage this little handy work-around:

```groovy
def someService = ...

@Singleton class GraphKeys {
  def personNodeType = 'person'
}

@Canonical
class Person {
  def name

  def insertStatement() {
    "${GraphKeys.instance.personKey}:${name}"
  }
}

def josh = new Person('josh')

someService.insert(GraphKeys.instance.personNodeType, josh)
```

...and voila. We use another class to hold our variables (you could easily have used an Enum with properties/values, too).
