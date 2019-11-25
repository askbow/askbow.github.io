---
layout: post
title:  "Dual-homing with Policy-Based Routing - a case study
date:   2015-04-20 14:07:00 +0100
categories: routing
---
[Policy-based routing](https://askbow.com/2015/04/13/policy-based-routing-ip/ "Policy-based") allows network administrator to stir traffic in directions different from the one chosen by [destination-based routing](https://askbow.com/2015/04/06/general-ip-routing/ "General") and its [routing protocols](https://askbow.com/tag/routing-protocols/). This can be useful in several scenarios, namely in dual-homing to different ISPs, as well as other special cases.

## Using policy-based routing for dual-homing

### General notes on dual-homing

The term Dual-homing in its most general meaning is used to refer to a situation, when connection to the same resource is built via two (or more) independent paths. Here, dual-homing is about having connection to another network (the Internet) via several Internet Service Providers.

There are usually several goals to reach with this type of connection to the Internet:

- Internet connection resiliency - with businesses world-wide relying more on Internet (even more so with rise of cloud services usage) to operate, Internet connection might be as important, as air; thus, many would prefer to install a secondary link in case of primary link failure;
- More bandwidth - when one connection isn\'t enough
- Cost optimization - some ISPs might charge you for traffic usage (*others just provide you with limited bandwidth, but with no limit on traffic besides the maximum possible time*bandwidth*), and some of those will charge differently for different traffic;
  > for example, I remember days when local (in-country) megabyte of traffic was way cheaper then foreign. At the same time, due to poor interconnectivity at local IX, traffic to a nearby city was routed trough an IX in Germany, thus treated as foreign.

Probably the best solution (design-wise) in this case is to get a provider-independent network and an autonomous system number, then use these to peer with both ISPs using BGP. This solution is usually flexible, scalable and it is relatively easy to support. On the other hand, it may cost more - especially with IPv4 address depletion at hand.

### Policy-based routing to the rescue

In cases when there are some reasons not to buy address space, and we want to use all of the links to the Internet simultaneously, policy-based routing (PBR) will help.

With PBR, network administrator is able:

- (*in some topologies*) ensure that traffic coming via one ISP will return via the same ISP
- route HTTP and FTP (or any other port) traffic to a certain ISP, while routing SMTP and DNS via another
- ensure that traffic from some users will be forwarded to a certain ISP, or even balanced shared L4 port wise

Have a web server that you need to always be visible trough only one ISP? PBR can do that.

## The case for PBR-based dual-homing

### Dual-homing topology

So, here is my example topology:

> *I\'m trying my best not to use completely textbook examples here, so I use a fairly simplified topology from my work experience*

[![Example](https://askbow.com/wp-content/uploads/2015/04/pbr-example2.png)](https://askbow.com/wp-content/uploads/2015/04/pbr-example2.png)

(*not shown: some less important redundant connections*) Here, Routers 1&2 are the border routers, and Router 3 performs NAT and firewalling between internet and LAN/DMZ. In that capacity, Router3 terminates all spare IPs in the two /27 networks which are provided by ISPs. Also, Router3 cannot perform policy-based routing itself for some reason.

Router 3 has two default routes, pointing at Routers 1&2 (only two, because I\'m keen on VRRP) in each of the /27 networks. ISP1 doesn\'t know (and thus doesn\'t route) ISP2\'s network, and vice versa.

ISP\'s gateways MUST have a route in their tables pointing traffic destined to respectful /27 networks towards our Routers 1&2.

We need to ensure, that traffic originated by Router3 from its IPs in ISP1\'s /27 network will always go towards ISP1 router which serves as a default gateway, and the same goes for ISP2\'s networks: always forward to ISP2 gateway.

### How policy-based routing is configured in this case

There are two elements that need to be configured for this to work:

1. Two access lists, to match traffic origins
2. Two route-maps, to do the PBR itself

An access list will look like this (Cisco IOS 15.x):

```
<pre access="" class="lang:python" decode:true="" list="" match="" source="" the="" title="An" to="">ip access-list standard ISP1-NET 
 permit 1.1.1.16 0.0.0.31
```

If there are more than one network owned by the same ISP, the respective ACL will contain either more lines (suggested for ease of management) or a summary network (may or may not increase speed, depending on platform).

A route map then will look like that:

```
<pre all="" class="lang:python" decode:true="" match="" route-map="" them="" title="A" to="">route-map ISP-FORWARD permit 10
 match ip address ISP1-NET
 set ip next-hop 1.1.1.1
```

Here, we match addresses listed in the access-list previously created, and then for any matching routes we set the next-hop to 1.1.1.1.

Another option would be to *set ip next-hop recursive.* The *recursive* keyword helps in case the next-hop is not adjacent (that is, not reachable directly on one of the Connected networks). Not our case, but still - nice to know.

Obviously, the same configuration is made for ISP2, using relevant addresses for networks and hosts/gateways.

After that, the route-map (it is possible to create multiple, but I found it more manageable to use just one) is applied to each of the *inside-looking* interfaces:

```
<pre a="" an="" class="lang:python" decode:true="" interface="" route-map="" title="Applying" to="">interface gi0/1.42
 ip policy route-map ISP-FORWARD
```

Moreover, it might be necessary to make traffic originated by the router itself to behave the same way:

```
<pre class="lang:python" decode:true="" route-map="" router-originated="" title="Apply" to="" traffic="">#ip local policy route-map ISP-FORWARD
```

### How policy-based routing works in this case

As illustrated by colorful arrows on the diagram above, PBR makes any traffic originating from orange subnets to be forwarded to the first ISP, and any traffic originated in the purple network to be forwarded to the second ISP. I.e. it follows this simple procedure:

1. A packet comes into ingress interface; **N.B.:** *route-maps, in PBR cases, are allways applied on ingress interfaces*
2. The route-map, applied at the interface has a PERMIT statement with a match, referencing an access list
3. IF the packet matches the any of the access-list statements, the SET directive will be applied, else - next route-map statement is evaluated
4. The SET directive commands our router to forward the packet to the next-hop stated.
5. The next-hop is evaluated and the egress interface is determined
6. The packet is forwarded to the next-hop out of the egress interface

The return traffic is forwarder normally, using normal destination-based routing process.

### How to live with that

As ISPs and inside networks are added and removed (that happens too), it is relatively easy to support this config:

- If an ISP gives you another network, it is easy to add to that ISP\'s respective ACL
- If, for some reason, another interface must be added, the policy-based routing-wise configuration might be copied from existing interfaces as-is;
- better still, make the Router3 terminate them in its logic and only add a new [static] route to Router 1&2 configurations pointing to Router3, and a new line to the ACL

There are some problems though. For example, Router3 must be able to decide from which IP to originate the traffic, how to maintain state for the flows traversing it and how to learn about failures in the upstream.

The scheme I\'ve drawn here protects from Router 1 or 2 failure (or failure of one of their interfaces). There are some parts missing (like, VRRP and IP SLA tracking) which are not relevant to the PBR case itself but are, in reality, present in the configuration.

The problem is that this technique doesn\\t help in case of ISP failure, especially in case of a partial failure (for example, an ISP looses connectivity to a half of the Internet) - we can\'t detect of work around such a failure (BGP peering with receiving full-views from both ISPs would help).

Thus the PBR option is far from perfect and should be used for dual-homing with caution and careful planning.