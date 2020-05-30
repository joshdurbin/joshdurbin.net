+++
title = "An ephemeral, reactive geospatial REST API"
description = "POC work leveraging Rxjava, Spock, Ratpack, and a reactive r-tree for geospatial points of interest near point functionality."
date = "2017-12-27"
tags = ["poc", "ratpack", "rxjava", "reactive"]
aliases = ["/projects/2017-12-home-network-monitoring/"]
+++

I spent a portion of my holiday break (mainly the plane rides) tinkering with
[Ratpack](https://ratpack.io/) and a reactive R-tree library. This is the second time I've tinkered with
this combination of tech. Unlike the [first](https://github.com/joshdurbin/places), which was benchmarking NoSQL geo-spatial
engine candidates; Mongo, Redis, Rethink, R-tree, and ElasticSearch, this
exploration/POC was meant to test the performance of a REST-enabled in-memory, ephemeral,
geospatial "engine" without a backing data store, related drivers, and the network overhead.

The [project](https://github.com/joshdurbin/ephemeral-reactive-geospatial-rest-api) mainly leverages the following:

1. [Ratpack](https://github.com/ratpack/ratpack)
2. David Moten's [immutable R-tree](https://github.com/davidmoten/Rtree) implementation
3. [wrk](https://github.com/wg/wrk) as a load-testing tool with [custom Lua scripts](https://github.com/joshdurbin/ephemeral-reactive-geospatial-rest-api/tree/load_test_scripts)

The objective was to see how hard I could push a REST framework, the geospatial engine,
and the (presumably reactive) drivers in both point insertions and queries.

I chose to tinker with David Moten's R-tree (again) as its a natural fit
for Ratpack. R-trees are a shoe in for geospatial tests in general. R-trees
entirely back or compliment other trees (b and quad) in popular geospatial engines
such as CouchDB, Mongo, and SQL alternatives; PostGIS, MySQL, Oracle Spatial, etc...
Not only that, but both are written from the ground up to support reactive principles. Yay!

The API supports the following operations on its points and place (w/ a name, address, etc...):

- The creation of points within the tree (w/ a JSON POST to `/api/v0/places`)
- Query of points within the tree within a certain bounded area (w/ a GET to `/api/v0/places/near/$lat/$long/$distanceInKm`)
- Retrieval of a single point based on its ID (w/ a GET to `/api/v0/places/$placeId`)
- Random retrieval of a single point (w/ a GET to `/api/v0/places/random`)

... and a few more endpoint detailed in the [handler chain](https://github.com/joshdurbin/ephemeral-reactive-geospatial-rest-api/blob/master/src/main/groovy/io/durbs/rtree/places/PlacesHandlerChain.groovy).

Usage of the R-tree is straightforward, with the bulk of logic in the [service implementation](https://github.com/joshdurbin/ephemeral-reactive-geospatial-rest-api/blob/master/src/main/groovy/io/durbs/rtree/places/RTreePlacesService.groovy).
In fact, the only tricky part handling the continued re-assigned of the volatile
variable holding the R-tree whenever a place/point is inserted.

To handle this, a Rx [PublishSubject](http://reactivex.io/RxJava/javadoc/rx/subjects/PublishSubject.html) is created and all mutating operations (inserts)
are done via interaction with the subject rather than the R-tree itself. The subject
handles the re-assignment of the tree data structure as it processes each stream (request).

```groovy
tree = R-treeBuilder.create()

subject = PublishSubject.create()
subject.subscribe(new Subscriber<Entry<IdAssignedPlace,Point>>() {

    @Override
    void onCompleted() {

    }

    @Override
    void onError(Throwable exception) {

        log.error(exception.getMessage(), exception)
    }

    @Override
    void onNext(final Entry<IdAssignedPlace, Point> entry) {

        log.debug("adding place '${entry.value().name}' w/ coordinates [${entry.value().latitude},${entry.value().longitude}]")
        tree = tree.add(entry)
    }
})
```

All other query and fetch operations are executed against the Observable streams
that derive from the tree. This results in query performance that's really quite astounding,
but at the cost of memory, CPU, and data durability. Query operations against the R-tree are
a bit costlier as they perform distance calculations against the returning Observable stream.

```groovy
Observable<PlaceWithDistance> findPlacesNear(Double latitude, Double longitude, Double searchRadius) {

  final Point geographicPoint = Geometries.pointGeographic(latitude, longitude)

  final Position queryPosition = Position.create(latitude, longitude)

  final Position north = queryPosition.predict(searchRadius, 0)
  final Position south = queryPosition.predict(searchRadius, 180)
  final Position east = queryPosition.predict(searchRadius, 90)
  final Position west = queryPosition.predict(searchRadius, 270)

  final Rectangle searchArea = Geometries.rectangle(west.getLon(), south.getLat(), east.getLon(), north.getLat())

  tree.search(searchArea)
    .filter({ Entry<IdAssignedPlace, Point> entry ->

      final Point point = entry.geometry()
      final Position position = Position.create(point.y(), point.x())

      queryPosition.getDistanceToKm(position) < searchRadius
    } as Func1)
    .limit(placesConfig.maxResults)
    .map({ Entry<IdAssignedPlace, Point> entry ->

      entry.value()
    } as Func1)
    .map( { IdAssignedPlace place ->

      new PlaceWithDistance(distance: Geometries.pointGeographic(place.latitude, place.longitude).distance(geographicPoint) * Constants.FIND_NEAR_DISTANCE_CALCULATION_MULTIPLIER, place: place)
    } as Func1)
  .bindExec()
}
```

For more information, see the [load test blog post]({{< ref "/posts/2017-12-load-testing-ephemeral-reactive-geospatial-rest-api.md" >}}).

Give Ratpack and David's R-tree a read over too. They're both light,
performant, explicit and generally easy to use and test against.

## See Also ##

- [Spatial Trees](http://skipperkongen.dk/files/spatialtrees_presentation.pdf) and the problems they solve
- My [places](https://github.com/joshdurbin/places) project that inspired this project
