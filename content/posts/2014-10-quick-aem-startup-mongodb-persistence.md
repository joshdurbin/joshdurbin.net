+++
date = "2014-10-14"
title = "Quick AEM 6 startup w/ MongoDB persistence"
tags = [ "aem", "mongo" ]
+++

A few quick notes on getting AEM 6 off the ground w/ MongoDB (on OS/X, specifically) ...

1.  Install mongodb via brew: `brew install mongodb`
2.  Unpack the AEM 6 JAR:&nbsp;`java -jar cq-quickstart-6.0.0.jar -unpack`
3.  Take note of the exploded `crx-quickstart` directory
4.  Modify the start script (`crx-quickstart/bin/start`) providing an additional runmode: `crx3mongo`
5.  Modify the JVM args adding the argument: `oak.mongo.uri` with the value&nbsp;`mongodb://localhost:27017` (update as needed)
6.  Start mongod:&nbsp;`mongod --dbpath /data/db --httpinterface --journal --directoryperdb --rest`
7.  Fire up AEM

It took my 1st generation MacBook Retina about 4-5 minutes to install and settle.

For more information, see [Jayan Kandathil](http://cq-ops.tumblr.com/)'s [post](http://cq-ops.tumblr.com/post/86895378084/how-to-run-aem-6-0-with-mongodb-2-6).
