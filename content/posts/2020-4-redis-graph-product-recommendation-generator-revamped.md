+++
title = "Product Recommendations in RedisGraph, Part 1.1: Better data loading"
date = "2020-04-18"
tags = ["graph", "redis", "recommendations", "data", "opencypher", "redisgraph"]
+++

This post is part of a [series]({{< ref "/tags/recommendations" >}}) on leveraging RedisGraph for product recommendations.

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

```shell script
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

Internally, the script runs through a number of steps to make the graph generation very, very quick compared to the last iteration. The dependency directives (Grapes) and imports are omitted.

{{<mermaid align="left">}}
graph LR
   setup --> a["more complex"] --> d & c--> d
{{< /mermaid >}}

1. CLI builder initializes to setup and provide input verification -- this takes a good number lines of code, unfortunately, but makes things pretty from a user / usability perspective

```groovy
def progressBarUpdateInterval = 200

// defaults must be strings for CliBuilder
def defaultDB = 'prodrec'
def defaultThreadCount = "${new SystemInfo().hardware.processor.physicalProcessorCount}"
def defaultMaxPotentialViews = '50'
def defaultMinPotentialViews = '0'
def defaultPercentageOfViewsToAddToCart = '25'
def defaultPercentageOfAddToCartToPurchase = '90'
def defaultMaxRandomTimeFromViewToAddToCartInMinutes = '4320'
def defaultMaxRandomTimeFromAddToCartToPurchased = '4320'
def defaultMaxPastDate = "${365 * 20}"
def defaultMaxPotentialPeopleToCreate = '5001'
def defaultMinPotentialPeopleToCreate = '5000'
def defaultMaxPotentialProductsToCreate = '1001'
def defaultMinPotentialProductsToCreate = '1000'
def defaultNodeCreationBatchSize = '500'
def defaultMaxTaxRate = '0.125'
def defaultMinTaxRate = '0.0'
def defaultMaxShipRate = '0.15'
def defaultMinShipRate = '0.0'
def defaultMaxPerProductPrice = '1000.00'
def defaultMinPerProductPrice = '0.99'
def defaultRedisHost = 'localhost'
def defaultRedisPort = '6379'

def cli = new CliBuilder(header: 'Commerce Graph Generator', usage:'generateCommerceGraph', width: -1)
cli.maxv(longOpt: 'maxPotentialViews', "The max number of potential views a person can have against a product [defaults to ${defaultMaxPotentialViews}]", args: 1, defaultValue: defaultMaxPotentialViews)
cli.minv(longOpt: 'minPotentialViews', "The min number of potential views a person can have against a product [defaults to ${defaultMinPotentialViews}]", args: 1, defaultValue: defaultMinPotentialViews)
cli.peratc(longOpt: 'percentageOfViewsToAddToCart', "The percentage of views to turn into add-to-cart events for a given person and product [defaults to ${defaultPercentageOfViewsToAddToCart}]", args: 1, defaultValue: defaultPercentageOfViewsToAddToCart)
cli.perpur(longOpt: 'percentageOfAddToCartToPurchase', "The percentage of add-to-cart into purchase events for a given person and product [defaults to ${defaultPercentageOfAddToCartToPurchase}]", args: 1, defaultValue: defaultPercentageOfAddToCartToPurchase)
cli.maxatct(longOpt: 'maxRandomTimeFromViewToAddToCartInMinutes', "The max random time from view to add-to-cart for a given user and product [defaults to ${defaultMaxRandomTimeFromViewToAddToCartInMinutes}]", args: 1, defaultValue: defaultMaxRandomTimeFromViewToAddToCartInMinutes)
cli.maxpurt(longOpt: 'maxRandomTimeFromAddToCartToPurchased', "The max random time from add-to-cart to purchased for a given user and product[defaults to ${defaultMaxRandomTimeFromAddToCartToPurchased}]", args: 1, defaultValue: defaultMaxRandomTimeFromAddToCartToPurchased)
cli.maxpd(longOpt: 'maxPastDate', "The max date in the past for the memberSince field of a given user [defaults to ${defaultMaxPastDate}]", args: 1, defaultValue: defaultMaxPastDate)
cli.maxpeeps(longOpt: 'maxPotentialPeopleToCreate', "The max number of people to create [defaults to ${defaultMaxPotentialPeopleToCreate}]", args: 1, defaultValue: defaultMaxPotentialPeopleToCreate)
cli.minpeeps(longOpt: 'minPotentialPeopleToCreate', "The min number of people to create [defaults to ${defaultMinPotentialPeopleToCreate}]", args: 1, defaultValue: defaultMinPotentialPeopleToCreate)
cli.maxprods(longOpt: 'maxPotentialProductsToCreate', "The max number of products to create [defaults to ${defaultMaxPotentialProductsToCreate}]", args: 1, defaultValue: defaultMaxPotentialProductsToCreate)
cli.minprods(longOpt: 'minPotentialProductsToCreate', "The min number of products to create [defaults to ${defaultMinPotentialProductsToCreate}]", args: 1, defaultValue: defaultMinPotentialProductsToCreate)
cli.ncbs(longOpt: 'nodeCreationBatchSize', "The batch size to use for writes when creating people and products [defaults to ${defaultNodeCreationBatchSize}]", args: 1, defaultValue: defaultNodeCreationBatchSize)
cli.maxtax(longOpt: 'maxTaxRate', "The max tax rate for an order [defaults to ${defaultMaxTaxRate}]", args: 1, defaultValue: defaultMaxTaxRate)
cli.mintax(longOpt: 'minTaxRate', "The min tax rate for an order [defaults to ${defaultMinTaxRate}]", args: 1, defaultValue: defaultMinTaxRate)
cli.maxship(longOpt: 'maxShipRate', "The max ship rate for an order [defaults to ${defaultMaxShipRate}]", args: 1, defaultValue: defaultMaxShipRate)
cli.minship(longOpt: 'minShipRate', "The min ship rate for an order [defaults to ${defaultMinShipRate}]", args: 1, defaultValue: defaultMinShipRate)
cli.maxppp(longOpt: 'maxPerProductPrice', "The max price per product [defaults to ${defaultMaxPerProductPrice}]", args: 1, defaultValue: defaultMaxPerProductPrice)
cli.minppp(longOpt: 'minPerProductPrice', "The min price per product [defaults to ${defaultMinPerProductPrice}]", args: 1, defaultValue: defaultMinPerProductPrice)
cli.db(longOpt: 'database', "The RedisGraph database to use for our queries, data generation [defaults to ${defaultDB}]", args: 1, defaultValue: defaultDB)
cli.tc(longOpt: 'threadCount', "The thread count to use [defaults to ${defaultThreadCount}]", args: 1, defaultValue: defaultThreadCount)
cli.rh(longOpt: 'redisHost', "The host of the Redis instance with the RedisGraph module installed to use for graph creation. [defaults to ${defaultRedisHost}]", args: 1, defaultValue: defaultRedisHost)
cli.rp(longOpt: 'redisPort', "The port of the Redis instance with the RedisGraph module installed to use for graph creation. [defaults to ${defaultRedisPort}]", args: 1, defaultValue: defaultRedisPort)
cli.h(longOpt: 'help', 'Usage Information')

// parse and validate options
def cliOptions = cli.parse(args)

if (!cliOptions) {
  cli.usage()
  System.exit(-1)
}

if (cliOptions.help) {
  cli.usage()
  System.exit(0)
}

def printErr = System.err.&println

// closure for validating min/max are discrete
def validateParams = { variable, min, max ->
  if (min == max) {
    printErr("Min and max values must be discrete. Current for ${variable} are ${min} and ${max}")
    cli.usage()
    System.exit(-1)
  }
}

def maxPotentialViews = cliOptions.maxPotentialViews as Integer
def minPotentialViews = cliOptions.minPotentialViews as Integer
validateParams('Potential Views', minPotentialViews, maxPotentialViews)

def percentageOfViewsToAddToCart = cliOptions.percentageOfViewsToAddToCart as Integer
def percentageOfAddToCartToPurchase = cliOptions.percentageOfAddToCartToPurchase as Integer
def maxRandomTimeFromViewToAddToCartInMinutes = cliOptions.maxRandomTimeFromViewToAddToCartInMinutes as Integer
def maxRandomTimeFromAddToCartToPurchased = cliOptions.maxRandomTimeFromAddToCartToPurchased as Integer
def maxPastDate = cliOptions.maxPastDate as Integer
def maxPotentialPeopleToCreate = cliOptions.maxPotentialPeopleToCreate as Integer
def minPotentialPeopleToCreate = cliOptions.minPotentialPeopleToCreate as Integer
validateParams('Created People', minPotentialPeopleToCreate, maxPotentialPeopleToCreate)

def maxPotentialProductsToCreate = cliOptions.maxPotentialProductsToCreate as Integer
def minPotentialProductsToCreate = cliOptions.minPotentialProductsToCreate as Integer
validateParams('Created Products', minPotentialProductsToCreate, maxPotentialProductsToCreate)

def nodeCreationBatchSize = cliOptions.nodeCreationBatchSize as Integer
def maxTaxRate = cliOptions.maxTaxRate as Double
def minTaxRate = cliOptions.minTaxRate as Double
validateParams('Tax Rate', minTaxRate, maxTaxRate)

def maxShipRate = cliOptions.maxShipRate as Double
def minShipRate = cliOptions.minShipRate as Double
validateParams('Ship Rate', minShipRate, maxShipRate)

def maxPerProductPrice = cliOptions.maxPerProductPrice as Double
def minPerProductPrice = cliOptions.minPerProductPrice as Double
validateParams('Product Price', minPerProductPrice, maxPerProductPrice)
```

2. Objects are declared that are used for generation and persistence; Person, Product, Order, and Event (a class used to represent edges in the system)

```groovy
// this is used for consistent label/type keys through the generation script
@Singleton class GraphKeys {
  def personNodeType = 'person'
  def productNodeType = 'product'
  def orderNodeType = 'order'
  def viewEdgeType = 'view'
  def addToCartEdgeType = 'addtocart'
  def transactEdgeType = 'transact'
  def containEdgeType = 'contain'
}

def globalOrderIdCounter = new AtomicInteger(0)

def mathContext = new MathContext(2, RoundingMode.HALF_UP)

@Canonical class Person {

  def id
  def name
  def address
  def age
  def memberSince

  def toCypherCreate() {
    "(:${GraphKeys.instance.personNodeType} {id: ${id}, name:\"${name}\",age:${age},address:\"${address}\",memberSince:\"${memberSince}\"})"
  }
}

@Canonical class Product {

  def id
  def name
  def manufacturer
  def msrp

  def toCypherCreate() {
    "(:${GraphKeys.instance.productNodeType} {id: ${id},name:\"${name}\",manufacturer:\"${manufacturer}\",msrp:'${msrp}'})"
  }
}

@Canonical class Order {

  def id
  def subTotal
  def tax
  def shipping
  def total

  def toCypherCreate() {
    "(:${GraphKeys.instance.orderNodeType} {id: ${id},subTotal:${subTotal},tax:${tax},shipping:${shipping},total:${total}})"
  }
}

@Canonical class Event {

  def product
  def type
  def time
}
```

3. Jedis opens a pool to redis and Redis Graph initializes itself from that pool.

```groovy
def db = cliOptions.database

// setup jedis and graph
def threadCount = cliOptions.threadCount as Integer
def config = new GenericObjectPoolConfig()
config.setMaxTotal(threadCount)
def jedisPool = new JedisPool(config, cliOptions.redisHost, cliOptions.redisPort as Integer)
def graph = new RedisGraph(jedisPool)

// index creation
graph.query(db, "create index on :${GraphKeys.instance.personNodeType}(id)")
graph.query(db, "create index on :${GraphKeys.instance.productNodeType}(id)")
graph.query(db, "create index on :${GraphKeys.instance.orderNodeType}(id)")
```

4. The script spins up a number of threads for creation of the foundational nodes (a) people and (b) products and waits for generation of the these objects from the faker

```groovy
def now = LocalDateTime.now()

// people and products have to be first created so we can query on them in a second round for orders and edges
def personProductQueue = new ConcurrentLinkedQueue()
def personProductLatch = new CountDownLatch(threadCount)

def queue = new ConcurrentLinkedQueue()
def peopleToCreate = random.nextInt(minPotentialPeopleToCreate, maxPotentialPeopleToCreate)
def productsToCreate = random.nextInt(minPotentialProductsToCreate, maxPotentialProductsToCreate)
def generationDone = new AtomicBoolean(false)
def createCount = new AtomicInteger(0)

// spin up threads to do the work
threadCount.times {

  Thread.start {

    while (personProductLatch.count > 0L) {

      def query = personProductQueue.poll()
      def batchCounter = 0
      def batchedInserts = []

      // as redis slows with load we become more efficient at batching nodes created against the graph
      while (query && batchCounter < nodeCreationBatchSize) {
        batchedInserts << query
        batchCounter++
        query = personProductQueue.poll()
      }

      if (batchedInserts) {
        graph.query(db, "CREATE ${batchedInserts.collect { record -> record.toCypherCreate()}.join(',')}")
        createCount.addAndGet(batchedInserts.size())
      }

      if (!query && generationDone.get()) {
        personProductLatch.countDown()
      }
    }
  }
}
```

5. A progressbar is initialized to show visual status of ^^^

```groovy
// thread used to track progress
Thread.start {

  def targetProgress = peopleToCreate + productsToCreate

  new ProgressBar("(:${GraphKeys.instance.personNodeType}) and (:${GraphKeys.instance.productNodeType})", targetProgress, progressBarUpdateInterval).withCloseable { progressBar ->

    while (personProductLatch.count > 0L) {
      progressBar.stepTo(createCount.get())
    }

    progressBar.stepTo(targetProgress)
  }
}
```

6. The faker begins generating (a) people and (b) products and the threads begin using `CREATE` statements against the graph

```groovy

// create the latch and second queue for processing
def peopleForOrderAndEdgesQueue = new ConcurrentLinkedQueue()
def orderAndEdgesLatch = new CountDownLatch(threadCount)

// generate the people and products, here is where the prior threads begin doing their work
peopleToCreate.times { num ->
  def address = mainThreadFaker.address()
  def person = new Person(id: num, name: "${address.firstName()} ${address.lastName()}", address: address.fullAddress(), age: random.nextInt(10, 100), memberSince: LocalDateTime.now().minusDays(random.nextInt(1, maxPastDate) as Long))
  personProductQueue.offer(person)
  peopleForOrderAndEdgesQueue.offer(person)
}

def products = new ArrayList(productsToCreate)
productsToCreate.times { num ->
  def product = new Product(id: num, name: mainThreadFaker.commerce().productName(), manufacturer: mainThreadFaker.company().name(), msrp: mainThreadFaker.commerce().price(minPerProductPrice, maxPerProductPrice) as Double)
  products << product
  personProductQueue.offer(product)
}
```

7. A gate, a latch, is reached while we wait for all ^^^ processing to finish...

```groovy
// signal to prior threads that generation is complete
generationDone.set(true)

// waiting for all threads to complete before continuing to order node and edge creation
personProductLatch.await()
```

8. We then iterate over the people who've been placed in another queue for processing across another "pool" (not a pool, technically) of threads
9. In each thread is where the majority of the random generation of edges resides, which is why we need to wait for those foundational nodes to be generated first. (This could have been done with `MERGE` operation, but...) Most of the RedisGraph operators in this section are `CREATE` and bounded creates (my words), specifically, `MATCH ... CREATE`.

```groovy
threadCount.times {

  Thread.start { thread ->

    def threadRandom = new SplittableRandom()

    while (!peopleForOrderAndEdgesQueue.empty) {

      def person = peopleForOrderAndEdgesQueue.poll()

      def views = threadRandom.nextInt(minPotentialViews, maxPotentialViews)
      def viewedProducts = [] as Set
      def minutesFromMemberSinceToNow = person.memberSince.until(now, ChronoUnit.MINUTES)

      // generate random value up to max potential views and drop those into a unique set
      views.times {
        viewedProducts << products.get(threadRandom.nextInt(products.size()))
      }

      def viewEvents = viewedProducts.collect { product ->
        new Event(product: product, type: GraphKeys.instance.viewEdgeType, time: person.memberSince.plusMinutes(threadRandom.nextInt(1, minutesFromMemberSinceToNow as Integer) as Long))
      }

      if (viewEvents) {

        // this pulls a percentage of viewed products into add to cart events
        def addedToCartEvents = viewEvents.findAll {
          threadRandom.nextInt(100) <= percentageOfViewsToAddToCart
        }.collect { event ->
          new Event(product: event.product, type: GraphKeys.instance.addToCartEdgeType, time: event.time.plusMinutes(threadRandom.nextInt(1, maxRandomTimeFromViewToAddToCartInMinutes) as Long))
        }

        // purchasedEvents
        def purchasedEvents = addedToCartEvents.findAll {
          threadRandom.nextInt(100) <= percentageOfAddToCartToPurchase
        }

        def addedToCartEventEdges = ''

        if (addedToCartEvents) {

          def joinedAddToCartEdges = addedToCartEvents.collect { event ->
            "(p)-[:${event.type} {time: '${event.time}'}]->(prd${event.product.id})"
          }.join(', ')
          addedToCartEventEdges = ", ${joinedAddToCartEdges}"

          if (purchasedEvents) {

            purchasedEvents.collate(threadRandom.nextInt(0, purchasedEvents.size())).collect { subSetOfPurchasedEvents ->

              def productsInOrder = subSetOfPurchasedEvents.collect { event -> event.product }
              def oldestPurchasedEvent = subSetOfPurchasedEvents.max { event -> event.time }
              def subTotalOfProducts = productsInOrder.collect { product -> product.msrp }.sum()
              def taxAddition = new BigDecimal(subTotalOfProducts, mathContext).multiply(new BigDecimal(threadRandom.nextDouble(minTaxRate, maxTaxRate), mathContext))
              def shipAddition = new BigDecimal(subTotalOfProducts, mathContext).multiply(new BigDecimal(threadRandom.nextDouble(minShipRate, maxShipRate), mathContext))

              def order = new Order(id: globalOrderIdCounter.incrementAndGet(), subTotal: subTotalOfProducts, tax: taxAddition, shipping: shipAddition, total: subTotalOfProducts + taxAddition + shipAddition)

              graph.query(db, "CREATE ${order.toCypherCreate()}")

              // we have to match across all the products
              def productMatchDefinitionParams = productsInOrder.collect { product ->
                "(prd${product.id}:${GraphKeys.instance.productNodeType})"
              }.join(', ')

              def productMatchCriteria = productsInOrder.collect { product ->
                "prd${product.id}.id=${product.id}"
              }.join(' AND ')

              def productEdges = productsInOrder.collect { product ->
                "(o)-[:contain]->(prd${product.id})"
              }.join(', ')

              def query = "MATCH (p:${GraphKeys.instance.personNodeType}), (o:${GraphKeys.instance.orderNodeType}), ${productMatchDefinitionParams} WHERE p.id=${person.id} AND o.id=${order.id} AND ${productMatchCriteria} CREATE (p)-[:${GraphKeys.instance.transactEdgeType}]->(o), ${productEdges}"

              graph.query(db, query)
            }
          }
        }

        // we have to match across all the products
        def productMatchDefinitionParams = viewEvents.collect { event ->
          "(prd${event.product.id}:${GraphKeys.instance.productNodeType})"
        }.join(', ')

        def productMatchCriteria = viewEvents.collect { event ->
          "prd${event.product.id}.id=${event.product.id}"
        }.join(' AND ')

        def viewedEdges = viewEvents.collect { event ->
          "(p)-[:${event.type} {time: '${event.time}'}]->(prd${event.product.id})"
        }.join(', ')

        def query = "MATCH (p:${GraphKeys.instance.personNodeType}), ${productMatchDefinitionParams} WHERE p.id=${person.id} AND ${productMatchCriteria} CREATE ${viewedEdges}${addedToCartEventEdges}"

        graph.query(db, query)
      }
    }

    orderAndEdgesLatch.countDown()
  }
}
```

10. A progressbar is initialized to show visual status of ^^^

```groovy
// progress tracker for order node and edge creation, separate thread isn't needed as main thread reaches this as prior threads spin up
new ProgressBar("(:${GraphKeys.instance.orderNodeType}), [:${GraphKeys.instance.viewEdgeType}], [:${GraphKeys.instance.addToCartEdgeType}], [:${GraphKeys.instance.transactEdgeType}], and [:${GraphKeys.instance.containEdgeType}]", peopleToCreate, progressBarUpdateInterval).withCloseable { progressBar ->

  while (orderAndEdgesLatch.count > 0L) {
    progressBar.stepTo(peopleToCreate - peopleForOrderAndEdgesQueue.size())
  }
}
```

The output of the progressbar and app itself looks like:

```
./generateCommerceGraph
(:person) and (:product) 100% │██████████████████████████████████████████████████████│ 6000/6000 (0:00:02 / 0:00:00)
(:order), [:view], [:addtocart], [:transact], and [:contain]  36% │██████▌           │ 1825/5000 (0:00:08 / 0:00:15)
```

Early next week I'll go over basic queries and cover the crude pass at product recommendations and the load test tooling created for those efforts.

Cheers!
