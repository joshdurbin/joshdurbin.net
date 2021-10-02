+++
title = "Product Recommendations in RedisGraph, Part 3: Query Load Testing"
date = "2020-05-15"
tags = ["graph", "redis", "recommendations", "data", "opencypher", "redisgraph"]
highlightjs = true
+++

This post is part of a [series]({{< ref "/tags/recommendations" >}}) on leveraging RedisGraph for product recommendations.

In this post we'll cover the script developed for concurrent load testing of product recommendation queries detailed in
[this post]({{< ref "/posts/2020-4-redis-graph-product-recommendation-generator-revamped.md" >}}).

### Concurrent RedisGraph

First let's talk about Redis -- it's single threaded. End of story, go home, right? Well, sort of. [RedisGraph](http://redisgraph.io)
is able to take advantage of the same [techniques](https://redislabs.com/blog/making-redis-concurrent-with-modules) that other [Redis Labs](http://redislabs.com)
projects like [RediSearch](https://oss.redislabs.com/redisearch/) to speed up concurrent processing of requests.

Though commands/operations in Redis happen atomically, RedisGraph internally uses a thread pool to offload some of the linear algebra done to convert
openCypher queries, engage in matrix multiplication on non-critical pathways, and (presumably) by frequently swapping 
contexts -- resulting in greater performance overall and preventing long-running queries from generating large scale bottlenecks.

### Queries in the Load Testing Tool

The tool written to strain and squeeze performance out of RedisGraph is the [productRecommendationQueryRunner](https://github.com/joshdurbin/redis-graph-commerce-poc/blob/master/productRecommendationQueryRunner).
The tool executes two queries; the first obtains the top 1,000 purchasers by default and then pumps their `person` ids into the second query; fetching all 
"products that share orders with products that user has ordered".

{{<highlightjs language="cypher">}}
match (p:person)-[:transact]->(o:order) return p.id, count(o) as orders order by orders desc limit 100
{{</highlightjs>}}

... returns a list of `person`-labeled node `id` property and pumps them into:

{{<highlightjs language="cypher">}}
match (p:person { id: ${personId} })-[:transact]->(:order)-[:contain]->(prod:product)
match (prod)<-[:contain]-(:order)-[:contain]->(rec_prod:product)
where not (p)-[:transact]->(:order)-[:contain]->(rec_prod)
return rec_prod.id, recprod.name order by indegree(prod) desc
{{</highlightjs>}}

The resulting output is a bar showing progress and time to completion. One complete the tool will output the min, average, and max
number of recommended products along with p50, p95, and p99 performance metrics.

### Installation

This script requires the installation of Java 11 and Groovy. Dependencies are automatically pulled from Maven repositories based on the Grape'd annotations.

- Java (11 OpenJDK)
- Groovy 3
- RedisGraph

### Obtaining and Basic Usage

Pull down the `productRecommendationQueryRunner` script itself:

```shell script
wget https://raw.githubusercontent.com/joshdurbin/redis-graph-commerce-poc/f13f4bffc7782ee74fccf51d5b78973836644c5f/productRecommendationQueryRunner
chmod u+x productRecommendationQueryRunner
```

At this point, with RedisGraph available at `localhost:6379`, you can just run `./productRecommendationQueryRunner` to load test against the graph.

```shell script
./productRecommendationQueryRunner 
Progress 100% │███████████████████████████████████████████████████████████████████████████│ 1000/1000 (0:00:42 / 0:00:00) 
Found a min number of recommended products of 141, avg of 693, and a max of 1674 for 1000 with a query performance p50 230ms, p95 442ms, p99 531ms
```

### RedisGraph concurrent performance during query 

I'm running RedisGraph in docker and running [ctop](https://github.com/bcicen/ctop) while the `productRecommendationQueryRunner` is running shows output like:

```shell script
  ctop - 23:25:58 PDT   3 containers                                                                                                               

     NAME                  CID                   CPU                   MEM                   NET RX/TX             IO R/W                PIDS

   ◉  awesome_gauss         0aa2d5b45a8c                    0%               115M / 1.94G     3M / 12M              0B / 0B               10       
   ◉  festive_greider       b4b196d21e40                   520%               42M / 1.94G     1M / 46M              0B / 0B               14
   ◉  pedantic_colden       ac816b00fbe7                    0%                1M / 1.94G      0B / 0B               0B / 0B               1
```

...showing that during the test RedisGraph (identified by the docker instance name `festive_greider`) was able to easily leverage
more than CPU core on my laptop.

### Advanced Usage

There are many fewer knobs to spin here compared to the graph generation tooling, but nonetheless there are a few. Like the generation script,
 the list of options can be output requesting the help menu `./productRecommendationQueryRunner --help`:

```shell script
./productRecommendationQueryRunner --help
usage: productRecommendationQueryRunner <args>
Concurrent RedisGraph Query Runner
 -db,--database <arg>        The RedisGraph database to use for the query [defaults to prodrec]
 -h,--help                   Usage Information
 -l,--limitResults <arg>     The default results limit.
 -rh,--redisHost <arg>       The host of the Redis instance with the RedisGraph module installed to use for graph creation. [defaults to localhost]
 -rp,--redisPort <arg>       The port of the Redis instance with the RedisGraph module installed to use for graph creation. [defaults to 6379]
 -tc,--threadCount <arg>     The thread count to use [defaults to 6]
 -tp,--topPurchasers <arg>   The number of top purchasers to query for [defaults to 1000]
```

### Query Tweaks

This tool is written to stress the Redis Graph instance and thus it returns a great amount of data -- more than you'd probably really want
in real life. There are some obvious optimizations / changes that you'd make if you were running a query like this in real life -- like returning
only the top N products from the recommendation query instead of all.

### Next up 

The next post will be a bit more detailed on performance where RedisGraph will be provisioned on cloud compute infrastructure and load tested
along with various input parameters to the [generateCommerceGraph](https://github.com/joshdurbin/redis-graph-commerce-poc/blob/bf3f81727dea7dc97b9732e5561fba78f4ea1c77/generateCommerceGraph)
and [productRecommendationQueryRunner](https://github.com/joshdurbin/redis-graph-commerce-poc/blob/f13f4bffc7782ee74fccf51d5b78973836644c5f/productRecommendationQueryRunner). Stay tuned!
 
