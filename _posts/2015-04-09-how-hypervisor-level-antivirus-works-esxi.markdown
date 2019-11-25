---
layout: post
title:  "How hypervisor-level antivirus works (ESXi)"
date:   2015-04-09 11:59:00 +0100
categories: security
---
There seems to be an ongoing battle between some people who want an antivirus in every virtual machine and some other people who do not see any benefit it wasting cycles (and memory) on it.

Recently, I discovered for myself that there is a middle-ground solution: to run an antivirus centrally for every virtual machine present on the host. Some people even go as far as to call it a hypervisor-level antivirus. Well, this description is just wrong - it does not work at the hypervisor level. It is a separate virtual machine which gets to check merely the filesystem operations happening in guest operating systems running on the same host.

> *A little note: I call it antivirus [software], but it might be called anti-malware as well. For this blogpost any possible distinction does not matter.*

### How they plug an antivirus to a hypervisor

The key to this solution in the world of VMware virtualization is the vShield bunch of products. A vShield Endpoint ESX module runs inside ESXi hypervisor to interact both with a special driver and the antivirus virtual machine.

> So, in a sense, there is a hypervisor-level machinery in the works and it assists in antivirus checks by passing messages and data between components (like a MUX), but that\'s all.

To make this work, there is an interaction between several components:

- VMware ESXi - the (obvious) hypervisor running the business of resource allocation and connecting things together. Any edition except Essentials;
- VMware vCenter Server - provides general administration facilities;
- VMware vShield Manger - manages vShield configuration, installs vShield Endpoint ESX module on ESXi hosts and registers the antivirus virtual machines;
- vShield Endpoint ESX module - interacts with the vShield Endpoint Thin Agent driver (installed in virtual machines with VMware Tools under VMCI Driver->vShield Drivers category) and with EPsec library in the antivirus virtual machine
- Antivirus solution management (control) panel - used to control virtual machines performing antivirus checks; provided by your antivirus software vendor;
- *Finally*, the antivirus virtual machine - contains vShield Endpoint EPSEC library and your trustworthy antivirus software

Here\'s some diagram to make the picture more complete:

[![How](https://askbow.com/wp-content/uploads/2015/04/avfile.png)](https://askbow.com/wp-content/uploads/2015/04/avfile.png)

### How this kind-of hipervisor-level antivirus system works in a nutshell

The interaction between components goes like this:

1. An application tries to open, save or execute a file on a virtual machine
2. The vShield Endpoint Thin Agent driver intercepts the filesystem call (just like a general antivirus does) and sends it to the vShield Endpoint ESX module via the VMCI API. If the request hits the cache, the cached result is returned to the application
3. The vShield Endpoint ESX module relays the request to the EPsec library in the antivirus VM, via VMCI API
4. Then the EPsec library sends this data to the antivirus engine. The antivirus requests file data and receives it either from the EPsec lib\'s cache or from the thin agent driver
5. The antivirus engine checks the file data for viruses and other malware 
  - If the file data is clean, then the OK result is sent back to the endpoint thin agent, and the application is able to complete the file operation
  - If the file data contains badware, the antivirus engine performs actions according to its parameters (for example, forbids the operation)

### Why this system is good

Well, the obvious good things about it are:

- You don\'t need to install antivirus agent software on your machines - *but you still need to make sure the Thin Agent Driver is installed*
- You centralize antivirus operations for a physical host, so you might be able to spend your physical host resources more efficiently
- Less administrative overhead - you manage one antivirus instance per ESXi host, in particular you need to upload updates to this special VM only
- There might be some licensing benefits from the antivirus vendor

### Now, why this antivirus scheme isn\'t so good

First of all, this solution works with file access and file access alone. As every antivirus vendor tells us nowadays,[[citation needed](#)] scanning files alone is not enough anymore.

Just look at any of today\'s antivirus vendor (at least the big ones) website: their general host virus protection solutions include packet inspection, real-time behavioral analysis, HIPS, device control, application control, sandboxing - everything but the kitchen sink.

So, this hypervisor-level antivirus is able to scan files and is limited to just that in its ability to protect VMs from threats out there.

Furthermore, even though they call it an agentless antivirus solution, the Thin Agent Driver is still an agent, which performs certain tasks: besides passing data to the ESXi vShield module, it caches results and returns an OK status for files excluded scanning. And you have to install it, just like an agent.

> Some sources suggest that the EPsec Thin Agent driver is Windows-only. I question the popular belief that Linux (or other) systems need no antivirus protection. The servers are ok - they are protected by careful and motivated admins. Windows servers usually are kept in a pretty good state as well, btw.
> 
> It is the lot of end users and workstations that needs protection, because users are careless and applications are not designed with security in mind.

Finally, this system introduces a lag into file system operations. I am not sure about the actual limits, but applied queuing theory suggests that since the antivirus virtual machine can handle only a limited number of transactions simultaneously (being limited by memory and CPU), then some file requests will be queued (delayed) for some time.

Look at this pretty diagram, illustrating timeline of a file operation in this system:

[![Timeline](https://askbow.com/wp-content/uploads/2015/04/vshield-endpoint-timeline.png)](https://askbow.com/wp-content/uploads/2015/04/vshield-endpoint-timeline.png)On this diagram we can see that the best case would be that the result is already in the local driver\'s cache (or excluded from scan). And the worst case happens when we get to scan the file data, and totally miss all the caches. The time it takes to scan is dynamic and depends on the antivirus system load at any given moment. Thus, not only we get a delay in file operations, but we get a significant jitter as well.

### Conclusion

Of course it is up to the systems and security engineers designing the system to decide whether this solution is viable and worth the implementation effort.

Personally, I would limit the use of this kind of protection to VDI systems with lots of [careless] users to protect from themselves.