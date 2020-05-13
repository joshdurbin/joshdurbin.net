+++
title = "Product Recommendations in RedisGraph, Part 2: openCypher Query Basics"
date = "2020-05-07"
tags = ["graph", "redis", "recommendations", "data", "opencypher", "redisgraph"]
mermaid = true
+++

This post is a continuation of the series on leveraging RedisGraph for product recommendations.

Back in Fall 2015 [Emil Eifrem](https://www.linkedin.com/in/emileifrem/), CEO/co-founder of [Neo4j](https://neo4j.com), [announced](https://neo4j.com/blog/open-cypher-sql-for-graphs/) [openCypher]() at
GraphConnect in San Francisco. OpenCypher, inspired by Cypher, was Neo4j's 3rd attempt at a graph query language. The growth and adoption of graph tech they experienced alongside other vendors began to expose and emphasize the need for a common
query language against Graphs -- someting similar to [SQL](https://en.wikipedia.org/wiki/SQL) for relational DBs. With Emil's announcement [openCypher](https://www.opencypher.org) was born. Here we'll talk about
how to use openCypher generally and in the context of the product recommendation POC / [RedisGraph](http://redisgraph.io).

### Graph Basics

I won't dive into an exhaustive explanation of graphs here (see [Wikipedia](https://en.wikipedia.org/wiki/Graph_(abstract_data_type))), but, in very broad strokes, they are structures composed of nodes and relationships. Relationships connect nodes
and can be directed. There are also [many different types](https://en.wikipedia.org/wiki/Graph_(discrete_mathematics)#Types_of_graphs) of graphs. Redis Graph specifically is a directed, labeled multigraph where both nodes and relationships are labeled (and, optionally, contain
properties). Generally speaking, graphs accel at deriving information from the interconnectedness of data -- as opposed to SQL or document systems where similar queries would require joining many tables or linking many collections.

Consider the following output from [RedisInsight](https://redislabs.com/redisinsight/) on a graph created from my commerce recommendation [POC tooling](http://github.com/joshdurbin/redis-graph-commerce-poc) (the process for
generating it is outlined in [this post]({{< ref "/posts/2020-4-redis-graph-product-recommendation-generator-revamped.md" >}})):

![image](/img/2020-5-redis-graph-product-recommendation-opencypher/subgraph.png)

Though it's not highlighted; the center-most, light-green node (with the giant red arrow pointing at it) is a node of type `person` and is the center point
for this graph expansion. In this example `person` has relationships with many `product` nodes by three paths: 

1. When the `person` _viewed_ a `product`
2. When the `person` _added a product_ to their cart
3. When the `person` placed (_transacted_) an `order` which _contained_ the product

In addition to these patterns in the graph, `view` and `addtocart` edges exist between nodes when a product is viewed or added to the cart. To keep the sample dataset as
realistic as possible, `addtocart` edges are only created from a subset of the `view` edges that exist between a `person` and `product` node. These
circumstances highlight Redis Graphs ability or utility as a [multigraph](https://en.wikipedia.org/wiki/Multigraph) -- when any node has parallel edges between itself and another node.  

This example contains only directed relationships, which are denoted by the arrows in the graph visualization.

### Graph Creation

The [graph generation](https://github.com/joshdurbin/redis-graph-commerce-poc/blob/master/generateCommerceGraph) tooling issues two types of commands against Redis Graph to seed and populate the graph:

1. The creation of nodes occurs via `create` statements. Three node "types" (not RedisGraph types, as they apply to edges/relationships), labels exist in the form of the following: `person`, `product`, and `order`. 
The following are the create statements used to insert these nodes into the graph.     
    
    ```
    CREATE (:person {
        id: 4,
        name:"Javier Bashirian",
        age:94,
        address:"36186 Koelpin Isle, Lake Rashidahaven, OH 78740",
        memberSince:"2011-01-07T07:49:21.274235"})
    ```
    
    Each `CREATE` statement takes a node enclosed by parenthesis with its label defined with a colon and the label ex: `person`. Properties for nodes are defined 
    within the bounds of left and right curly brackets `{ }`. Each property must contain a string key separated by a colon and the value, which must equal one of
    the [supported types](https://oss.redislabs.com/redisgraph/cypher_support/#types). Multiple properties are separated by commas.
    
    Note: In these examples `id` is a property the application is setting on the node, which can be used for querying/matching,
    but is not the unique ID of the node in the graph itself -- an important distinction. The internal ID of a node or
    relationship can be obtained via the [Scalar function](https://oss.redislabs.com/redisgraph/commands/#scalar-functions) `id()`.
    
    In the following statement two nodes of the label `product` are created with their various properties.      
            
    ```
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
    ```
   
    Per the openCypher spec, nodes are supposed to support multiple labels and edges or relationships are supposed to support multiple types. As of this writing,
    RedisGraph does not support at least multiple node labels, but there is an [issue](https://github.com/RedisGraph/RedisGraph/issues/910) tracking that enhancement.
    I'm uncertain of 
    
    Our final node type created in this POC is of label `order`:    
         
    ```
    CREATE
        (:order {
            id: 2,
            subTotal:1565.85,
            tax:176,
            shipping:208,
            total:1949.85}
        )
    ```        

2. The next crucial, and, arguably more important part of this graph creation is the relationships, the edges. As previously covered they do the connecting,
the entire reason for having a graph in the first place.

    Here the tooling is instructing the graph to execute a sort of conditional create where its first matching nodes of various types, then creating
    a relationship between those nodes. `MATCH` is used to tell the graph to find the nodes, a path, relationships, etc...  
      
    Consider the following statement:  
    
    ```
    MATCH
        (p:person), (o:order), (prd3:product), (prd0:product)
    WHERE
        p.id=4 AND o.id=2 AND prd3.id=3 AND prd0.id=0
    CREATE
        (p)-[:transact]->(o), (o)-[:contain]->(prd3), (o)-[:contain]->(prd0)
    ```
    
    This query instructs the graph to match and find nodes with labels `person`, `order`, and `product`. These look similar to the create statements before except the
    labels have a leading identifier prior to the colon and label declaration. These identifiers, termed an [alias](https://oss.redislabs.com/redisgraph/commands/#match), `p`, `o`, `prd3`, and `prd0` are exclusive to the query. They could be
    _anything_ -- graph generation app uses sensible names for these aliases, though, which is why `prd3` and `prd0` are used. These aliases in and of themselves
    do not restrict us to "product 3" and "product 0". The restrictions to those product ids come in the `WHERE` segment of the query: 
    
    ```
    WHERE p.id=4 AND o.id=2 AND prd3.id=3 AND prd0.id=0
    ```
    
    ...where alias `p`, which is a node label `person`, with property `id` equal to `4`, etc...
    
    **Only when** this conditional is met do we create our edges, our relationships:
    
    ```
    CREATE(p)-[:transact]->(o), (o)-[:contain]->(prd3), (o)-[:contain]->(prd0)
    ```
    
    The `CREATE` statement here is creating three edges of `type`, `transact`, and (2x) `contain`. It's worth noting that
    these edges _could_ contain properties -- we'll see an example of this in our next example. The `create` statements
    are leveraging the same aliases we used for our `MATCH` statement, which now makes it a little more clear why they are
    explicitly named `p`, `o`, `prd3`, and `prd0`.
    
    The prior `MATCH` statement associated our `person`-labeled node with the created `order`-labeled node and the order
    with the `product`-labeled nodes the `order` contains. The next statement associates our `person` node with `product` nodes they've viewed and added to their carts.    
    
    ```
    MATCH
        (p:person), (prd1:product), (prd3:product), (prd2:product), (prd0:product), (prd4:product)
    WHERE
        p.id=4 AND prd1.id=1 AND prd3.id=3 AND prd2.id=2 AND prd0.id=0 AND prd4.id=4
    CREATE
        (p)-[:view {time: '2018-07-18T15:54:21.274235'}]->(prd1),
        (p)-[:view {time: '2013-04-16T02:10:21.274235'}]->(prd3),
        (p)-[:view {time: '2012-10-22T15:51:21.274235'}]->(prd2),
        (p)-[:view {time: '2012-11-05T18:29:21.274235'}]->(prd0),
        (p)-[:view {time: '2017-09-02T15:22:21.274235'}]->(prd4),
        (p)-[:addtocart {time: '2013-04-19T02:01:21.274235'}]->(prd3),
        (p)-[:addtocart {time: '2012-11-07T14:43:21.274235'}]->(prd0)
    ```
    
    This example has a few more aliases, but primarily differs from the other in that a property is set on the edge, on the relationship. Note that the syntax
    for doing so is identical to the nodes. 
    
    ```
    (p)-[:addtocart {time: '2012-11-07T14:43:21.274235'}]->(prd0)
    ```
   
    Types and syntax are the same for relationships as for nodes.
    
### Querying the Graph

Note: some queries include query times. The graph used for these queries and all subsequent examples was created [by the generator](https://github.com/joshdurbin/redis-graph-commerce-poc/blob/master/generateCommerceGraph).  

1. Query the distinct labels in the graph. The following query will match any node, any match aliased with `n`, returning
distinct results from the `labels()` function.

    ```
    127.0.0.1:6379> graph.query prodrec "match (n) return distinct labels(n)"
    1) 1) "labels(n)"
    2) 1) 1) "person"
       2) 1) "product"
       3) 1) "order"
    3) 1) "Query internal execution time: 10.429400 milliseconds"
    ```
   
   An alternative approach to this specific query is calling the `db.labels()` function. This is an optimized function and should be preferred for this type of query. 
   
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
    be preferred for this type of query.   
    
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

    ```
    127.0.0.1:6379> graph.query prodrec "match (p:person {id: 3}) return id(p)"
    1) 1) "id(p)"
    2) 1) 1) (integer) 3
    3) 1) "Query internal execution time: 0.389400 milliseconds"    
    ```
7. Obtain the id of a relationship or edge. This query is similar to aforementioned query in ex: #2; match
any directed relationship, aliased with `e`, between an un-aliased source and destination node of any label,
returning the ID of the relationship via the `id` function.
    
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

    ```
    match
        (p:person)-[:transact]->(o:order)
    return p.id, p.name, count(o) as orders
    order by orders desc limit 3
    ``` 

    ```
    127.0.0.1:6379> graph.query prodrec "match (p:person)-[:transact]->(o:order) return p.id, p.name, count(o) as orders order by orders desc limit 3"
    1) 1) "p.id"
       2) "p.name"
       3) "orders"
    2) 1) 1) (integer) 98
          2) "Abel Heathcote"
          3) (integer) 15
       2) 1) (integer) 1875
          2) "Masako Smith"
          3) (integer) 14
       3) 1) (integer) 1172
          2) "Antoinette Von"
          3) (integer) 14
    3) 1) "Query internal execution time: 45.437700 milliseconds"   
    ```
     
2. Return the top 3 most ordered products (`id` and `name` properties). This query will match a path where a node of label `order` with
    a directional edge of type `contain` with an alias `c` to a node of label `product` with alias `p`. The `id`, `name`, and count as `count` are returned and limited to 3.

    ```
    match
        (:order)-[c:contain]->(p:product)
    return p.id, p.name, count(c) as count
    order by count limit 3
    ```

    ```
    127.0.0.1:6379> graph.query prodrec "match (:order)-[c:contain]->(p:product) return p.id, p.name, count(c) as count order by count limit 3"
    1) 1) "p.id"
       2) "p.name"
       3) "count"
    2) 1) 1) (integer) 696
          2) "Ergonomic Aluminum Pants"
          3) (integer) 15
       2) 1) (integer) 206
          2) "Synergistic Silk Knife"
          3) (integer) 15
       3) 1) (integer) 313
          2) "Heavy Duty Plastic Plate"
          3) (integer) 16
    3) 1) "Query internal execution time: 89.558300 milliseconds"   
    ```

3. Return the top 3 most viewed products (`id` and `name` properties). This query will match a path where a node of label `person` with
a directional edge of type `view` with alias `v` to a node of label `product` with alias `p`. The `id`, `name`, and count as `count` are returned and limited to 3.

    ```
    match
        (:person)-[v:view]->(p:product)
    return p.id, p.name, count(v) as count
    order by count limit 3
    ```

    ```
    127.0.0.1:6379> graph.query prodrec "match (:person)-[v:view]->(p:product) return p.id, p.name, count(v) as count order by count limit 3"
    1) 1) "p.id"
       2) "p.name"
       3) "count"
    2) 1) 1) (integer) 37
          2) "Incredible Marble Coat"
          3) (integer) 82
       2) 1) (integer) 518
          2) "Mediocre Iron Plate"
          3) (integer) 82
       3) 1) (integer) 814
          2) "Heavy Duty Wool Plate"
          3) (integer) 87
    3) 1) "Query internal execution time: 155.783600 milliseconds"   
    ```
   
4. Return the 3 fewest purchased products (`id` and `name` properties). This query will match a path where a node of label `order` with
a directional edge of type `contain` with alias `c` to a node of label `product` with alias `p`. The `id`, `name`, and count as `count` are returned in descending order and limited to 3.

    ```
    match
        (:order)-[c:contain]->(p:product)
    return p.id, p.name, count(c) as count
    order by count desc limit 3
    ```       

    ```
    127.0.0.1:6379> graph.query prodrec "match (:order)-[c:contain]->(p:product) return p.id, p.name, count(c) as count order by count desc limit 3"
    1) 1) "p.id"
       2) "p.name"
       3) "count"
    2) 1) 1) (integer) 65
          2) "Enormous Wooden Pants"
          3) (integer) 47
       2) 1) (integer) 797
          2) "Synergistic Cotton Knife"
          3) (integer) 44
       3) 1) (integer) 442
          2) "Fantastic Aluminum Clock"
          3) (integer) 43
    3) 1) "Query internal execution time: 115.636500 milliseconds"   
    ```
       
5. Return products not purchased. This query will match a path for every node of label `product` with alias `p` where no node of label `order` with directional edge
type `contain` points to said `product`-labeled node `p`. Return the count.
    
    ```
    match
        (p:product)
    where not
        (:order)-[:contain]->(p)
    return count(p)
    ```    
    
    ```
    127.0.0.1:6379> graph.query prodrec "match (p:product) where not (:order)-[:contain]->(p) return count(p)"
    1) 1) "count(p)"
    2) (empty array)
    3) 1) "Query internal execution time: 357.291000 milliseconds"   
    ```
   
    The default settings of the script mean that this particular query usually does not return results.
    
6. Queries can be strung together linearly using [`with`](https://oss.redislabs.com/redisgraph/commands/#with) allowing for their individual execution
and results handling. Consider the following example where we want the count of `view`, `addtocart`, and `contain`-typed edges along with a `product`-labled node's `name`.

    ```
    match
        (prod:product {id: 393})<-[c:contain]-(:order)
    with
        prod, count(c) as orders
    match
        (prod)<-[atc:addtocart]-(:person)
    with
        prod, orders, count(atc) as addstocarts
    match
        (prod)<-[v:view]-(:person)
    with
        prod, orders, addstocarts, count(v) as views
    return
        prod.name, orders, addstocarts, views      
    ```
   
    The initial `product`-labeled node aliased with `prod` is matched based off the `id` property set to `393`.
    
    Note the pattern of "match with". Each `match` statement runs like a pipeline, passing the parameters to the next.     
    
    - In the first we create a path from `order`-labeled nodes via the `contain`-typed edge with an alias `c`.
    The alias exists so we can count and pass the value along with the `product`-labeled node aliased as `prod`.
    - In the second we create a path from the passed `prod` from a `person`-labeled node via the `addtocart`-typed edge with an alias `atc`. Because we want the
    add to cart count and the order count we are passing out what we took in.
    - In the final "match with" we take the three inputs and create a path from the passed `prod` from a `person`-labeled node via the `view`-typed edge with an
    alias `v`.
    
    The final return statement returns the product name along with each value.  
   
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

    ```
    match
        (prod:product)<-[c:contain]-(:order)
    with
        prod, count(c) as orders
    match
        (prod)<-[atc:addtocart]-(:person)
    with
        prod, orders, count(atc) as addstocarts
    match
        (prod)<-[v:view]-(:person)
    with
        prod, orders, addstocarts, count(v) as views
    merge
        (prod)
    on match set
        prod.num_orders = orders, prod.num_adds_to_carts = addstocarts, prod.num_views = views
    return
        prod.id, orders, addstocarts, views
    ```
   
    This query very similar to the first except for a few ways. The initial `match` statement doesn't single out a particular product.
    A merge statement exists on the `prod` alias for the `product`-labeled node that was "match with"-ed three times. On a match, the
    statement takes the passed values at establishes them on the `product`-labeled node itself. The values are returned along with
    the product id.

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
     406) 1) (integer) 393
          2) (integer) 30
          3) (integer) 33
          4) (integer) 131
    1000) 1) (integer) 999
          2) (integer) 27
          3) (integer) 28
          4) (integer) 126
    3) 1) "Properties set: 3000"
    2) "Query internal execution time: 263.394600 milliseconds"    
    ```      
    
The next section will get into more powerful paths through the nodes and edges in the system and begin to show the power of openCypher and the graph.            
           
### Basic Product Recommendation Queries

The following query as adapted from the queries in [this gist](https://gist.githubusercontent.com/adam-cowley/79923b538f6851d30e08/raw/abe673493fafba9f2cfd43cba0d7c6b334e6b01e/neo4j-northwind-recommendation).

There are a number of ways you could extract meaningful information about what to recommend to a user. 

1. Find products that are in orders that have common products placed by person id 294. Limit results to 1 (so this blog post isn't terribly long).

    ```
    match
        (p:person)-[:transact]->(:order)-[:contain]->(:product)<-[:contain]-(:order)-[:contain]->(prd:product)
    where
        p.id=294
    return distinct prd limit 1
    ``` 

    This statement really shows the benefit of cypher over something like SQL, which would require many joins to accomplish the same query. Here we trace the path,
    telling the graph to match `people`-labeled nodes, through the `product`-labeled nodes they've purchased to the `product`-labeled nodes of all orders that
    shared that `product` in common.
    
    This query only aliases the `person` labeled node at the start `p` and the `product` labeled node at the end `prd`. Given, though,
    that `p` only exists so that can use it to filter based on its id in the `where` clause.
    
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
    
    ```
    match
        (p:person { id: 294 })-[:transact]->(:order)-[:contain]->(:product)<-[:contain]-(:order)-[:contain]->(prd:product)
    return distinct prd   
    ```   
    
    Note that the query returns the alias for `product`, `prd`, itself and therefor we get all the details for that node.
    
2. The following is a slight variation of the last example where we remove the users purchased products from the list of recommendations.

    ```
    match
        (p:person { id: 294 })-[:transact]->(:order)-[:contain]->(prod:product)
    match
        (prod)<-[:contain]-(:order)-[:contain]->(rec_prod:product)
    where not
        (p)-[:transact]->(:order)-[:contain]->(rec_prod)
    return distinct rec_prod.id, rec_prod.name
    ```
   
    This statement shows us splitting up the work between multiple `match` directives. In the first we resolve the `person`-labeled node
    with the `id` property set to `294` and return all the `product`-labeled nodes in said persons orders aliased by `prod`.
    
    In the second we take each `product`-labled node (`prod` alias) and look for all "products of products" where that product is not in a
    transacted order by said person.
   
    ```
    127.0.0.1:6379> graph.query prodrec "match (p:person { id: 294 })-[:transact]->(:order)-[:contain]->(prod:product) match (prod)<-[:contain]-(:order)-[:contain]->(rec_prod:product) where not (p)-[:transact]->(:order)-[:contain]->(rec_prod) return distinct rec_prod.id, rec_prod.name limit 1"
    1) 1) "rec_prod.id"
       2) "rec_prod.name"
    2) 1) 1) (integer) 1
          2) "Gorgeous Plastic Wallet"
    3) 1) "Query internal execution time: 2.397600 milliseconds"   
    ```
   
    This query is marginally more useful but we still really need additional data and/or queries or partial queries to rank the recommendations.
    
### Followup Query Post     
                       
The post following this will be on easy ways to concurrently stress Redis and RedisGraph with the prior query. The post following that will dig deeper into
additional queries and usage of additional data, like product reviews, to better target / identify which products to share users. We'll be answering questions like:

"Find all products viewed by people who have purchased the thing that the current user is looking at..."

Stay tuned!                                  

### Additional Reading

1. the [Cypher Query Language Reference (Version 9)](https://s3.amazonaws.com/artifacts.opencypher.org/openCypher9.pdf) from the [OpenCypher resources](http://www.opencypher.org/resources)
2. the [Cypher Style Guide]() also from the [OpenCypher resources](http://www.opencypher.org/resources)
3. [Redis Graph Commands](https://oss.redislabs.com/redisgraph/commands/) for a full list of supported commands
4. [Redis Graph Cypher Coverage](https://oss.redislabs.com/redisgraph/cypher_support/) for notes on what commands aren't covered (yet) by Redis Graph
5. [learn x in y minutes (cypher)](https://learnxinyminutes.com/docs/cypher/)
  