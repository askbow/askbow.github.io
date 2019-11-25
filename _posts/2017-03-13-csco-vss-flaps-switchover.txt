---
layout: post
title:  "Cisco 6500 VSS flap during SSO switchover if some features are configured"
date:   2017-03-13 20:30:00 +0100
categories: switching
---
There is an interesting problem with Cisco 6500 VSS clusters: generally, switchover between nodes is fast enough and only a few packets are lost. NSF&SSO algorithms help a lot to achieve that. But if you configure a feature that doesn\'t support SSO for some reason, the flap becomes more noticeable. In this post I\'m trying to make an educated guess of what is happening.

### Background information on VSS and SSO

Cisco Virtual Switching System (VSS) for 6500 and 4500 is a clustering technology which is meant to increase network availability.
[![basic](https://askbow.com/wp-content/uploads/2017/03/picture-1024x656.png)](https://askbow.com/wp-content/uploads/2017/03/picture.png)

VSS is fairly well [documented by the vendor](https://www.cisco.com/c/en/us/td/docs/solutions/Enterprise/Campus/VSS30dg/campusVSS_DG/VSS-dg_ch3.html), and the basic idea is that the supervisors in two chassis form a distributed system with one being master (*Active*) and the other slave (*Hot-standby*).

The active supervisor maintains routing and other protocol neighborships and synchronizes its Forwarding Information Base [FIB] (i.e. CEF datastructures) to the standby.

In the event of switchover, for whatever reason (*switchover command, active powerdown...*), the standby will detect the loss of its neighbor. It will then assume the active role and go through NSF / SSO recovery procedures.

NSF (Non-Stop Forwarding) basically works because the FIB datastructures are synchronized, and the dataplane remains populated, thus able to forward traffic. That gives SSO enough time for soft-recovery of controlplane.

Most of the protocols at the moment can go through SSO without problems.

At the same time, some specific features cannot. A clear example for that is Enhanced Object Tracking (EOT), [which does not support SSO](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipapp/configuration/15-mt/iap-15-mt-book/iap-eot.html#GUID-A294B6B0-7E5A-4579-BED1-68DFBC8886D8). If we use it to directly control other protocols (*that do support SSO*), SSO becomes broken for them. For example, refer to this Cisco bug (*which is not really a bug, more like a reply to an inquiry*): [CSCui37233](https://bst.cloudapps.cisco.com/bugsearch/bug/CSCui37233/?referring_site=bugquickviewredir) (*requires CCO login*)

### 
 What are the reasons for that EOT flapping during SSO

There are two main thing that follow from documented SSO algorithms:

First, during switchover event, the new active supervisor starts by initializing the processes (i.e. protocols) that support SSO. This is necessary so that the neighborships are not torn down and to notify the neighbors that we are undergoing a soft-reboot. At the same time, the NSF process keeps FIB (CEF) separated from RIB. After the new active supervisor fully boots, all processes are started normally to perform their duties.

Second, as I said before, a VSS cluster does not keep RIB synchronized between the nodes. Process memory (i.e. BGP or EIGRP datastructures) isn\'t synchronized either (*thus the need for NSF to keep FIB separated from RIB until all processes start normally*).

### What happens inside the supervisor during that kind of switchover

> **Disclaimer**: as EOT (*track commands*) do not support SSO, their relationship with SSO/NSF isn\'t publicly documented. So what follows are my general thoughts and speculations on what might be happening inside. I do not work for Cisco at the moment of this writing and cannot have access to that kind of detail, so don\'t rely on my words too much. I\'m probably wrong.

Initially, before the switchover event, the two supervisors had their FIB (CEF datastructures) synchronized. The routing table (RIB) is not, nor are the datastructures of any routing protocols running.

> *I\'ll assume EIGRP further on, but the program flow described should be very similar for other protocols as well. My other assumption is that the EOT is set to track a presence of a route in the routing table. This makes my argument so much easier! But it should also work in a very similar fashion for other track variations.*

As the switchover happens, new active supervisor takes control. It already has a working copy of CEF, so the data continues to flow [*almost*] without interruption. Also it is important here that at that moment, the RIB\'s state is undetermined (*it is empty for all practical purposes*). The supervisor continues to boot.

As it finishes booting, it starts the processes and they commence their work. Because IOS process scheduling is a variation on run-to-completion with FIFO discipline, every process will have a chance to do something useful (if it is not interrupted). Then it will either finish or return the CPU to the scheduler and go into waiting state.

In that time, the EIGRP process would say hello to it\'s neighbors and notify them (with a special flag) that it is undergoing a soft restart and needs data from them. After sending out these messages, the EIGRP goes to waiting state. The reason is that network communication will take time and it can\'t block the CPU for that long.

Other processes start as well, EOT included. EOT starts and following its configuration, checks for the tracked objects (*per my assumption, it is a route in RIB*). It will see that the route isn\'t there (*which is normal, as the RIB is empty*). EOT sets a down flag and notifies any process that is subscribed to that event via IPC (*inter-process communication*). It then relinquishes CPU control to the scheduler, which runs other processes in the queue.

At some point, some process (*which might in general support SSO very well*, *like BGP, OSPF, EIGRP, HSRP*) receives EOT\'s notification via IPC and acts accordingly, as configured. It might drop a neighborship, change priority, or filtering. What ever that is, **the state changes**.

After some time, the EIGRP process receives the updates from its neighbors, computes the routes, and sends that update to the RIB management process. RIB management process fills out the routing table using static routes and data it receives from routing protocols like EIGRP.

Notice that all that time, the system is fully capable of forwarding packets, as FIB was sort of frozen (i.e. the NSF was active). *But because of the state change above, the neighboring devices might have ceased to forward traffic through VSS pair or tore down neighborship relationship.*

When the RIB is complete and all the SSO-supporting processes report that they are done, the synchronization of RIB->FIB is reestablished, and eventually the FIBs of all CPUs, linecards, PFC/DFC are updated accordingly. The SSO/NSF procedure is complete.

Later on, at some point, the EOT process is scheduled to run again. When running, it sees that the tracked object (the route) is present; it puts it into *up* state and notifies the subscribed processes.

When the subscribed process runs, it reacts to the track state change by **changing its own state back**.

And we observe a flap, documented in the Cisco support case referenced above.

### Discussion

Notice here that because the order of process execution isn\'t really deterministic in IOS, we can\'t say that the order of events will always be the same. But the fact that, unlike EOT, a routing process like EIGRP depends on IO (*to send and receive packets*) and will have to wait before it can ready an update to RIB, lets us assume that any other process (including EOT) will have a chance to run at least once and do their duties as configured.

Given that a routing database update is rather big, it needs some quality time with IO to be fully loaded (CPU time-wise). I think it is logical to further assume that there are several transitions of the routing process (EIGRP) between running and waiting before it eventually synchronizes. That definitely gives enough time for the scenario I describe.

As a result, we can observe the flapping event, despite SSO/NSF working correctly. The inability of EOT to work with SSO is documented by Cisco (I reference just one document above, but there are others), so flapping is not a bug but really a feature.