+++
title = "Home Network Monitoring"
description = "Project to poll, report and track cable modem stats and general network performance stats (like ICMP)"
date = "2017-12-06"
tags = ["dynamodb", "apigateway", "terraform", "groovy", "scraping"]
aliases = ["/projects/2017-12-an-ephemeral-reactive-geospatial-rest-api/"]
+++

Every once and a while my Comcast-based cable service gets into a little bit of a wonky state.
It works, but then, it does, but then it does, but then it doesn't... I'm sure it's probably a
unique experience.

Anyway, when it does, I've found that I have nearly no leg to stand on when citing ICMP response
times, download rates, and cable modem stats and events over time. All I can give the "support
staff" is what's happening *now*.

Way back in June of 2016 I wrote a [Groovy script](https://github.com/joshdurbin/network-status-monitor/blob/scripts/ScrapeMotoSBStats.groovy) to scrape the stats off my Motorola Surfboard modem
but did nothing with the project after I set it down. After some network caddywhompusness a few weeks
back I picked the project back up, but a bit more formally. Based on the work in the script a year prior, I
wrote a Java/Groovy app that would poll, store, and push the data up to AWS DynamoDB for time-series storage.

DynamoDB is often the data store of choice for my side projects as it costs me virtually nothing for storage
and I can easily "develop" REST APIs via API Gateway with [velocity](http://velocity.apache.org) template pass-thru
to DynamoDB.

Normally I try and make projects more extendable or applicable to a wider range of audiences, configuration
options, etc... but time is an ever-growing precious resource for this now 35-year-old! :)

The app runs on a local network Raspberry PI and leverages a single worker thread within Quartz to execute a series of
[jobs](https://github.com/joshdurbin/network-status-monitor/tree/master/src/main/groovy/io/durbs/netstatus/collection/job) on a specified interval:

- A cable modem log scraper
- A cable modem up and downstream stats scraper
- An "ICMP" tracer -- ICMP is in quotes because I open a socket, layer 4, connection to the DNS servers on the DNS port instead of ICMP.

... search for decent ICMP options from the JVM and you'll understand why I did this (remember, precious time).

The app runs these jobs, stores the data locally, in memory, then runs another job and tries the flush the objects
to DynamoDB via another Quartz job.

A [terraform module](https://github.com/joshdurbin/terraform-network-status-monitor-storage-and-api) exists to create the DynamoDB
tables:

- net_stat_tracker_downstream_modem_stats
- net_stat_tracker_upstream_modem_stats
- net_stat_tracker_echo_responses
- net_stat_tracker_modem_logs

... the necessary keys, IAM profiles and roles, API Gateway resources and what not. Leveraging AWS DynamoDB and API Gateway while
using the service pass-thru integration method allows me to capture and record this data, pull it, visualize it, and use it as evidence
for network engineering at essentially no cost to me.

## Sample Data

Cable modem downstream stat data:

```json
{
  "timestamp": "2017-12-05T21:24:57.379",
  "channel": "8",
  "frequency": "555000000 Hz",
  "powerLevel": "2 dBmV",
  "signalToNoiseRatio": "39 dB",
  "modulation": "QAM256",
  "totalUnerroredCodewords": "31733001042",
  "totalCorrectableCodewords": "25",
  "totalUncorrectableCodewords": "695"
}
```

Cable modem upstream stat data:

```json
{
  "timestamp": "2017-12-05T21:24:57.379",
  "channel": "4",
  "frequency": "38700000 Hz",
  "powerLevel": "46 dBmV",
  "rangingServiceId": "12429",
  "modulation": "[3] QPSK [3] 64QAM",
  "symbolRate": "5.120 Msym/sec",
  "rangingStatusSuccessful": ""
}
```

Cable modem log events:

```json
{
  "timestamp": "2017-12-05T21:24:57.500",
  "code": "R02.0",
  "logTimestamp": "1970-01-01T08:00:27.000Z",
  "message": "No Ranging Response received - T3 time-out;CM-MAC=3c:df:a9:44:5b:f6;CMTS-MAC=00:01:5c:63:14:59;CM-QOS=1.1;CM-VER=3.0;",
  "priority": "3-Critical"
}
```

and ICMP response times:

```json
[
  {
    "timestamp": "2017-12-04T18:24:56.904",
    "endpoint": "8.8.8.8",
    "success": "",
    "time": "19"
  },
  {
    "timestamp": "2017-12-01T22:24:56.590",
    "endpoint": "141.142.2.2",
    "success": "",
    "time": "74"
  },
  {
    "timestamp": "2017-12-02T18:24:57.010",
    "endpoint": "4.2.2.2",
    "success": "",
    "time": "16"
  }
]
```

## Todos

- Change the "ICMP" job to timeout more quickly
- Update the API Gateway velocity templates to support pagination and query support for things like; distinct endpoints, average, etc...

