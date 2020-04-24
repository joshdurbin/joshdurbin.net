+++
date = "2016-06-28T17:57:18-07:00"
title = "AWS IoT w/ RPI and Sense HAT (Part 1)"
tags = [ "aws", "python", "internet of things"]
aliases = [ "/blog/aws-iot-w/-rpi-and-sense-hat-part-1/", "/blurbs/aws-iot-w/-rpi-and-sense-hat-part-1/" ]
+++

This is the first part of a series of posts aimed at the integration of the [AWS Internet of Things (IoT)](https://aws.amazon.com/iot/) service offering and the developer friendly [Raspberry PI](https://www.raspberrypi.org/) platform. These few posts will make use of the Raspberry PI’s officially supported [HATs](https://www.raspberrypi.org/blog/introducing-raspberry-pi-hats/); the [Sense HAT](https://www.raspberrypi.org/products/sense-hat/). The Sense HAT provides an 8x8 LED grid, accelerator, mag, gyro, temp, humidity, and pressure sensors.

[Raspbian](https://www.raspberrypi.org/downloads/raspbian/), the officially supported, Debian-based Linux distro that commonly runs on the Raspberry PI, has the necessary Python SDKs required to use the Sense HAT.

The goal for this post is to create the necessary AWS IoT resources and publish / subscribe to the [MQTT](http://mqtt.org/) topic made available by AWS IoT. At the end of this post you’ll be transmitting JSON payloads with sensor data to AWS IoT. Simultaneously on the same device, you’ll consume said data and write the temperature on the 8x8 LED grid.

To get started; you’ll need the following:

1.  A Raspberry PI (I’m using a RPI 3, but you could use a 2 or a B+)
2.  A Sense HAT module installed on your PI
3.  Raspbian
4.  An AWS account

Let’s get started!

## AWS Setup

We’re going to kick it off with our AWS setup. First off, we need to create a “thing” and the related resources and relationships so that our “thing” can work.

Just like all other AWS offerings, there are two ways to do this, via the CLI or via the web interface. I’ll walk through the CLI and provide links to the supporting AWS documentation for Web/GUI shots if you prefer to do things that way...

**Note:** I assume you have the AWS CLI installed and configured; if not please do so.

1.  First we need to create our “thing”, to do so, issue the following command (web instructions [here](http://docs.aws.amazon.com/iot/latest/developerguide/create-device.html)): `aws iot create-thing --thing-name=rpisensehat`

2.  Next we’ll create a policy. **Note:** The policy I’m providing in the command block is a completely open policy. Issue the following command (web instructions [here](http://docs.aws.amazon.com/iot/latest/developerguide/create-iot-policy.html)): `aws iot create-policy --policy-name rpisensehatpolicy --policy-document '{ "Version": "2012-10-17", "Statement": [{ "Effect": "Allow", "Action":["iot:*"], "Resource": ["*"] }] }'`

3.  Next we’ll create our certificates required for TLS communication with AWS IoT. Issue the following command (web instructions [here](http://docs.aws.amazon.com/iot/latest/developerguide/create-device-certificate.html)): `aws iot create-keys-and-certificate --set-as-active --certificate-pem-outfile certificate.pem --private-key-outfile privateKey.pem | grep certificateArn | sed -e "s/^.*arn/arn/" -e "s/\".*$//" > certificateArn.arn`

4.  Next, we need to attach the aforementioned certificate to the policy. Issue the following commands (web instructions [here](http://docs.aws.amazon.com/iot/latest/developerguide/attach-policy-to-certificate.html)): `aws iot attach-principal-policy --policy-name rpisensehatpolicy --principal $(<certificateArn.arn)`

5.  Finally, similarly, we need to attach the aforementioned certificate to the "thing" itself. Issue the following commands: `aws iot attach-thing-principal --principal $(<certificateArn.arn) --thing-name rpisensehat`

You'll need to download the AWS Root certificate from [here](https://iotworkshop.s3.amazonaws.com/rootCA.pem).

... and with that we're done with our AWS IoT resource setup.

## Libraries

Next let's install the library responsible for interacting with our RPI and AWS IoT; [Paho MQTT](http://www.eclipse.org/paho/clients/python/). Execute the command below to install Paho using the Python package installer, `pip`. Issue the following command: `sudo pip install paho-mqtt`

## Script Walkthrough

Next let's take a look at the scripts:

* The [`awsiotpub.py`](https://github.com/joshdurbin/aws-iot-rpi-sense-hat/blob/master/awsiotpub.py) is responsible for connecting to the AWS IoT topic, reading sensor data, generating JSON, and publishing the JSON to the topic. This script sends data to the MQTT topic `environmentData` every 10 seconds.
* The [`awsiotsub.py`](https://github.com/joshdurbin/aws-iot-rpi-sense-hat/blob/master/awsiotsub.py) is responsible for connecting to the AWS IoT topic (`environmentData`, listening, consuming messages and displaying the temperature within the message on the 8x8 LED grid.

## Download scripts and certs

Next we'll need to download the scripts and certificates to our raspberry PI. Use `curl -0 $url` or `wget` to fetch the scripts. You *probably* ran the AWS certificate generation scripts via the AWS CLI on your Windows / MacOS device. Use `scp` or some other file transport mechanism to move the certificates onto the raspberry PI.

## Modification of scripts

Finally, update the certificate file references in the `awsiotpub.py` and `awsiotsub.py` scripts. The file paths in the scripts are currently relative and can be absolute if need be. Change accordingly.

## Execution

This is the easy part -- make sure you have permissions to access the GPIO pins, required for the Sense HAT (see gotchas #1).

First, I recommend you run the subscription script -- execute `python awsiotsub.py`.

Second, run the publisher `python awsiotpub.py` and wait a few seconds. The publisher will display its payload, ex: `{"device": {"cpuTemperature": "43.5"}, "environment": {"pressure": 1010.93310546875, "temperature": {"basedOnPressure": 28.395832061767578, "basedOnHumidity": 29.6092472076416}, "humidity": 39.20086669921875}}`

Lastly, I recommend you connect to the `environmentData` topic via the [AWS Iot MQTT web client](http://docs.aws.amazon.com/iot/latest/developerguide/view-mqtt-messages.html). It allows you to both subscribe and publish data to the topic.

... and with that we've established reading sensor data and round-trip data transmission. The next post will focus on wiring up Lambdas to react to the data and send other more meaningful messages to the consumer for display on the Sense HAT's 8x8 LED grid.

Stay tuned!

## Notes and gotchas

1. In order to use these scripts on your device you'll need to ensure you have the proper permissions to access the Sense HAT. I believe your user should be in the `gpio` group, but I haven't verified this. If you *still* have the `pi` user that works, comes with default, root-like permissions.
2. Make sure you're operating within the same AWS region at all times, otherwise things won't work.
3. Client names must always be unique (note the script's client IDs)
