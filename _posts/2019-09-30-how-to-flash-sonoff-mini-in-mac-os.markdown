---
layout: post
title: How to flash Sonoff Mini in MacOS
date: 2019-09-30 08:00:00 +0200
description: How to flash Sonoff Mini in MacOS without Sonoff DIY tool and without soldering
img: sonoff-mini-flash.jpg
tags: [sonoff, sonoff mini, tasmota, MacOS]
---

How to user REST API to flash Sonoff Mini with Tasmota.

## Requirements
* chceck if eWelink firmware in Sonoff Mini is updated to at least version 3.1. If not, connect Sonoff Mini to Wifi using eWelink app and then update firmware using same app
* physicaly open the device cover and connect DIY jumper
* create Wifi with SSID `sonoffDiy` and password `20170618sn` (both are case sensitive!)
* connect Sonoff Mini to this network
* run HTTP server on sonoffDiy network (idealy nginx) - HTTP server must support the Range request header

## Find the Sonoff Mini on network
Look in administration of you WiFi router to find your Sonoff Mini IP Address `deviceIP` . Using utility `dns-sd` find Zeroconf detals, specifically `deviceID` - look for `Instance Name`.
{% highlight Bash %}
dns-sd -B _ewelink._tcp
{% endhighlight %}

It will look like `eWeLink_100000140e`. 100000140e is your `deviceID`.

## Download Tasmota

On HTTP server download Tasmota to root folder of HTTP server:
{% highlight Bash %}
sudo wget http://thehackbox.org/tasmota/release/sonoff.bin -P /var/www/html
{% endhighlight %}

And count SHA256 `<SHA256>` of firmware binary file `sonoff.bin`:
{% highlight Bash %}
shasum -a 256 sonoff-basic.bin
{% endhighlight %}

It will look like `c01b39c7a35ccc3b081a3e83d2c71fa9a767ebfeb45c69f08e17dfe3ef375a7b`.

## Update firmware using OTA

* check if your Sonoff Mini is connected:
{% highlight Bash %}
curl http://<deviceIP>:8081/zeroconf/signal_strength -XPOST --data '{"deviceid":"<deviceID>","data":{} }'
{% endhighlight %}

if response looks like this, everything is ok:

{% highlight Json %}
{ 
    "seq": 2, 
    "error": 0, 
    "data": { 
      "signalStrength": -67 
    } 
 }
{% endhighlight %}

* Check if `otaUnlock` is true:
{% highlight Bash %}
curl http://<deviceIP>:8081/zeroconf/info -XPOST --data '{"deviceid":"<deviceID>","data":{} }'
{% endhighlight %}
Response:
{% highlight JSON %}
{ 
	"seq": 2, 
	"error": 0,
	"data": {
	"switch": "on",
	"startup": "stay",
	"pulse": "on",
	"pulseWidth": 2000,
	"ssid": "eWeLink",
	"otaUnlock": true
	}
 }
{% endhighlight %}

If `otaUnlock` is false just use:
{% highlight Bash %}
curl http://<deviceIP>:8081/zeroconf/ota_unlock -XPOST --data '{"deviceid":"<deviceID>","data":{} }'
{% endhighlight %}

* Update firmware using OTA:
{% highlight Bash %}
curl http://<deviceIP>:8081/zeroconf/ota_flash -XPOST --data '{"deviceid":"<deviceID>","data":{"downloadUrl": "http://<HTTPserverIP>/sonoff.bin", "sha256sum": "<SHA256>"} }'
{% endhighlight %}

And wait abou 30 seconds.

Once the firmware is successfully uploaded and Sonoff Mini restarts, look for sonoff-xxxx WiFi. You should know the rest.

## Links
[API Documentation for Sonoff DIY](https://github.com/itead/Sonoff_Devices_DIY_Tools/blob/master/SONOFF%20DIY%20MODE%20Protocol%20Doc%20v1.4.md)