+++
title = "AWS Serverless Canary Sensor Capture, 1st release"
date = "2017-03-22T01:19:19-07:00"
tags = [ "aws", "canary", "infrastructure as code"]
aliases = [ "/blog/aws-serverless-canary-sensor-capture-1st-release/", "/blurbs/aws-serverless-canary-sensor-capture-1st-release/" ]
+++

Tonight I dropped the first of a few commits/releases of a TF module aimed at pulling and cheapily storing [Canary security](https://canary.is)
device sensor data (temp, humidity, air quality) on AWS using Lambda and DynamoDB.

Over the next few days I'll add error handling to the API calls, add token refresh support, and an API Gateway implementation
that will allow for securely querying the data. The ultimate goal is to plot the historical data on graphs, for fun. duh! :)

For more information, checkout the readme on the [github project](https://github.com/joshdurbin/aws_canary_sensor_capture).
