# Chapter 7 - Foundations 

Design for production means thinking about production issues as first-class concerns. 
- the production network, which might be considerably different from your development environment
- logging and monitoring, runtime control, and security 
- designing for the people who do operations, e.g. a dedicated ops team or devops
-- "Operators are users, too" 
-- Interact with your system through its configuration, control, and monitoring interfaces 

Layers of responsibility: 

#### Operations
Security, availability, capacity, status, communication

#### Control Plane
System monitoring, deployment, anomaly detection, features

#### Interconnect
Routing, load balancing, failover, traffic management

#### Instances
Services, processes, components, instance monitoring

#### Foundation
Hardware, VMs, IP addresses, physical network

## Networking in the Data Center and the Cloud

Different than development networks
- redundancy
- security
- virtualization

### NICs and Names

Host names can be defined in two ways 

*OS identifier*
The name an operating system uses to identify itself - output of “hostname” command.\
The administrator of the machine can set that hostname and the “default search domain.”\
Fully qualified domain name (*FQDN*): hostname + search domain 

*External name of the system.*
Other computers expect to connect to the target machine using this hostname.\
When a program tries to connect to a particular hostname, it resolves that name via DNS.\
DNS resolves the desired name to an IP address.

No technical requirement for the FQDN of a machine and the one the DNS has for its IP address to match.\
The machine FQDN can be “spock.example.com” but have a DNS mapping as “mail.example.com” and “www.example.com.” 
 
A machine uses its hostname to identify the whole machine, while a DNS name identifies an IP address.\
Multiple DNS names can resolve to the same IP address. 

For load-balanced services, a DNS name can resolve to multiple IP addresses (many-to-many).\
But the machine still acts as if it has exactly one hostname.

*Many utilities and programs assume that the machine’s self-assigned FQDN is a legitimate DNS name that resolves back to itself. 
Generally true for development machines and false for production services.*

There’s another many-to-many relationship in the mix as well. 
A single machine may have multiple network interface controllers (NICs).\
If you run ifconfig/ipconfig you’ll probably see several NICs listed.\
Each NIC can be attached to a different network.\
Each active NIC gets an IP address on its particular network ("multihoming").\
Nearly every server in a data center will be multihomed.

A dev box usually has multiple NICs for the sake of mobility. 
- a wired Ethernet port (for desktops/laptops that have wired Ethernet).
- Wi-Fi 
- A loopback NIC is a virtual device that handles 127.0.0.1

Data center machines are multihomed for different purposes. 
- enforce security by separating administration and monitoring onto a different network
- improve performance by segmenting high-volume traffic away from production traffic 

These networks have different security requirements.
An application that is not aware of the multiple network interfaces could end up accepting connections from the wrong networks.\
E.g. administrative connections from the production network or offer production functionality over the backup network.

#### Example 
*Production traffic interfaces* 
Handle the applications functionality.\
E.g. for a web server,  these handle the incoming requests and send the replies back.

By connecting the production interfaces to different switches, the server can be configured for high availability.\
The two interfaces might be load balanced, or they might be set up as a failover
pair.\
Two different IP addresses will get packets to this server, i.e. there's probably DNS entries for both addresses.\
E.g. this machine has both its own internal hostname (the hostname command output) but from the outside, more than one name reaches this host.

The two interfaces could also have been setup in a *bonding/teaming* configuration 
- common configuration 
-  both interfaces share a common IP address
- OS ensures that an individual packet goes out over only one interface
- can be configured to automatically balance outbound traffic or to prefer one link or the other
- Bonded interfaces that connect to different switches require some additional configuration on the switches, or else routing loops can result

*Backup interface*
Because backups transfer huge volumes of data in bursts, they can clog up a production network.\
Therefore, good network design for the data center partitions the backup traffic onto its own network segment. 
- sometimes handled by separate switches 
- sometimes just by separate VLANs on the production switches

With backup traffic partitioned off from the production network, application users don’t necessarily suffer when the backups run.\
(they might, if the server doesn’t have enough I/O bandwidth to process backups and application traffic at the same time. Nevertheless, users of other applications don’t suffer when this server is being backed up.)

*Admin interface*

Many data centers have a specific network for administrative access.
- security measure
--  Services such as SSH can be bound only to the administrative interface and are therefore not accessible from the production network 
-- Offers protection if a firewall gets breached or the server handles an internal application and doesn’t sit behind a firewall

### Programming for Multiple Networks
 
By default, an application that listens on a socket will listen for connection attempts on any interface.\
Language libraries always have overtly simple version of listening on a socket that just opens a socket on every interface on the host.\
*It is important that we instead specify which IP address we are opening the socket for:*

```
ln, _ := net.Listen("tcp", ":8080") // incorrect
ln, _ := net.Listen("tcp", "spock.example.com:8080") // correct
```

*To determine which interfaces to bind to, the application must be told its own name or IP addresses.* 
This is a big difference with multihomed servers. 
In development, the server can always call its language-specific version of getLocalHost(). 
On a multihomed machine, this simply returns the IP address associated with the server’s internal hostname. 
This could be any of the interfaces, depending on local naming conventions. 
*Server applications that need to listen on sockets must add configurable properties to define to which interfaces the server should bind.*

## Physical Hosts, Virtual Machines, and Containers

### Physical Hosts

CPUs are similar in development and data centers 
- multicore Intel or AMD x86 processor running in 64-bit mode 
- Clock speeds 

Data centers have gone from being all about vertical scaling to horizontal.

Some workloads require large amounts of RAM in the box (e.g. graph processing).\
The other specialized workload is GPU computing. Some algorithms are “embarrassingly parallel,” so it makes sense to run them across thousands of vector-processing cores.

Data center storage comes in many forms and sizes.\
A development machine typically has more storage than one of your data center hosts will have.\
The typical data center host only has enough storage to hold a bunch of virtual machine images and offer some fast local persistent space.\
Most of the bulk space will be available either as SAN or NAS, i.e. not actually on the host.  

To an application running on the host, though, both of them just look like another mount point or drive letter.\
Your application doesn’t need to care too much about what protocol the storage speaks. 
Just measure the throughput to see what you’re dealing with.\
Bonnie 64 will give you a reasonable view with a minimum of fuss.

### Virtual Machines in the Data Center


### Virtual Machines in the Cloud

Virtualization gives developers a common hardware appearance across all the physical configurations in the data center.\
It lets data center managers pack all those extra web servers running at 5 percent utilization into a high-density, high-utilization, easily managed whole.

Performance is however unpredictable.\
Many virtual machines can reside on the same physical hosts and rarely move between hosts, because it’s disruptive to the guest.\
The "host OS” is the one that really runs on hardware and provides the virtualization features.\
The “Guest OS” run in the virtual machines.

Physical hosts are usually oversubscribed.\
I.e. the physical host may have 16 cores, but the total number of cores allocated to VMs on the host is 32.\
That host would be 200 percent subscribed or 100 percent oversubscribed.\
If all those applications receive requests at the same time, just through random chance, then there’s not enough CPU to go around.

Almost any resource on the host can be oversubscribed, especially CPU, RAM,and network.\
Regardless of resource, it leads to contention among VMs and random slowdowns for all.\
Virtually impossible for the guest OS to monitor for this.

When designing applications to run in virtual machines you must make sure that they’re not sensitive to the loss or slowdown of any one host

Watch out for:
- Distributed programming techniques that require synchronous responses from the whole cluster for work to proceed
- “Special” machines like cluster managers or lock managers, unless
another machine can take over without reconfiguration
- Subtle dependency on request or event ordering—nobody designs this into a system, but it can creep in unexpectedly

Virtual machines make clock problems worse.\
Incorrect mental model of the clock as being monotonic and sequential is common.\
I.e. a program that samples the system clock may get the same value twice, but it’ll never get a value less than a prior response.\
Not even true for a clock on a physical machine. 

On a virtual machine it can be much worse.\
*Between two calls to examine the clock, the virtual machine can be suspended for an indefinite span of real time.\*
It might even be migrated to a different physical host that has a clock skew relative to the original host.\
*A clock on a virtual machine is not necessarily monotonic or sequential.*\
The virtualization tools compensates by having the VM query the host so the VM can update its OS clock whenever it wakes up.\
That keeps the VM’s OS clock synced with the host’s OS clock.\
From an application perspective, this makes the clock jump around even more. 

*Don’t trust the OS clock. If external, human time is important, use an external source like a local NTP server.*

### Containers in the Data Center
Containers are widely used in data centers.\
Provides process isolation and packaging of a virtual machine together with a developer-friendly build process.\
Ensures that production matches QA.

Containers in the data center act a lot like virtual machines in the cloud.\
Any individual container only has a short-lived identity.\
*This, it should not be configured on a per-instance basis.*\
This can cause interesting effects with older monitoring systems that need to be reconfigured and bounced every time a machine is added or removed.

A container won’t have much local storage, so the application must rely on external storage for files, data, and maybe even cache.

The most challenging part of running containers in the data center is the network.\
By default, a container doesn’t expose any of its ports (on its own virtual interface) on the host machine.\
You can selectively forward ports from the container to the host, but then you still have to connect them from one host to another.

One common pattern that’s developing is the overlay network.\
Uses virtual LANs (VLANs) to create a virtual network just among the containers.\
The overlay network has its own IP address space and does its own routing with software switches running on the hosts.\
Within the overlay network, control plane software manages the whole ensemble of containers, VLANs, IPs, and names.


It is also tricky to make sure enough container instances of the right types are on the right machines.\
Containers are meant to come and go—part of their appeal is their very fast startup time (range of milliseconds). 

Containers are automatically operated by control plane software.\
We describe our desired load out of the containers, and the software spreads container meringue across the physical hosts.\
The control software should know about the geographic distribution of the hosts, so it can allocate instances regionally for low latency while maintaining availability in case you lose a data center.

The control plane should ideally also schedule container instances and manage their network settings. 

There's many competeting solutions for running containers in data centers - 
Kubernetes, Mesos, and Docker Swarm are attacking both the networking and allocation problem.

When you design an application for containers, remember:
- The whole container image moves from environment to environment.
-- *Can’t hold things like production database credentials. Credentials all have to be supplied to the container* 
-- A 12-factor app handles this naturally 
-- If not using that style, consider injecting configuration on container start-up 
-- In either case, use password vaulting
- externalize networking 
-- Container images should not contain hostnames or port numbers 
-- Setting needs to change dynamically while the container image stays the same 
-- Links between containers are all established by the control plane when starting them up
- Containers are meant to start and stop rapidly
-- Avoid long startup/initialization sequences. 
-- Production servers take many minutes to load reference data or to warm up caches are not suited for containers 
-- *Aim for a total startup time of one second*
- Notoriously hard to debug an application running inside a container. 
-- Just getting access to log files can be a challenge 
-- *Containerized applications, even more than ordinary ones, need to send their telemetry out to a data collector*

### Virtual Machines in the Cloud

Evident now that traditional applications can run in the cloud. 
Requires some “lift and shift” efforts tho.\
A cloud native system will have better operational characteristics, especially in terms of availability and cost.

Any individual virtual machine in the cloud has worse availability than any individual physical machine (assuming equally skilled data center engineering and operations). 
- A cloud VM runs atop a physical host, but with an extra operating system in the middle. 
- Can be started or stopped without notice by the management APIs (i.e. “control plane” software.) 
- Shares the physical host with other VM's and may contend for resources 
-- Common in AWS that machines are randomly killed
- Ephemeral machine identity. A machine ID and its IP address are only there as long as the machine keeps running
-- Most traditional application configurations keep hostnames or IP addresses in config files
-- In AWS, a VM’s IP address changes on every boot 
--- If your application needs to keep those addresses in files, then you have to rent Elastic IP addresses from Amazon 
---  Works well enough until you need a lot of them. A basic AWS account has a limit on how many addresses it can procure
- VMs have to “volunteer” to do work, rather than having a controller dole the work out.
- A new VM should be able to start up and join whatever pool of workers handles load. 
-- For HTTP requests, autoscaling and load balancers (either elastic load balancers or application load balancers) are the way to go 
--For asynchronous load, use competing consumers on a queue.

- The default cloud VM network interface is one NIC with a private IP address. 
-- There’s a limit to how much traffic a single NIC can support, based on the number of sockets available
-- Socket numbers only range from 1 to 65535, so at best a single NIC can support about 64,000 connections
-- Can be necessary to setup more production NICs to handle more simultaneous
connections 
- A dedicated  NIC is for monitoring and management traffic is a good idea 
-- In particular, it’s a bad idea to have SSH ports available on front-end NICs for every server
-- Set up a single entrypoint (a “bastion”/“jumphost” server) with strong logging on SSH connections and then use the private network to get from there to other VMs

### Containers in the Cloud
Containers on cloud VMs combine the challenges of both containers and the cloud.\
The containers have short-lived, ephemeral identities.\
Connecting them means linking ports across different VMs, possibly in different zones or regions.\
Designing individual services to run in this kind of deployment is not that much different from designing them to run in containers in the data center.\
Most of the big challenges arise from building those containers into a whole system. 
*Using containers push complexity out of the boxes and into the control plane.*

##### Virtual LANs for Virtual Machines
The idea of a VLAN is to multiplex Ethernet frames on a single wire but let the switch treat them like they came in from totally separate networks.\
The VLAN tag is a number from 1 to 4,094 that nestles into the physical routing portion of the header.

The operating system that runs a NIC can create a virtual device assigned to a virtual LAN.\
All the packets sent by that device will have that VLAN ID in them.\
That also means the virtual device must have its own IP address in a subnet assigned to that VLAN.

VXLAN takes the same idea but runs it at “layer 3,” meaning it’s visible to IP on the host.\
It also uses 24 more bits in the IP header, so a physical network can have more than 16 million VXLANs.

Virtualization and containers increasingly rely on software switches (instead of hardware) to handle dynamic updates.\
It will be common to see software switches running on the hosts, presenting a complete network environment to the containers that
- Allows containers to “believe” they’re on isolated networks
- Supports load-balancing via virtual IPs
- Uses a firewall as a gateway to the external network

For now our container systems have to provide their own loadbalancing and need to be told which IP addresses and ports their peers are on.

#### The 12 factor app 
Succinct description of a cloud-native, scalable, deployable application.\
Also makes a great checklist for application developers not running on the cloud.

The “factors” identify different potential impediments to deployment and recommended solutions:
- Codebase
-- Track one codebase in revision control. Deploy the same build to every environment.
- Dependencies
-- Explicitly declare and isolate dependencies.
- Config
-- Store config in the environment.
- Backing services
-- Treat backing services as attached resources.
- Build, release, run
-- Strictly separate build and run stages.
- Processes
-- Execute the app as one or more stateless processes.
- Port binding
-- Export services via port binding.
- Concurrency
-- Scale out via the process model.
- Disposability
-- Maximize robustness with fast startup and graceful shutdown.
- Dev/prod parity
-- Keep development, staging, and production as similar as possible.
- Logs
-- Treat logs as event streams.
- Admin processes
-- Run admin/management tasks as one-off processes.