+++
title = "Evaluating container CPU workloads in Nomad"
date = "2019-11-29"
tags = ["nomad", "containers", "prometheus", "groovy"]
+++

[Nomad](https://www.nomadproject.io) has [been](https://endler.dev/2019/maybe-you-dont-need-kubernetes/) [getting](https://news.ycombinator.com/item?id=22034291) a [lot](https://news.ycombinator.com/item?id=19467067) of [press](https://hn.algolia.com/?q=don%27t+need+Kubernetes+) lately with its ecosystem partners [Consul](https://www.consul.io) and [Value](https://www.vaultproject.io). You probably already know, but Nomad is a scheduler of various sorts; docker containers, java processes, other task drivers... It simply executes jobs on fleets of clients/worker nodes, passing parameters about those tasks/task groups/jobs to the underlying task drivers. Resource constraints are defined on [`resource stanza`](https://www.nomadproject.io/docs/job-specification/resources.html) for CPU, memory, network and GPU settings.

When a task attempts to use more resources than its allocated, like a process reads an exceptionally large file into memory when the file is normally tiny, the task driver and underlying sub-system (like [Docker](https://www.nomadproject.io/docs/drivers/docker.html)) might OOM kill the allocation (the placement of the task). CPU restrictions are a bit trickier, though, first because Nomad runs by default with a config called
`cpu_hard_limit` set to false. This means that compute workloads can burst beyond their allocated, scheduled, and placed amounts. There are benefits to this bustiness, but there are also drawbacks. Benefits are given to applications with unpredictable workloads that are prone to sudden spikes in utilization. A cluster with a bunch of workloads that use over their allocation and those that are chronically under should be balanced.

Balancing is hard in Nomad. Metric reporting to systems like [prometheus](https://www.nomadproject.io/guides/operations/monitoring-and-alerting/prometheus-metrics.html) shows the CPU utilization for a task as a percentage of a CPU core on that client; i.e. %3.2 or %191.4. Comparing this value against the MHz in the resource stanza for the task is difficult. That's why I wrote [nomadWorkloadCPUActualsReport](https://github.com/joshdurbin/nomad-workload-cpu-actuals-report-generator).

This tool queries Nomad for its clients and their CPU clockspeeds, all the given tasks across the fleet of machines, then compares and cross references that information about task CPU utilization. The tool then renders this data in the same format, MHz, as the resource stanza letting
operators know what workloads are chronically use more or less than their expected, configured amount. The tool renders in three colors; red for any workload over 2x allocation, yellow for anything up to 2x allocation, and green for anything under allocation (to zero). Yellow and green render stronger, bolder, when they're close to their limits 2x and 0, respectively. This allows users to quickly identify how things are behaving within a given period. Information about the [shape of the data](https://brownmath.com/stat/shape.htm) is additionally rendered like std dev, kurtosis, and skewness.

![image](/img/2019-11-nomad-cpu-actuals-report-generation/output.jpg)

The tool can hit multiple environments at once, supports sleep operations, supports fallback clock calculations, and by default queries for the past hour, day, week, month, and, optionally 90 days.

Here's a sample output of the application:

```
usage: ./nomadWorkloadCPUActualsReport -e <comma delimited environments> <other args>
Nomad Workload CPU Actuals Report
 -avg,--avgKnownNomadClients           Use instead of --nomadFallbackClock to average the known Nomad clients for historical workloads
 -d1d,--disableOneDayQuery             Disable 1 day query
 -d1h,--disableOneHourQuery            Disable 1 hour query
 -d30d,--disableThirtyDayQuery         Disable 30 day query
 -d7d,--disableSevenDayQuery           Disable 7 day query
 -e,--environments <arg>               Environments (comma delimited)
 -e90d,--enableNinetyDayQuery          Enable 90 day query
 -fbc,--nomadFallbackClock <arg>       The approximate clockspeed of out-of-service Nomad nodes that were used to run historical workloads tracked in Prometheus [defaults to 2300]
 -h,--help                             Usage Information
 -nc,--nomadTLSCertFilename <arg>      Nomad TLS Key Filename [defaults to %env%.nomad.key]
 -nca,--nomadTLSCACertFilename <arg>   Nomad TLS CA Certificate Filename [defaults to nomadca.crt]
 -nh,--nomadHost <arg>                 Nomad host [defaults to https://nomad.service.%env%-dc1.consul:4646]
 -nk,--nomadTLSKeyFilename <arg>       Nomad TLS Certificate Filename [defaults to %env%.nomad.crt]
 -ph,--prometheusHost <arg>            Prometheus host [defaults to http://prometheus-read.service.%env%-dc1.consul:9090]
 -qst,--querySleepTime <arg>           Query sleep time in seconds [defaults to 0]
Environment queries are run in parallel to reduce report generation time. Use %env% to inject environment into --nomadTLSKeyFilename, --nomadTLSCertFilename, --nomadHost, --prometheusHost
```

Check it out! It only requires Groovy, Prometheus and Nomad.
