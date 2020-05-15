+++
title = "Product Recommendations in RedisGraph, Part 1: Data loading"
date = "2020-01-16"
tags = ["graph", "redis", "recommendations", "data", "opencypher", "redisgraph"]
highlightjs = true
+++

This post is part of a [series]({{< ref "/tags/recommendations" >}}) on leveraging RedisGraph for product recommendations.

There are so many persistence stores in the world; relational, document, key-value, time series, graph, and on and on...
Of all those types, graphs excel at deriving the manner of the interconnectedness of data. This is why social networks
are backed by graphs, why fraud systems are often graph-based, and why recommendation engines of most kinds are too!

The first graph system I built, or, really prototyped is the actual usage, was a page recommendation system for a client
using [AEM](https://www.adobe.com/marketing/experience-manager.html). The system leveraged
[Neo4j](https://neo4j.com) and used edge compute to track clients, the cached pages, and track what pages they'd clicked
through, the content characteristics of the page, and made live recommendations on subsequent page
request. It was awesome. Neo4j, though, is expensive for non-enterprise merchants, and it don't
go anywhere. That was 6 years ago and now there are many other contenders;

- [dgraph](https://dgraph.io)
- [TigerGraph](https://www.tigergraph.com)
- [Amazon Neptune](https://aws.amazon.com/neptune/)
- [JanusGraph](https://janusgraph.org)

... the list, the contenders, goes on and on...

I few months back I stumbled on [RedisGraph](http://redisgraph.io) and thought I'd give it a shot for another
PoC or really a "learn [Cypher](https://en.wikipedia.org/wiki/Cypher_(query_language))" project, or, more accurately,
a "learn [openCypher](https://www.opencypher.org)". RedisGraph is, as one might imagine, built atop
[redis](http://redis.io), the [swiss army knife](https://en.wikipedia.org/wiki/Swiss_Army_knife) of the computing world.
The advantages to that are numerous:

- redis is fast
- redis is scalable
- redis is easily useful for many different types of workloads
- redis is ... simple.

### RedisGraph

From the creators of RedisGraph, it is "the first queryable Property Graph database to use sparse matrices to represent the adjacency matrix in graphs and linear algebra to query the graph." ...which basically means it very, very, very efficiently
stores your vertices and edges and their properties in memory for very fast query execution.

[RedisLabs](https://redislabs.com) is the primary maintainer of the RedisGraph module, and in addition to it
they make a slick and full-featured web UI. The UI, [RedisInsight](https://redislabs.com/redisinsight/), is awesome for interacting with your graph and is quite similar to the visualizations supported by Neo4j. The UI is not _just_ for RedisGraph, but
for RedisSearch, Time Series, etc... RedisInsight can be obtained through their website, but, more conveniently (in my opinion) via docker:

`docker run -p 8001:8001 -it --rm redislabs/redisinsight`

### A Product Recommendations data model

Alright, let's switch gears and get into the modeling part of this proof of concept. First, the vertices and edges:

- person (id, name, address, age, memberSince)
- product (id, name, manufacturer, msrp)
- order (id, subTotal, tax, shipping, total)

and the following edges are supported:

- view (timestamp)
- addtocart (timestamp)
- transact
- contain

The relationships are as follows:

1. Person views product
2. Person adds product to cart
3. Person transacts order which contains products

![image](/img/2020-1-redis-graph-product-recommendation-part-1-data-loading/subgraph.png)

The data generation is fairly realistic. People's names and address are believable as are products thanks to [Java Faker](http://dius.github.io/java-faker). Products bought are randomly
split between 1-to-n orders of a subset of those added to cart of a subset of those viewed.

Orders contain valid sums of products associated with them, etc...

Second, let's go over the scripts themselves. There are two; one that connects to the RedisGraph instance and creates all the vertices
and edges programmatically and is representative of what you might do in actual (Java) code deployment. The second generates the same data, but writes
to CSV files to be used by RedisGraph's [bulk load](https://github.com/RedisGraph/redisgraph-bulk-loader) utility. The following are the links
to the two utilities:

- [create local commerce graph](https://github.com/joshdurbin/redis-graph-commerce-poc/blob/beaafc2f003bb6f41b68c9211814c98951403f28/1-create_local_commerce_graph.groovy)
- [create CSVs for bulk import](https://github.com/joshdurbin/redis-graph-commerce-poc/blob/beaafc2f003bb6f41b68c9211814c98951403f28/2-create-csvs-for-bulk-import.groovy)

### Graph Generation

There are variables common to both scripts. In fact, the only non-common variable is the batch size parameter for the first, which should not be
modified. The other tuning parameters for data generation are as follows:

```groovy
def maxPotentialViews = 50
def minPotentialViews = 0
def percentageOfViewsToAddToCart = 25
def percentageOfAddToCartToPurchase = 15
def maxRandomTimeFromViewToAddToCartInMinutes = 4320
def maxRandomTimeFromAddToCartToPurchased = 4320
def maxPastDate = 365 * 20
def maxPotentialPeopleToCreate = 1_001
def minPotentialPeopleToCreate = 1_000
def maxPotentialProductsToCreate = 1_001
def minPotentialProductsToCreate = 1_000
def nodeCreationBatchSize = 500
def maxTaxRate =  0.125
def minTaxRate = 0.0
def maxShipRate = 0.15
def minShipRate = 0.0
def minPerProductPrice = 0.99
def maxPerProductPrice = 1000.00
```

Of these, the four you'll probably be most interesting in tuning are the min/max people and products:

- `minPotentialPeopleToCreate` / `maxPotentialPeopleToCreate`
- `minPotentialProductsToCreate` / `maxPotentialProductsToCreate`

Right now the min and max of both are set to 1000 and 1001, respectively, meaning the generators create 1000 of each people and products. For the performance samples that follow I have the people set to min/max of 25000/25001.

#### Programmatic Generation of the Graph

I'm going to assume Docker familiarity with all these examples.

1. Run `redisgraph` with a data volume, allowing us to save our graph: `docker run -p 6379:6379 -it --rm -v redis-data:/data redislabs/redisgraph:edge`. If you've been running this for a while, it occasionally helps to explicitly pull the `redislabs/redisgraph:edge` tag.
2. With [groovy](http://groovy-lang.org) installed, run `groovy 1-create_local_commerce_graph.groovy`. The script will output what its doing as it executes -- it won't leave you in the dark.

#### Bulk Loading Generation of the Graph

1. Run `redisgraph` with a data volume, allowing us to save our graph: `docker run -p 6379:6379 -it --rm -v redis-data:/data redislabs/redisgraph:edge`. If you've been running this for a while, it occasionally helps to explicitly pull the `redislabs/redisgraph:edge` tag.
2. Clone the bulk loader utility: `git clone git@github.com:RedisGraph/redisgraph-bulk-loader.git`
3. Change directory into the newly cloned bulk loader: `cd redisgraph-bulk-loader`
4. Create a Python virtual env for this work: `python3 -m venv redisgraphloader`
5. Step into the venv: `source redisgraphloader/bin/activate`
6. Install the dependencies for the bulk loader: `pip install -r requirements.txt`
7. Pull down the groovy script `wget https://raw.githubusercontent.com/joshdurbin/redis-graph-commerce-poc/master/2-create-csvs-for-bulk-import.groovy`
8. Execute the groovy script, to generate the CSVs for the vertices and edges: `groovy 2-create-csvs-for-bulk-import.groovy`

Note the structure of the CSVs:

```bash
(redisgraphloader) ➜  redisgraph-bulk-loader git:(master) ✗ head -n2 *.csv
==> addtocart.csv <==
src_person,dst_product,timestamp
0,158328,2016-09-25T03:53:36.160952

==> contain.csv <==
src_person,dst_order
175000,160230

==> order.csv <==
_internalid,id,subTotal,tax,shipping,total
175000,0,681.33,65.28,61.88,808.49

==> person.csv <==
_internalid,id,name,address,age,memberSince
0,0,Gertrude Baumbach,Apt. 677 4680 Jae Estate Lake Vincenzo SC 77433-9279,65,2008-05-26T09:21:36.160952

==> product.csv <==
_internalid,id,name,manufacturer,msrp
150000,0,Sleek Leather Car,Orn and Sons,21.99

==> transact.csv <==
src_person,dst_order
0,175000

==> view.csv <==
src_person,dst_product,timestamp
0,158328,2016-09-24T02:27:36.160952
```

The `person`, `product`, and `order` vertices have an internal, unique ID that's used as part of the import process. The leading `_` in the first
row of the import CSV denotes that the field should **not** be imported into the graph.

9. Run the bulk loader: `python bulk_insert.py prodrec-bulk -n person.csv -n product.csv -n order.csv -r view.csv -r addtocart.csv -r transact.csv -r contain.csv` (read [usage](https://github.com/RedisGraph/redisgraph-bulk-loader) for more info)

#### Performance, trade offs

The bulk loader is much faster than the programmatic creator as it uses the `GRAPH.BULK` operator (see [more](https://github.com/RedisGraph/RedisGraph/blob/master/src/bulk_insert/bulk_insert.c) and [more](https://github.com/RedisGraph/RedisGraph/issues/709)).

  For smaller data sets the difference is negligible, but for larger data sets the `MATCH` criteria on edge creation starts to become expensive, especially as the `product` vertex count bounds upward.

Here's an example of the first, programmatic script run and timing:

`Finished creating 25000 'person', 1000 'product', 27897 'order', 234727 'view' edges, 61070 'addtocart' edges, 27897 'transact' edges, and 40181 'contain' products edges in 626164 ms...`

... which is ~10 min 30 seconds.

The bulk loader, with roughly the same outcomes (in terms of counts) has the following output summarization:

```
person  [####################################]  100%
25000 nodes created with label 'person'
1000 nodes created with label 'product'
order  [####################################]  100%
27966 nodes created with label 'order'
view  [####################################]  100%
235988 relations created for type 'view'
addtocart  [####################################]  100%
61288 relations created for type 'addtocart'
27966 relations created for type 'transact'
contain  [####################################]  100%
40327 relations created for type 'contain'
Construction of graph 'test' complete: 53966 nodes created, 365569 relations created in 12.129039 seconds
```

### Play.

#### Accessing your graph

First off, you can always, simply access your graph using the Redis command line. The following
is the docker execution of the Redis command line utility: `docker run -it --network host --rm redis redis-cli -h 127.0.0.1`

Once you're connected, you can easily save the graph to the data volume attached by Docker.

```
127.0.0.1:6379> save
OK
(0.53s)
```

...and you can query for data using a query like:

{{<highlightjs language="cypher">}}
match (p:person) where p.id=200 return p.name
{{</highlightjs>}}

Executed:

```
127.0.0.1:6379> graph.query prodrec "match (p:person) where p.id=200 return p.name"
1) 1) "p.name"
2) 1) 1) "Denver Jerde"
3) 1) "Query internal execution time: 4.937000 milliseconds"
```

**Note**: Keep in mind the programmatic script writes to the graph `prodrec`, while the bulk utility writes to `prodrec-bulk`.

#### Visualizing your graph

This assumes you've already got the RedisInsight container running. If not; `docker run -p 8001:8001 -it --rm redislabs/redisinsight`

Once you've got that running, do the following:

1. Point your browser at [`http://localhost:8001`](http://localhost:8001)
2. Accept the EULA
3. Create a connection to your RedisGraph instance at, in my case, `docker.for.mac.localhost`
4. Connect to the console
5. Select RedisGraph and select either the `prodrec` or `prodrec-bulk` graphs.

Now, a few things to be aware of when you're using RedisInsigh for RedisGraph:

- Here we don't need to prefix calls to Redis using `graph.query`
- When you return a vertex from a query, say:

{{<highlightjs language="cypher">}}
match (p:person) where p.id=199 return p
{{</highlightjs>}}

...a visualization of the graph will render.

![image](/img/2020-1-redis-graph-product-recommendation-part-1-data-loading/visual-part-0.png)

- You can double click on any vertex and it will show you the connecting edges to other vertices in the system.

![image](/img/2020-1-redis-graph-product-recommendation-part-1-data-loading/visual-part-1.png)

- If you don't render a vertex, data will be listed in table format.

![image](/img/2020-1-redis-graph-product-recommendation-part-1-data-loading/visual-part-2.png)

### Summary

A post in the next week or two will dive into queries against this data to actually generate product recommendations. Stay tuned!
