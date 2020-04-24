+++
title = "AEM OSGi Service and Component Monitoring"
description = "Adobe Enterprise Manager OSGi service and component monitoring"
date = "2014-03-10"
tags = ["aem", "osgi", "devops", "gpars", "monitoring"]
aliases = [ "/projects/2014-03-aem-service-monitoring/"]
+++

AEM is a juggernaut in the content and experience management space. It is used far
and wide from nonprofits to the most profitable companies our modern world has seen.
AEM is also an incredible monolith in the budding world of microservices and soon
the up-coming world of functions as a service.

Engineers and users of the AEM as a platform write code which resides within bundles.
These bundles create little pockets, islands, where the code runs and is able to specifically
import and expose code, libraries, interfaces, APIs. It also allows for service
and OSGi component definitions that can dictate their operational conditions and interlocking
service dependencies/references.

It is quite typical for customer needs to justify creating extensive APIs and services
to use for web content, mobile content, and API services to anything in-between,
like retail displays and internet of things.

AEM also lives somewhat in its own a world -- a world where open tech and products often
require additional work for import. If an API or persistence engine driver doesn't
provide OSGi manifests, then the system (Apache Felix) doesn't know how to load it.
This is often the case. Couple this with the fact that AEM, formerly CQ, has been around for a little
over a decade, and it's own feature richness and you end up painting a picture. A
picture of a self-contained ecosystem that occasionally yearns for the coasts of its
neighbors (i.e. other tech).

Configuration management, property encryption are an example of this. AEM leverages
the OSGi property configs coupled with run modes to layer together something that
resembles Hashicorps [Consul](https://www.consul.io), etcd, or other key-value stores.
Live config changes in say, production, can yield random and in-consistent outcomes
from the merged configs like those stored in the JCR. Cases like this introduce
service and component availability issues within an otherwise healthy AEM appserver monolith.

## Monitoring ##

And so most devops, cloud engineering, etc... teams monitor an instances uptime by
pinging prescribed endpoints, monitoring for disk usage, free space, CPU load, etc...
Monitoring memory utilization requires something that can introspect the JVM as
the 8-16-32 gigs of RAM you allocate to the JVM running AEM are taken pretty right
away at startup.

I had been on a number of high profile projects where we ran into random service
outages and performance inconsistencies. These were typically caused by continuous
code delivery mechanisms causes bundles to rise and fall along with their components
and services. Frequent installations in and of itself are not nefarious, but
occasionally set a chain of events in motion that can render inconsistent service
states. That is why in April of last year I created a crude monitoring framework for
monitoring OSGi services and components; the [osgi-service-monitor](https://github.com/Citytechinc/osgi-service-monitor).

## OSGI Service Monitor ##

At its heart, the monitor set to loosely bind to _other_ service references, and
provide an ecosystem consistent polling "function" that allowed for other service
implementations to watch for success and failure states. The monitors were termed
`ServiceMonitor`s and the watchers were termed `NotificationDeliveryAgents`. The `ServiceMonitor`s
crucially obtained, checked for, and guarded against missing, faulty, or failed service
references and would report state to the `NotificationDeliveryAgents`. This system
worked well for a variety of CITYTECHs clients at the time.

Despite decent adoption internally, the system was in need of configuration options,
defined polling intervals, dashboards, outcome persistence, etc... This would be critical
in releasing such a framework to a broader audience, though that was never done.

## Canary ##

Early in 2014 I set out to create the second version, a complete re-write of what was formerly
the [osgi-service-monitor](https://github.com/Citytechinc/osgi-service-monitor).
I called this new project [AEM Canary](https://github.com/joshdurbin/aem-canary). Out of the box,
Canary set to build on the old monitor framework covering the following functionality:

- Monitor functionality with service annotation support defining when and how a monitor fires
- Notification agent support to handle and react to failures for a particular monitor or monitors
once a defined threshold had been met or exceeded
- Response handlers meant to react to poll responses, regardless of the state
- Persistence to the JCR
- Additional JMX data
- Useful [monitors](https://github.com/joshdurbin/aem-canary/tree/master/src/main/groovy/com/citytechinc/aem/canary/services/monitor) OOTB (blocked agent queue, sling event monitoring, exception listeners, etc...)
- A graphical dashboard for viewing and interacting with the monitors and response data

Under the hood, Canary was far more complex than the 1st iteration of the framework.
The core of the framework used an Actor pattern to handle message passing between the monitors, wrapped in
[actors](https://github.com/joshdurbin/aem-canary/tree/master/src/main/groovy/com/citytechinc/aem/canary/services/manager/actors).
A sort of "mission control"-type service bridge the gap between services coming up and down
in the OSGi ecosystem with the generated actors. Quartz was utilized in providing
the critical timed execution, polling.

The actor framework, provided by [GPars](http://www.gpars.org), allowed each unit to fire
and communicate with one another via the actor's inbox without worrying about
concurrency or timing usage. Several years later, after development, I think it's
fair to admit that an Actor framework was overkill and perhaps unnecessary.

In the end, by the time I was close to finishing AEM Canary, AEM 6.0 was launched
with a health check project that had made its way into the product. The project was
pulled into the Sling[https://sling.apache.org] ecosystem as [Sling Health Check](https://sling.apache.org/documentation/bundles/sling-health-check-tool.html).
I ended up more or less dropping the project, moving on to work on some non-AEM-based platforms, like
building mobile backends in PlayFramework. Oh. Actors again, you say? Yes.

Anyway, it was a fun side project. A project I wrote about a few times and tried to
promote, but at the end of the day was a good learning lesson in engineer maturity,
framework maturity, and toolchain adoption by a community.

## See Also ##

- a [Canary Framework](http://slides.com/joshdurbin/canary-framework#/) introduction
