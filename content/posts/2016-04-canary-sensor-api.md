+++
date = "2016-04-24T15:55:37-07:00"
title = "Canary Sensor API"
tags = [ "canary" ]
aliases = [ "/blog/canary-sensor-api/", "/blurbs/canary-sensor-api/" ]
+++

Canary doesn’t a have an official API for accessing your sensor data and otherwise control or access your Canary. I’ll have another post in a few days demoing how you might gain access to, store, and plot your sensor data long-term.

A quick tip, though, if you’d like access to your data;

1.  login to the web console and inspect the REST requests made from their Angular app
2.  note the auth token in your local storage -- be sure all subsequent requests to the endpoints have this token set as a value to the Authorization header w/ the Bearer schema
3.  you should see a call (or calls) to [https://my.canary.is/api/readings/12345 ](https://my.canary.is/api/readings/12345)- where `12345` is the ID of each canary device (if you do not, inspect the payload from [https://my.canary.is/api/locations](https://my.canary.is/api/locations) -- inspect the devices array)

Once you have your set of IDs, you can poll for data -- my tinkering seems to indicate that new sensor values are available _at most_ once per ten minutes.

Then you can do something like this...

`while true; do curl 'https://my.canary.is/api/readings/12345' -H 'Authorization: Bearer YOURTOKEN' -w "\n"; sleep 600; done`

... which yields a series JSON payload of:

    {"air_quality":0.9912101429827669,"temperature":20.410450630999627,"humidity":58.39233714976209}
    {"air_quality":0.9912101429827669,"temperature":20.410450630999627,"humidity":58.39233714976209}
    {"air_quality":0.9912101429827669,"temperature":20.410450630999627,"humidity":58.39233714976209}
