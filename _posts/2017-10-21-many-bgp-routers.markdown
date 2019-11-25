---
layout: post
title:  "How many BGP routers can a big AS have?"
date:   2017-10-21 13:52:00 +0100
categories: sandbox
---
For iBGP number of peers (*i.e. the number of BGP routers inside an AS*), the only significant limiting factor is that iBGP peers must be fully meshed (N.B.: not directly interconnected! An iBGP peering can span all the hops you can fit into the IP TTL field) - because it is the only way for iBGP to prevent loops.

The impact of each BGP peer is an open TCP connection, some memory, and occasionally some processing to do and then some administrative burden.

> *How many connections?*
> 
> *$latex \\frac{n \\times (n-1)}{2}$*
> 
>  *- That\'s quadratic complexity:*

[![](https://askbow.com/wp-content/uploads/2017/10/bgp-full-mesh-300x198.png)](https://askbow.com/wp-content/uploads/2017/10/bgp-full-mesh.png)

To overcome iBGP scalability problems, two approaches were developed:

- Confederations
- BGP Route Reflectors

### BGP Confederations

*Confederations* is basically splitting your AS into several sub-ASes. A confederated AS looks like a single entity to its eBGP peers, even though each individual router might belong to a different sub-AS.

[![](https://askbow.com/wp-content/uploads/2017/10/bgp-confederations.png)](https://askbow.com/wp-content/uploads/2017/10/bgp-confederations.png)

Routers prevent loops inside confederation by using a special CONFED version of AS_PATH. Just like AS_PATH, its CONFED counterparts can be of two types: _SET and _SEQ.

Importantly, we must fully mesh BGP routers inside a sub-AS. Basically, a sub-AS is just an AS in its own right.

Downside: loss of detailed routing information when we cross sub-AS boundary.

Where is it logical to put confederations in production? My guess would be, large enterprises. It is kind of normal for one company to own several AS numbers - that usually happens as a result of corporate mergers and acquisitions. At the same time, such company might want to present itself as a single entity to any outside network.

### BGP Route reflectors

*Route reflectors* allow you to build a hierarchy of routers. A route-reflector client router doesn\'t know that it works with a route reflector - it\'s a normal iBGP peering for a client. Thus client\'s algorithm is the same as in fully-meshed iBGP system.

[![](https://askbow.com/wp-content/uploads/2017/10/bgp-route-reflectors-300x198.png)](https://askbow.com/wp-content/uploads/2017/10/bgp-route-reflectors.png)

A route reflector (RR) acts a little differently though. That is because its clients are not fully meshed. So, RR (*almost*) always sends updates to its clients, even when received from another client.

In order to prevent loops, a CLUSTER_ID attribute is used by route reflectors.

Notice that we must fully mesh Route reflectors between each other. And for redundancy, we must install at least two.

Moreover, we sometimes would place route reflectors outside of traffic paths. That way, we can use a cheaper router (still has to receive all the routers we have). It is possible thanks to BGP\'s third party next hop feature.

Downside: loss of detailed routing information, because RR will only send the best routes to its clients. Hence, possible suboptimal routing.

> Interestingly, BGP RRs are the basic idea behind some SDN implementations. Basically, the RR (SDN control server) is filling client\'s routing tables via BGP.

### Finally,

Both schemes allow ASes to grow to hundreds of routers and more, and the two schemes can be used in parallel if desired. The Route reflectors method is perhaps the most widely deployed. The reason is it is easier to design, setup, and support. Also, it allows to build a multi-tier routing hierarchy (*core-aggregation-edge, for example*) with minimal effort both initially and during scaling.