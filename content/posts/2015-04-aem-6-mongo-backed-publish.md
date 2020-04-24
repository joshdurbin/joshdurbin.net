+++
date = "2015-04-10"
title = "AEM 6.x mongo-backed, clustered publish architecture"
tags = [ "aem" , "mongo" ]
+++

AEM is a monolith...a big, big monolith. In addition to being a monolith, or possibly because it's
a monolith, AEM fails to adhere to a n-tier or classic three tier web architecture incorporating
a presentation, domain logic, and persistence layers.
Traditionally, at least up until the next-gen [JCR implementation](https://en.wikipedia.org/wiki/Content_repository_API_for_Java) [Oak](https://jackrabbit.apache.org/oak/),
AEM bundled all these and more into one, containing:

- presentation logic
- internal and external data
- business logic
- integrations
- configurations and secrets

In some ways this affords operations engineers and SREs
great flexibility and ensures that issues with one publish node can't affect the others.
Bad nodes can't affect other nodes because they're not actually cross communicating.
The nodes act alone with a load balancer spreading traffic across the members and accept
content and code changes from author instances. The following diagram depicts this...

![image](/img/2015-04-aem-6-mongo-backed-publish/rep_agents_diagram.jpg)

The configured replication agents on author (one agent per target publisher) are essentially
a sling-managed queue with payloads or payload descriptors serializing and transmitting data
to configured endpoints on the publisher.

**Note**: If you're thinking "oh what about reverse replication?", don't.

A potential downside of this design is scale out efficiency and speed during un-planned load events.
It will take time to provision new server nodes and copy data if each publisher contains a
few terrabytes of data. Additionally, it's not feasible to do a full tree replication from author when new, fresh
publishers come online.

Enter... mongo. Ohhh Mongo. People love to hate Mongo, but I don't. I especially find it funny when people refuse
to use Mongo and create their own "unstructured, document-like" schemas instead of splitting ownership, concerns, etc...
across a variety of data stores. Anyway... AEM 6.x Oak JCR implementation additionally supports a Mongo persistence kernel
kernel. The [official](https://docs.adobe.com/content/docs/en/aem/6-1/deploy/platform/aem-with-mongodb.html) Adobe docs recommend
that publishers **never** be used with Mongo perrsitence kernels enabled, but I disagree with this. I think a Mongo-based
publisher installation can be beneficial for operators.

A few quick points about Mongo and AEM:

- MongoMK is **definitely** slower than TarMK for nearly all operations
- TarMK forces a union between app server, code, db (see aforementioned single tier reference)
- MongoMK based installations require all app servers to be in communication with a Mongo cluster
- Mongo clusters are at minimum a size of 3

Why do I disagree with this? Operationally, particularly in certain scaling scenarios (like see-sawing traffic and usage
patterns), AEM app servers could very rapidly be provision and decommisioned based on load, all while leveraging
a robust back-end Mongo data infrastructure. While this breaks the normal pattern for AEM publish instances,
it puts AEM in line with other 2-3-n tier applications, which is a good thing. The architecture then looks more like
this (with more mongo instances than two):

![image](/img/2015-04-aem-6-mongo-backed-publish/rep_agents_diagram_2.jpg)

Adobe and the Apache Jackrabbit teams are all iteratively improving on the new MongoMK kernel
and I expect the feasibility of this sort of architecture to grain traction over future releases.
That is, I'd expect performance between the TarMK and MongoMK kernels to improve.

One caveat to this type of architecture configuration with AEM 6.1 is that the existing replication
agent implementations still expect a single target node. If multiple replication agents exist, targeting
multiple publish instances backed by Mongo, AEM author will throw a lot of nasty duplicate document errors
upon replication. This occurs because each instance receives the document and tries to write it to the shared
Mongo document repo anew.

Thus, in order to further realize "the dream" of splitting the data / code and app, a custom replication agent would
need to exist that can health check and load balance across a set of endpoints (where any endpoint can successfully
ack a transmission and that transmission be considered done and persisted). Right now, AEM doesn't have that, so build
your own or use TarMK Publishers.