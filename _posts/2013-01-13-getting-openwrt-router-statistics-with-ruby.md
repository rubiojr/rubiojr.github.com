---
layout: post
title: "Getting OpenWRT router stats with ruby"
description: ""
tagline: ""
category: hack 
tags: [ruby, openwrt, luci, excon]
---

The following code has been tested with **OpenWrt Backfire 10.03.1**.

[OpenWRT](http://openwrt.org) routers are pretty awesome, and one of the things 
you can do is retrieve router statistics from 
[LuCI](http://wiki.openwrt.org/doc/techref/luci) using 
Ruby and an HTTP client library like [Excon](http://github.com/geemus/excon).

## Authentication and router status

Authenticate, get router status and the required auth data.

{% highlight ruby %} 
require 'excon'
require 'pp'
require 'yajl'

# Auth parameters
username = 'root'
password = 'secret'
host = '192.168.1.1'
# Will parse returned JSON content
parser = Yajl::Parser.new

#
# Authenticate
#
# Also returns router's dashboard status information, i.e. 
# the info you see in the router 'Status -> Overview' dashboard
#
# Returned keys
# ["conncount",
#  "leases",
#  "wan",
#  "membuffers",
#  "connmax",
#  "memfree",
#  "uptime",
#  "wifinets",
#  "memtotal",
#  "localtime",
#  "memcached",
#  "loadavg"
# ]

r = Excon.post "http://#{host}/cgi-bin/luci?status=1", 
               :body => "username=#{username}&password=#{password}",
               :headers => { 'Content-Type' => 'application/x-www-form-urlencoded' }

# Do something with the data
# pp parser.parse(r.body)
{% endhighlight %}  

The request returns auth info we will use in further requests:

{% highlight ruby %}
# Get sysauth cookie and session path, will use it to send
# other requests
sysauth, path = r.headers['Set-Cookie'].split
path = path.split('=')[1..-1].join('=')
{% endhighlight %}

## Available statistics

Pretty much all the graphs you can see if you got to **'Overview -> Realtime Graphs'**
in the OpenWRT web interface.

### Network traffic 

Data used by the graphs available in **'Status -> Realtime Graphs -> Traffic'** 

{% highlight ruby %}
# Network traffic
#
# Returned data format: time, rx bytes, rx packets, tx bytes, tx packets
#
# Data sample:
# [
#   [1358095466, 1482617413, 101161697, 1420821587, 62541131],
#   [1358095467, 1482626173, 101161703, 1420830431, 62541137],
#   ...
# ]
r = Excon.get "http://192.168.66.1/#{path}/admin/status/realtime/bandwidth_status/eth0.1", 
              :headers => { 'Cookie' => sysauth }
# Do something with the data
pp parser.parse(r.body)
{% endhighlight %} 

### System load

Data used by the graphs available in **'Status -> Realtime Graphs -> Load'** 

{% highlight ruby %}
#
# System load
#
# Returned data format: time, 1min load, 5min load, 15min load
#
# Data sample:
# [
#   [1358098335, 36, 40, 56],
#   [1358098336, 36, 40, 56],
#   ...
# ]
r = Excon.get "http://#{host}/#{path}/admin/status/realtime/load_status", 
              :headers => { 'Cookie' => sysauth }
# Do something with the data
pp parser.parse(r.body)
{% endhighlight %}

### Connections status

Data used by the graphs available in **'Status -> Realtime Graphs -> Connections'** 

{% highlight ruby %}
# 
# Connections Status
#
# Returned data format:
#
# statistics: TIME, # udp connections, # tcp connections, # other connections
# connections: returns a Hash, self descriptive.
#
# TCP and UDP number of connections is always 0, looks like a minor bug in this
# version of OpenWRT (1.0)
# Data sample:
#
# {
#   "connections" => [
#      {"bytes"=>"746295",
#       "src"=>"192.168.1.4",
#       "sport"=>"36865",
#       "layer4"=>"tcp",
#       "dst"=>"1.2.3.4",
#       "dport"=>"333",
#       "layer3"=>"ipv4",
#       "packets"=>"10674"},
#      ...
#   ],
#   "statistics" => [
#     [1358097511, 0, 0, 48],
#     ...
#    ]
# }
#    
#
r = Excon.get "http://#{host}/#{path}/admin/status/realtime/connections_status", 
              :headers => { 'Cookie' => sysauth }
# connections_status does not return valid JSON apparently, fix it
r.body.gsub! /(connections|statistics)/, '"\\1"'
# Do something with the data
pp parser.parse r.body
{% endhighlight %}

### Wireless interface status

Data used by the graphs available in **'Status -> Realtime Graphs -> Wireless'** 

{% highlight ruby %}
#
# Wireless interface wl0 status
# 
# Returned data format: time, rate, rssi (signal strength), noise
#
# Data sample:
# [
#  [1358099217, 61650, 213, 167],
#  [1358099218, 61650, 213, 166],
#  ...
# ]
#
r = Excon.get "http://#{host}/#{path}/admin/status/realtime/wireless_status/wl0", 
              :headers => { 'Cookie' => sysauth }
# Do something with the data
pp parser.parse r.body
{% endhighlight %}

## Source code used in this post

Available here: [https://gist.github.com/4525640](https://gist.github.com/4525640)
