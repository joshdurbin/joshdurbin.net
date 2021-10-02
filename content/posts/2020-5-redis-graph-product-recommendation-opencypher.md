+++
title = "Product Recommendations in RedisGraph, Part 2: openCypher Query Basics"
date = "2020-05-12"
tags = ["graph", "redis", "recommendations", "data", "opencypher", "redisgraph"]
highlightjs = true
+++

This post is part of a [series]({{< ref "/tags/recommendations" >}}) on leveraging RedisGraph for product recommendations.

It's a hot time for Redis! [RedisConf 2020](http://redisconf.com) is happening virtually and there are many interesting topics on Redis and RedisGraph. Check it out!

Back in Fall 2015 [Emil Eifrem](https://www.linkedin.com/in/emileifrem/), CEO/co-founder of [Neo4j](https://neo4j.com), [announced](https://neo4j.com/blog/open-cypher-sql-for-graphs/) [openCypher]() at
GraphConnect in San Francisco. OpenCypher, inspired by Cypher, was Neo4j's 3rd attempt at a graph query language. The growth and adoption of graph tech they experienced alongside other vendors began to expose and emphasize the need for a common
query language against Graphs -- someting similar to [SQL](https://en.wikipedia.org/wiki/SQL) for relational DBs. With Emil's announcement [openCypher](https://www.opencypher.org) was born. Here we'll talk about
how to use openCypher generally and in the context of the product recommendation proof of concept (POC) I've been building with [RedisGraph](http://redisgraph.io).

This post is broken up into the following sections...

- A very brief overview of graphs
- Specifics of this e-Commerce-related graph generation POC
- Basics of querying
- Queries related to common questions of e-Commerce data
- Product recommendation queries
- What's next
- Additional reading materials 

### Graph Basics

[Graphs](https://en.wikipedia.org/wiki/Graph_(abstract_data_type)) are, very simply put, structures composed of nodes and relationships, or edges. Relationships connect nodes
and can be directed. There are also [many different types](https://en.wikipedia.org/wiki/Graph_(discrete_mathematics)#Types_of_graphs) of graphs. Redis Graph is a directed, labeled, [multigraph](https://en.wikipedia.org/wiki/Multigraph) where both nodes and relationships are typed --
nodes with labels and edges with types. Nodes and edges can and often do contain properties like columns in a SQL-db or keys in a document store. Generally
speaking, graphs excel at deriving information from the interconnectedness of data -- as opposed to SQL or document systems where similar queries would require joining many tables or linking many collections.

Consider the following output from [RedisInsight](https://redislabs.com/redisinsight/) on a graph created from my commerce recommendation [POC tooling](http://github.com/joshdurbin/redis-graph-commerce-poc) (the process for
generating it is outlined in [this post]({{< ref "/posts/2020-4-redis-graph-product-recommendation-generator-revamped.md" >}})):

![image](/img/2020-5-redis-graph-product-recommendation-opencypher/subgraph.png)

The centered, light green node with the giant red arrow pointing at it is a `person`-labeled node and is the center point
for this graph expansion. In total, there are three node "varieties" created in this POC:

1. `person`-labeled nodes (again, shown in light green)
2. `product`-labeled nodes shown in dark green
3. `order`-labeled nodes in dark blue

The edges in the system exist to represent the following actions or connections: 

1. When a `person` _views_ a `product` -- shown as yellow directed lines
2. When a `person` _adds a product_ to their cart -- shown as cyan directed lines 
3. When a `person` places (_transacts_) an `order` which _contains_ a product -- shown as blue directed lines

To keep the sample dataset as realistic as possible, `addtocart` edges are only created from a subset of the `view` edges
that exist between a `person` and `product` node.  

All of the relationships in this POC and post are directed, which are denoted by the arrows in the graph visualization. 

### Graph Generation

The [graph generation](https://github.com/joshdurbin/redis-graph-commerce-poc/blob/bf3f81727dea7dc97b9732e5561fba78f4ea1c77/generateCommerceGraph) tooling runs through two phases to seed and populate the graph:

1. The creation of independent nodes occurs via `create` statements. Three node "varieties" are created: `person`, `product`, and `order`. Technically the "variety" of a node is referred to as a label. 
We'll step through distinct examples of each node - label creation. The following are the create statements used to insert these nodes into the graph.     
    
    {{<highlightjs language="cypher">}}
CREATE (:person {
    id: 4,
    name:"Javier Bashirian",
    age:94,
    address:"36186 Koelpin Isle, Lake Rashidahaven, OH 78740",
    memberSince:"2011-01-07T07:49:21.274235"})
    {{</highlightjs>}}
    
    The [`CREATE`](https://oss.redislabs.com/redisgraph/commands/#create) statement takes a node enclosed by parenthesis with its label defined with a colon and the label ex: `person`. Properties for nodes are defined 
    within the bounds of left and right curly brackets `{ }`. Each property must contain a string key separated by a colon and the value, which must equal one of
    the [supported types](https://oss.redislabs.com/redisgraph/cypher_support/#types). Multiple properties are separated by commas.
    
    In these examples `id` is a property the application is setting on the node, which can be used for querying/matching,
    but is not the unique ID of the node in the graph itself -- an important distinction. The internal ID of a node or
    relationship can be obtained via the [Scalar function](https://oss.redislabs.com/redisgraph/commands/#scalar-functions) `id()`.
    
    In the following statement two nodes of the label `product` are created with their various properties.      
            
    {{<highlightjs language="cypher">}}
CREATE
    (:product {
        id: 0,
        name:"Mediocre Granite Shirt",
        manufacturer:"Hoppe-Fahey",msrp:'812.68'}
    ),
    (:product {
        id: 3,
        name:"Awesome Iron Hat",
        manufacturer:"Willms, Mills and Wolf",
        msrp:'753.17'}
    ))
    {{</highlightjs>}}
   
    Per the openCypher spec, nodes are supposed to support multiple labels and edges or relationships are supposed to support multiple types. As of this writing,
    RedisGraph does not support at least multiple node labels, but there is an [issue](https://github.com/RedisGraph/RedisGraph/issues/910) tracking that enhancement.   
    
    Our final node type created in this POC is of label `order`:    
         
    {{<highlightjs language="cypher">}}
CREATE
    (:order {
        id: 2,
        subTotal:1565.85,
        tax:176,
        shipping:208,
        total:1949.85}
    )
    {{</highlightjs>}}        

2. The next crucial component of this graph creation is the relationships/the edges -- they do the connecting,
the entire reason for having a graph in the first place.

    Here the tooling is instructing the graph to execute a sort of conditional create where its first matching nodes of various types, then creating
    a relationship between those nodes. [`MATCH`](https://oss.redislabs.com/redisgraph/commands/#match) is used to tell the graph to find the
    nodes, a path, relationships, etc...  
      
    Consider the following statement:  
    
    {{<highlightjs language="cypher">}}
MATCH (p:person), (o:order), (prd3:product), (prd0:product)
WHERE p.id=4 AND o.id=2 AND prd3.id=3 AND prd0.id=0
CREATE (p)-[:transact]->(o), (o)-[:contain]->(prd3), (o)-[:contain]->(prd0)
    {{</highlightjs>}}
    
    This query instructs the graph to match and find nodes with labels `person`, `order`, and `product`. These look similar to the create statements before except the
    labels have a leading identifier prior to the colon and label declaration. These identifiers, termed an alias, `p`, `o`, `prd3`, and `prd0` are exclusive to the query.
    The aliases could be _anything_. However, the graph generation tooling uses sensible names for these aliases, though, which is why `prd3` and `prd0` are used. These aliases in and of themselves
    do not restrict us to "product 3" and "product 0". The restrictions to those product ids come in the `WHERE` segment of the query: 
    
    {{<highlightjs language="cypher">}}
WHERE p.id=4 AND o.id=2 AND prd3.id=3 AND prd0.id=0
    {{</highlightjs>}}
    
    ...where alias `p`, which is a node label `person`, with property `id` equal to `4`, etc...
    
    Only when this conditional is met do we create our edges, our relationships:
    
    {{<highlightjs language="cypher">}}
CREATE (p)-[:transact]->(o), (o)-[:contain]->(prd3), (o)-[:contain]->(prd0)
    {{</highlightjs>}}
    
    The `CREATE` statement here is creating three edges of types: `transact` and `contain`. The `create` statements
    are leveraging the same aliases we used for our `MATCH` statement, which now makes it a little more clear why they are
    explicitly named `p`, `o`, `prd3`, and `prd0`.
    
    The prior `MATCH` statement associated our `person`-labeled node with the created `order`-labeled node and the order
    with the `product`-labeled nodes the `order` contains. The next statement associates our `person` node with `product` nodes they've viewed and added to their carts.    
    
    {{<highlightjs language="cypher">}}
MATCH (p:person), (prd1:product), (prd3:product), (prd2:product), (prd0:product), (prd4:product)
WHERE p.id=4 AND prd1.id=1 AND prd3.id=3 AND prd2.id=2 AND prd0.id=0 AND prd4.id=4
CREATE
    (p)-[:view {time: '2018-07-18T15:54:21.274235'}]->(prd1),
    (p)-[:view {time: '2013-04-16T02:10:21.274235'}]->(prd3),
    (p)-[:view {time: '2012-10-22T15:51:21.274235'}]->(prd2),
    (p)-[:view {time: '2012-11-05T18:29:21.274235'}]->(prd0),
    (p)-[:view {time: '2017-09-02T15:22:21.274235'}]->(prd4),
    (p)-[:addtocart {time: '2013-04-19T02:01:21.274235'}]->(prd3),
    (p)-[:addtocart {time: '2012-11-07T14:43:21.274235'}]->(prd0)
    {{</highlightjs>}}
    
    This example has a few more aliases, but primarily differs from the other in that a property is set on the edge, on the relationship. Note that the syntax
    for doing so is identical to the nodes. 
    
    {{<highlightjs language="cypher">}}
(p)-[:addtocart {time: '2012-11-07T14:43:21.274235'}]->(prd0)
    {{</highlightjs>}}
   
    Types and syntax are the same for relationships as for nodes.
    
### Querying the Graph

Next we'll zip through some examples of running queries and functions on the graph. Each example is broken into an easier to read version of the query
and the execution of the query using the redis command line tooling.  

1. Query the distinct labels in the graph. The following query will match any node, any match aliased with `n`, returning
distinct results from the `labels()` function.

    {{<highlightjs language="cypher">}}
match (n) return distinct labels(n)
    {{</highlightjs>}}

    Executed:

    ```
    127.0.0.1:6379> graph.query prodrec "match (n) return distinct labels(n)"
    1) 1) "labels(n)"
    2) 1) 1) "person"
       2) 1) "product"
       3) 1) "order"
    3) 1) "Query internal execution time: 10.429400 milliseconds"
    ```
   
   An alternative approach to this specific query is calling the `db.labels()` function. This is an optimized function and should be preferred for this type of query -- note the difference in response time.
   
   Executed: 
   
   ```
   127.0.0.1:6379> graph.query prodrec "call db.labels()"
   1) 1) "label"
   2) 1) 1) "person"
      2) 1) "product"
      3) 1) "order"
   3) 1) "Query internal execution time: 0.179300 milliseconds"   
   ```
   
2. Query the distinct edges or relationships in the graph. The following query will match any directed relationship,
aliased with `e`, between an un-aliased source and destination node of any label, returning distinct types from the `type()` function.

    {{<highlightjs language="cypher">}}
match ()-[e]->() return distinct type(e)
    {{</highlightjs>}}
    
    Executed:

    ```
    127.0.0.1:6379> graph.query prodrec "match ()-[e]->() return distinct type(e)"
    1) 1) "type(e)"
    2) 1) 1) "view"
       2) 1) "addtocart"
       3) 1) "transact"
       4) 1) "contain"
    3) 1) "Query internal execution time: 200.475200 milliseconds"   
    ```

    An alternative approach to this specific query is calling the `db.relationshipTypes()` function. Just like the other DB function call, this is also optimized and should
    be preferred for this type of query -- note the difference in response time.   
    
    ```
    127.0.0.1:6379> graph.query prodrec "call db.relationshipTypes()"
    1) 1) "relationshipType"
    2) 1) 1) "view"
       2) 1) "transact"
       3) 1) "contain"
       4) 1) "addtocart"
    3) 1) "Query internal execution time: 0.166000 milliseconds"   
    ```    
   
3. Query the distinct labels in the graph and obtain their counts -- similar to the aforementioned query in ex: #1, with an added `count`.

    {{<highlightjs language="cypher">}}
match (n) return distinct labels(n), count(n)
    {{</highlightjs>}}

    Executed:

    ```
    127.0.0.1:6379> graph.query prodrec "match (n) return distinct labels(n), count(n)"
    1) 1) "labels(n)"
       2) "count(n)"
    2) 1) 1) "order"
          2) (integer) 11663
       2) 1) "person"
          2) (integer) 5000
       3) 1) "product"
          2) (integer) 1000
    3) 1) "Query internal execution time: 7.088800 milliseconds"   
    ```
   
4. Query the distinct edges (or relationships) in the graph -- similar to the aforementioned query in ex: #2, with an added `count`     

    {{<highlightjs language="cypher">}}
match ()-[e]->() return distinct type(e), count(e)
    {{</highlightjs>}}
    
    Executed:
    
    ```
    127.0.0.1:6379> graph.query prodrec "match ()-[e]->() return distinct type(e), count(e)"
    1) 1) "type(e)"
       2) "count(e)"
    2) 1) 1) "addtocart"
          2) (integer) 30995
       2) 1) "contain"
          2) (integer) 28250
       3) 1) "transact"
          2) (integer) 11663
       4) 1) "view"
          2) (integer) 119276
    3) 1) "Query internal execution time: 177.944000 milliseconds"   
    ```    
    
5. Obtain the id of a node. The following query will match a node of type `person` with the query alias `p`
and the conditional, property, `id` == `3` returning the actual ID of the node via the `id` function.

    {{<highlightjs language="cypher">}}
match (p:person) where p.id=3 return id(p)
    {{</highlightjs>}}
    
    Executed:

    ```
    127.0.0.1:6379> graph.query prodrec "match (p:person) where p.id=3 return id(p)"
    1) 1) "id(p)"
    2) 1) 1) (integer) 3
    3) 1) "Query internal execution time: 0.455100 milliseconds"   
    ```
   
    In this case the id of the node and the nodes property `id` are identical. The graph creation utility is multi-threaded, though, and not all nodes
    share that same trait. Ex:
   
    ```
    127.0.0.1:6379> graph.query prodrec "match (p:person) where p.id=493 return id(p)"
    1) 1) "id(p)"
    2) 1) 1) (integer) 497
    3) 1) "Query internal execution time: 0.627600 milliseconds"   
    ``` 
    
6. Obtain the id of a node. In this query, match statement is identical to the prior one, but here we
specify the required property match in the node itself rather than with a `where` clause.

    {{<highlightjs language="cypher">}}
match (p:person {id: 3}) return id(p)
    {{</highlightjs>}}
   
    Executed:

    ```
    127.0.0.1:6379> graph.query prodrec "match (p:person {id: 3}) return id(p)"
    1) 1) "id(p)"
    2) 1) 1) (integer) 3
    3) 1) "Query internal execution time: 0.389400 milliseconds"    
    ```
7. Obtain the id of a relationship or edge. This query is similar to aforementioned query in ex: #2; match
any directed relationship, aliased with `e`, between an un-aliased source and destination node of any label,
returning the ID of the relationship via the `id` function.

    {{<highlightjs language="cypher">}}
match ()-[e]->() return id(e) limit 1
    {{</highlightjs>}}
    
    Executed:
    
    ```
    127.0.0.1:6379> graph.query prodrec "match ()-[e]->() return id(e) limit 1"
    1) 1) "id(e)"
    2) 1) 1) (integer) 41
    3) 1) "Query internal execution time: 0.418200 milliseconds"    
    ```         

### e-Commerce Product-related Queries

1. Return the top 3 people (`id` and `name` properties) who have created the most orders. This query will match a path where a node of label `person` with
alias `p` with a directional edge of label `transact` to a node of label `order` with alias `o`. The `id`, `name`, and count as `orders` are returned in
descending order and limited to 3.

    {{<highlightjs language="cypher">}}
match (p:person)-[:transact]->(o:order)
return p.id, p.name, count(o) as orders
order by orders desc limit 3
    {{</highlightjs>}}
    
    Executed: 

    ```
    127.0.0.1:6379> graph.query prodrec "match (p:person)-[:transact]->(o:order) return p.id, p.name, count(o) as orders order by orders desc limit 3"
    1) 1) "p.id"
       2) "p.name"
       3) "orders"
    2) 1) 1) (integer) 1388
          2) "Marion Mante"
          3) (integer) 17
       2) 1) (integer) 4977
          2) "Birdie Oberbrunner"
          3) (integer) 17
       3) 1) (integer) 1871
          2) "Towanda Funk"
          3) (integer) 16
    3) 1) "Query internal execution time: 55.890400 milliseconds"  
    ```
     
2. Return the top 3 most ordered products (`id` and `name` properties). This query will match a path where a node of label `order` with
    a directional edge of type `contain` with an alias `c` to a node of label `product` with alias `p`. The `id`, `name`, and count as `count` are returned and limited to 3.

    {{<highlightjs language="cypher">}}
match (:order)-[c:contain]->(p:product)
return p.id, p.name, count(c) as count
order by count desc limit 3
    {{</highlightjs>}}
    
    Executed:

    ```
    127.0.0.1:6379> graph.query prodrec "match (:order)-[c:contain]->(p:product) return p.id, p.name, count(c) as count order by count desc limit 3"
    1) 1) "p.id"
       2) "p.name"
       3) "count"
    2) 1) 1) (integer) 592
          2) "Rustic Paper Hat"
          3) (integer) 44
       2) 1) (integer) 203
          2) "Sleek Silk Lamp"
          3) (integer) 43
       3) 1) (integer) 987
          2) "Intelligent Steel Computer"
          3) (integer) 43
    3) 1) "Query internal execution time: 120.565500 milliseconds"
    ```

3. Return the top 3 most viewed products (`id` and `name` properties). This query will match a path where a node of label `person` with
a directional edge of type `view` with alias `v` to a node of label `product` with alias `p`. The `id`, `name`, and count as `count` are returned and limited to 3.

    {{<highlightjs language="cypher">}}
match (:person)-[v:view]->(p:product)
return p.id, p.name, count(v) as count
order by count desc limit 3
    {{</highlightjs>}}
    
    Executed:

    ```
    127.0.0.1:6379> graph.query prodrec "match (:person)-[v:view]->(p:product) return p.id, p.name, count(v) as count order by count desc limit 3"
    1) 1) "p.id"
       2) "p.name"
       3) "count"
    2) 1) 1) (integer) 189
          2) "Aerodynamic Wool Wallet"
          3) (integer) 151
       2) 1) (integer) 39
          2) "Incredible Silk Lamp"
          3) (integer) 148
       3) 1) (integer) 713
          2) "Small Concrete Lamp"
          3) (integer) 148
    3) 1) "Query internal execution time: 160.342500 milliseconds" 
    ```
   
4. Return the 3 fewest purchased products (`id` and `name` properties). This query will match a path where a node of label `order` with
a directional edge of type `contain` with alias `c` to a node of label `product` with alias `p`. The `id`, `name`, and count as `count` are returned in descending order and limited to 3.

    {{<highlightjs language="cypher">}}
match (:order)-[c:contain]->(p:product)
return p.id, p.name, count(c) as count
order by count limit 3
    {{</highlightjs>}}    
    
    Executed:   

    ```
    127.0.0.1:6379> graph.query prodrec "match (:order)-[c:contain]->(p:product) return p.id, p.name, count(c) as count order by count limit 3"
    1) 1) "p.id"
       2) "p.name"
       3) "count"
    2) 1) 1) (integer) 763
          2) "Small Bronze Computer"
          3) (integer) 12
       2) 1) (integer) 984
          2) "Fantastic Plastic Gloves"
          3) (integer) 13
       3) 1) (integer) 266
          2) "Durable Linen Clock"
          3) (integer) 14
    3) 1) "Query internal execution time: 125.494000 milliseconds" 
    ```
       
5. Return products not purchased. This query will match a path for every node of label `product` with alias `p` where no node of label `order` with directional edge
type `contain` points to said `product`-labeled node `p`. Return the count.
    
    {{<highlightjs language="cypher">}}
match (p:product)
where not (:order)-[:contain]->(p)
return count(p)
    {{</highlightjs>}}
    
    Executed:    
    
    ```
    127.0.0.1:6379> graph.query prodrec "match (p:product) where not (:order)-[:contain]->(p) return count(p)"
    1) 1) "count(p)"
    2) (empty array)
    3) 1) "Query internal execution time: 357.291000 milliseconds"   
    ```
   
    The default settings of the script mean that this particular query usually does not return results.
    
6. Queries can be strung together linearly using [`with`](https://oss.redislabs.com/redisgraph/commands/#with) allowing for their individual execution
and results handling. Consider the following example where we want the count of `view`, `addtocart`, and `contain`-typed edges along with a `product`-labled node's `name`.

    {{<highlightjs language="cypher">}}
match (prod:product {id: 393})<-[c:contain]-(:order)
with prod, count(c) as orders
match (prod)<-[atc:addtocart]-(:person)
with prod, orders, count(atc) as addstocarts
match (prod)<-[v:view]-(:person)
with prod, orders, addstocarts, count(v) as views
return prod.name, orders, addstocarts, views      
    {{</highlightjs>}}
   
    The initial `product`-labeled node aliased with `prod` is matched based off the `id` property set to `393`.
    
    Note the pattern of "match with". Each `match` statement runs like a pipeline, passing the parameters to the next.     
    
    - In the first we create a path from `order`-labeled nodes via the `contain`-typed edge with an alias `c`.
    The alias exists so we can count and pass the value along with the `product`-labeled node aliased as `prod`.
    - In the second we create a path from the passed `prod` from a `person`-labeled node via the `addtocart`-typed edge with an alias `atc`. Because we want the
    add to cart count and the order count we are passing out what we took in.
    - In the final "match with" we take the three inputs and create a path from the passed `prod` from a `person`-labeled node via the `view`-typed edge with an
    alias `v`.
    
    The final return statement returns the product name along with each value.
    
    Executed:  
   
    ```
    127.0.0.1:6379> graph.query prodrec "MATCH (prod:product {id: 393})<-[c:contain]-(:order) WITH prod, count(c) as orders MATCH (prod)<-[atc:addtocart]-(:person) WITH prod, orders, count(atc) as addstocarts MATCH (prod)<-[v:view]-(:person) WITH prod, orders, addstocarts, count(v) as views return prod.name, orders, addstocarts, views"
    1) 1) "prod.name"
       2) "orders"
       3) "addstocarts"
       4) "views"
    2) 1) 1) "Gorgeous Cotton Shoes"
          2) (integer) 30
          3) (integer) 33
          4) (integer) 131
    3) 1) "Query internal execution time: 5.319900 milliseconds"
    ```
        
7. [Merge](https://oss.redislabs.com/redisgraph/commands/#merge) statements ensure a path exists in the graph and allow you to take
some action when "on match" or "on create" conditions are met. Take the following example where instead of computing the values from
the last example each time, we wanted to periodically set them on the nodes themselves.     

    {{<highlightjs language="cypher">}}
match (prod:product)<-[c:contain]-(:order) with prod, count(c) as orders
match (prod)<-[atc:addtocart]-(:person)
with prod, orders, count(atc) as addstocarts
match (prod)<-[v:view]-(:person)
with prod, orders, addstocarts, count(v) as views
merge (prod)
  on match set prod.num_orders = orders, prod.num_adds_to_carts = addstocarts, prod.num_views = views
return prod.id, orders, addstocarts, views
    {{</highlightjs>}}
   
    This query very similar to the first except for a few ways. The initial `match` statement doesn't single out a particular product.
    A merge statement exists on the `prod` alias for the `product`-labeled node that was "match with"-ed three times. On a match, the
    statement takes the passed values at establishes them on the `product`-labeled node itself. The values are returned along with
    the product id.
    
    Executed (with omitted nodes):

    ```
    127.0.0.1:6379> graph.query prodrec "MATCH (prod:product)<-[c:contain]-(:order) WITH prod, count(c) as orders MATCH (prod)<-[atc:addtocart]-(:person) WITH prod, orders, count(atc) as addstocarts MATCH (prod)<-[v:view]-(:person) WITH prod, orders, addstocarts, count(v) as views merge (prod) ON MATCH SET prod.num_orders = orders, prod.num_adds_to_carts = addstocarts, prod.num_views = views return prod.id, orders, addstocarts, views"
    1) 1) "prod.id"
       2) "orders"
       3) "addstocarts"
       4) "views"
    2) 1) 1) (integer) 0
          2) (integer) 29
          3) (integer) 33
          4) (integer) 127
   
    // omitted nodes //
   
     406) 1) (integer) 393
          2) (integer) 30
          3) (integer) 33
          4) (integer) 131
   
    // omitted nodes //
   
    1000) 1) (integer) 999
          2) (integer) 27
          3) (integer) 28
          4) (integer) 126
    3) 1) "Properties set: 3000"
    2) "Query internal execution time: 263.394600 milliseconds"    
    ```
   
    Here we query for the node via:
    
    {{<highlightjs language="cypher">}}
match (p:product) where p.id=333 return p    
    {{</highlightjs>}}
    
    ... in RedisInsight to produce and show the results from the `merge`.
    
    ![image](/img/2020-5-redis-graph-product-recommendation-opencypher/merged_prod_node.png)
             
The next section will get into more powerful paths through the nodes and edges in the system and begin to show the power of openCypher and the graph.            
           
### Basic Product Recommendation Queries

There are a number of ways you could extract meaningful information about what to recommend to a user. These next few examples highlight a very crude
approach to recommendations where we want to show all the `product` nodes that share orders in common with the `product` nodes in the `transacted` 
`order` nodes for a given person. In the first example the person targeted is identified by their `id` property `294`.   

1. Find products that are in orders that have common products placed by person id 294. Limit results to 1 (so this blog post isn't terribly long).

    {{<highlightjs language="cypher">}}
match (p:person)-[:transact]->(:order)-[:contain]->(:product)<-[:contain]-(:order)-[:contain]->(prd:product)
where p.id=294
return distinct prd limit 1
    {{</highlightjs>}} 

    This statement really shows the benefit of cypher over something like SQL, which would require many joins to accomplish the same query. Here we trace the path,
    telling the graph to match `people`-labeled nodes, through the `product`-labeled nodes they've purchased to the `product`-labeled nodes of all orders that
    shared that `product` in common.
    
    This query only aliases the `person` labeled node at the start `p` and the `product` labeled node at the end `prd`. Given, though,
    that `p` only exists so that can use it to filter based on its id in the `where` clause.
    
    Executed:
    
    ```
    127.0.0.1:6379> graph.query prodrec "match (p:person)-[:transact]->(:order)-[:contain]->(:product)<-[:contain]-(:order)-[:contain]->(prd:product) where p.id=294 return distinct prd limit 1"
    1) 1) "prd"
    2) 1) 1) 1) 1) "id"
                2) (integer) 5000
             2) 1) "labels"
                2) 1) "product"
             3) 1) "properties"
                2) 1) 1) "id"
                      2) (integer) 1
                   2) 1) "name"
                      2) "Gorgeous Plastic Wallet"
                   3) 1) "manufacturer"
                      2) "Terry and Sons"
                   4) 1) "msrp"
                      2) "42.46"
    3) 1) "Query internal execution time: 1.889600 milliseconds"   
    ```
  
    This could be re-written as:
    
    {{<highlightjs language="cypher">}}
match (p:person { id: 294 })-[:transact]->(:order)-[:contain]->(:product)<-[:contain]-(:order)-[:contain]->(prd:product)
return distinct prd   
    {{</highlightjs>}}   
    
    This query returns the alias for `product`, `prd`, itself and therefor we get all the details for that node.
    
2. The following is a minor variation over the last example where we go a step further to remove the users purchased products from the list of recommendations.

    {{<highlightjs language="cypher">}}
match (p:person { id: 294 })-[:transact]->(:order)-[:contain]->(prod:product)
match (prod)<-[:contain]-(:order)-[:contain]->(rec_prod:product)
where not (p)-[:transact]->(:order)-[:contain]->(rec_prod)
return distinct rec_prod.id, rec_prod.name
    {{</highlightjs>}}
   
    This statement shows us splitting up the work between multiple `match` directives. In the first we resolve the `person`-labeled node
    with the `id` property set to `294` and return all the `product`-labeled nodes in said persons orders aliased by `prod`.
    
    In the second we take each `product`-labled node (`prod` alias) and look for all "products of products" where that product is not in a
    transacted order by said person.
    
    Executed:
   
    ```
    127.0.0.1:6379> graph.query prodrec "match (p:person { id: 294 })-[:transact]->(:order)-[:contain]->(prod:product) match (prod)<-[:contain]-(:order)-[:contain]->(rec_prod:product) where not (p)-[:transact]->(:order)-[:contain]->(rec_prod) return distinct rec_prod.id, rec_prod.name limit 1"
    1) 1) "rec_prod.id"
       2) "rec_prod.name"
    2) 1) 1) (integer) 1
          2) "Gorgeous Plastic Wallet"
    3) 1) "Query internal execution time: 2.397600 milliseconds"   
    ```
     
    This query is marginally more useful than the last but there's a lot of room for improvement. One such improvement might be to sort 
    and pick only the most popular products to return from the subset.

3. Returning the most popular 3 products as defined by inbound relationships to each product -- this query builds on the former.

    {{<highlightjs language="cypher">}}
match (p:person { id: 294 })-[:transact]->(:order)-[:contain]->(prod:product)
match (prod)<-[:contain]-(:order)-[:contain]->(rec_prod:product)
where not (p)-[:transact]->(:order)-[:contain]->(rec_prod)
return rec_prod.id, rec_prod.name
order by indegree(prod) desc limit 3  
    {{</highlightjs>}}
   
    Here we layer in a crude attempt at finding the popularity of the product by using the [node function](https://oss.redislabs.com/redisgraph/commands/#node-functions)
    that returns the number of inbound connections, or relationships, or edges for each product.
    
    Executed:
    
    ```
    127.0.0.1:6379> graph.query prodrec "match (p:person { id: 294 })-[:transact]->(:order)-[:contain]->(prod:product) match (prod)<-[:contain]-(:order)-[:contain]->(rec_prod:product) where not (p)-[:transact]->(:order)-[:contain]->(rec_prod) return rec_prod.id, rec_prod.name order by indegree(prod) desc limit 3"
    1) 1) "rec_prod.id"
       2) "rec_prod.name"
    2) 1) 1) (integer) 33
          2) "Mediocre Steel Gloves"
       2) 1) (integer) 54
          2) "Small Linen Wallet"
       3) 1) (integer) 55
          2) "Heavy Duty Silk Pants"
    3) 1) "Query internal execution time: 132.870600 milliseconds" 
    ```
   
    If the query is re-written to return the `product`-labeled nodes themselves we can produce a graph in RedisInsight.
    
    {{<highlightjs language="cypher">}}
match (p:person { id: 294 })-[:transact]->(:order)-[:contain]->(prod:product)
match (prod)<-[:contain]-(:order)-[:contain]->(rec_prod:product)
where not (p)-[:transact]->(:order)-[:contain]->(rec_prod)
return rec_prod
order by indegree(prod) desc limit 3  
    {{</highlightjs>}}
   
    ![image](/img/2020-5-redis-graph-product-recommendation-opencypher/indegree_query.png)
    
    And selecting one of those products, like the "Heavy Duty Silk Pants", shows an expansion of all that product's connections. RedisInsight allows exports
    of graph views which is very, very slick.
    
    ![image](/img/2020-5-redis-graph-product-recommendation-opencypher/indegree_graph_expansion_export.png)     
    
    Another optimization on this query might be to calculate, rank products recommended based on a rating that `people` give `products`. Or,
    iterating on this solution and instead of getting the inbound connection count, get the count for each known edge type and weight them. It might
    also be useful to have abandoned cart or removal from cart events in place of add to cart events. As it is currently written, the graph
    generation tooling code generates a random set of `view` edges for products and from that a subset is created for `addtocart` edges and from that
    another subset is created that are "contained" in an order. Tracking "negative" events, particular the two aforementioned ones, might be more
    useful than the existing `addtocart` edge.
        
4. The final example of recommendations is a query inspired by a work colleague of mine, [Matthew Huckaby](https://www.linkedin.com/in/matthew-huckaby-928ab02/) (on [Twitter](http://twitter.com/makingElements) and [GH](https://github.com/mhuckaby)).
In this query we want to find the top 3 products (`id` and `name`) viewed by people who have purchased queried product, id `393`.
   
    {{<highlightjs language="cypher">}}
match (prod:product)<-[v:view]-(p:person)-[:transact]->(:order)-[:contain]->(:product { id: 393 })
return prod.id, prod.name, count(v) as count
order by count desc limit 3       
    {{</highlightjs>}}
   
    In this query we specify a path to match against that points in both directions -- which can be a bit mind blowing the first
    time you try an decipher... the cypher. Ha.
    
    Let's start in the "center" with our `person`-labeled node with the alias `p`. To the right we're asking for `transact`-typed edges pointing inbound
    to `order`-labeled nodes. The order nodes should have `contain`-typed edges pointing inbound to a `product`-labeled node. This node is the node we query
    against with the `id` property set to `393`. Note that only the `person` node is aliased because everything else is just to query / filter for
    match against.
    
    To the left of the `person` node we have an inbound relationship requirement of `view` with the alias `v` inbound to a `product` node
    with the alias `prod`.
    
    The return statement pulls the product `id` and `name` properties in addition to the count of `v` which gives us `views` within the path match.
    
    From there things are smooth sailing with ordering and limits.
    
    Executed:            
   
    ```
    127.0.0.1:6379> graph.query prodrec "match (prod:product)<-[v:view]-(p:person)-[:transact]->(:order)-[:contain]->(:product { id: 393 }) return prod.id, prod.name, count(v) order by count(v) desc limit 3"
    1) 1) "prod.id"
       2) "prod.name"
       3) "count(v)"
    2) 1) 1) (integer) 393
          2) "Practical Wool Hat"
          3) (integer) 35
       2) 1) (integer) 17
          2) "Small Marble Gloves"
          3) (integer) 5
       3) 1) (integer) 266
          2) "Practical Cotton Shoes"
          3) (integer) 5
    3) 1) "Query internal execution time: 10.226500 milliseconds"
    ```                
   
### What's next?

I hope you found this informative. Tinkering with Cypher/openCypher definitely requires clearing some mental hurdles in understanding. I find
using a mix of technical docs with specific, broken down examples a great way of reinforcing learning.

The next few posts will be on the topics of performance and further optimizations to the query.   

### Additional Reading

The following documents are really useful for getting started with things. It is important to node that Redis Graph is implementing most but (yet) all of the openCypher commands and spec.

1. the [Cypher Query Language Reference (Version 9)](https://s3.amazonaws.com/artifacts.opencypher.org/openCypher9.pdf) from the [OpenCypher resources](http://www.opencypher.org/resources)
2. the [Cypher Style Guide](https://s3.amazonaws.com/artifacts.opencypher.org/M15/docs/style-guide.pdf) also from the [OpenCypher resources](http://www.opencypher.org/resources)
3. [Redis Graph Commands](https://oss.redislabs.com/redisgraph/commands/) for a full list of supported commands
4. [Redis Graph Cypher Coverage](https://oss.redislabs.com/redisgraph/cypher_support/) for notes on what commands aren't covered (yet) by Redis Graph
5. [learn x in y minutes (cypher)](https://learnxinyminutes.com/docs/cypher/)

Cheers and Happy RedisConf day two!   