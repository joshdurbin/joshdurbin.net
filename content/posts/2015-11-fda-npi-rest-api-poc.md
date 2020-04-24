+++
title = "Reactive FDA NDC and CMS NPI REST services via Ratpack"
description = "POC work leveraging Ratpack, Mongo, and ETL processes for data ingestion, transformation, and serving to Cordova and content clients."
date = "2015-03-10"
tags = ["poc", "ratpack", "rxjava", "reactive"]
aliases = ["/projects/2015-11-fda-npi-rest-api-poc/"]
+++

Back in March a large government health entity approached us regarding
the development of a series of new web properties and mobile apps. These
"tools" were meant to be used by providers and patients alike for the entity's
context-specific lookup and reference of prescription drugs and national
health provider data.

I worked on POC development efforts for what ended up being about forty hours of
work spread over 4 weeks. The work involved leveraging Ratpack, a new performance
focused, ultra non-blocking JVM web framework.

The REST APIs were meant to run on Heroku with very limited resources and
overhead cost, all while providing the mobile and CMS content ingestion teams
consistent/reliable performance. The work consisted of four parts:

1. The Federal Drug Administration (FDA) National Drug Registry (NDC) REST
[ETL scripts](https://github.com/joshdurbin/fda-ndc-rest) meant to pick up and consume compressed static files and refresh Mongo-backed
storage with updates.
2. A reactive, Ratpack-based, [REST service](https://github.com/joshdurbin/fda-ndc-rest) leveraging Mongo and Redis
for caching with individual requests wrapped within Netflix Hystrix command,
fallback, rate limit framework for added resiliency.
3. Centers for Medicare and Medicaid Services (CMS) National Provider
Identifier (NPI) REST [ETL scripts](https://github.com/joshdurbin/cms-npi-rest) essentially doing the same as their FDA
NDC counterparts.
4. A reactive, Ratpack-based, [REST service](https://github.com/joshdurbin/cms-npi-rest) leveraging Mongo but without
Hystrix integration/command wrapping.

Ratpack's non-blocking methods and RxJava integration make it fun to play
with and undeniably faster than the Servlet-based Apache Sling system
I'm used to these days. Using Observable streams definitely requires more thought
than just throwing Runnable or Callables at an ExecutorService and waiting
for your Futures or callbacks.

The reactive extensions project has frameworks for most languages and
[this guide](https://gist.github.com/staltz/868e7e9bc2a7b8c1f754) is great for understanding how these systems
work.

Take a look at the work, tear it apart (#noTests :-()!
