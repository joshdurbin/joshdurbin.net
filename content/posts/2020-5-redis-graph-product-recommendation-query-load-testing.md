+++
title = "Product Recommendations in RedisGraph, Part 2: Query Load Testing"
date = "2020-05-11"
tags = ["graph", "redis", "recommendations", "data", "opencypher", "redisgraph"]
+++

This post is a continuation of the series on leveraging RedisGraph for product recommendations.

```groovy
def progressBarUpdateInterval = 200

// defaults must be strings for CliBuilder
def defaultGraphDB = 'prodrec'
def defaultThreadCount = "${new SystemInfo().hardware.processor.physicalProcessorCount}"
def defaultRedisHost = 'localhost'
def defaultRedisPort = '6379'
def defaultTopNumberOfPurchasers = '1000'

def cli = new CliBuilder(header: 'Concurrent RedisGraph Query Runner', usage:'productRecommendationQueryRunner -e <comma delimited environments> <other args>', width: -1)
cli.db(longOpt: 'database', "The RedisGraph database to use for the query [defaults to ${defaultGraphDB}]", args: 1, defaultValue: defaultGraphDB)
cli.tc(longOpt: 'threadCount', "The thread count to use [defaults to ${defaultThreadCount}]", args: 1, defaultValue: defaultThreadCount)
cli.h(longOpt: 'help', 'Usage Information')
cli.rh(longOpt: 'redisHost', "The host of the Redis instance with the RedisGraph module installed to use for graph creation. [defaults to ${defaultRedisHost}]", args: 1, defaultValue: defaultRedisHost)
cli.rp(longOpt: 'redisPort', "The port of the Redis instance with the RedisGraph module installed to use for graph creation. [defaults to ${defaultRedisPort}]", args: 1, defaultValue: defaultRedisPort)
cli.tp(longOpt: 'topPurchasers', "The number of top purchasers to query for [defaults to ${defaultTopNumberOfPurchasers}]", args: 1, defaultValue: defaultTopNumberOfPurchasers)
cli.l(longOpt: 'limitResults', "The default results limit.", args: 1)

// parse and validate options
def cliOptions = cli.parse(args)

def printErr = System.err.&println

if (!cliOptions) {
  cli.usage()
  System.exit(-1)
}

if (cliOptions.help) {
  cli.usage()
  System.exit(0)
}

def db = cliOptions.db

// setup jedis and graph
def threadCount = cliOptions.tc as Integer
def config = new GenericObjectPoolConfig()
config.setMaxTotal(threadCount)
def jedisPool = new JedisPool(config, cliOptions.redisHost, cliOptions.redisPort as Integer)
def redisGraph = new RedisGraph(jedisPool)

// query to get the top 1,000 person ids with the most orders
def personIdsToOrderCounts = redisGraph.query(db, "match (p:person)-[:transact]->(o:order) return p.id, count(o) as orders order by orders desc limit ${cliOptions.topPurchasers}")
def personIds = personIdsToOrderCounts.collect {
  it.values.first() as Integer
}

// queue is used to track results coming back from the worker threads
def resultsQueue = new ConcurrentLinkedQueue()

// latch is used to denote to the progress bar when things should be complete
def latch = new CountDownLatch(threadCount)

@Canonical class RecommendedProducts {

  def personId
  def products
  def queryTime
}

@Canonical class Product {

  def id
  def name
}

// this is used to generate a reaslistic max value for the progressbar
def expectedNumberOfQueueEntries = personIds.size()
def queueOfPeopleToQueryForProductRecommendations = new ConcurrentLinkedQueue(personIds.shuffled())

// thread generation
threadCount.times {

  Thread.start {

    while (!queueOfPeopleToQueryForProductRecommendations.isEmpty()) {

      def personId = queueOfPeopleToQueryForProductRecommendations.poll()

      try {

        // ask the graph for the product ids and names found in the placed orders of other users who share product purchase histories with a given user, person id
        def query = """match (p:person { id: ${personId} })-[:transact]->(:order)-[:contain]->(prod:product)
                       match (prod)<-[:contain]-(:order)-[:contain]->(rec_prod:product)
                       where not (p)-[:transact]->(:order)-[:contain]->(rec_prod)
                       return distinct rec_prod.id, rec_prod.name"""

        if (cliOptions.limitResults) {
          query += " limit ${cliOptions.limitResults}"  
        }

        def recommendedProductsQuery = redisGraph.query(db, query)
        def recommendedProducts = recommendedProductsQuery.results.collect {
          new Product(it.values().first(), it.values().last())
        }

        // get the query details and offer them to the queue for reporting
        def queryTime = recommendedProductsQuery.statistics.getStringValue(Label.QUERY_INTERNAL_EXECUTION_TIME).takeBefore(' ')
        resultsQueue.offer(new RecommendedProducts(personId, recommendedProducts, queryTime))

      } catch (Exception e ) {
        printErr("error processing ${personId}")
      }
    }

    latch.countDown()
  }
}

// counts don't require individual insert tracking, so we use the memory efficient summary stats
def counts = new SummaryStatistics()

// times does require it, so we can get the p50, p95, p99
def times = new DescriptiveStatistics()

new ProgressBar('Progress', expectedNumberOfQueueEntries, progressBarUpdateInterval).withCloseable { progressBar ->

  while (latch.count > 0L) {

    def recommendedProducts = resultsQueue.poll()

    if (recommendedProducts) {

      counts.addValue(recommendedProducts.products.size() as Integer)
      times.addValue(recommendedProducts.queryTime as Double)
      progressBar.step()
    }
  }
}

println "Found a min number of recommended products of ${counts.min as Integer}, avg of ${counts.mean as Integer}, and a max of ${counts.max as Integer} for ${counts.n} with a query performance p50 ${(times.getPercentile(50.0) as String).takeBefore('.')}ms, p95 ${(times.getPercentile(95.0) as String).takeBefore('.')}ms, p99 ${(times.getPercentile(99.0) as String).takeBefore('.')}ms"
```