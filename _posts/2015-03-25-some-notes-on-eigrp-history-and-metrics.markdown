---
layout: post
title:  "Some notes on EIGRP history and metrics"
date:   2015-03-25 18:45:54 +0100
categories: routing
math: true
---
I\'ve decided to listen to some advice and use blogging as a learning aid. And to begin with, I\'d like to tackle on the subject of EIGRP. It\'s no secret that EIGRP is Cisco\'s beloved proprietary (it is published as an IETF draft, but apparently public version lacks a feature or two) routing protocol, and thus it is the most discussed in Cisco Certification R&S Curriculum.

> The timeline for this post is based on the contents of BRKIPM-2444 CiscoLive session. Sadly, Cisco does not preserve historical content anymore, so I'm removing the dead link.

# Begining of EIGRP

The history of EIGRP starts with a paper titled [Loop-free routing using diffusing computations](https://dl.acm.org/doi/pdf/10.1109/90.222913 "https://doi.org/10.1109/90.222913") by J. J. Garcia-Lunes-Aceves published in IEEE/ACM Transactions on Networking in 1993.

In this paper, the DUAL algorithm is presented and evaluated in terms of correctness, performance and production of loop-free routes. The algorithm is based on several definitions (taken from [Ivan Pepelnjak](https://www.ipspace.net/About_Ivan_Pepelnjak)\'s book EIGRP Network Design Solutions):

> - Downstream router - the router that is closer to the destination
> - Upstream router - the router that is further away from the destination
> - Reported distance - the distance reported to the current router from a neighbour
> - Feasibility distance - minimal computed distance to destination
> - Feasibility condition - met if the reported distance is strictly lower than the feasibility distance
> - Feasible successor - a downstream router, potential next-hop, guaranteed to be on a loop-free path (=not Upstream). Must meet feasibility condition.
> - Successor - the next-hop router toward the destination. A Feasible successor on the lowest cost path to destination.

## When EIGRP is stuck

In 1998, the Stuck-In-Active condition handling algorithm was rewritten (the SIA rewrite point on the EIGRP timeline). 

Before the rewrite, certain failure conditions (for example, unidirectional link causes loss of Reply in transmission) would lead to a Neighbour Relationship reset after the Active timer (3 mins by default) expiration. And the relationship might be torn down several hops away from the actual failure.

After rewrite, the default active timer became halved. After its expiration, a new type of query - SIA Query is sent. If the next neighbour acknowledges it, the timer is reset, the relationship continues. The relationship nearest to the failure in its turn will inevitably fail at some point, the SIA query is cleared and a reply is propagated back to its initiator.

> A picture is worth a thousand words:
> 
> [caption id=attachment_13 align=alignnone width=800\][![EIGRP](https://askbow.com/wp-content/uploads/2015/03/eigrp-sia.png)](https://askbow.com/wp-content/uploads/2015/03/eigrp-sia.png) It looked better in my head[/caption]

## Handling new topologies

In 1999, EIGRP was extended with support of a special kind of topology - the Hub&Spoke. It is the one, where the Stub functionality was intended to be used. The idea was to solve another problem with Queries: in a hub and spoke topology large enough, hub routers would have to send queries to every spoke (and wait for Replies, right).

First, it is not a very scalable process per se, and second, those spokes that happen to have two connections to Hubs are thus equipped for resiliency, not so they can be used as a transit between hubs.

Therefore, there is no reason for a hub router to learn about routes (or query for) through any of these spokes. A stub router needs only to advertise certain routes to the hub (e.g. connected routes; other options available: summary, static or nothing [receive-only]) and tell it of its Stub status.

> Notably, this is the very feature of EIGRP that did not make it to the IETF draft. The reason for this seems to be another Cisco feature - the DMVPN.

## Metrics

**Classic metrics**: minimum Bandwidth, sum of Delays and (disabled by default) link Reliability and Load. These "vector" metrics are weighted using the K-values and mixed in the following formula to calculate the Composite Metric: 

$$ CM = [(k_1 \times Bandwidth + \frac{K_2 \times Bandwidth}{256 - Load} + (K_3 \times Delay)) \times \frac{K_5}{K_4 + Relyability}] \times 256 $$

here, 

$$Bandwidth = \frac{10^7}{MinBandwidth}$$

$$Delay = \frac{\sum delays}{10 ms} $$

Load, Reliability $\in [0;255]$ and $K_5=0,~~ \frac{K_5}{K_4 + Reliability} = 1$

The constant reference to 256 / 255 is not random. The classic metrics are bounded by 32-bit.

Notably, the original paper uses just the term "Cost", thus I assume CM must have been a separate development. By terming them "vector metrics", this refers back to the original paper however.

# Millennium Edition

The early 2000s, it appears, was one of the most fruitful for EIGRP features. The PE/CE deployment scenario (service provider stuff), support for NSF/SSO, route-maps and third-party next hop were developed.

## PE/CE and Site of Origin

Introduced in 2000, PE/CE scenario is meant for peering with SP who interconnects two or more our sites via its MPLS/BGP infrastructure.

Here, normally, at Site1 PE router redistributes EIGRP->BGP and at Site2 iPE redistributes BGP->EIGRP. That makes the routes to appear as External in EIGRP topology.

The idea of PE/CE feature set is to make those routes appear as EIGRP Internal, i.e. make the SP transparent to our routing. This is performed by attaching to routes (in BGP) extended communities with EIGRP metrics, and using them at the other side to reconstruct original EIGRP routing information.

This construct makes SP network look like a zero-cost link between the CE routers. To solve potential routing loops, the routes are tagged at PE with a Site of Origin (SoO, introduced in 2005) tag, and CEs reject marked routes.

## Non-Stop-Forwarding

2001\'s NSF is a Cisco\'s way to continue forwarding (in data plane), while the control plane is experiencing failure and undergoing recovery procedures; the new Graceful Restart (GR) feature allowed an EIGRP router to notify its neighbours of the problems/restart in order not to tear relationships while EIGRP control is unavailable (no Hello/Ack exchange).

The signalling is done via an EIGRP Update packet with both Init and RS (restart) bits set. The router undergoing restart would send its Hellos with RS bit set until it completes GR. In the same time, all of its neighbours would be sending it relevant routing information.


## Nerd Knobs: traffic engineering

In the year 2003, Route-maps were enhanced to allow for metric manipulation and route tagging.

The application is clear - traffic engineering, i.e. enforcement of path preference for certain flows. Or ability for route-map statement to match on metric values (example: match metric 1000 deviation 100 will match routes with Metrics from 900 to 1100).

Then in 2004 **3rd-party-next-hop** feature was added. Third-party next hop allows for a hub to advertise routes through his neighbours to other neighbours. Apparently, in certain topologies, when there is a link, but no EIGRP neighbour relation between routers for some reason.

> An example of such topology would be the case of redistribution between two routing protocols, both running on a shared segment: In this way, the next hop is preserved in redistribution from broadcast networks.
> [![EIGRP](https://askbow.com/wp-content/uploads/2015/03/eigrp-next-hop-not-self.png)](https://askbow.com/wp-content/uploads/2015/03/eigrp-next-hop-not-self.png)

Another example of this feature usage is - guess what - Dynamic Multipoint Virtual Private Networks (DMVPN). There were other notable additions, like new SNMP MIBs and Bidirectional Forwarding Detection (BFD) support during the same year 2004.

**Summary metric** was added in 2009. The idea is to advertise just a summary route (say, a /22) instead of all component routes (here, a number of /24 networks).

As every of the component routes may have a different metric, a strict method of summary route metric determination is required. Cisco\'s engineers decided that the smallest metric among the component routes will be employed as the summary route metric.

The metric is updated every time the component route\'s metrics are updated (and the new lowest is elected to represent the summary). This might present a problem if there are many links, their metrics change often.

Thus, the summary-metric feature was introduced, allowing the router\'s administrator to explicitly set the metric for the summary regardless of the component route metrics.


## Reliability improvement

In the year 2003, the 3-way handshake was added. Basically, Cisco created its own TCP with regard to EIGRP neighbour relationship, introducing a new state - Pending. The process goes this way:

1. When a router receives the 1st multicast Hello, it places the relevant neighbour in the Pending state and sends a unicast Update with the Init bit set. While in Pending state, no Queries or actual routing information is transmitted;
2. Upon receiving of Update with Init, router replies with an Update with the Init and Ack bits set. That way, the 1st router will receive both an Update AND an acknowledgement of its own initial Update;
3. If the 1st router receives that Update+Init+Ack, it moves its new neighbour out of the Pending state and starts to transmit topology information.
4. If that Update+Init+Ack (more importantly, Ack) was not received, any other communication (Hellos) from a neighbour in Pending is ignored. Several retransmission attempts are made until timeout.



## A prelude to SD-WAN

In 2005, support for EIGRP was added to PIX firewalls. EIGRP itself was updated to support IPv6 and (mentioned earlier) Site of Origin tagging. In 2006, DMVPN was introduced, which relies heavily on EIGRP features in its operation. Notably, performance Routing (PfR) was added to Cisoc IOS about the same time.

Personally, I think that PfR was created to realise the ideas of the failed attempt to use dynamic metrics in EIGRP.

Dynamic metrics (Load and reliability were proposed as feasible candidates) easily cause positive feedback loops in EIGRP logic an result in traffic constantly churning between links. 

I suppose that it gets synchronized just like TCP\'s fallback to slow start for many flows.



# After 2010

In 2010, a new authentication type was added - the SHA2 HMAC. In 2011 the advances in underlying transport technologies (yay, 100G Ethernet!) lead to the introduction of EIGRP Wide Metrics.

## Wide Metrics

**Wide metrics** are more intricate. First of all, the Delay and Bandwidth values now cross the uint32 boundary.

They still employ the same basic vectors, but mix them in a very different bowl, so to say:

$$ WM = [ (k_1 \times Throughput 
          + \frac{K_2 \times Throughput}{256 - Load} 
          + (K_3 \times Latency) 
          + (K_6 \times Extended) )
       \times \frac{K_5}{K_4 + Reliability }
       ] \times 256 $$

here, throughput is calculated using maximum theoretical throughput:

$$ MaxThroughput = K_1 \times \frac{EIGRP_{BANDWIDTH} \times EIGRP_{WIDESCALE}}{Bandwith} $$

$$NetThroughput = [MaxThroughput + (\frac{K_{2}\times MaxThroughput}{256 - Load})]$$

These values are only used by the local router. Original numbers are sent to neighbours. The Latency here is calculated using 

$$ Latency = k_3 \times \frac{Delay \times EIGRP_{WIDESCALE}}{EIGRP_{DELAYPICO}} $$

where, for interfaces under 1 Gbps, Delay is:

$$ Delay = InterfaceDelay \times EIGRP_{DELAYPICO}$$

and for interfaces beyond 1 Gbps:

$$Delay = \frac{EIGRP_{BANDWIDTH} \times EIGRP_{DELAYPICO}}{InterfaceBandwidth}$$
