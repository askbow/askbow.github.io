---
layout: post
title:  "Why Hulc LED process consumes so much CPU on 2960 platforms?"
date:   2017-10-17 14:51:00 +0100
categories: switching
---
In this post I\'ll try to make an educated guess about what happens with Hulc LED process and why it appears to consume 20-30% CPU on Cisco 2960(S/X/XR/RX) switches.

(*N.B.: the issue appears to be present on Cisco 3750 / 3560 platforms as well*)

# Symptoms

If you monitor your switch via SNMP, you may quickly notice constantly elevated CPU at about 20-40 % total. To investigate further, you get relevant command output from the device: #show process cpu sort

And the result looks something like this:

[![](https://askbow.com/wp-content/uploads/2017/10/sh-proc-cpu-hulk-led-e1508217172316.png)](https://askbow.com/wp-content/uploads/2017/10/sh-proc-cpu-hulk-led.png  "sh proc cpu")

or what we could call **Angry Hulc** process ;-)

# What is Cisco HULC?

According to Cisco document 64641 (*it\'s public somewhere on cisco.com*):

> The Mirage is based on HULC hardware architecture and the Sasquatch switching ASIC chipset from DSBU. HULC is a hardware architecture that you use to build low cost, stackable, 10/100/1000 Ethernet switches.
>  ...
> **Lord Of The Rings (LOTR)** - The the first release of the HULC based platforms and software based on the Sasquatch ASIC chip set from DSBU. The Mirage is based on LOTR platforms from DSBU.

* N.B.: DSBU - Data Switching Business Unit, a part of Cisco*)
* Also, notice the LOTR reference; for contrast, 4500 platform was built by Star Wars geeks

A quick look around shows that a lot of stuff is based on that machinery, starting with 3750 / 3560 and 2960 series, but later included SMB series like Express 500 switches, industrial Ethernet series. I\'d also assume that some spoils of that development went into 3850, 4500, 6800IA and later systems.

> *This gave me pause, because, judging by the docs and CiscoLive sessions, the stacks of 3750 and 2960 series differ quite a lot.*

It is reasonable to assume that all the multitude of hulc and /H.*/ (*HLPIP, HRPC, HL3U, etc.*) processes in IOS are talking to hardware in this architecture. For example, from [docs](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst2960/software/release/12-2_55_se/configuration/guide/scg_2960/swtrbl.html):

> - Hulc Forwarding TCAM Manager (HFTM)
> - Hulc Quality of Service (QoS)/access control list (ACL) TCAM Manager (HQATM)

We can also infer that commands under show platform stack talk directly to HULC.

## Ok, what\'s Hulc LED process anyway?

According to the description of the two most relevant Cisco bugs (CSCtg86211 and CSCtn42790), Hulc LED process is the thing that monitors port states, including PoE, transceiver (*think SFP*) status and sets the LED indicators accordingly. It also communicates with the MODE button, and resets the switch to factory default if you press it for too long:

```
%SYS-7-NV_BLOCK_INIT: Initialized the geometry of nvram
%SYS-5-RELOAD: Reload requested by Hulc LED Process. Reload Reason: Reason unspecified.
```

(*see Cisco FN - 63722 and bug CSCuj69384 for insight into why this is important*)

Sadly, the description in these bugs doesn\'t go beyond *this is an expected behavior*.

> ***Disclaimer**: This isn\'t publicly documented. What follows are my general thoughts and speculations on what might be happening inside. I do not work for Cisco at the moment of this writing and cannot have access to that kind of detail, so don’t rely on my words too much. I’m probably wrong.*

## How come it consumes so much CPU?

Here comes my theory. This might be completely bogus and is definitely based on a lot of assumptions, some of which are made up. But I tried to keep it realistic.

**Hulc LED process does not consume your CPU. It\'s an illusion created by processes\' waiting state.**

Taking into account that much of today\'s IOS has Linux blood in it\'s veins, that\'s not hard to imagine. That way, the command `show process cpu` and its derivatives don\'t show us actual CPU usage, but something closer in spirit to Linux load average: a decaying average over three time windows, in case of Cisco IOS - 5 Sec, 1 Min, 5 Min.

If we[ look closely](https://github.com/torvalds/linux/blob/master/kernel/sched/loadavg.c) at how data points for calculating load average are collected,

```c
long calc_load_fold_active(struct rq *this_rq, long adjust)
{
	long nr_active, delta = 0;

	nr_active = this_rq->nr_running - adjust;
	nr_active += (long)this_rq->nr_uninterruptible;

	if (nr_active != this_rq->calc_load_active) {
		delta = nr_active - this_rq->calc_load_active;
		this_rq->calc_load_active = nr_active;
	}

	return delta;
}
```

we could see that it takes load measurements for processes not only in the running state, but also in the *uninterruptible* wait state.

> *I would like to thank [Brendan Gregg and his post](http://www.brendangregg.com/blog/2017-08-08/linux-load-averages.html) for giving me this idea. His is a great writeup about the problem of load average in Linux.*

Let\'s consider what the Hulc LED process is doing.

It is communicating with peripheral (*from CPU\'s point of view*) devices: the ports are [probably] connected to the CPU complex via some serial bus. Hulc LED process polls every port. It does it by pulling port status register and setting the command register to blink the LED.

> Or something similar. It\'s a logical assumption from the fact that the more ports the switch has, the more load Hulc LED exhibits.

This is your basic IO operation. IO operations are slow and need to be completed atomically (*i.e. without interruption*) to prevent corruption / inconsistency. Hence, it\'s reasonable to assume that during this operation the process is put to TASK_UNINTERRUPTABLE state.

> Refer to [Understanding Linux Process States by Yogesh Babar, RedHat](https://access.redhat.com/sites/default/files/attachments/processstates_20120831.pdf) for details on what different states mean.

## The LED is not that bright

Despite what I\'ve said so far, Hulc LED process can still potentially consume too much CPU. That might be a symptom of faulty SFP modules, link flapping, or some very specific hardware problems in the switch. It\'s possible to think about several kinds of problems that will result in longer delay on system buses.

This would result in [apparent] very high (*beyond 30% of CPU*) consumption by Hulc LED process.

> The 30% here is rule-of-thumb-level-arbitrary (*also given in table 3 [here](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst3750/software/troubleshooting/cpu_util.html) so my guess is in the right ballpark*). Although in other Cisco documentation (*see BRKCRS-3141 2011 for example*) they state normal levels for some devices, you should always consider your environment and do a baseline analysis.

Moreover, there is also bug CSCvd68472 which can make Hulc LED process consume au pair with Hulc running con up to 100% CPU.

## Conclusion

To summarize, most of the time Hulc LED process on Cisco 3750/2960 platforms does not actually consume CPU for 20-30%, but rather is mostly waiting for its IO syscalls to finish for all that time. The system displays it as usage, because of the specifics of the algorithm used to calculate the load.

> I can counter my own argument: this might actually be a quirk of run-to-completion FIFO discipline used by Cisco IOS scheduler and have nothing to do with Linux.

Also notice, that this Cisco fixed this behavior in 15-something IOS branch for many hardware platforms, so your mileage may vary.