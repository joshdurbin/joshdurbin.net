+++
date = "2017-04-11T06:44:27-07:00"
title = "Evaluating API Gateway as a Proxy to internal AWS resources via Lambda and HTTP Proxy"
chartEnabled = true
tags = [ "aws", "cdn", "infrastructure as code"]
aliases = [ "/blog/evaluating-api-gateway-as-a-proxy-to-internal-aws-resources-via-lambda-and-http-proxy/", "/blurbs/evaluating-api-gateway-as-a-proxy-to-internal-aws-resources-via-lambda-and-http-proxy/" ]
+++

I've spent the last few weeks at work investigating and evaluating API Gateways to drop in front of our present
architecture. One of the candidates for evaluation was Amazon's [API Gateway](https://aws.amazon.com/api-gateway/).
I had used API Gateway in the past for little things here and there, but never as a "simple" proxy layer to existing
infrastructure. I set up a simple test and wrote a [bunch of code](https://github.com/joshdurbin/aws_api_gateway_proxy_vs_lambda_vs_direct_elb_performance_benchmark) to generate the necessary infrastructure and executed
 the tests...

To start, let me review what API Gateway is -- from ten thousand feet, API Gateway allows for rapid development of
REST-based endpoints by defining resource(s), method(s) and integration(s) of those methods. On the backend, API Gateway
integrates with:

1. AWS Lambdas
- AWS resources, "direct" via [templating](http://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-mapping-template-reference.html) (like DynamoDB, S3)
- Mock request/response systems
- An HTTP Proxy to a single, publicly available endpoint

In addition to concrete resources (ex: `/person`) and explicit HTTP methods (ex: `GET /person`), API Gateway allows for
[proxy resources](http://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-set-up-simple-proxy.html) and `ANY` HTTP method (hereby referred to just as "Proxy resources").
Proxy resources (ex: `/{proxy+}`) allow API Gateway to pass requests for that resource and any of its dependents to one of the two available integration methods:

1. A lambda
- An HTTP Proxy to a publicly available endpoint

I set out to compare the performance benchmarks with [a few simple tests](https://github.com/joshdurbin/aws_api_gateway_proxy_vs_lambda_vs_direct_elb_performance_benchmark/blob/master/siege_load_test.sh) last week. The goal was to compare the
  performance of proxying requests via the two aforementioned integration options in addition to direct communication
  with an ELB.

## Proxying with Lambdas

Proxying with lambdas is straightforward -- API Gateway forwards requests to the Lambda which _can_ be written to
  do _anything_ with those requests. For this test the lambda opens a connection to the internally available nginx instance,
  however, it could just as easily be a redis cluster, mongo, or postgres instance.

![image](/img/2017-04-initial-performance-benchmarks-python-api-gateway-proxy-vs-elb/api_gateway_proxy_lambda.png)

From a security standpoint, Lambdas are an attractive option because they allow API Gateway to be the isolating barrier
  from the outside world to the internal networks. This is accomplished by attaching an AWS Security Group and Subnet
  to the proxying [lambda function](https://github.com/joshdurbin/aws_api_gateway_proxy_vs_lambda_vs_direct_elb_performance_benchmark/proxy.py).

A static endpoint is configured in our POC Lambda which routes to a single web server. In reality, our function would route to an internal
  ELB, or routing layer (e.g. [kong](https://getkong.org/), [zuul](https://github.com/Netflix/zuul), [nginx](https://nginx.org/) etc...).
  VPC attached Lambdas do incur startup costs (measured in time) and are limited by the number of ENIs that can be bound within
  the assigned subnets.

The lambda grants flexibility, but requires maintenance, tuning, and "warming", presumably via AWS CloudWatch events.

If your API returns binary content, you'll need additional logic handling and API Gateway response type configuration for a
  Lambda integration. I did not go through this effort for these tests.

## Proxying via HTTP

In addition to proxying with Lambdas, API Gateway allows for proxying to **public** HTTP endpoints. The proxy integration
  method coupled with client certificates and request signing allows API Gateway to provide its value-add before it
  shuttles traffic up-stream.

![image](/img/2017-04-initial-performance-benchmarks-python-api-gateway-proxy-vs-elb/api_gateway_proxy_client_certificate.png)

Unlike Proxying via Lambda, Proxying via HTTP should require next to little maintenance and tuning. However, the logical architecture
  for this approach smells -- validating certificates requires additional resources in front of infrastructure. It would
  be awesome if Amazon updated their Application Load Balancer to verify signed requests.

## Test Specifics

To get a rough idea of how these two approaches would perform I set up a [simple test](https://github.com/joshdurbin/aws_api_gateway_proxy_vs_lambda_vs_direct_elb_performance_benchmark/blob/master/siege_load_test.sh)... The test leverages an
nginx instance with various pages, paths, and sizes. The nginx instance is in a private subnet and is not
accessible from the outside world.

A public non-SSL-enabled ELB exists to shuttle traffic from the outside world to the nginx box.

The HTTP Proxy test does **not** use client certificate signing as I didn't want to go through the work of having the private,
  nginx target instance verify said requests.

The test against the Lambda-based API Gateway instance was run three times to benchmark the benefits of memory and network
  capacity increases. Each test had two subnets spanning availability zones `us-west-2a` and `us-west-2b`. The amount
  of memory and network capacity (in addresses for ENI attachment) scaled with each test -- the first with 128 MB memory
  and ~254 addresses, the second with 512 MB memory and ~512 addresses, and the final with 1024 MB memory and ~1024 addresses.  

The following tests were executed using [wrk](https://github.com/wg/wrk) on a dedicated EC2 instance, each for
  30 seconds, 25 threads, and 2 connections per thread (for a total of 50 per test):

1. directly against the ELB (ELB -> nginx)
- against the Lambda-256 proxying API (api gateway -> lambda -> nginx)
- against the Lambda-512 proxying API (api gateway -> lambda -> nginx)
- against the Lambda-1024 proxying API (api gateway -> lambda -> nginx)
- against the HTTP proxying API (api gateway -> http proxy -> ELB -> nginx)

The resource allocations for the lambdas were very generous given the VPC, ENI capacity formula provided by AWS [here](http://docs.aws.amazon.com/lambda/latest/dg/vpc.html):

```text
Projected peak concurrent executions * (Memory in GB / 1.5GB)
```  

## Proxy POC results

The ELB performance establishes the baseline of throughput for our nginx instance, which is pretty good for
  such a little machine. At approximately %22 the performance of the ELB, its clear that the
  Proxy-based API Gateway incurs greater overhead. The Lambda-based API Gateway instances fared even worse, unfortunately.

{{< 2017-04-initial-performance-benchmarks-python-api-gateway-proxy-vs-elb >}}

For the Lambda-based API Gateways, the increase in memory (and presumably CPU performance) along with additionally
  available network addresses seemed to allow greater throughput. However, in at least two tests ("Lambda-512" and "Lambda-1024"),
  the Lambdas were over-allocated network address capacity and memory and only improved marginally.
