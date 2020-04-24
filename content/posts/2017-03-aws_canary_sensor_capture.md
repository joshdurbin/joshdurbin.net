+++
title = "Canary Sensor Data Capture, Serverless, on AWS"
date = "2017-03-26T20:38:33-07:00"
tags = [ "aws", "canary", "infrastructure as code"]
aliases = [ "/blog/canary-sensor-data-capture-serverless-on-aws/", "/blurbs/canary-sensor-data-capture-serverless-on-aws/" ]
+++

If you're reading this, you likely own a Canary and you're likely one of the
 [many people](https://twitter.com/search?f=tweets&vertical=default&q=%40canary%20api&src=typd) on twitter
 requesting an API from Canary. For at least a year Canary has acknowledged those requests and redirected them to
 product. As consumers, we have yet to see an API or any concrete movements towards one.

The only interface Canary has that is close to an API are the calls their [angular-based webapp](https://my.canary.is/)
 makes when you login to the dashboard. The dashboard displays your devices, who is home, and a snapshot of the current sensor readings for each device.

That interface or "private API", let's say, has undergone a variety of changes over time, which I've
 written about [here]({{< ref "/posts/2015-10-reading-sensor-data-from-canary.md" >}}) and [here]({{< ref "/posts/2016-04-canary-sensor-api.md" >}}).

Between the writing of those two posts Canary made a change that seems to have disallowed querying past sensor data.
 The current "private API" only seems interested in pulling current sensor readings for your devices.

So what if you want this data for yourself? The most reasonable thing I've come up with is a private, secure polling and storage
 mechanism for your Canary device's sensors (temp, humidity, air quality).

A [few days ago]({{< ref "/posts/2017-03-first-release-canary_sensor_capture.md" >}}) I wrote about my first drop of the [aws_canary_sensor_capture](https://github.com/joshdurbin/aws_canary_sensor_capture)
 project which aims to achieve just that. This post is a more comprehensive walkthrough of what is all provided
 and what you need to set something similar up for yourself.  

What does this thing do?
------------------

Generally speaking, the project leverages the serverless capabilities provided by Amazon Web Services (AWS) to securely poll,
 store, and govern access to your Canary device sensor data. The project uses [Terraform](https://www.terraform.io/) to declare
 the AWS resources that compose the solution -- with the addition of a bit of python code in the form of an AWS Lambda.

AWS Usage spans the following key areas:

* [KMS](https://aws.amazon.com/kms/) for encryption of your Canary account password and encrypted storage of the bearer token received from the Canary "private API"
* [API Gateway](https://aws.amazon.com/api-gateway/) which provides the endpoint to access your sensor data, by device ID
* [Lambda](https://aws.amazon.com/lambda/) for execution of the aforementioned python script
* 2x [DynamoDB](https://aws.amazon.com/dynamodb/) tables; one for metadata and one for sensor data storage
* [Cloudwatch](https://aws.amazon.com/cloudwatch/) event rules for scheduled execution of the Lambda

![image](/img/2017-03-aws_canary_sensor_capture/aws_canary_sensor_capture.png)

So what's all happening under the hood and when?

* The Cloudwatch event rules kick off the lambda. The lambda decrypts what it needs and checks to see if a metadata record
exists in DynamoDB. If one does, it uses this data to poll each of the devices listed in the metadata record and stores those
records in the another, sensor data DynamoDB table. If one does not, it will obtain a bearer token and the device IDs referenced
in the account then proceed with fetching sensor data.
* The API Gateway exists to allow a user (you) access to your data. The API Gateway supports read-only operations against
your DynamoDB-hosted sensor data. There are no lambdas involved for these requests as request/response data flows through
mapping templates only as it flows to and from the backend DynamoDB REST APIs.
* KMS is used at the appropriate times to decrypt/encrypt your Canary account password and stored bearer token.

Obtaining and Usage
------------------

The project source code is hosted [here](https://github.com/joshdurbin/aws_canary_sensor_capture) and is easily
leveraged in your own terraform project by declaring a module, specifying the source, and required input variables. For
example:

```json
module "canary" {
  source = "github.com/joshdurbin/aws_canary_sensor_capture"
  kms_arn = "arn:aws:kms:us-west-2:abc123abc123:key/aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee"
  canary_username = "bobsdinner@gmail.com"
  canary_encrytped_password = "..."
}

output "my_canary_sensor_capture_api_keys" {
  value = "${module.canary.api_keys}"
}

output "my_canary_sensor_capture_api_gateway_endpoint" {
  value = "${module.canary.api_gateway_endpoint}"
}
```

**Note:** There are additional, optional inputs to the [aws_canary_sensor_capture](https://github.com/joshdurbin/aws_canary_sensor_capture)
 module, which you can read about on the project readme.

The single, required resource dependency not provisioned for you within the `aws_canary_sensor_capture` module is the KMS
 key. You must create this key prior to usage of the module **and** leverage it to encrypt your Canary account password. The KMS
 ARN and encrypted account password are required variable definitions for the module. If you're unsure how to do this,
 check the instructions listed in the [module readme](https://github.com/joshdurbin/aws_canary_sensor_capture).

Accessing your sensor data
------------------

Terraform will create an API Gateway instance and API keys. The usage example above declares outputs that
 print the generated API keys and endpoint. These are optional! If you do not declare these outputs you will need to
 obtain them via the AWS API, Terraform state inspection, etc...

When you Terraform apply with the [aws_canary_sensor_capture](https://github.com/joshdurbin/aws_canary_sensor_capture) module
 in use you'll get an output block declaring your endpoint and keys...

```text
Outputs:

my_canary_api_gateway_endpoint = https://abc123defg.execute-api.us-west-2.amazonaws.com/production
my_canary_api_keys = [
    ABC123ABC123ABC123ABC123ABC123ABC123ABC1
]
```

There are two types of data to access:

1. Canary device IDs
*  Canary device sensor reading data

To access your device IDs, issue an HTTP GET request to the endpoint returned as an output from the [aws_canary_sensor_capture](https://github.com/joshdurbin/aws_canary_sensor_capture) module
  with the additional trailing `/devices` resource. Ex:

`curl -H "x-api-key: ABC123ABC123ABC123ABC123ABC123ABC123ABC1" https://abc123defg.execute-api.us-west-2.amazonaws.com/production/devices`

... which will return:

```json
{
    "deviceIds": ["12345"]
}    
```

Finally, to access a devices data, issue an HTTP GET request to the same endpoint with the additional device ID resource. Ex:

`curl -H "x-api-key: ABC123ABC123ABC123ABC123ABC123ABC123ABC1" https://abc123defg.execute-api.us-west-2.amazonaws.com/production/devices/12345`

... which will return:

```json
{
  "readings": [
    {
      "time": "2017-03-22T08:17:54.088573",
      "air_quality": "0.7394286281135604",
      "humidity": "73.37273965536741",
      "temperature": "18.44126831862636"
    },
    {
      "time": "2017-03-22T09:15:51.981384",
      "air_quality": "0.7774637438918911",
      "humidity": "73.23038767477068",
      "temperature": "18.25387227927232"
    }
  ]
}
```  

Security
------------------

The AWS KMS key **you** define is central to the encryption/decryption and storage of your credentials to Canary. As the
 instructions state, you define this key and use it to encrypt your password, which you pass to the Terraform module.
 Terraform passes it, encrypted, as a parameter to the Python Lambda function.

The Lambda function uses the provided KMS key to decrypt your password and uses it with your username to fetch a bearer
 token whenever one isn't already stored in the DynamoDB metadata table. The bearer token stored in DynamoDB is
 encrypted by the Lambda function when it's stored for later use. This is done because the tokens Canary currently
 issues are relatively long-lived (at least a week).

The DynamoDB metadata table additionally stores your account's device IDs, which are **not** encrypted. The sensor
 DynamoDB data storage table contains the deviceID and timestamp as a composite key with the sensor data for that
 device at that time. None of the sensor data is stored encrypted.

The API Gateway endpoints require auth tokens and are only accessible securely over the web.

Terraform Security Note
------------------

Terraform stores a wealth of data behind each resource that may or may not be visible to you (you can inspect by
describing state resources). If you're using remote storage with Terraform, you **must enable server-side encryption**.
