+++
title = "Product Recommendations in RedisGraph, Part 2: openCypher Query Basics"
date = "2020-05-09"
tags = ["graph", "redis", "recommendations", "data", "opencypher", "redisgraph"]
+++

This post is a continuation of the series on leveraging RedisGraph for product recommendations.

Back in Fall 2015 [Emil Eifrem](https://www.linkedin.com/in/emileifrem/), CEO/co-founder of [Neo4j](https://neo4j.com), [announced](https://neo4j.com/blog/open-cypher-sql-for-graphs/) [openCypher]() at
GraphConnect in San Francisco. OpenCypher, inspired by Cypher, was Neo4j's 3rd attempt at a graph query language. The growth and adoption of graph tech they experienced alongside other vendors began to expose and emphasize the need for a common
query language against Graphs -- someting similar to [SQL](https://en.wikipedia.org/wiki/SQL) for relational DBs. With Emil's announcement [openCypher](https://www.opencypher.org) was born. Here we'll talk about
how to use openCypher generally and in the context of the product recommendation POC / [RedisGraph](http://redisgraph.io).

### Graph Basics

I won't dive into an exhaustive explanation of graphs here (see [Wikipedia](https://en.wikipedia.org/wiki/Graph_(abstract_data_type))), but, in very broad strokes, they are structures composed of nodes and relationships. Relationships connect nodes
and can be directed. There are also [many different types](https://en.wikipedia.org/wiki/Graph_(discrete_mathematics)#Types_of_graphs) of graphs. Redis Graph specifically is a directed, labeled multigraph where both nodes and relationships are labeled (and, optionally, contain
properties). Consider the following output from [RedisInsight](https://redislabs.com/redisinsight/) on a graph created from my commerce recommendation [POC tooling](http://github.com/joshdurbin/redis-graph-commerce-poc):

![image](/img/2020-5-redis-graph-product-recommendation-opencypher/subgraph.png)

The graph from the image above was created via the process outlined in [this post]({{< ref "/posts/2020-4-redis-graph-product-recommendation-generator-revamped.md" >}}).

Though it's not highlighted; the center-most, light-green node (with the giant red arrow pointing at it) is a node of type `person` and is the center point
for this graph expansion. In this example `person` has relationships with many `product` nodes by three paths: 

1. When the `person` _viewed_ a `product`
2. When the `person` _added a product_ to their cart
3. When the `person` placed (_transacted_) an `order` which _contained_ the product

In addition to these patterns in the graph, `view` and `addtocart` edges exist between nodes when a product is viewed or added to the cart. To keep the sample dataset as
realistic as possible, `addtocart` edges are only created from a subset of the `view` edges that exist between a `person` and `product` node. These
circumstances highlight Redis Graphs ability or utility as a [multigraph](https://en.wikipedia.org/wiki/Multigraph) -- when any node has parallel edges between itself and another node.  

This example contains only directed relationships, which are denoted by the arrows in the graph visualization.

### open(cypher) Basics

This post will cover usage of a variety of Cypher commands, why they're used, and specific examples in the context of a product recommendation POC/example. 



### Graph Creation

The [graph generation](https://github.com/joshdurbin/redis-graph-commerce-poc/blob/master/generateCommerceGraph) tooling issues two types of commands against Redis Graph to seed and populate the graph:

1. The creation of independent nodes via `create` statements. Three node "types" (not RedisGraph types, as they apply to edges or relationships **only**) exist in the form of the following labels: `person`, `product`, and `order`. 
The following are the create statements used to insert these nodes into the graph.     
    
    - `CREATE (:person {id: 4, name:"Javier Bashirian",age:94,address:"36186 Koelpin Isle, Lake Rashidahaven, OH 78740",memberSince:"2011-01-07T07:49:21.274235"})`        
    - `CREATE (:product {id: 0,name:"Mediocre Granite Shirt",manufacturer:"Hoppe-Fahey",msrp:'812.68'}), (:product {id: 3,name:"Awesome Iron Hat",manufacturer:"Willms, Mills and Wolf",msrp:'753.17'})`     
    - `CREATE (:order {id: 2,subTotal:1565.85,tax:176,shipping:208,total:1949.85})`
    
    Each `CREATE` statement takes a node enclosed by parenthesis with its label defined with a colon and the label ex: `person`. Multiple
    nodes can be created with a single statement with nodes separated with commas.
    
    Properties for nodes are defined within the bounds of left and right curly brackets `{ }`. Each property must contain a string key separated by a colon and the value, which must equal one of
    the [supported types](https://oss.redislabs.com/redisgraph/cypher_support/#types). Multiple properties are separated by commas.  
    
    Here a node of label `person` is created with the properties `id: 4`, `name: Javier Bashirian`, `age:94`, etc...
    
    Note: In these examples `id` is a property the application is setting on the node,
    which can be used for querying/matching, but is not the unique ID of the node in the graph itself -- an important distinction. The internal ID of a node or relationship can be obtained
    via the [Scalar function](https://oss.redislabs.com/redisgraph/commands/#scalar-functions) `id()`.
    
    Nodes do not require labels or properties. For example: `CREATE ()` is a valid query. 

2. The next crucial and arguable more important part of this graph creation is the relationships, the edges -- they do the connecting, the entire reason for having a graph in the first place.

    Here the tooling is instructing the graph to execute a sort of conditional create where its first matching nodes of various types, then creating a relationship between those nodes. `MATCH` is used
    to tell the graph to find the nodes, a path, relationships, etc...  
      
    Consider the following statement:  
    
    `MATCH (p:person), (o:order), (prd3:product), (prd0:product) WHERE p.id=4 AND o.id=2 AND prd3.id=3 AND prd0.id=0 CREATE (p)-[:transact]->(o), (o)-[:contain]->(prd3), (o)-[:contain]->(prd0)`
    
    This query instructs the graph to match and find nodes with labels `person`, `order`, and `product`. These look similar to the create statements before except the
    labels have a leading identifier prior to the colon and label declaration. These identifiers, termed an [alias](https://oss.redislabs.com/redisgraph/commands/#match), `p`, `o`, `prd3`, and `prd0` are exclusive to the query. They could be
    _anything_ -- graph generation app uses sensible names for these aliases, though, which is why `prd3` and `prd0` are used. These aliases in and of themselves
    do not restrict us to "product 3" and "product 0". The restrictions to those product ids come in the `WHERE` segment of the query: 
    
    `WHERE p.id=4 AND o.id=2 AND prd3.id=3 AND prd0.id=0`
    
    ...where alias `p`, which is a node label `person`, with property `id` equal to `4`, etc...
    
    **Only when** this conditional is met do we create our edges, our relationships:
    
    `CREATE (p)-[:transact]->(o), (o)-[:contain]->(prd3), (o)-[:contain]->(prd0)`
    
    The `CREATE` statement here is creating three edges of `type`, `transact`, and (2x) `contain`. It's worth noting that
    these edges _could_ contain properties -- we'll see an example of this in our next example. The `create` statements
    are leveraging the same aliases we used for our `MATCH` statement, which now makes it a little more clear why they are
    explicitly named `p`, `o`, `prd3`, and `prd0`.
    
    The prior, full `MATCH` statement associated our person with the created order and the order
    with the products it contains. The next statement associates our person with products they've viewed and added to their carts.    
    
    `MATCH (p:person), (prd1:product), (prd3:product), (prd2:product), (prd0:product), (prd4:product) WHERE p.id=4 AND prd1.id=1 AND prd3.id=3 AND prd2.id=2 AND prd0.id=0 AND prd4.id=4 CREATE (p)-[:view {time: '2018-07-18T15:54:21.274235'}]->(prd1), (p)-[:view {time: '2013-04-16T02:10:21.274235'}]->(prd3), (p)-[:view {time: '2012-10-22T15:51:21.274235'}]->(prd2), (p)-[:view {time: '2012-11-05T18:29:21.274235'}]->(prd0), (p)-[:view {time: '2017-09-02T15:22:21.274235'}]->(prd4), (p)-[:addtocart {time: '2013-04-19T02:01:21.274235'}]->(prd3), (p)-[:addtocart {time: '2012-11-07T14:43:21.274235'}]->(prd0)`
    
    This example has a few more aliases, but primarily differs from the other in that a property is set on the edge, on the relationship. Note that the syntax
    for doing so is identical to the nodes. 
    
    `(p)-[:addtocart {time: '2012-11-07T14:43:21.274235'}]->(prd0)`
    
### Querying the Graph

Note: some queries include query times. The graph used for these queries and all subsequent examples was created [by the generator](https://github.com/joshdurbin/redis-graph-commerce-poc/blob/master/generateCommerceGraph).  

1. Query the distinct labels in the graph
    - `match (n) return distinct labels(n)` (9.7 ms) -- match any node, any match aliased with `n`, returning distinct results from the `labels()` function
    - `call db.labels()` (1.4 ms)
2. Query the distinct edges or relationships in the graph
    - `match ()-[e]->() return distinct type(e)` (195 ms) -- match any directed relationship, aliased with `e`, between an un-aliased source and destination node of any label, returning distinct types from the `type()` function  
    - `call db.relationshipTypes()` (.2 ms)    
3. Query the distinct labels in the graph and obtain their counts
    - `match (n) return distinct labels(n), count(n)` -- similar to the aforementioned query in ex: #1, with an added `count`
4. Query the distinct edges (or relationships) in the graph     
    - `match ()-[e]->() return distinct type(e), count(e)` -- similar to the aforementioned query in ex: #2, with an added `count`
5. Obtain the id of a node or edge
    - `match (p:person) where p.id=3 return id(p)` -- match a node of type `person` with the query alias `p` and the conditional, property, `id` == `3` returning the actual ID of the node via the `id` function
    - `match (p:person {id: 3}) return id(p)` -- this query, match statement is identical to the prior one, but here we specify the required property match in the node itself rather than with a `where` clause 
    - `match ()-[e]->() return id(e) limit 1` -- similar to aforementioned query in ex: #2; match any directed relationship, aliased with `e`, between an un-aliased source and destination node of any label, returning the ID of the relationship via the `id` function    

### e-Commerce Product-related Queries

1. Return the top 5 people who have created the most orders
    - `match (p:person)-[:transact]->(o:order) return p.id, count(o) as orders order by orders desc limit 5` -- match a node of label `person` with alias `p` with a directional edge of label `transact` to a node of label `order` with alias `o`
2. Return the top 20 most ordered products id and name
    - `match (:order)-[c:contain]->(p:product) return p.id, p.name, count(c) as count order by count limit 20`
3. Return the top 20 most viewed products
    - `match (:person)-[v:view]->(p:product) return p.id, p.name, count(v) as count order by count limit 20`
4. Return the 5 fewest purchased products
   - `match (:order)-[c:contain]->(p:product) return p.id, p.name, count(c) as count order by count desc limit 5`    
4. Return products not purchased
    - `match (p:product) where not (:order)-[:contain]->(p) return count(p)`    
5. Show all items viewed by visitors that have purchased something    
           
### Basic Product Recommendation Queries

Note: Giving credit where credit is due -- I adapted, or took inspiration, from the query listed in [this gist](https://gist.githubusercontent.com/adam-cowley/79923b538f6851d30e08/raw/abe673493fafba9f2cfd43cba0d7c6b334e6b01e/neo4j-northwind-recommendation).

1. Find products that are in orders that have common products placed by person id 294
    - `match (p:person)-[:transact]->(:order)-[:contain]->(:product)<-[:contain]-(:order)-[:contain]->(prd:product) where p.id=294 return distinct prd`
    
    This statement really shows the benefit of cypher over something like SQL, which would require many joins to accomplish the same query. Here we trace the path,
    telling the graph to match `people`-labeled nodes, through the `product`-labeled nodes they've purchased to the `product`-labeled nodes of all orders that
    shared that `product` in common.
    
    This query only aliases the `person` labeled node at the start `p` and the `product` labeled node at the end `prd`. Given, though,
    that `p` only exists so that can use it to filter based on its id in the `where` clause.
    
    This could be re-written as `match (p:person { id: 294})-[:transact]->(:order)-[:contain]->(:product)<-[:contain]-(:order)-[:contain]->(prd:product) return distinct prd`.   
    
    This query returns the alias for `product`, `prd`, itself and therefor we get all the details for that node. Ex:
    
    ```
   127.0.0.1:6379> graph.query prodrec "match (p:person { id: 294})-[:transact]->(:order)-[:contain]->(:product)<-[:contain]-(:order)-[:contain]->(prd:product) return distinct prd limit 1"
   1) 1) "prd"
   2) 1) 1) 1) 1) "id"
               2) (integer) 5000
            2) 1) "labels"
               2) 1) "product"
            3) 1) "properties"
               2) 1) 1) "id"
                     2) (integer) 3
                  2) 1) "name"
                     2) "Gorgeous Plastic Pants"
                  3) 1) "manufacturer"
                     2) "Rohan LLC"
                  4) 1) "msrp"
                     2) "771.38"
   3) 1) "Query internal execution time: 4.133600 milliseconds"
    ```
2. An iteration of the former, where the products in the orders that user placed are excluded 
    - `match (p:person { id: ${personId} })-[:transact]->(:order)-[:contain]->(prod:product) match (prod)<-[:contain]-(:order)-[:contain]->(rec_prod:product) where not (p)-[:transact]->(:order)-[:contain]->(rec_prod) return distinct rec_prod.id, rec_prod.name`     
3. Find all items viewed by visitors who have purchased the thing that the current user is looking at

### Bonus Examples

The following example is 

1. aasdf
    - `MATCH (p:person)-[:transact]->(:order)-[:contain]->(prd1:product) WITH p, count(prd1) as total MATCH (p)-[:transact]->(o:order)-[:contain]->(prd2:product) WITH p, total, prd2, count(o) as orders MERGE (p)-[rated:RATED]->(prd2) ON CREATE SET rated.rating = orders/total ON MATCH SET rated.rating = orders/total RETURN p.name, p.name, orders, total, rated.rating`    

### Additional Reading

1. the [Cypher Query Language Reference (Version 9)](https://s3.amazonaws.com/artifacts.opencypher.org/openCypher9.pdf) from the [OpenCypher resources](http://www.opencypher.org/resources)
2. the [Cypher Style Guide]() also from the [OpenCypher resources](http://www.opencypher.org/resources)
3. [Redis Graph Commands](https://oss.redislabs.com/redisgraph/commands/) for a full list of supported commands
4. [Redis Graph Cypher Coverage](https://oss.redislabs.com/redisgraph/cypher_support/) for notes on what commands aren't covered (yet) by Redis Graph
  