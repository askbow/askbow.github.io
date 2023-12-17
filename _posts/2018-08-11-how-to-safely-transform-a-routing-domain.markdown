---
layout: post
title:  "How to safely transform a routing domain"
date:   2018-09-11 13:36:00 +0100
categories: automation, design, ospf, python, routing
---
As part of my job as a Senior Network Engineer, I develop procedures for undertakings of varying complexity. In this post I\'m describing a technique that greatly simplifies any project where a routing domain is expected to churn (*i.e. neighborships going up and down, routes flapping*), when such event is undesirable.

### Motivation

I developed this technique for a client running a critical network operating 24x7 with data flowing across ten timezones. The prime objective was minimization of packet loss during the procedure, so that the real-time application can continue.

> *Personally, I consider this application way too brittle and in a huge need of redesign. But this is beyond the point of this blog.*

The original project I developed this procedure for was segmentation of a flat OSPF (*i.e. single-area*) network into multiple areas of different types. However it is general enough so we can easily adapt it to other similar projects.

### Design options for transforming a routing domain

We start with a routing domain in state A and want transform it into state B, without loosing connectivity in the process.

[![](https://askbow.com/wp-content/uploads/2018/08/routing-domain-A-B-300x163.png)](https://askbow.com/wp-content/uploads/2018/08/routing-domain-A-B.png)

> *Why do that? To isolate less-stable WAN churn from more-stable LAN, for one. Also remember that in OSPF you can effectively enforce policy only at ABRs/ASBRs, so segmentation may make sense for you.*

There are a few general ways we could go about that:

1. Schedule a maintenance window, do the job as quickly as possible
  Good: just do the core job
  Bad: will drop packets in the process
2. spin up a parallel routing domain temporarily over the same network
  Good: low command count to apply on device, automatic workings of a routing protocol taking care of connectivity all the way
  Bad: possible routing loops, need to account for existing routing policy (*redistirbution, filtering, costs, etc*)
3. convert existing routing tables into static routes and use them
  Good: routing tables already assumed loop-free and based on policy
  Bad: there are hundreds, thousands of routes, overwhelming volume

Luckily, the overwhelming volume part is easily solved (*or so I thought, see below*) with automation!

Hence, I decided that we must convert the existing routing tables into long lists of static routes, which we add to configurations of every device in the network we\'re working on.

### But there are thousands of them routes!

Python to the rescue!

The script I wrote to wrangle this task is on GitHub: [https://github.com/askbow/networking-tools/blob/master/routep.py](https://github.com/askbow/networking-tools/blob/master/routep.py)

> Note: this is an old post; I wrote this script before TextFSM came to my attention; the script essentially implements a single-purpose finite-state-machine to parse text input ("screen-scraping").
> Nowadays, just use TextFSM. 


The basic idea of the script is this:

1. load show ip route output from file
2. parse it line-by-line into a dictionary, where keys are prefixes and values are lists of nexthops and interfaces
3. optionally optimize the routes where safely possible
4. go through the dictionary and print static route commands to default output

The result is a neat list of all the routes in the routing domain (*of which this device is aware of*) with a high administrative distance.

The most gruesome challenge for me when writing this script was the sheer inconsistency of Cisco IOS and ASA products of different versions in terms of show ip route output structure.

> *If the network was build of just one type of device running one version of software, the whole script would\'ve been thee times shorter and basically consist of a single RegEx match to extract the information I need. My script is ugly because it must parse the ugly.*

### Known issues

This simple method was generally successful, simplifying procedures for more than a dozen projects. There were however some operational challenges I must make you aware of.

First, the high metric I\'ve chosen as default for the script is not optimal in some topologies. Such topologies tend to be complex and the problem lies on intersection of several routing domains. For example iBGP takes precedence with a lowed AD, setting routes across a path. Adjust the script accordingly.

Second, the route optimization the script employs is very straightforward: it aggregates adjacent prefixes where possible and drops any equal-cost routes from the lists. This usually reduces the length of the resulting command list several times over. Yet, like with any aggregation, you loose detailed routing information and that may introduce some additional risks.

Both of these in my practice had only played out in *more-complicated-then-usual* topologies. Your mileage may vary, be prepared and double check.

### The procedure to safely transform a routing domain

With all that being said, here\'s an outline of a procedure which is based on the method described here.

1. collect fresh show ip route outputs from all devices in the immediate routing domain (*i.e. there might be no point in scraping those behind an aggregation/summarization wall*)
2. parse them through the script to get static route commands
3. apply static route commands to all devices
4. do the main job (*i.e. change the routing protocol, change OSPF areas*)
5. check everything that your routing domain is back up as expected (*make a checklist ahead of time!*)
6. remove the static routes

Looks simple to me and *It does the job*.