---
layout: post
title:  "How many spares do you need?"
date:   2018-05-11 16:08:00 +0100
categories: sandbox
---
In designing a network, there is a question that is often missing an answer or at best, answered using some rule-of-thumb. How many spare units you should include in your BOM? Actually, do you need them at all?

> **Disclaimer**: *I won\'t be covering any of the really complex models. People who need them probably know about spare part forecasting and procurement more than I do. But some simple models are useful in general network design work, so here\'s my take on it.*

### TL;DR:

It depends. The lower mean time to recovery (MTTR) you want, the more chance there is that you need to have on-site spares. The lower the MTTR, the higher the availability you get.

## Why discuss spares?

Let\'s go with a top-down approach here.

There can be several business drivers for really high network availability. A few examples:

- network downtime cost is **very** high - think of a broker connecting to an exchange, or medical equipment during remote procedures (*these will become more and more common over the years*)
- regulatory / compliance - rules imposed by regulatory body (*industry association, state department*) upon your information system in general and by extension on the network
- tight SLAs with customers (*who then have cost / compliance or other stuff for their reasons*)

To see why we may consider spare parts as part of high availability equation, let\'s go a little deeper.

### What is availability

Availability in its general mathematical form depends on two factors:

- MTBF - mean time between failures; many people confuse MTBF with how long a given specimen will work for. A better, more practical understanding of it goes like this: if a vendor has sold 1 000 000 units (*power supplies for example*) with MTBF 1 000 000 hours, then on average they will be sending one replacement unit every hour.
- MTTR - mean time to recover [from failure] - how long it takes to fix a problem

The availability is usually taken as A = \\frac{MTBF}{MTBF + MTTR} and the result might look something like 0.99818231.

> There\'s a comprehensive article on that topic over at Packet Pushers: [Reliability Basics- Part1 by Diptanshu Singh](http://packetpushers.net/reliability-basics-part1/). There\'s no point in repeating all of that math background here.

What it means in practical terms is, you can compute the expected (*notice expected - it\'s all a matter of statistics*) downtime by taking T_d = (1 - A) \\times T, where T is your time budget (*most people use a Gregorian year here, as expressed in minutes or seconds*).

### How do you increase your network availability?

There are several ways to push availability up:

- get more reliable equipment (*i.e. increase MTBF*)
- add redundancy (*think failover/cluster/VSS/vPC/stack/VRRP etc*) - also sometimes called structural reliability
- decrease MTTR

Now, there is some technological level after which it is prohibitively expensive to increase MTBF, plus there\'s a natural trade-off between features (=complexity) and reliability.

It is also hard to do redundancy: it increases complexity even further and introduces a separate set of distributed systems problems (*for example, firewall cluster and VSS state machines have a lot of moving parts*). Although some people may push it higher than that, most settle for something manageable, like running two units in parallel.

Seems like the only thing left to do is to try to decrease MTTR.

### Side note: what network equipment MTBF looks like?

Enterprise-class ethernet switches (fixed) seem to have their MTBF converged around 250000 - 400000 hours (*at least based on datasheets referencing Telcordia parts-count methods*). Individual linecards for modular switches have about the same figures.

Fixed routers are in the same ballpark or higher. Servers are ususally considered to have a lower MTBF, around 75000 hours, while the appliances (like many firewalls, which are basically stripped-down servers) expect to have something around 100000-150000 hours of MTBF.

You should always refer to your vendor/manufacturer if you need exact datapoints for precise calculation.

## What is MTTR?

In general, MTTR consists of several components:

- Failure detection time
- Problem diagnose time
- Repair time
- Time to test and confirm restoration of service

> Many times, Time to actually repair is time to reboot the device (i.e. in the order of 5-10-20-30 minutes) or remove a config line (in the order of 1-5 minutes). On the other end of spectrum is replacing a whole half-rack-high modular switch (0.5-4 hours) . Notice here also that time spread increases with complexity. A corollary to that is for lower MTTR you might want to minimize complexity.

In special circumstances, like remote sites, you might also add to the mix:

- Engineering team delivery to the site to perform repairs (*for unmanned site*)
- Time to deliver spares to the site (*if sent separate from the repairs team*) - which is also the case if you don\'t have any spares at all

### How do you decrease MTTR?

Before we dive deeper into the whole spare part business, let\'s cover other ways to decrease MTTR first.

> If you think about it, *redundancy* is actually a way to decrease MTTR taken to its absolute: the spare unit takes over automatically with minimum switchover delay feasible.

First things first, depending on your economics and technology, you optimize MTTR down by decreasing detection time. You do it with all sorts of monitoring/telemetry, regular health check-ups and planned maintenance procedures. Same approach works for decreasing diagnostics time. You prepare and use checklists, configuration management procedures [i.e. you always know if any change was made prior to failure], automation. Last but not least - you invest in people by training them. You can also make your critical sites manned 24x7x365, i.e. hire more people.

Similarly, time to actually repair something depends again on procedures and people, but there are limits to that.

#### Other replacement options

At some point, you will need to replace failed equipment. You don\'t necessarily need spares for that:

- *Warranty* - many honest manufacturers will cover (although without any real SLA) their products for some reasonable time (or for the product\'s lifetime, i.e. until they declare its End of Live)
- *Service contracts* - these include not only replacements, but also have some SLA attached to them - for example shipping the replacement part on the next business day (mean time delivery will be at least 32 hours), the next day (24 hours), in 4 hours (5 hours)

> Time estimates here are rough and include some reasonable same-postcode-expedite-delivery. No vendor has a warehouse in every area, so add some time allowance for UPS / DHL / FedEx to reach you.

As far as I know, 4 hour shipping SLA is the top speed available from most vendors. Sometimes, if you have enough leverage, you can squeeze a little more (*down to 1 hour maybe*) from your local vendor\'s VAR.

Here we arrive to the final point: if you need to go further down the timescale, you have to have on-site spares.

## The economic effect of having spares

First of all, spares cost money directly, that is - you need to buy them (*and possibly cover them with appropriate contracts as well*).

Then you need to store them, and spend some time regularly testing them (*a once-a-year [or more often for more critical systems] smoke test*). Moreover, from financial point of view, spares are stale capital [not exact term, sorry], that is by buying something (*to sit in your warehouse*) you throw out your ability to employ that capital otherwise. And that, in short, can make some of your financial KPIs look not as good.

On the other hand, spares relax service contract requirements. For example, instead of covering your whole park of 1000 access points with 24x7@4hrs contracts, with ten spares stored in a wire closed you would only need to get 8x5@NBD contracts.

All in all, your mileage will vary, and this motivator is worth a due consideration.

## Spare part kit

Your spare part inventory consists of one or more spare part kits. Spare part kits are collections of spare parts which serve a particular site or a group of sites. As such, we can distinguish between:

- *local kits* - serve one site, stored on that site (zero delivery time)
- *group kits* - serve a set of sites (usually grouped by geography). Vendor\'s warehouses that ship you a spare part based on service contract can be considered an example of that
- *multilevel kits* - some combination of the above

### Replenishing spare part kit

Another way to classify spare part kits is the way you top them up (i.e. how you drive your *spare part procurement process*). Basically, you can do it in any of these ways:

- *never* - you load up everything you will ever need and fly to the edge of the Solar system.
- *regularly* - every year (month, quarter, other set interval) you buy a bucket of transceivers.
- *waterline* - as soon as the number of spares goes down to some predefined level (waterline) below the base, you buy more to restore status quo.

> There are special cases and combinations of these, but they make sense only for some level of sophistication of supply and support organizations. For example, military organizations probably have very complex schemes with specific goals (given that most of the interesting math for reliability and spare part calculation was initially [and still is] developed for army\'s needs).
> 
> I expect organizations such as Google and AWS, as well as network equipment vendors, who also happen to have a lot of data about IT systems reliability, to have developed their own complex spare kit configurations as well.

Down the line, I\'ll be covering a generic case of a local kit which we replenish on a waterline basis.

### How do you evaluate your spare part kit?

> Sorry for another interruption, but answering this question early will make later understanding easier.

There are two important metrics of spare part kits:

- *spare kit efficiency* - basically, how many hot devices you are covering with each spare
- *spare kit readiness* - this is a statistical measure of probability to find necessary spare in the kit at the moment of hot device failure

#### Efficiency

Efficiency can be calculated as Q = 1 - S / N, where S is the number of spares, and N is total number of devices protected with these spares. The higher - the better (*lower economic loss*).

#### Readiness

Readiness is a little more complex. For waterline replenishment discipline it goes in two steps:

1. calculate minimum spares needed as S_{min} = \\frac{N \\times T_t }{MTBF} - *see notes in the next section*
2. insert the result into this slightly bigger formula:

R = 1 - \\frac{S_{min}^{m + 2}}{(S - m + S_{min})\\times((1 + S_{min})^{m + 1})}

where S - your spares base level, m - your waterline level.

Obviously, you want your Spare Kit readiness as high as possible.

In practice, you would find your own balance between efficiency and readiness by solving some optimization problem specific to your needs.

### Calculating minimum spares required

This formula comes as a result of developments in mathematical modeling and queuing theories. By modeling spare parts kit as a queue and failure rate as Poisson independent events, it can be shown that for a general case:

S_{min} = \\frac{N \\times T_t }{MTBF}

T_t here is the mean time it takes to replenish the kit, i.e. for the vendor to deliver on the contract (see above).

> There\'s a side result from same sciences. We can say with high confidence that this kit will be optimal for many typical cases as well. That is, it maximizes both efficiency and readiness at the same time. I\'m not sure if this holds for all cases.

### Calculating the Total number of spares required

How many spares would you need if you expect never to replenish the kit?

S_T = T \\times N / MTBF, where T is total expected lifetime.

By taking 1/MTBF we effectively convert it to failure rate, which we then multiply by total system time budget (expressed in machine-hours).

The result of this calculation is also the maximum number of spares. Between the minimum and this maximum, you must choose a point that makes sense to you. A good way to start is to set some expectations about the kit\'s readiness and efficiency and crunch the numbers, trying to maximize both.

### Conclusion

Maintaining a spares inventory is a good way to lower MTTR in a highly available system. The models I list here might not be perfectly refined, and they certainly don\'t take into account every situation possible, but they produce a fair estimate.