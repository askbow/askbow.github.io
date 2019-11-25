---
layout: post
title:  "Cisco UCS HyperFlex FE class notes"
date:   2016-07-16 21:45:00 +0100
categories: sandbox
---
Recently I attended a two-day class on Cisco HyperFlex by Tomaz Klancnik from NIL. It was quite interesting and packed with information about this new Cisco\'s system. Here are the notes I took during the lectures and the labs.

### General Notes about Cisco HyperFlex

- Not developed internally; Not a complete acquisition / spin-in; software developed by Springpath [https://springpathinc.com/resources.php](https://springpathinc.com/resources.php)
  - Springpath name can be seen in some of the log messages, despite big Cisco logo at the top
  
  
  - Looks like it runs on java

- Silo breaking narrative

- Cisco Hyperflex differs by integrating network into the solution, gets you faster deployment of the system

- rumor has it, Cisco wants to catch up to nutanix in two year span

- Vmware lockin now, KVM in the roadmap

- Comm latency is expected to be in order of disk write latency
- Max.nodes 8, but there is a promise of 16 (near future)

- The whole Hyperflex line is still in it\'s infancy really, will develop more capabilities as it matures - *if Cisco doesn\'t kill it like UCS M-series before*

- ESXi comes with the boxes, customized image

- All-flash nodes are coming

- The FIs are the usual ones, it\'s more politics that they require to buy a new pair

- Most of the system management is done through vcenter, but some things can only be done in CLI

- Rolling upgrade procedure with non-disruptive VM movement (DRS / vmotion)

- 3 (2)-node data spreading across the cluster
  - Lenient policy - at least one node up is enough
  
  
  - Strict policy - at least two nodes up for a healthy cluster

- Requires external shared services (vcenter, DHCP, NTP, DNS, AD) be present at deployment time

- Uses Cisco VIC1227 in common nodes

- Cluster deployment takes like 25 mins
- Can\'t mix c220 and c240 at the time

- Deduplication / compression is always on

- 48 / 72 GB of RAM is consumed on each node by the cluster process (Controller VM)

- Re-balancing is delayed up to an hour

- failover is done at vmware software level

- Only vNICs to be active-active are vm-network(a,b)
   others include hv-mgmt(a,b), hv-vmotion(a,b), storage-data(a,b)

- Hyperflex Integration with ACI in the future: get rid of FIs, connect HX nodes to ACI leafs directly

- Use cases:
  - VDI - primary marketing effort focus, low upfront cost, can put an nVidia GRID K1 / K2 GPU inside a node
  
  
  - Server virtualization - scaling, resiliency
  
  
  - Test & dev - agile provisioning and stuff
  
  
  - Not really meant for high-performance low-latency scenario, more focused on cases of simplicity

### Dirty details aka deep dive

- Data integrity was a design goal
- Deduplication done inline in the cache->HDD path
- Expect 30-50% compression ratio
- Callhome and telemetry - autosupport feature

- Controller VM on each node takes care of caching, deduplication etc
- VAAI (vmware vstorage API for array integration) plugin offloads snapshot / cloning
- IOvisor module handles disk IO, data striping to 3 locations, presents NFS abstraction to ESXi

- Backend FS is transactional, waits for 3-node confirmation before purging cache

- Pointer-based log-like data writes to disks; reversion (to a snapshot) is just changing metadata block back
  - Interesting: it creates a special snapshot called SENTINEL for each VM:
      [![The](https://askbow.com/wp-content/uploads/2016/07/cisco-ucs-hx-vm-snapshot-sentinel-300x231.png)](https://askbow.com/wp-content/uploads/2016/07/cisco-ucs-hx-vm-snapshot-sentinel.png)
- A cleaner process runs periodically to purge old chunks
- Space reserve for metadata / reversion copies at 88% (recommended, not enforced) of raw capacity-> important for capacity planning
- They don\'t use erasure coding methods to spread data - just copies
- Cache is pooled across the nodes - *no guarantee of locality*
- No public information from Cisco yet on expected IOPS (performance-wise) - *I was able to find some data on the Springpath website, see link above*
- Compute-only nodes still must have all the software components in place
- The controller VM is bound to a host and looks like this: 
  [![Cisco](https://askbow.com/wp-content/uploads/2016/07/cisco-ucs-hx-vm-controllervm-300x231.png)](https://askbow.com/wp-content/uploads/2016/07/cisco-ucs-hx-vm-controllervm.png)
- If the controller VM drops dead, you loose HDD / cache of that node; but no data loss is expected, as there are copies of the same data somewhere else in the cluster
- No data migration expected on vmotion event - less network load
- Single subnet per HyperFlex cluster, leverage FI-local switching
- When a node fails, for 2 hrs nothing happens - just in case it comes back (*for example in an upgrade scenario*); after that the cluster re-balances
- Cluster expansion is done using the Installer VM, the same used for deployment
- Upgrades require DRS and disable auto-healing capability; nodes are added one-by-one

#### Data virtualization

- pNode - physiscal node
- Vnode - logical node, many to a pnode
  - Cache vnode
  - Data vnode - uniform resource utilization, load balancing, expedite rebuilds
  - Metadata vnode
- IO Ack after cache commit (write log)
- Write-through is default - Data being written is placed to read cache immediately on ACK - optimization for some cases when written data is read back right away
- Write-back is possible if preferred
- Data and metadata can be placed to different nodes

#### Data optimization

- Dedup saves 20-30% space
- Compress saves 30-50% space
- Best-effort data optimization to prevent performance penalty
- Performed locally at each node
- Data read
  - VM asks for data
  - Iovisor module looks up for data
  - Multi-level caching
- Log-structured filesystem
  - Data chunks are fixed-size

### UCS Components in HyperFlex

- UCS manager runs on UCS fabric interconnect (FI) as usual
- FabricInterconnect is just a switch here - as usual
- N.B.: FEX was named so because of Switch Extender abbreviation, true story
- Dedicated VLANs between UCS servers and FIs for UCS manager communication
- Uniqueness of identifiers is ensured through pools (MAC pool, UUID pool, WWN pool, ...)
- Hyperflex at the moment supports UCS6200 FIs
- Selection of UCS components for HX is based on market popularity
- UCS 2.2(6f) is the minimum supported version

### Data platform install, HX cluster deployment

- *CVD is coming*
- Capacity planning ○ #HDD \\times Size \\times #Nodes
  - Metadata overhead, ENOSPC buffer +12% (it is 10% fudged) - also see notes on 88%
  - Divide by replication factor
  - Dedup + compress at 20%
- Prereqs:
  - Download OVA from cisco
  - Needs vcenter pre-existing, can be moved in later
  - Create manually a DC cluster in vmware for HX
  - Set IP addresses on ESXi nodes
  - Put nodes to maintenance mode prior deployment
  - Soft:
      - UCS 2.2(6f)
      - ESXi 6.0 U1b
      - Hxdp 1.7.1 (hyperflex data platform itself)
  - Hard
      - At least two 10Gb line per server
  - Host
      - Adder to a vcenter cluster
      - Added to a vcenter datacenter
      - Same VLAN ID
  - Network
      - Allow for 4 vlans to be used
      - Ports: http, https, kvm, ssh…
      - Create an IP pool, one IP per host
      - NTP, DNS, TZ set
  - Hypervisor
      - Management, storage / data,storage IP per ESXi node
      - Storage controller VM thus has two IP addresses: mgmt and data
      - Same goes for the Installer appliance you\'ve downloaded
      - Having DRS enabled is a good idea - *not sure if a prereq*
      - Vmotion and HA must be enabled
      - User VMs must be version 9 or later
- Everything UCS-wise is predefined - policies and profiles[![cisoc](https://askbow.com/wp-content/uploads/2016/07/cisco-ucs-hx-vm-deployment1-300x231.png)](https://askbow.com/wp-content/uploads/2016/07/cisco-ucs-hx-vm-deployment1.png)[![cisoc](https://askbow.com/wp-content/uploads/2016/07/cisco-ucs-hx-vm-deployment2-300x231.png)](https://askbow.com/wp-content/uploads/2016/07/cisco-ucs-hx-vm-deployment2.png) [![cisoc](https://askbow.com/wp-content/uploads/2016/07/cisco-ucs-hx-vm-deployment3-300x231.png)](https://askbow.com/wp-content/uploads/2016/07/cisco-ucs-hx-vm-deployment3.png)
- Do some other stuff - connect to hosts, set ESXi IP address, add nodes to cluster[![cisco](https://askbow.com/wp-content/uploads/2016/07/cisco-ucs-hx-vm-cluster-300x231.png)](https://askbow.com/wp-content/uploads/2016/07/cisco-ucs-hx-vm-cluster.png)
- Click validate, then Deploy[![cisco](https://askbow.com/wp-content/uploads/2016/07/cisco-ucs-hx-cluster-delpoyment1-300x231.png)](https://askbow.com/wp-content/uploads/2016/07/cisco-ucs-hx-cluster-delpoyment1.png) [![cisco](https://askbow.com/wp-content/uploads/2016/07/cisco-ucs-hx-cluster-delpoyment2-300x231.png)](https://askbow.com/wp-content/uploads/2016/07/cisco-ucs-hx-cluster-delpoyment2.png) [![cisco](https://askbow.com/wp-content/uploads/2016/07/cisco-ucs-hx-cluster-delpoyment3-300x231.png)](https://askbow.com/wp-content/uploads/2016/07/cisco-ucs-hx-cluster-delpoyment3.png)
- HX data platform does its cluster creation automagically
  - Scripted factory configuration
  - Configures FI ports, QoS, server policies
  - Creates an organization in FI
  - Configures 3 VLANs for fabric a / b
      - MTU 1500 for mgmt and netw, 9000 for vMotion and storage
      - 4 vSwitches are created in ESXi
      - vMotion prefers FI-b
      - Storage Data traffic prefers FI-a
      - VM network MUST be user-defined
- Wait an hour (we had enough time for lunch during the lab part while the automation was doing its job)
- N.B.: parts of configuration can be imported from a JSON file
- After all is done, re-login to vSphere client, go to vCenter inventory Cisco HX Data Platform, et voila:[![cisco-ucs-hyperflex](https://askbow.com/wp-content/uploads/2016/07/cisco-ucs-hx-vm-storage1-300x231.png)](https://askbow.com/wp-content/uploads/2016/07/cisco-ucs-hx-vm-storage1.png) [![cisco](https://askbow.com/wp-content/uploads/2016/07/cisco-ucs-hx-vm-storage2-300x231.png)](https://askbow.com/wp-content/uploads/2016/07/cisco-ucs-hx-vm-storage2.png)
- Adding a host to the cluster is a similar process

#### Scalability

- Independent compute node scaling
- Add nodes to the cluster
- Add compute-only node
- Add disks to nodes (possible with UCS C240)

#### Data service

- Cloning, snapshots, scheduled snapshots
- VmWare limits to 30 snapshots per VM
- Cloning is basically a writable snapshot for that matter
- Up to 256 clones per job - all done in parallel

#### performance monitoring

- HCIbench or IOmeter can be used
- https://<cluster_ip>/perf - internal tool, shows IOPS, latency, bandwidth per cluster and per-host
- Similar data available via vCenter client interface along with alarms and event log

### Management

- Everything is done via vCenter / plugin
- Cluster expansion is through the deployment VM
- Capacity and dedup performance can be seen in vCenter
- Some troubleshooting stuff can be done via controller VM CLI:[![cisco-ucs](https://askbow.com/wp-content/uploads/2016/07/cisco-ucs-hx-cluster-cli1-300x231.png)](https://askbow.com/wp-content/uploads/2016/07/cisco-ucs-hx-cluster-cli1.png) [![cisco-ucs](https://askbow.com/wp-content/uploads/2016/07/cisco-ucs-hx-cluster-cli2-300x231.png)](https://askbow.com/wp-content/uploads/2016/07/cisco-ucs-hx-cluster-cli2.png)
- Logs it produces:[![cisco](https://askbow.com/wp-content/uploads/2016/07/cisco-ucs-hx-cluster-cli-logs-300x231.png "hyperflex")](https://askbow.com/wp-content/uploads/2016/07/cisco-ucs-hx-cluster-cli-logs.png)
- `Stcli cluster info` - The output is extensive, shows all the health status, versions, IP addresses etc

### Disaster Recovery

- No built-in tool
- External tools like SRM, Veeam
- The vCenter plugin that comes with the system provides a neat scheduled VM snapshot feature
- Resiliency
  - Replication factor - 2 or 3
      - Loose part of raw capacity, 1/n remains available
  - Re-balance - self-heal and capacity changes
  - Failures
      - Simultaneous - a failure while the system is not healthy - puts system offline (read-only)
      - Non-simultaneous - a failure while the system Is healthy after self-healing
  - Fault-tolerance - cant tolerate a failure that puts system offline
  - Policy -
      - Strict is the default now
      - Lenient is promised to be the default in the future; only option for replication factor of 2
- You MUST have more than a half of your cluster alive for it to work - avoid split-brain
- Rebuild from node failure auto starts in 2 hrs, but can be initiated from the CLI
- Disk / SSD failure healing starts in 1 min
  - If a disk fails after a node failure, larger timeout will be used
  - Caching SSD failure - lost cache capacity, no service interruption expected
  - Housekeeping SSD failure - node becomes compute-only

### Disclaimer

I work for a Cisco partner VAR. The HX FE class I attended was provided free of charge for me and, to the best of my knowledge, for my employer. All opinions expressed are my own, nobody asked me to write about all this.

Also, any performance data is only relevant at the time of publication. These things might change quickly, i.e. your mileage may vary.