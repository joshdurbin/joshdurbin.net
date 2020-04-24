+++
date = "2015-05-18T16:48:12-07:00"
title = "Sling Background Servlets / Jobs. Future Feature?"
tags = [ "aem", "sling", "jcr" ]
+++

Among the many new features that dropped over a year+ ago in AEM 6.x is the feature: Sling Background Servlets and Jobs. There is little documentation of this feature -- most of which is in [JIRA tickets](http://permalink.gmane.org/gmane.comp.apache.sling.devel/17038) from the past.

As you might expect, this feature aims to provide some mechanism for the background processing of Servlet requests and perhaps other internal Jobs. At first I thought the job was somehow related to the introduction of Jetty 8.x and [servlet spec 3.0](http://download.oracle.com/otndocs/jcp/servlet-3.0-fr-oth-JSpec/)′s async feature(s). Further investigation seems to indicate they are not related.

The Background Servlet and Jobs feature allows a caller to add a query parameter “`sling:bg`” (is configurable) with a value of “`true`” to their request. The [BackgroundServletStarterFilter](https://github.com/sling-git/org.apache.sling.bgservlets/blob/master/src/main/java/org/apache/sling/bgservlets/impl/BackgroundServletStarterFilter.java) “captures” said request and begins an atypical flow.

The filter constructs a new request based off the original and invokes the destination servlet in a separate thread pool. The resulting stream of the background invocation is stored within the JCR within `/var/bg/jobs` in a path that is a unique? representation of the request timestamp...

The caller is made aware of this path via a more timely response -- obviously independent of the background processing. It is the callers responsibility to poll for their output, or results, via their stream location.

For example:

A call to “[http://localhost:4502/content/dam.json”](http://localhost:4502/content/dam.json?sling:bg=true) yields the result:

    {"jcr:primaryType":"sling:OrderedFolder","jcr:mixinTypes":["mix:lockable","rep:AccessControllable"],"jcr:createdBy":"admin","jcr:created":"Thu Apr 02 2015 11:00:23 GMT-0700"}

Not ground breaking, right? Now request the same resource with the “`sling:bg`” param: “[http://localhost:4502/content/dam.json?sling:bg=true”](http://localhost:4502/content/dam.json?sling:bg=true). Sling responds with a 302 to http://localhost:4502/var/bg/jobs/2015/05/18/14/54/93534.json with a body of:

    {"info":"Background job information","jobStreamPath":"/var/bg/jobs/2015/05/18/14/54/93534/stream.json"}

Note: The extension of the job output tends to match the extension of the calling request -- special cases exists, like for binary assets (images, etc...). In this case it’s JSON because we originally requested JSON. If your original request was in HTML you’d get something more like:

    Background job
    <link rel="stream" href="/var/bg/jobs/2015/05/18/16/05/15047/stream.html"><h1>Background job information</h1>
    Job output available at
    <a href="/var/bg/jobs/2015/05/18/16/05/15047/stream.html">/var/bg/jobs/2015/05/18/16/05/15047/stream.html</a>
    

... which is obviously much more difficult to parse.

## KNOCK. KNOCK. HELLLLLO? ARE YOU DONE YET?!? ##

Up to this point, this all made sense -- although I must express the fact that I have great reservations regarding the practical applications and performance of such an approach.

So, what happens we issue another request to get our ouput at `/var/bg/jobs/2015/05/18/14/54/93534/stream.json`? Answer: **nothing**.

I can’t seem to find any output data for any of the resource-based background requests I’ve made to out-of-the-box Sling servlets (JSON/HTML/XML).

Furthermore, one background request alone will run an unlimited number of jobs that grow my local repo at ~4GB an hour. Stopping/starting the instance and perhaps the executing thread pool stops the processing and growth.

You can examine these background requests in the “Background Servlets & Jobs” view found under the “Main” menu in the AEM Web Console. **Note**: The plugin seems to load all the background job data in an attempt to return the most recent 100 (give or take) jobs. So if you've let this thing run to generate gigs of data it will take a while to load...

## TURN. IT. OFF! ##

As far as I can tell, an authorized request to your instance with the parameter/value “`sling:bg=true`” is all it takes to execute an unbounded number of background jobs and fill `/var/bg/jobs` with resulting nodes.

Additionally, it's unclear if the results nodes have the appropriate access controls in place.

Most production deployments of AEM will disallow all or only allow a few, very specific query parameters to hit their dispatcher and underlying publisher nodes.

A wise response might be to configure the Apache Sling Background Requests Filter such that if it fires only on a very specific, very private key or, better yet, disable the Filter altogether.

