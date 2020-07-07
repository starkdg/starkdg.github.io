---
layout: post
title: "A Better Way to Redis GeoHash"
tagline: "Reventis: A Redis Module for Spatio-temporal Event Indexing"
description: "Redis Module for geospatial-temporal event indexing"
author: starkdg
date: 2020-06-30
use_math: true
tags: [geohash, spatial-temporal, event, indexing, geospatial, reventis, space-filling-curves, hilbert-curves]
---


![reventis](/resources/post_4/reventis.png)


Redis, an in-memory data store cache, is a popular database for many
applications.  One of its many unique features is the `geohash` set of
commands.  These commands allow you to add locations to a sorted set,
find the distance between any two locations, or even get a list of
all locations that fall within a given radius of a particular location.
A typical usage might run as follows:
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

Then, to get all distances within 5 miles of hartford-ct, just do:

```
127.0.0.1:6379> georadiusbymember myplaces hartford-ct 5 mi
1) "westhartford-ct"
2) "wethersfield-ct"
3) "hartford-ct"
4) "easthartford-ct"
```

To get the distance in miles between colchester-ct and mansfield-ct:

```
127.0.0.1:6379> geodist myplaces colchester-ct mansfield-ct mi
"16.0341"
```

To lookup individual locations' coordinates:

```
127.0.0.1:6379> geopos myplaces colchester-ct mansfield-ct
1) 1) "-72.33974844217300415"
   2) "41.56418901188302328"
2) 1) "-72.23406940698623657"
   2) "41.78234485790384412"
```

Indeed, the Redis geohash can be extremely useful tool for web applications
offering location-based services.  

However, that's about all it can do. It lacks many other features
of a full spatial-temporal or object tracking index that would
no doubt be highly useful. 


## Reventis


[Reventis](https://github.com/starkdg/reventis) - a portmanteau of the
words, Redis and Events - is a Redis Module that introduces a native
data structure capable of indexing events or point locations in time
and space.  Events can be inserted into the data structure
by its geo-spatial coordinates along with beginning and ending timestamps.
Then all events within a given geographical area and timespan can be queried.
The sequence mirrors the geohash commands.  Here is an example:

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
```

The `insert` command is followed by a key string, the longitude, latitude, beginning date-time,
end-date-time, and finally, a short descriptive string of the event.  The command returns an integer
identifier for that new event. 

The `queryradius` command retrieves all the events inserted within a 10 mile radius of South
Windsor, Ct for June 1, 2020.

```
127.0.0.1:6379> reventis.queryradius myplaces -72.594856 41.814437 10 mi 06-01-2020 8:00 06-01-2020 16:00
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


### What if you want to retrieve only certain kinds of events?

With Reventis, inorder to further filter events, you can assign
integer categories 1 to 64 to each event. Multiple categories per event are possible.
Simply invoke the `addcategory` command with the key, the assigned
id number for that event, followed by a list of categories you wish to assign.

```
reventis.addcategory myplaces 942729159590019073 10 20 30
reventis.addcategory myplaces 942950844413706242 10 20
reventis.addcategory myplaces 943188820988133379 55 60
```

Then you can query in the same way as above with the desired categories
appended onto the end of the command.  The following queries the myplaces
key for categories 10 and 20.

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


By attaching an object integer identifier to an event, Reventis provides the ability to
track objects. With the `update` command, a new event is inserted to the data
structure with a given object ID provided.  In this way, a chain of events can
be tracked with a common object identifier.
The update command is followed by a key string, longitude, latitude, timestamp,
an object id, and finally, a descriptive string for the update.  For example, 

```
127.0.0.1:6379> reventis.update mytracks -72.514046 41.823773 06-01-2020 8:00 100 "avery st., SW, CT"
(integer) 2692244187721629697
127.0.0.1:6379> reventis.update mytracks -72.679270 41.754280 06-01-2020 9:00 100 "seymour st. Htfd, CT"
(integer) 2692649354707730434
127.0.0.1:6379> reventis.update mytracks -72.673778 41.773129 06-01-2020 10:30 100 "Rensselaers, Htfd, CT"
(integer) 2692919245505495043
127.0.0.1:6379> reventis.update mytracks -72.672338 41.762721 06-01-2020 11:15 100 "Uconn Htfd, CT"
(integer) 2693312935547633668
127.0.0.1:6379> reventis.update mytracks -72.760321 41.762886 06-01-2020 13:00 100 "Front St, Htfd, CT"
(integer) 2693757132817039365
127.0.0.1:6379> reventis.update mytracks -72.554229 41.825927 06-01-2020 17:00 100 "Wapping, SW, CT"
(integer) 2694272864704200710
127.0.0.1:6379> reventis.update mytracks -72.604337 41.695416 06-01-2020 9:30 200 "Sherman, Rd, Glastonbury, CT"
(integer) 2694635424033406983
127.0.0.1:6379> reventis.update mytracks -72.689379 41.748207 06-01-2020 11:00 200 "Trinity College, Htfd.,CT"
(integer) 2694958883051143176
127.0.0.1:6379> reventis.update mytracks -72.668197 41.762560 06-01-2020 12:00 200 "Convention Center, Htfd.,CT"
(integer) 2695270279792492553
127.0.0.1:6379> reventis.update mytracks -72.671490 41.762741 06-01-2020 14:30 200 "Bears Smokehouse BBQ, Htfd, CT"
(integer) 2695768111295037450
127.0.0.1:6379> reventis.update mytracks -72.463723 41.638888 06-01-2020 19:00 200 "Denler, Dr - Marlborough, CT"
(integer) 2696103043806396427
127.0.0.1:6379> reventis.update mytracks -72.568387 41.685582 06-02-2020 11:00 200 "Glastonbury, CT"
(integer) 2696432004334157836```
```

Now we can query for all updates that fall within a 5 mile radius of Hartford, Ct.

```
127.0.0.1:6379> reventis.queryobjradius mytracks -72.675968 41.763890 5 mi 06-01-2020 06:00 06-01-2020 23:00
1) 1) "Front St, Htfd, CT"
   2) (integer) 2693757132817039365
   3) (integer) 100
   4) "-72.760321000000005"
   5) "41.762886000000002"
   6) "06-01-2020 13:00:00\x00"
2) 1) "seymour st. Htfd, CT"
   2) (integer) 2692649354707730434
   3) (integer) 100
   4) "-72.679270000000002"
   5) "41.754280000000001"
   6) "06-01-2020 09:00:00\x00"
3) 1) "Convention Center, Htfd.,CT"
   2) (integer) 2695270279792492553
   3) (integer) 200
   4) "-72.668197000000006"
   5) "41.762560000000001"
   6) "06-01-2020 12:00:00\x00"
4) 1) "Rensselaers, Htfd, CT"
   2) (integer) 2692919245505495043
   3) (integer) 100
   4) "-72.673777999999999"
   5) "41.773128999999997"
   6) "06-01-2020 10:30:00\x00"
5) 1) "Uconn Htfd, CT"
   2) (integer) 2693312935547633668
   3) (integer) 100
   4) "-72.672337999999996"
   5) "41.762720999999999"
   6) "06-01-2020 11:15:00\x00"
6) 1) "Trinity College, Htfd.,CT"
   2) (integer) 2694958883051143176
   3) (integer) 200
   4) "-72.689379000000002"
   5) "41.748207000000001"
   6) "06-01-2020 11:00:00\x00"
7) 1) "Bears Smokehouse BBQ, Htfd, CT"
   2) (integer) 2695768111295037450
   3) (integer) 200
   4) "-72.671490000000006"
   5) "41.762740999999998"
   6) "06-01-2020 14:30:00\x00"
```

Or get a list of objects identifiers that intersect a particular area:

```
127.0.0.1:6379> reventis.trackallradius mytracks -72.682303 41.765600 5 mi 06-01-2020 5:00 06-01-2020 23:30 
1) (integer) 100
2) (integer) 200
```

There is also the `reventis.hist` commmand which allows you to get the history of any object - either the
entire history or only for a designated time duration.  


## But How Well Does it Scale?


The all important question!

Since Reventis is able to store each event/object in a balanced search tree, query response
time complexity is kept within logarithmic bounds.  In the following graph, four different
query sizes are plotted over increasing index size.  The x-axis is for N, the total number
of indexed events; the y-axis is response time in milliseconds.  As you can see, most queries
stabilize to under 1ms over increasing N.  Query #4 does not stabilize, because it is
literally the size of the state of Texas for over a year in time duration.  Also, the test adds
uniformly distributed random events, so larger query regions  do contain proportionally  more
retrieved results.  However, this ordinarily does not happen in practice,
since actual data is usually more clustered. 

![QueryResults](/resources/post_4/results.png)


## Summary

Reventis introduces a robust set of commands for the management of spatio-temporal point
data, and presents an efficient and invaluable solution for any applications that need to manage
location-based data.

More documentaion on the various commands is available [here](https://github.com/starkdg/reventis).
A client-side library of functions is also available to automate interactions with Redis.  

There is also an interesting application of Reventis to the GDELT - Global Database of Events, Locations
and Tones - dataset on the project README.  I'll leave further exploration for a future
blog post.  




