---
layout: post
title:  "Paper: Scanning the Internet for Liveness"
date:   2018-06-14 09:20:00 +0100
categories: worth_reading, networking, research
---
An interesting paper where the authors are building a better way to scan the Internet.

[https://sheharbano.com/assets/publications/ccr18-scan-liveness.pdf](https://sheharbano.com/assets/publications/ccr18-scan-liveness.pdf)

*Shehar Bano et al. Scanning the Internet for Liveness // ACM SIGCOMM Computer Communication Review, Volume 48 Issue 2, April 2018*

> Liveness—whether or not a target IP address responds to a probe packet—is a nuanced concept without a simple yes/no answer. Responsiveness directly depends on the probe type, the configuration of the targeted host, as well as on firewalling and filtering behaviors at the edge or within networks.

Key findings include:

> - TCP and UDP probes increase the population responsive over ICMP by 18%,
> - comprehensively capturing reply traffic (i.e., taking into account negative reply packets) increases the responsive population by more than 13%,
> - TCP stacks do not consistently respond with a TCP Rst for non-available services—in our measurements only 24% of hosts with an active TCP stack respond to all the probes,
> - our concurrent scans allow us to identify nearly 2M tarpits that would bias measurements that do not take them into account, and
> - we report on the correlation of responsiveness across protocols uncovering potential filtering practices.

Other things I found interesting:

> - probe redundancy [sending deferred repeated probes] increases the population of active IP addresses by 2.2%
> - scans recorded 487M network alive IPs (IPall) out of 3.6B probed.
> - they see that ICMP Echo probes are most effective in discovering network active IPs, revealing 79% of IPall, followed by TCP probes.
> - they found that 16% of IPall can only exclusively be discovered via TCP, and a small but significant ≈2% can only be discovered via UDP probes.