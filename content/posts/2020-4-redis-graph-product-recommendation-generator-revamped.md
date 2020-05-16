+++
title = "Product Recommendations in RedisGraph, Part 1.1: Better data loading"
date = "2020-04-18"
tags = ["graph", "redis", "recommendations", "data", "opencypher", "redisgraph"]
+++

This post is part of a [series]({{< ref "/tags/recommendations" >}}) on leveraging RedisGraph for product recommendations.

This post is to discuss changes to the [graph generator](https://github.com/joshdurbin/redis-graph-commerce-poc/blob/master/generateCommerceGraph). I've put
a decent chunk of effort into tweaking the performance, configurability, etc.. of the generator and you might ask yourself why? Why spend time
generating fake data? Well, a few things:

1. Earlier this month RedisGraph [released version 2.0](https://redislabs.com/blog/introducing-redisgraph-2-0/) which includes a ton of really great
updates and performance/stability wins. That said, though RedisGraph is continually improving its resiliency and stability, it does occasionally
bonk and when it does, so does your data. So, it can be helpful to understand the most efficient way of getting data into RedisGraph,
performance bottlenecks, etc...
2. Altering the "shape" and "depth" of data drastically changes performance impact of queries, DB snapshotting, sync and startup performance, etc...
3. Though the data is fake, it is realistic in that:
  - people (customers) have join dates and orders are always after
  - add to cart events are a subset of product view events
  - purchased events are randomized across orders for a customer and they only purchase products from the pool of products added to their cart
  - pricing is realistic, cost, shipping, tax, etc...

### Installation

This script requires the installation of Java 11 and Groovy. Dependencies are automatically pulled from Maven repositories based on the Grape'd annotations.

- Java (11 OpenJDK)
- Groovy 3
- RedisGraph

### Obtaining and Usage

Pull down the `generateCommerceGraph` script itself:

```shell script
wget https://raw.githubusercontent.com/joshdurbin/redis-graph-commerce-poc/master/generateCommerceGraph
chmod u+x generateCommerceGraph
```

At this point, with RedisGraph available at `localhost:6379`, you can just run `./generateCommerceGraph` to produce a graph. The graph
generation will be shown in two steps (1) to create the `person` and `product` nodes and (2) to create the occasional `order` node and all
the connecting edges.  

```shell script
./generateCommerceGraph
(:person) and (:product) 100% │███████████████████████████████████████████████████████████│ 6000/6000 (0:00:02 / 0:00:00) 
(:order), [:view], [:addtocart], [:transact], and [:contain] 100% │███████████████████████│ 5000/5000 (0:00:20 / 0:00:00) 
Finished creating 5000 'person', 1000 'product', 11662 'order', 118721 'view' edges, 30835 'addtocart' edges, 11662 'transact' edges, and 28007 'contain' products edges.
```

There are a number of interesting knobs in the graph generation you can tweak; knobs in probability and lower/upper bounds of things, such as:

- The (min/max) number of people created
- The (min/max) number of products created
- The (min/max) views for a person and product
- The percentage of views for a person which “upgrade” to add to cart events
- The percentage of add to cart events which “upgrade” to purchased events
- The max random time from viewed to add to cart
- The max random time from add to cart to purchased
- The (min/max) product price, tax rate for orders, ship rate for orders
- Other utility things, such as the redis endpoint details, batch sizes for commits, thread count, etc…

An exhaustive list of options can be output requesting the help menu `./generateCommerceGraph --help`:

```shell script
./generateCommerceGraph --help
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

### Peering Behind the Curtain

Assuming that you're running RedisGraph in Docker as described in the last post in the series, you can easily connect a Redis client to the instance
and run `monitor` to see what commands are hitting the RedisGraph instance as it creates nodes and relationships.

```shell script
➜  ~ docker run -it --network host --rm redis redis-cli -h 127.0.0.1
127.0.0.1:6379> monitor
OK
1589518567.962973 [0 172.17.0.1:39190] "graph.QUERY" "prodrec" "create index on :person(id)" "--COMPACT"
1589518567.980113 [0 172.17.0.1:39190] "graph.QUERY" "prodrec" "create index on :product(id)" "--COMPACT"
1589518567.984275 [0 172.17.0.1:39190] "graph.QUERY" "prodrec" "create index on :order(id)" "--COMPACT"
1589518568.577378 [0 172.17.0.1:39190] "graph.QUERY" "prodrec" "CREATE (:person {id: 5, name:\"Thomas Murazik\",age:51,address:\"77917 Laurena Manor, Durganstad, NC 21122\",memberSince:\"2001-12-22T21:56:08.558384\"})" "--COMPACT"
1589518568.587073 [0 172.17.0.1:39204] "graph.QUERY" "prodrec" "CREATE (:person {id: 4, name:\"Reid Pouros\",age:74,address:\"Suite 726 867 Micheal Walks, North Jere, HI 37697-5232\",memberSince:\"2015-01-11T21:56:08.556150\"})" "--COMPACT"
1589518568.588166 [0 172.17.0.1:39206] "graph.QUERY" "prodrec" "CREATE (:person {id: 2, name:\"Amos Pfeffer\",age:71,address:\"Apt. 009 7298 Murazik Inlet, New Bea, AK 80809\",memberSince:\"2015-02-21T21:56:08.552424\"})" "--COMPACT"
1589518568.589285 [0 172.17.0.1:39200] "graph.QUERY" "prodrec" "CREATE (:person {id: 3, name:\"Warren Lebsack\",age:18,address:\"43074 McGlynn Manors, Lake Juliustown, NC 81978\",memberSince:\"2008-11-09T21:56:08.554088\"})" "--COMPACT"
1589518568.590025 [0 172.17.0.1:39202] "graph.QUERY" "prodrec" "CREATE (:person {id: 1, name:\"Nakia Medhurst\",age:74,address:\"Suite 751 4111 Goldner Plaza, Bergstrommouth, CT 82975-2395\",memberSince:\"2010-12-17T21:56:08.550949\"})" "--COMPACT"
1589518568.590740 [0 172.17.0.1:39190] "graph.QUERY" "prodrec" "CREATE (:person {id: 6, name:\"Lia Cormier\",age:36,address:\"7354 Rowe Burgs, East Garrettville, KY 36935-1834\",memberSince:\"2014-02-17T21:56:08.561472\"}),(:person {id: 7, name:\"Kami Barrows\",age:73,address:\"23466 Farrell Lights, New Rosechester, AL 16156-1743\",memberSince:\"2005-01-24T21:56:08.563581\"}),(:person {id: 8, name:\"Miquel Hoeger\",age:48,address:\"80943 Kayleen Garden, East Gregorio, WA 35273-0323\",memberSince:\"2015-10-17T21:56:08.565609\"}),(:person {id: 9, name:\"Sonny Bechtelar\",age:71,address:\"91795 Randy Course, Jeremyton, ME 73938\",memberSince:\"2002-08-27T21:56:08.568028\"})" "--COMPACT"
1589518568.591365 [0 172.17.0.1:39210] "graph.QUERY" "prodrec" "CREATE (:person {id: 0, name:\"Dominique Boyle\",age:80,address:\"Apt. 903 0812 Williamson Garden, Lake Justineberg, MN 66033-9020\",memberSince:\"2006-03-29T21:56:08.343678\"})" "--COMPACT"
1589518568.598528 [0 172.17.0.1:39204] "graph.QUERY" "prodrec" "CREATE (:person {id: 10, name:\"Gilberto McKenzie\",age:94,address:\"1170 Candida Ways, Sengerstad, IL 14026-2275\",memberSince:\"2019-03-04T21:56:08.572586\"}),(:person {id: 11, name:\"Fredric Davis\",age:71,address:\"Apt. 167 19608 Reichel Squares, Lake Courtneychester, VT 29560\",memberSince:\"2011-07-23T21:56:08.576044\"}),(:person {id: 12, name:\"Aaron DuBuque\",age:77,address:\"57721 Evangelina Prairie, South Hueyfurt, MA 50369\",memberSince:\"2006-07-13T21:56:08.579893\"}),(:person {id: 13, name:\"Jaymie Hackett\",age:78,address:\"Suite 086 6574 Price Branch, North Duane, NE 13841-6676\",memberSince:\"2020-01-10T21:56:08.581632\"})" "--COMPACT"
1589518568.599720 [0 172.17.0.1:39206] "graph.QUERY" "prodrec" "CREATE (:person {id: 14, name:\"Bong Harber\",age:16,address:\"Suite 091 4974 Milo Stream, Port Aldobury, CA 13766-7737\",memberSince:\"2016-08-10T21:56:08.583292\"})" "--COMPACT"
1589518568.607787 [0 172.17.0.1:39190] "graph.QUERY" "prodrec" "CREATE (:person {id: 15, name:\"Shoshana Shanahan\",age:49,address:\"Suite 244 6170 Bryanna Parks, South Wilfredo, OH 14122-4197\",memberSince:\"2015-08-08T21:56:08.589217\"})" "--COMPACT"
1589518568.612820 [0 172.17.0.1:39206] "graph.QUERY" "prodrec" "CREATE (:person {id: 16, name:\"Jonathon Ward\",age:66,address:\"Suite 517 605 Arlie Greens, Wiegandborough, TX 71956-0802\",memberSince:\"2019-03-03T21:56:08.593040\"})" "--COMPACT"
1589518568.613971 [0 172.17.0.1:39204] "graph.QUERY" "prodrec" "CREATE (:person {id: 17, name:\"Dino Fisher\",age:32,address:\"Apt. 627 97371 Ashly Terrace, North Darwinside, WI 92060-5607\",memberSince:\"2010-11-27T21:56:08.596312\"})" "--COMPACT"
1589518568.615046 [0 172.17.0.1:39190] "graph.QUERY" "prodrec" "CREATE (:person {id: 18, name:\"Lashandra Koch\",age:54,address:\"Apt. 716 750 Bartoletti Springs, South Delta, LA 87610-1813\",memberSince:\"2003-07-20T21:56:08.597432\"})" "--COMPACT"
1589518568.616852 [0 172.17.0.1:39210] "graph.QUERY" "prodrec" "CREATE (:person {id: 19, name:\"King Yost\",age:34,address:\"950 Robel Meadows, Reichelburgh, NM 86969\",memberSince:\"2015-05-04T21:56:08.600070\"})" "--COMPACT"
1589518568.621137 [0 172.17.0.1:39202] "graph.QUERY" "prodrec" "CREATE (:person {id: 20, name:\"Shanta Bogisich\",age:68,address:\"Suite 000 6685 Sporer Circle, West Robert, MT 48202-9460\",memberSince:\"2015-08-24T21:56:08.603044\"})" "--COMPACT"
1589518568.623978 [0 172.17.0.1:39206] "graph.QUERY" "prodrec" "CREATE (:person {id: 21, name:\"Daniel Abshire\",age:18,address:\"0790 Koss Neck, New Dedeside, OK 23288-6408\",memberSince:\"2019-02-27T21:56:08.606614\"})" "--COMPACT"
1589518568.626905 [0 172.17.0.1:39210] "graph.QUERY" "prodrec" "CREATE (:person {id: 22, name:\"Clementina Kunze\",age:97,address:\"8772 Johnson Branch, New Estelahaven, UT 48620\",memberSince:\"2007-12-08T21:56:08.608716\"})" "--COMPACT"
```

### Don't forget to Save

Just like anything else in Redis, it's easy enough to point-in-time snapshot data to disk via `save`:

```shell script
➜  ~ docker run -it --network host --rm redis redis-cli -h 127.0.0.1
127.0.0.1:6379> save
OK
(0.68s)
```

...and, if you're running RedisGraph in a docker image with a volume specified, the next time you start your generated graph data will be there.

```shell script
➜  ~ docker run -p 6379:6379 -it --rm -v redis-data:/data redislabs/redisgraph
1:C 15 May 2020 04:59:39.532 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
1:C 15 May 2020 04:59:39.533 # Redis version=5.0.9, bits=64, commit=00000000, modified=0, pid=1, just started
1:C 15 May 2020 04:59:39.533 # Configuration loaded
                _._                                                  
           _.-``__ ''-._                                             
      _.-``    `.  `_.  ''-._           Redis 5.0.9 (00000000/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._                                   
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
 |    `-._   `._    /     _.-'    |     PID: 1
  `-._    `-._  `-./  _.-'    _.-'                                   
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |           http://redis.io        
  `-._    `-._`-.__.-'_.-'    _.-'                                   
 |`-._`-._    `-.__.-'    _.-'_.-'|                                  
 |    `-._`-._        _.-'_.-'    |                                  
  `-._    `-._`-.__.-'_.-'    _.-'                                   
      `-._    `-.__.-'    _.-'                                       
          `-._        _.-'                                           
              `-.__.-'                                               

1:M 15 May 2020 04:59:39.536 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
1:M 15 May 2020 04:59:39.537 # Server initialized
1:M 15 May 2020 04:59:39.537 # WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Redis. To fix this issue run the command 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' as root, and add it to your /etc/rc.local in order to retain the setting after a reboot. Redis must be restarted after THP is disabled.
1:M 15 May 2020 04:59:39.546 * <graph> Thread pool created, using 6 threads.
1:M 15 May 2020 04:59:39.547 * Module 'graph' loaded from /usr/lib/redis/modules/redisgraph.so
1:M 15 May 2020 04:59:45.442 * DB loaded from disk: 5.894 seconds
1:M 15 May 2020 04:59:45.442 * Ready to accept connections
```

### Next up

In a few weeks I'll have a post on querying and making sense of the data. Stay tuned!