+++
date = "2015-04-27"
title = "AEM 6.x mongo-backed clustered author setup"
description = "Steps for setting up a mongo-backed AEM 6.x cluster for an authoring environment."
tags = [ "aem", "mongo" ]
+++

There's been a bit of internal discussion regarding the setup and load test of AEM 6.x (with specific interest on 6.1) with Mongo as persistence storage. The following outlines the instantiation and configuration of a (2) node AEM Author cluster backed by a (3) member mongo replica set consisting of PRIMARY, SECONDARY, and ARBITER cluster members.

### INITIAL AEM AUTHOR CONFIGURATION, STAND-ALONE MONGO CONFIGURATION ###

1. Create directory to store AEM author instance 0, e.g.: `author0`
2. Create directory a stand-alone directory for mongo storage (we’ll later convert this), e.g.: `mongo0`
3. Start mongo on the default port `27017`, executing: `mongod --dbpath mongo0/`
4. Copy or sym-link to AEM jar, license file within `author0` and unpack. Execute: `java -jar cq-quickstart-beta-6.1.0-load23-preload1.jar -unpack`
5. Modify script to use CRX3 Mongo, and point to stand-alone instance initiated in Step #3
  1. Append runmode `crx3mongo`
  2. Append mongo connection details via JVM argument key: `oak.mongo.uri` value: `mongodb://localhost:27017`
6. Start AEM and wait for installation to complete

### PREPARE FOR REPLICA SET CONFIGURATION ###

1. Shutdown AEM
2. Shutdown mongod running from `mongo0` on port `27017`

### DEFINE MONGO REPLICA SET ###

1. Create a directory for mongo replica set node 2, e.g.: `mongo1`
2. Create a directory for mongo replica set, arbiter node, e.g.: `mongo2_arb`
3. Start the stand-alone mongo instance with the `replSet` option set to `aem6`, e.g.: `mongod --dbpath mongo0/ --replSet aem6`
4. Start the node 2 mongo instance, e.g.: `mongod --dbpath mongo1/ --replSet aem6 --port 27018`
5. Start the arbiter mongo instance, e.g.: `mongod --dbpath mongo2_arb --replSet aem6 --port 27019`
6. Open a mongo shell and complete the replica set definition
  1. Initialize the replica set: `rs.initiate(rsconf = {_id: "aem6", members: [{_id: 0, host: "localhost:27017"}]})`
  2. Add replica 2: `rs.add("localhost:27018")`
  3. Add the arbiter: `rs.addArb("localhost:27019")`
7. Check status, wait until you have a PRIMARY, SECONDARY, and ARBITER node: `rs.status()`

### UPDATE INITIAL AEM AUTHOR INSTANCE MONGO ###

1. Modify AEM start script to point to additional, non-arbiter node
  1. Append host and port (e.g. `localhost:27018`) of additional node via JVM argument key `oak.mongo.uri`. Host:ports are comma delimited. The value of `oak.mongo.uri` should now equal `mongodb://localhost:27017,localhost:27018`
2. Start AEM Author

### DEFINE ADDITIONAL AEM AUTHOR INSTANCE ###

1. Create directory to store AEM author instance 1, e.g.: `author1`
2. Copy or sym-link to AEM jar, license file within `author1` and unpack. Execute: `java -jar cq-quickstart-beta-6.1.0-load23-preload1.jar -unpack`
3. Modify script to use CRX3 Mongo, and point to stand-alone instance initiated in Step #2
  1. Append runmode `crx3mongo`
  2. Change bind port from `4502` to `5502`
  2. Append mongo connection details via JVM argument key: `oak.mongo.uri` value: `mongodb://localhost:27017,localhost:27018`
4. Start AEM and wait for installation to complete

### SHUTDOWN PROCEDURES ###

1. Shutdown all AEM instances
2. Shutdown all MongoDB instances
