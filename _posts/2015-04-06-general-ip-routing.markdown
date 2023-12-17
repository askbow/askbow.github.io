---
layout: post
title:  "General IP Routing"
date:   2015-04-06 17:40:00 +0100
categories: routing
math: true
mermaid: true
---
Internet Protocol Routing, nowadays commonly known as L3 Switching, is part of the process of forwarding an IP packet from Source to Destination. Interestingly, it happens more often then commonly understood: even on a common subnet we often need to make an IP routing decision.

### What happens inside IP Routing?

Let\'s suppose that routing process is a black box:

```mermaid

flowchart LR

I(Ingres) -->|daddr| R{Routing Decision}
R -->|in A| A[Egress 1]
R -->|in B| B[Egress 2]
R -->|in C| C[Egress 3]
R -->|else| D[Drop]

```

Basically, we are taking some kind of function $R$, which takes a packet as an input and puts a packet to the output. Or not (more about that not further). The interesting thing here is that there are several outputs and each of them serves a set of destinations. The function in question, in fact, gives us the direction as a result.

> Thus, the routing decision is made on a per-destination basis and the routing decision is a vector towards destination.

So, our function might look like this:

$$ R(Ingres) = \Bigg\left\{ $$

Here, we have two additional sets: $A$ and $B$. If the destaddr (destination address) part of the Ingres is part of the set A, the Ingres will be sent in the $Egress_1$

So far, so good. Well, not quite so, as there are two problems with this notation:

1. What if the destaddr is both in A and B?
2. What if the destaddr is neither in A nor in B?

The solution to the first question is that the set which is the most specific, that is - contains less elements of all the destination sets, is selected.

For example, if $A = {a_1, a_2,a_3,a_4}$ and $B = {a_1,\dots,a_{255}}$ and $Ingres_{destaddr}=a_3$, then the first Direction will be used.

And as for the case of the destination address being in none of our sets, the result must be to destroy the input:

$$R(INPUT) = \Bigg\left\{\begin{array}{l l}Direction_1 &\quad\text{if } INPUT_{destaddr}\text{ }\in A\\Direction_2&\quad\text{if }INPUT_{destaddr}\text{ }\in B\\DROP&\quad\text{if } INPUT_{destaddr}\text{ }\notin ALL\text{, } ALL\supseteq\{A,B\} \end{array}$$

Basically, DROP here is just another Direction - the bit bucket, as some people call it.

The form I\'ve been using so far is far from being convenient. As some people might know, you can describe a function in several ways: in the form of a formula, as a drawing, or as a table.

### The routing table

Everything presented in $\Latex$ so far may be described more conveniently as a table.

| $INPUT_{destaddr}$ | Direction |
|---------------------|-----------|
| $\in A$ | Direction1 |
| $\in B$ | Direction2 |
| $\notin$A or B 
(nowhere else) | bit bucket, e.g. DROP |

This presentation is beneficial in that it efficiently shows a lot of destaddr-direction pairs in a uniform way. It doesn\'t matter that much when all we have is three simple routes. But consider the size of the Internet: at the time of this writing, the IPv4 routing table contains more than 500000 destination sets (i.e. networks).

> In the first book I\'v read about routing, it was a textbook called Networking basics (that book contains X.25, ATM, Frame Relay and other arcane stuff), the author said that there were more than 100000 prefixes in the Internet routing table at the time. Puts things in a perspective.

Besides, the table representation correlates well with what happens in the memory of a router.

> Not so much, actually. While initially it might have been true, today\'s routers (aside from ASICs) store forwarding information in the form of binary trees (or tries, to be more precise). A binary tree allows for a quick decision based on the destaddr bit-by-bit representation. A trie differs in that it has a third option: don\'t care.

### Enough of this general stuff, and you\'re doing it wrong by the way.

Alright. Now, to get back into the world of IP Routing, certain replacements are in order to make my notation useful.

- The destaddr is an IP address of a destination;
- The sets A, B are some networks, known to our router
- The directions are Next Hops (this is too specific, but let\'s stick with it)
- The function is the Routing Table

So, the table might be rewritten in this way (addresses are just examples, any resemblance to a real-world address is pure coincidence):

| Network | Next Hop |
|---------|----------|
| 192.168.9.0/24 | 192.168.1.1 |
| 192.168.16.0/22 | 192.168.9.1 |
| (nowhere else) | DROP |

I did a certain thing with this table, to demonstrate that the routing process is iterative:

1. A packet comes in with destaddr=192.168.17.45
2. Look up 192.168.17.45 in the Routing Table.
3. Per our Routing Table, it must be forwarded to 192.168.9.1!
4. Now, to reach 192.168.9.1: 
  1. Look up 192.168.9.1 in the Routing Table.
  2. Per our Routing Table, it must be forwarded to 192.168.1.1!
5. Send the packet in the general direction of 192.168.1.1

This is rather simple. Let\'s get more serious, shall we:

| Network | Next Hop |
|---------|----------|
| 192.168.1.0/24 | serial interface 0 |
| 192.168.9.0/24 | 192.168.1.1 |
| 192.168.16.0/22 | 192.168.9.1 |
| (nowhere else) | DROP |

The process is a little more long:

1. A packet comes in with destaddr=192.168.17.45
2. Look up 192.168.17.45 in the Routing Table.
3. Per our Routing Table, it must be forwarded to 192.168.9.1!
4. Now, to reach 192.168.9.1: 
  1. Look up 192.168.9.1 in the Routing Table.
  2. Per our Routing Table, it must be forwarded to 192.168.1.1!
  3. Now, to reach 192.168.1.1: 
      1. Look up 192.168.1.1 in the Routing Table.
      2. Per our Routing Table, it must be forwarded out of our serial interface #0!
5. Send the packet out the selected interface (serial 0)

This is a simple example, but I think it drives the general idea home: we need to store very specific information in the routing table to be able to route packets around. Ideally, each route\'s direction is represented by a pair of {Next Hop, Interface}.

> For next hop inforamtion we keep a separate table: the adjacency table. It usually contains information on how to send the packet to the next hop. For example, for Ethernet nexthops, it will contain destination MAC address and other data to fill in layer 2 header.

### How to reach the stars

So far, so good. But what happens if we do not want to DROP the packet when we do not know the destination? The best idea would be to pass the packet to some other router, which we presume to know that destination. Enter the Default Routing.

The Default Route - the destination of last resort is the least specific route inserted in the routing table just before the DROP. As a result, it is inserted instead the DROP sentence, because as the Default Route is the least specific (it contains ALL routes in the IP-verse), the DROP almost never gets executed.

With the default route my imaginary routing table will look like this:

| Network | Next Hop |
|---------|----------|
| 192.168.1.0/24 | serial interface 0 |
| 192.168.9.0/24 | 192.168.1.1 |
| 192.168.16.0/22 | 192.168.9.1 |
| 0.0.0.0/0
(anywhere else) | 192.168.1.2 |
| (nowhere else) | DROP |

Therefore, if a packet comes in with destination 1.1.1.1, it will be forwarded to 192.168.1.2 via serial interface 0, per the process described in the previous section.

### Populating the routing table

Next problem we have is how to fill these tables. The obvious way is to fill every row manually.
In many systems, where other methods are not available, this method - called Static Routing is used.

> There is a curious problem at the intersection of static routing and the process of building adjacency tables:
> If the direction of a route, especially of a default route (but any network big enough would suffice), is set to be an interface (without any particular nexthop IP) and this interface is of a broadcast multiaccess (like, Ethernet) type, the router will need to resolve L2 address for each destination out that route.
> 
> The L2 addresses for Ethernet networks are resolved using Address Resolution Protocol (ARP), and the situation will result in an ARP request for each destination believed (per the Routing Table) to directly behind that interface.
> 
> Now, the other feature in action here would be the [Proxy ARP](https://www.cisco.com/c/en/us/support/docs/ip/dynamic-address-allocation-resolution/13718-5.html). A proxy ARP-enabled router will reply with an ARP reply for each destination it can reach.
> 
> That way, our router will be receiving an ARP per each destination and store it in its memory. Effectively, that poor router will be soon storing adjacencies for most hosts of the destination network of the badly constructed route.

But there are other methods:

- Automatic configuration
- Dynamic routing protocols

The way of automatic configuration is to receive routes from neighboring routers via some autoconfiguration mechanism. For example, in IPv4, a router may receive routes via DHCP; in IPv6 routers send Router Advertisement messages to tell everybody that they exist and route packets. Another example would be ICMP redirect which allows one router to tell another that a third one has a better route to the intended destination.

> I forgot to mention. In this framework, any host is a router, as it must make at least one routing decision: send elsewhere, loop to itself of drop.

The routes installed this way are considered Static by the runtime, but they are not stored and expire upon restart of the routing process (or entire router for that matter).

Dynamic routing protocols are different. They all presume, that there is some kind of relationship between participants. This is usually called neighbour relationship.

Every routing protocol fills the best information it can find into the routing table. That information must be first received from the neighbours. To produce the information to pass to neighbours, each router, at a minimum, will consider it\'s connected interfaces.

> Not quite so and entirely configurable. But there is no other information, besides Redistribution, which I\'ll leave for another day.

The routing protocols considered in the present-day CCIE RnS exams are:

- [EIGRP](//askbow.com/2015/03/25/some-notes-on-eigrp-history-and-metrics/)
- OSPF
- IS-IS (briefly)
- RIP
- BGP

If several protocols are used all at once, the routing table might be populated by several directions to the same set of destinations. How does the router choose what route to use?

### Administrative distance

The administrative distance is used as the measure of how trustworthy a source of routing information is considered to be. In Cisco routers, the default administrative distances are set according to this table:

| Connected | 0 |
|-----------|---|
| Static | 1 |
| eBGP | 20 |
| EIGRP (internal) | 90 |
| IGRP | 100 |
| OSPF | 110 |
| IS-IS | 115 |
| RIP | 120 |
| EIGRP (external) | 170 |
| iBGP | 200 |
| (Never enters the routing table) | 255 |

So, the actual Routing Table (from previous examples) should look more like this:

| Network | Next Hop | AD |
|---------|----------|----|
| 192.168.1.0/24 | serial interface 0 | 0 |
| 192.168.9.0/24 | 192.168.1.1 | 1 |
| 192.168.16.0/22 | 192.168.9.1 | 1 |
| 0.0.0.0/0
(anywhere else) | 192.168.1.2 | 1 |
| (nowhere else) | DROP | 254 |

(all my routes are static)

Moreover, each Routing protocol provides its own metric - a number measuring how good this route is, in relation to other routes provided by the same routing protocol.