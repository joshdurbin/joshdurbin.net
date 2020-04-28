+++
title = "Building a product recommendation engine using RedisGraph and OpenCypher, Part 2: Understanding OpenCypher"
date = "2020-04-24"
tags = ["graph", "redis", "recommendations", "data", "opencypher"]
draft = true
+++

This is a long post on understanding the data generated from the [previously discussed]() in the series graph generator.

Covering things a second time; what is a property graph?

What is a graph?

Why redisgraph?

- storage adjacency lists vs matrix

What benefits does Redisgraph provide?

Basics of querying

- cypher declarative language allowing you to specify what you want rather than how


Show a photo of that graph.

List the nodes, list the edges, the counts.

`match (n) return distinct labels(n), count(n)`

`graph.query prodrec "match ()-[e]-() return distinct type(e), count(e)`



Drawbacks thusfar
