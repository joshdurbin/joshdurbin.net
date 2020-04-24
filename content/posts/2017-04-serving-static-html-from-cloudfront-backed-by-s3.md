+++
date = "2017-04-17T17:12:44-05:00"
title = "Serving static HTML from Cloudfront backed by a web-hosting enabled S3 bucket"
tags = [ "aws", "cdn", "infrastructure as code"]
aliases = [ "/blog/serving-static-html-from-cloudfront-backed-by-a-web-hosting-enabled-s3-bucket/", "/blurbs/serving-static-html-from-cloudfront-backed-by-a-web-hosting-enabled-s3-bucket/" ]
+++

Two weeks ago I transitioned my personal site, this site, to an SSL-based, secure only site. Prior to the transition,
the site was served from s3 storage with web hosting enabled. Now the site is served from CloudFront backed by an
s3 origin. I had done this before for work, for clients, but never with static web hosting. During this exercise, I
found a documentation gap in the process, specifically with Terraform...

By default, S3 does not enable web hosting which means queries must directly match files and directories. There is
no "fuzzy space" in between a hit and miss against the bucket contents. Such a setup is generally leveraged in cases
where requests are made with confidence against the bucket, typically by a caller who has a particular asset path in
mind.

S3 web hosting adds a layer of functionality to a particular bucket. This functionality allows for a directory
listing, custom HTTP responses, etc... [Directory listings](https://wiki.apache.org/httpd/DirectoryListings) allow for
requests against directories to render based on the configured index page, typically a child resource (ex `index.html`).
For example, an S3 bucket with web hosting enabled and a configured index page of `index.html` would respond to the
request `GET /path/to/dir` by resolving and returning `/path/to/dir/index.html`. This sort of configuration is
necessary for many content systems, static web hosting systems, etc...

## S3 Configuration

When an S3 bucket is configured for web hosting, two HTTP endpoints are generated -- the regular bucket endpoint and the
  web hosting enabled bucked endpoint. It might seem obvious, but the web hosting functionality only applies to the
  ... you guessed it, web hosting endpoint. Take the following configuration:

```yaml
resource "aws_s3_bucket" "example_bucket" {

  bucket = "joshs_example_bucket"

  website {

    error_document = "404.html"
    index_document = "index.html"
  }
}
```

... it generates two endpoints:

1. The bucket HTTP endpoint
2. The web hosting enabled bucket HTTP endpoint :

## Cloudfront configuration

Cloudfront's role in this architecture is to take the defined distribution; origin, cache parameters, edge lambdas,
  etc and distribute that content to edges. Cloudfront turns to its upstream origins (S3 or ELBs) to resolve cache
  misses. In the most basic of configurations with S3, defining an `origin` with a `domain_name` pointed at your S3
  bucket will suffice. Ex:

```yaml
resource "aws_cloudfront_distribution" "example_distribution" {

  origin {

    domain_name = "s3-bucket-http-endpoint"
    origin_id = "your_origin_id"
  }

  ...

}
```

This configuration works, though it does not take advantage of the web hosting functionality configured on our target
  bucket. The most obvious path of resolution is to change the `origin` `domain_name` to the web hosting enabled
  HTTP endpoint for our bucket. This works if you execute this from the AWS CLI or via the AWS Web Console. The
  same operation from Terraform, however, **does not work** -- AWS complains that the endpoint is not a valid S3 bucket.

It turns out that when you use the AWS CLI or AWS Console to alter your Cloudfront Origin, the origin is no longer
  considered of type "s3". To resolve this in Terraform, we have to leverage the `custom_origin_config` block within
  our `origin`, within our `aws_cloudfront_distribution` resource. This configuration:

```yaml
  origin {

    domain_name = "s3-bucket-http-endpoint"
    origin_id = "your_origin_id"
  }
```

... becomes this configuration:

```yaml

  custom_origin_config {

    origin_protocol_policy = "http-only"
    http_port = 80
    https_port = 443
    origin_ssl_protocols = ["TLSv1", "TLSv1.1", "TLSv1.2"]
  }
```

After waiting a bit for the CloudFront distribution to finish rollout, we should now see requests for directories
  resolve to their `index.html` files (when they exist).

If you'd like to see a complete example of this, including security settings and what not, check out my personal
  infrastructure starting with the [cloudfront.tf](https://github.com/joshdurbin/aws-infrastructure/blob/master/cloudfront.tf).
