+++
title = "Load testing an in-memory, reactive, geospatial REST API"
date = "2017-12-28"
tags = ["wrk", "ratpack", "rxjava", "reactive"]
chartEnabled = true
aliases = [ "/blog/load-testing-an-in-memory-reactive-geospatial-rest-api/", "/blurbs/load-testing-an-in-memory-reactive-geospatial-rest-api/"]
+++

Yesterday I wrote [a project writeup]({{< ref "/posts/2017-12-an-ephemeral-reactive-geospatial-rest-api.md" >}})
on building an R-tree-based reactive REST API for geospatial data and queries. This blog post
will cover the query load testing efforts for the REST API. First though, a quick refresher...

R-trees have been around for decades and are used commonly with quad-trees, b-trees, and geohash
algorithms in a variety of spatial engines from PostGIS to Redis. I’ve done [extensive](https://github.com/joshdurbin/places)
no-SQL load testing of geospatial engines in the past, but haven’t written about it.
A few weeks ago a use case arose and I thought I’d dust off the aforementioned project,
tweak it, and load test it again, with many dependent project revisions and change in the benchmarking tool.

The previous benchmarking tool I used pretty consistently was siege. I used to love [siege](https://github.com/JoeDog/siege).
It works well load testing servlet-based systems that, at least in my case, weren’t quick enough
to cause hiccups in siege. However, if you point siege at a crazy fast system, you’ll end up tweaking
siege so much that its hard to gauge if its (lower) results are in fact valid. This is true,
especially when comparing to commercial load testing tools or open source ones like [ab](https://httpd.apache.org/docs/2.4/programs/ab.html), [gatling](https://gatling.io/), [wrk](https://github.com/wg/wrk), etc...

Not to mention I’ve done a bit of Lua lately w/ Kong and Redis and wrk happens to have scripting
support for it. Most importantly, wrk is able to exert and maintain high levels of concurrency
and threading against targets.

## Insertion Testing ##

The [load_test_scripts](https://github.com/joshdurbin/ephemeral-reactive-geospatial-rest-api/tree/load_test_scripts) branch of the project
contains shell and Lua scripts for both inserting and querying data. The insertion aims to fill the tree with a precise number of entries (653,796)
which are real places found in the bay area and available [here](https://github.com/joshdurbin/ephemeral-reactive-geospatial-rest-api/tree/data).

The script loads the file reads each line containing a JSON object of a place and then posts that data to the prescribed endpoint.

```
function file_exists(file)
  local f = io.open(file, "rb")
  if f then f:close() end
  return f ~= nil
end

function lines_from(file)
  if not file_exists(file) then return {} end
  lines = {}
  for line in io.lines(file) do
    lines[#lines + 1] = line
  end
  return lines
end

placesPerLine = lines_from("bayareaplaces.json")

if #placesPerLine <= 0 then
  print("insert_places: no places found")
  os.exit()
end

counter = 0

request = function()

  local place = placesPerLine[counter]
  counter = counter + 1

  if counter > #placesPerLine then
    os.exit()
  end

  wrk.method = "POST"
  wrk.headers["Content-Type"] = "application/json"
  wrk.body = place

  return wrk.format(nil, "/api/v0/places")
end
```

The `insert_places.sh` script executes wrk w/ the aforementioned lua script and inserts the data, exiting once it processes the final POST.

## Query Testing ##

The `query_places.sh` script uses the points creating a rectangle over the bay area and randomly generating coordinates within the said rectangle. This allows me to avoid threading within Lua which reduces drag and memory concerns with wrk.

I commit number blasphemy by generating random whole numbers and concatenating them to produce coordinate pairs for querying...

```
local random = math.random

request = function()

  wrk.method = "GET"

  local distance = "5"

  local latWhole = random(37, 42)
  local latDec = random(random(0,999999))

  local longWhole = random(118, 123)
  local longDec = random(random(0,999999))

  local resource = "/api/v0/places/near/" .. latWhole .. "." .. latDec .. "/-" .. longWhole .. "." .. longDec .. "/" .. distance

  return wrk.format(nil, resource)
end
```

The REST api defines two query nearby endpoints – one which uses a programmed default radius and one that accepts a radius. When a radius is supplied, its evaluated against a max allowable radius and limited to the max should it exceed it.

The test was then run on the following radiuses for 30 seconds, using 50 connections across 10 worker threads:

- default (1/2 kilometer)
- 2 kilometer
- 5 kilometer
- 10 kilometer
- 25 kilometer
- 50 kilometer

## Query Testing Results ##

{{< 2017-12-load-testing-ephemeral-reactive-geospatial-rest-api >}}

The results show that the R-tree backed REST system is blazingly fast. Even with a 50km query area and 3x the
data returned of the 1/2km query area, the API returns at a rate of ~13.5k a second w/ 22ms response time.
At the other end, a more reasonable query area of 1/2km, the API returns at a rate of ~35k a second w/
~3 ms latency.

The graph below shows heap usage for the app during insertion (in black) and each of the six query testing phases.
The start of each test is denoted by dashed lines and the termination denoted by a solid line. The colors for
the query tests correlate with those in the previous, interactive graph.

![image](/img/2017-12-load-testing-ephemeral-reactive-geospatial-rest-api/vm-heap-analysis.png)

As the query area grows, the time required to traverse the tree increases with the nodes visited. The
likelihood of greater data returned means increased latency across the board. The impact on the heap is
and its visualization corroborates this.

The tests of this R-tree implementation far exceed what Mongo, ES, Redis, and rethink could do in my [places](https://github.com/joshdurbin/places)
project. It just so happens that I need a super fast query and pretty fast insertion performance on
unimportant/non-durable/ephemeral location data and I might build a more mature solution around this.

I'll write if I do. Until then, cheers to the new year: 2018!
