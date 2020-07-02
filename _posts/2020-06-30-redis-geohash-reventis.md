---
layout: post
title: "Reventis&trade;: A Better Redis GeoHash"
description: Redis Module for geospatial-temporal event indexing
author: starkdg
date: 2020-06-30
use_math: true
tags: [geohash, spatial-temporal, event, indexing, longitude, latitude]
---

![reventis](/resources/post_4/reventis.png)

Redis, an in-memory data store cache, is a popular database that can
serve many uses.  One of the unique features is the `geohash` set of
commands.  These allow you to add locations to a sorted set, find the distance
between any two locations, or even get a list of all locations that
fall within a given radius of a location. A typical example might
go as follows:
<!--more-->

```
127.0.0.1:6379> geoadd myplaces -72.677169 41.761841 hartford-ct
(integer) 1
127.0.0.1:6379> geoadd myplaces -72.620889 41.764183 easthartford-ct
(integer) 1
127.0.0.1:6379> geoadd myplaces -72.667689 41.702897 wethersfield-ct
(integer) 1
127.0.0.1:6379> geoadd myplaces -72.746997 41.749971 westhartford-ct
(integer) 1
127.0.0.1:6379> geoadd myplaces -72.339749 41.564189 colchester-ct
(integer) 1
127.0.0.1:6379> geoadd myplaces -72.234068 41.782345 mansfield-ct
(integer) 1
```

To get all distances within 5 miles of hartford-ct, just do:

```
127.0.0.1:6379> georadiusbymember myplaces hartford-ct 5 mi
1) "westhartford-ct"
2) "wethersfield-ct"
3) "hartford-ct"
4) "easthartford-ct"
```

To get the distance  between colchester-ct and mansfield-ct:

```
127.0.0.1:6379> geodist myplaces colchester-ct mansfield-ct mi
"16.0341"
```

To lookup individual location, use geopos:

```
127.0.0.1:6379> geopos myplaces colchester-ct mansfield-ct
1) 1) "-72.33974844217300415"
   2) "41.56418901188302328"
2) 1) "-72.23406940698623657"
   2) "41.78234485790384412"
```

Indeed, the Redis geohash can be extremely useful tool for web applications
offering location based services.  

However, that's about all it can do. It lacks many other features
of a full spatial-temporal or object tracking index that would
no doubt be highly useful. 

## Reventis

[Reventis](https://github.com/starkdg/reventis) - a portmanteau of the
words, Redis and Events - is a Redis Module that introduces a native
data structure capable of indexing events, or point locations in time
and space.  Events can be inserted into the data structure
by its geo-spatial coordinates along with beginning and end timestamps.
Then all events within a given geographical area and timespan can be queried.
The sequence mirrors the geohash commands and goes something like this:

```
127.0.0.1:6379> reventis.insert myplaces -72.569023 41.839269 06-01-2020 10:00 06-01-2020 12:00 "southwindsor-ct"
(integer) 942729159590019073
127.0.0.1:6379> reventis.insert myplaces -72.534727 41.770793 06-01-2020 11:00 06-01-2020 11:35 "manchester-ct"
(integer) 942950844413706242
127.0.0.1:6379> reventis.insert myplaces -72.565348 41.907065 06-01-2020 11:15 06-01-2020 11:20 "eastwindsor-ct"
(integer) 943188820988133379
127.0.0.1:6379> reventis.insert myplaces -72.310697 41.958829 06-01-2020 10:30 06-01-2020 11:05 "stafford-ct"
(integer) 943534029388972036
127.0.0.1:6379> reventis.insert myplaces -72.480640 41.806255 06-02-2020 10:00 06-02-2020 11:00 "buckley-manchester"
(integer) 943880954251706373
127.0.0.1:6379> reventis.querybyradius myplaces -72.594856 41.814437 10 mi 06-01-2020 8:00 06-01-2020 16:00
1) 1) "manchester-ct"
   2) (integer) 942950844413706242
   3) "-72.534727000000004"
   4) "41.770792999999998"
   5) "06-01-2020 11:00:00\x00"
   6) "06-01-2020 11:35:00\x00"
2) 1) "southwindsor-ct"
   2) (integer) 942729159590019073
   3) "-72.569023000000001"
   4) "41.839269000000002"
   5) "06-01-2020 10:00:00\x00"
   6) "06-01-2020 12:00:00\x00"
3) 1) "eastwindsor-ct"
   2) (integer) 943188820988133379
   3) "-72.565348"
   4) "41.907065000000003"
   5) "06-01-2020 11:15:00\x00"
   6) "06-01-2020 11:20:00\x00"
```

As you can see, the query retrieves all the events inserted within
a 10 mile radius of South Windsor, Ct for June 1, 2020.

### Categories

What if you want to retrieve only certain kinds of events?

Reventis affords the ability to assign integer categories from 1 to 64 to
each event. Simply invoke the `addcategory` command with the key, the assigned
id number for that event, followed by a list of categories you wish to assign.

```
reventis.addcategory myplaces 942729159590019073 10 20 30
reventis.addcategory myplaces 942950844413706242 10 20
reventis.addcategory myplaces 943188820988133379 55 60
```

Then you can query in the same way as above with the desired categories appended onto the end.

```
127.0.0.1:6379> reventis.querybyradius myplaces -72.594856 41.814437 10 mi 06-01-2020 8:00 06-01-2020 16:00 10 20
1) 1) "manchester-ct"
   2) (integer) 942950844413706242
   3) "-72.534727000000004"
   4) "41.770792999999998"
   5) "06-01-2020 11:00:00\x00"
   6) "06-01-2020 11:35:00\x00"
2) 1) "southwindsor-ct"
   2) (integer) 942729159590019073
   3) "-72.569023000000001"
   4) "41.839269000000002"
   5) "06-01-2020 10:00:00\x00"
   6) "06-01-2020 12:00:00\x00"
```

## Object Tracing 

By attaching an object ID number to an event, Reventis provides the ability to
track objects. With the `update` command, a new event is inserted to the data
structure with a given object ID provided.  In this way, a chain of events can
be tracked.

```
127.0.0.1:6379> reventis.update myplaces -72.254756 41.807999 06-01-2020 11:00 100 "class 1"
(integer) 1270559233548550150
127.0.0.1:6379> reventis.update myplaces -72.242718 41.804041 06-01-2020 12:00 100 "lunch"
(integer) 1270837990485983239
127.0.0.1:6379> reventis.update myplaces -72.452282 41.835916 06-01-2020 15:00 100 "work"
(integer) 1271442546606538760
```

## How Well Does it Scale?

Since Reventis is able to store each event/object in a balanced search tree, query response
time complexity is within logarithmic complexity.  In the following graph, four different
query sizes are plotted over increasing index size.  The x-axis is for N, the total number
of indexed events; the y-axis is response time in milliseconds.  As you can see, most queries
stabilize to around 1ms for ever increasing N.  Query #4 does not stabilize, because it is
literally the size of the state of Texas for a year time duration.  Also, the test adds
uniformly distributed random events, so larger queries will have more results to retrieve
the more random events are inserted.  

![Query Results](/resources/post_4/results.png)


## Summary

Reventis introduces a robust set of commands for the management of spatio-temporal point
data, and can be of invaluable service to a number of web application services.  


More information is available on the [github repos](https://github.com/starkdg/reventis).





