---
layout: post
title: "Getting OpenWRT router stats with ruby"
description: ""
tagline: ""
category: hack 
tags: [ruby, openwrt, luci]
---

## Authentication and router status

Authenticate, get router status and requequired auth data.

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
# Also returns dashboard router status information, i.e. 
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

# pp parser.parse(r.body)
{% endhighlight %}  

The request returns some useful info we will use in further requests

{% highlight ruby %}
# Get sysauth cookie and session path, will use it to send
# other requests
sysauth, path = r.headers['Set-Cookie'].split
path = path.split('=')[1..-1].join('=')
{% endhighlight %}

## Available info

### Network interface stats

{% highlight ruby %}
# Network interface stats
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
