+++
title = "Building a product recommendation engine using RedisGraph and OpenCypher, Part 1.1: Better data loading"
date = "2020-04-18"
tags = ["graph", "redis", "recommendations", "data", "concurrency"]
+++

This is a follow-up post to [one back in January]({{< ref "/posts/2020-1-redis-graph-product-recommendation-part-1-data-loading.md" >}}) related to using RedisGraph and OpenCypher to build a product recommendations engine. In that post I mentioned I'd have a follow up on the queries and OpenCypher basics, but this post is not that. That's coming next week. I Promise!

This post is to discuss changes to the [graph generator](https://github.com/joshdurbin/redis-graph-commerce-poc/blob/master/generateCommerceGraph). I've put a decent chunk of effort into tweaking the performance, configurability, etc.. of the generator and you might ask yourself why? Why spend time generating fake data? Well, a few things:

1. Earlier this month RedisGraph [released version 2.0](https://redislabs.com/blog/introducing-redisgraph-2-0/) which includes a ton of really great updates and performance/stability wins. That said, though RedisGraph is continually improving its resiliency and stability, it does occasionally bonk and when it does, so does your data. So, it can be helpful to understand the most efficient way of getting data into RedisGraph, performance bottlenecks, etc...
2. Altering the "shape" and "depth" of data drastically changes performance impact of queries, DB snapshotting, sync and startup performance, etc...
3. The data is as realistic as I could make fake data; ex:
  - people (customers) have join dates and orders are always after
  - add to cart events are a subset of product view events
  - purchased events are randomized across orders for a customer and they only purchase products from the pool of products added to their cart
  - pricing is realistic, cost, shipping, tax, etc...

So, what sort of things can we tweak about the graph creation?

1. The (min/max) number of people created
2. The (min/max) number of products created
3. The (min/max) views for a person and product
4. The percentage of views for a person which "upgrade" to add to cart events
5. The percentage of add to cart events which "upgrade" to purchased events
6. The max random time from viewed to add to cart
7. The max random time from add to cart to purchased
8. The (min/max) product price, tax rate for orders, ship rate for orders
9. Other utility things, such as the redis endpoint details, batch sizes for commits, thread count, etc...

Running the utility is straight forward, all you need is Java 11 (I'm using OpenJDK 11) and Groovy 3.x. There are many maven dependencies which are loaded automatically via the Grape directives in the script.

Here's a snapshot, output of the help menu for the script. The script requires no parameters and will use the defaults stated in the help menu below.

```
./generateCommerceGraph -h
usage: generateCommerceGraph
Commerce Graph Generator
 -db,--database <arg>                                         The RedisGraph database to use for our queries, data generation [defaults to prodrec]
 -h,--help                                                    Usage Information
 -maxatct,--maxRandomTimeFromViewToAddToCartInMinutes <arg>   The max random time from view to add-to-cart for a given user and product [defaults to 4320]
 -maxpd,--maxPastDate <arg>                                   The max date in the past for the memberSince field of a given user [defaults to 7300]
 -maxpeeps,--maxPotentialPeopleToCreate <arg>                 The max number of people to create [defaults to 5001]
 -maxppp,--maxPerProductPrice <arg>                           The max price per product [defaults to 1000.00]
 -maxprods,--maxPotentialProductsToCreate <arg>               The max number of products to create [defaults to 1001]
 -maxpurt,--maxRandomTimeFromAddToCartToPurchased <arg>       The max random time from add-to-cart to purchased for a given user and product[defaults to 4320]
 -maxship,--maxShipRate <arg>                                 The max ship rate for an order [defaults to 0.15]
 -maxtax,--maxTaxRate <arg>                                   The max tax rate for an order [defaults to 0.125]
 -maxv,--maxPotentialViews <arg>                              The max number of potential views a person can have against a product [defaults to 50]
 -minpeeps,--minPotentialPeopleToCreate <arg>                 The min number of people to create [defaults to 5000]
 -minppp,--minPerProductPrice <arg>                           The min price per product [defaults to 0.99]
 -minprods,--minPotentialProductsToCreate <arg>               The min number of products to create [defaults to 1000]
 -minship,--minShipRate <arg>                                 The min ship rate for an order [defaults to 0.0]
 -mintax,--minTaxRate <arg>                                   The min tax rate for an order [defaults to 0.0]
 -minv,--minPotentialViews <arg>                              The min number of potential views a person can have against a product [defaults to 0]
 -ncbs,--nodeCreationBatchSize <arg>                          The batch size to use for writes when creating people and products [defaults to 500]
 -peratc,--percentageOfViewsToAddToCart <arg>                 The percentage of views to turn into add-to-cart events for a given person and product [defaults to 25]
 -perpur,--percentageOfAddToCartToPurchase <arg>              The percentage of add-to-cart into purchase events for a given person and product [defaults to 90]
 -rh,--redisHost <arg>                                        The host of the Redis instance with the RedisGraph module installed to use for graph creation. [defaults to localhost]
 -rp,--redisPort <arg>                                        The port of the Redis instance with the RedisGraph module installed to use for graph creation. [defaults to 6379]
 -tc,--threadCount <arg>                                      The thread count to use [defaults to 6]
 ```

Internally, the script runs through a number of steps to make the graph generation very, very quick compared to the last iteration.

1. The CLI builder initializes to setup and provide input verification -- this takes a good number lines of code, unfortunately, but makes things pretty from a user / usability perspective
2. Objects are declared that are used for generation and persistence; Person, Product, Order, and Event (a class used to represent edges in the system)
3. Jedis opens a pool to redis and Redis Graph initializes itself from that pool.
4. The script spins up a number of threads for creation of the foundational nodes (a) people and (b) products and waits for generation of the these objects from the faker
5. A progressbar is initialized to show visual status of ^^^
6. The faker begins generating (a) people and (b) products and the threads begin using `CREATE` statements against the graph
7. A gate, a latch, is reached while we wait for all ^^^ processing to finish..........
8. We then iterate over the people who've been placed in another queue for processing across another "pool" (not a pool, technically) of threads
9. In each thread is where the majority of the random generation of edges resides, which is why we need to wait for those foundational nodes to be generated first. (This could have been done with `MERGE` operation, but...) Most of the RedisGraph operators in this section are "CREATE" and bounded creates (my words), specifically, "MATCH ... CREATE".
10. A progressbar is initialized to show visual status of ^^^

The output of the progressbar and app itself looks like:

```
./generateCommerceGraph
(:person) and (:product) 100% │██████████████████████████████████████████████████████│ 6000/6000 (0:00:02 / 0:00:00)
(:order), [:view], [:addtocart], [:transact], and [:contain]  36% │██████▌           │ 1825/5000 (0:00:08 / 0:00:15)
```

Early next week I'll go over basic queries and cover the crude pass at product recommendations and the load test tooling created for those efforts.

Cheers!
