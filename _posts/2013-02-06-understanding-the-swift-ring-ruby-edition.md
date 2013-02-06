---
layout: post
title: "Understanding the Swift Ring, Ruby Edition"
description: ""
category: hack
tags: [openstack, swift, ruby]
---
{% include JB/setup %}

## Intro

In my crusade to understand the way Swift's ring works, I've 
translated some of the python code from swift.common.ring and
swift.common.utils to ruby.

My goal is simple: understand the decisions Swift's proxy makes
to upload a file 'bar.txt' to a container named 'foo', using my
Swift account. That is:

* Which storage nodes will store the file?
* What happens (roughly) when you use the following 
  Swift's client commands?

```
    swift -A https://identity.openstack.mycloud/v2.0/ \
          -V 2 -U admin@mycloud.net -K secret \
          post foo

    swift -A https://identity.openstack.mycloud/v2.0/ \
          -V 2 -U admin@mycloud.net -K secret \
          upload foo bar.txt
```

Swift creates **partitions** (directories) in every storage node to
store the data (accounts, containers, objects) and when you upload
**objects** (files) and/or create **containers**, the proxy needs to know
which server (or servers, since we're using 3 replicas) to send the
files and where to store the data (the target partition).

This is the full path to an object, stored in a Swift storage node:

```
/srv/node/1/objects/165875/d00/a1fce4a02ddf6cca933a314c9a169d00/1360155539.49394.data
```

**165875:** the partition number

**d00:** the last three hex digits of the account hash

**a1fce4a02ddf6cca933a314c9a169d00:** the account hash

**1360187777.49394:** the object name in disc (timestamp, seconds since 1970-01-01 00:00:00 UTC)

All this information is hardcoded in Swift's ring files, so let's 
read and understand it.

## The code

Swift's ring format has changed recently (Folsom, Swift 1.7.0 IIRC)
and now we don't need Python pickle to deserialize the ring anymore.

See https://github.com/openstack/swift/commit/f8ce43a21891ae2cc00d0770895b556eea9c7845

### De-serializing the ring file

Reading the ring file:

```ruby
require 'zlib'
require 'json'
require 'pp'

# Ruby implementation of deserialize_v1
# from swift.common.ring.RingData
#
# Reads a ring file (account|container|object).ring.gz
# and returns a Hash like:
#
#     { "devs" => [
#         {
#           "zone"   => 1, 
#           "weight" => 1.0, 
#           "ip"     => "10.130.1.179", 
#           "port"   => 6002, 
#           "meta"   => "",
#           "device" => "1", 
#           "id"     => 0
#          },
#          ...
#        ],
#        "part_shift"          => 14,
#        "replica_count"       =>3,
#        "replica2part2dev_id" =>[]
#     }
#
# @see https://github.com/openstack/swift/blob/master/swift/common/ring/ring.py
# @param [String] ring file path (i.e. account.ring.gz)
# @return [Hash] the ring hash
def deserialize_v1(file)
  ring = Zlib::GzipReader.open(file)

  # read the magic number
  #
  # returns R1NG in swift >= 1.6.0
  magic = ring.read(4)

  # Read the ring version
  #
  # read unsigned short (2 bytes), big endian
  # using String.unpack
  version = ring.read(2).unpack('S>').first

  # read the ring json structure
  #
  # { "devs" => [
  #     {
  #       "zone"   => 1, 
  #       "weight" => 1.0, 
  #       "ip"     => "10.130.1.179", 
  #       "port"   => 6002, 
  #       "meta"   => "",
  #       "device" => "1", 
  #       "id"     => 0
  #      },
  #      ...
  #    ],
  #    "part_shift"          => 14,
  #    "replica_count"       =>3,
  #    "replica2part2dev_id" =>[]
  # }
  #
  # read unsigned int (4 bytes), big endian
  json_len = ring.read(4).unpack('L>').first
  ring_hash = JSON.parse ring.read(json_len)
  ring_hash['replica2part2dev_id'] = []

  # calculate ring's partition count
  #
  # Left shift operator, same as 2^part_power
  #
  # 32 is the lenght in bytes of an MD5 hash used
  # by Swift
  partition_count = 1 << (32 - ring_hash['part_shift'])

  replica_count = ring_hash['replica_count']
  replica_count.times do |i|
    ring_hash['replica2part2dev_id'] << ring.read(2*partition_count).unpack('S*')
  end
  ring_hash
end
```

### Calculating the partition number

Given some data from the deserialized ring (part_shift), the 
swift_hash_path_suffix from /etc/swift/swift.conf and the full path
to an object (i.e. /account/container/object.txt), return the
partition the object belongs to.

```ruby
#
# Given a swift_hash_path_suffix (from /etc/swift/swift.conf),
# the part_shift (from the ring structure), an account, container
# and object, return the partition number the account/container/object
# belongs to
#
# @param [String] swift_hash_path_suffix from /etc/swift/swift.conf
# @param [Integer] parth_shift
# @param [String] account path
# @param [String] container path
# @param [String] object path
# @return [Integer] the partition number
#
def get_part(swift_hash_path_suffix, part_shift, account, container = nil, object = nil)
  require 'digest/md5'
  path = [account, container, object].compact
  key = Digest::MD5.digest("/" + path.join('/') + swift_hash_path_suffix )
  key.unpack('L>').first >> part_shift
end
```

### Getting the list of primary nodes

Given the ring hash from deserialize_v1 and the partition
number, get the list of nodes responsible for that partition.


```ruby
# Get the nodes responsible for a given partition
#
# @param [Hash] the ring Hash returned by deserialize_v1
# @param [Ingeter] the partition number
# @return [Array] the list of nodes
def get_part_nodes(ring_hash, part)
  seen_ids = []
  nodes    = []
  ring_hash['replica2part2dev_id'].each do |r|
    if !seen_ids.include?(r[part])
      nodes << ring_hash['devs'][r[part]]
      seen_ids << r[part]
    end
  end
  nodes
end
```

## A real world example

Let's say we want to upload an object 'bar.txt' to the container 'foo':

1. Create the container

   ```
   swift -A https://identity.openstack.cloud/v2.0/ \
         -V 2 -U admin@mycloud.net -K secret \
         post foo 
   ```

2. Upload the bar.txt file to foo

   ```
   swift -A https://identity.openstack.cloud/v2.0/ \
         -V 2 -U admin@mycloud.net -K secret \
         upload foo bar.txt
   ```

And now we wan't to know in which nodes that 'bar.txt' file
is located. We'll need the account ID, so let's print it:

1. Get my account ID


   ```
   swift -A https://identity.openstack.cloud/v2.0/ \
         -V 2 -U admin@mycloud.net -K secret \
         stat | grep Account
   ```

This will print something like:

```
Account: AUTH_32f7e581cdc44cb8a079f1b7632a9bae
```

Let's use **get\_part** and **get\_part\_nodes** methods to retrieve
the info:

```ruby
# read the ring from account.ring.gz
ring = deserialize_v1('account.ring.gz')


# Calculate the partition number for /AUTH_25f7e581cdc44cb8a079f1b7632a9bae/foo/bar.txt
part = get_part 'aqjwoeeMeeMetaeLuy',                    # swift_hash_path_suffix
                ring['part_shift'],                      # part_shift
                'AUTH_32f7e581cdc44cb8a079f1b7632a9bae', # account ID
                'foo',                                   # container
                'bar.txt'                                # object

# The partition number is 165875 in my case

# Get the nodes where the partition will be located
nodes = get_part_nodes(ring, part)

# the returned nodes will be an array with 3 nodes, something like:
# [{"zone"=>1, "weight"=>1.0, "ip"=>"1.2.3.4", "port"=>6002, "meta"=>"", "device"=>"sdb", "id"=>17}, 
#  {"zone"=>2, "weight"=>1.0, "ip"=>"1.2.3.5", "port"=>6002, "meta"=>"", "device"=>"sda", "id"=>5},
#  {"zone"=>3, "weight"=>1.0, "ip"=>"1.2.3.6", "port"=>6002, "meta"=>"", "device"=>"sdc", "id"=>1}]
```

**UNFINISHED:** this post needs to be polished and finished.

## Related

**Ring Overview**

http://docs.openstack.org/developer/swift/overview_ring.html

**How the ring works in OpenStack Swift**

http://swiftstack.com/blog/2012/11/21/how-the-ring-works-in-openstack-swift/


