---
layout: post
title:  "What happens if you start a Cisco 6500 switch without the fan module?"
date:   2017-05-30 12:44:00 +0100
categories: switching
---
Recently, I tested a Cisco 6500 switch in a fan-less configuration, to see how long it can go.

> **DISCLAIMER**: **DO NOT TRY TO DO IT**. This is a stupid idea and it will void warranty / would be a perfectly valid reason for Cisco to decline RMA (*in my opinion at least*). Running a switch without fans will be directly damaging to active components (*ASICS, TCAM, CPU etc*) and increase wear-out of passive ones (*capacitors etc*). I did it in the lab so you don\'t have to.

Initially, I would\'ve guessed that you can run the system w/o the fan module for about five minutes. That should give enough time to [replace](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst6500/hardware/Chassis_Installation/Cat6500/6500_ins/04remrep.html#49856) (swap, or clean) the fan module if needed.

### Fanless test results

That\'s not exactly the case:

1. right away, as the CPUs heated up, the system became slow to respond to direct console connection
2. in a minute, I\'ve got these logs:

`*May 30 07:24:53.619: %C6KENV-SW2_SP-4-INSUFFCOOL: Module 1 cannot be adequately cooled`

`*May 30 07:24:53.707: %C6KENV-SW2_SP-4-INSUFFCOOL: Module 3 cannot be adequately cooled`

`*May 30 07:24:55.539: %C6KENV-SW2_SP-4-FANCOUNTFAILED: Required number of fan trays is not present`

`*May 30 07:25:20.215: %C6KENV-SW2_SP-4-MINORTEMPALARM: switch 2 RP 5/0 inlet temperature crossed threshold #1(=50C. It has exceeded normal operating temperature range.`

`*May 30 07:27:07.183: %C6KENV-SW2_SP-4-MINORTEMPALARM: switch 2 module 5 asic-1 temperature crossed threshold #1(=. It has exceeded normal operating temperature range.`

`*May 30 07:27:26.387: %C6KENV-SW2_SP-4-MINORTEMPALARM: switch 2 EARL 5/0 outlet temperature crossed threshold #1(=. It has exceeded normal operating temperature range.`

In short, it took a minute w/o fans for the test system to start overheating.

The EnvMon (*as far as I understand*) would shut the system down if it went too far above the temperature range red line. But I didn\'t go that far, because this wasn\'t the purpose of my test.

### Why we need the fans

A supervisor in a Cisco 6500 switch may consume around 250-500 W from a power supply. Most of that energy is actually burned away, translating into heat *(thermal energy)*.

The heat is dissipated by the chip surface into the environment. The bigger the surface, the more heat a chip can dissipate. The radiators glued on top of the chips increase the dissipating surface.

The format of the chassis cards dictates the size of the radiators installed on the chips: they are necessarily small to fit in the slot height.

Hence, to provide enough cooling (i.e. take enough heat from the chip and its radiator) we need to force air through. To that end, we use fans (*I won\'t go into liquid cooling here, but the basic principle is the same*).

By removing the fan, I let the heat stay in the chip. I wasn\'t able to find a datasheet on the SR71000 processor (*owned by Broadcom, who are too shy to publish anything*) used as SP and RP in Sup720, but as a reference, [Intel CPUs](https://www-ssl.intel.com/content/www/us/en/processors/core/2nd-gen-core-lga1155-socket-guide.html) (*I\'m specifically choosing to look at an older generation Intel CPUs here, as I hope the tech of SR71000 is about the same age*) are tested up to about 70 degrees Celsius. Given the warning at 50 degrees I\'ve got in the logs, that seems to be a reasonable estimate.

### Test environment:

1. WS-C6506-E with Sup720
2. Two Gigabit linecards inserted so to have at least a single free slot between any two cards
3. linecard slot blanks removed
4. it was an isolated lab system without any traffic to load it
5. AC in the room providing a steady 23 degrees Celsius