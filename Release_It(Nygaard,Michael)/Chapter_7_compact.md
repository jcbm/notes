# Chapter 7
- Beware of differences in production and dev environments - mainly network 
- 2 types of host names 
- Server multihoming 
- High availability NIC config 
- Bonding/teaming NIC config 
- VMs in DCs 
- Containers in DCs
- VMs in the cloud 
- Containers in the cloud 

Important to think about production during development
- Very different networks
- logging and monitoring, runtime control, and security
- keep operations in mind - interact with your system through configuration, control, and monitoring interfaces

## Networking in the Data Center and the Cloud

Differs from development networks
- redundancy
- security
- virtualization

### NICs and Names
Host names can be defined in two ways
- OS identifier
-- name the OS uses to identify itself
-- set by admin along with a "default search domain"
-- Fully qualified domain name (FQDN): hostname + search domain
- External name of the system
-- name resolved by DNS by others 

Multiple DNS names can resolve to the same IP address (*:1)
For load-balanced services, a DNS name can resolve to multiple IP addresses (1:*).
The individual machine still acts as if it has exactly one hostname.

Many programs assume that the machine’s self-assigned FQDN is a legitimate DNS name that resolves back to itself. 
Generally true for development and false for production.


A single machine may have multiple network interface controllers (NICs).
Action: Run ifconfig/ipconfig to list NICs
Each NIC is attached to a different network and is associated with an IP address on its particular network ("multihoming").
Most servers in a data center are multihomed.
- more secure - separates administration and monitoring onto a different network
- improved performance by segmenting high-volume traffic away from production traffic

Beware: An application that is not aware of the multiple network interfaces could end up accepting connections from the wrong networks.

Example of multiple NICs:
- Production NICs 
- Backup NIC
- Admin NIC

Use multiple production traffic interfaces to handle the applications functionality (e.g. 2).
**High availability**:
Connect the production NICs to different switches.
Can either be
- load balanced
- set up as a failover pair
NIC correspond to two different IP adresses. DNS must be set up with entries for both.

**Bonding/teaming configuration**
- interfaces share a common IP address
- OS ensures that an individual packet goes out over only one interface
- can be configured to automatically balance outbound traffic or to prefer one link or the other

Backup interface - backups transfer huge volumes of data in bursts - can clog up network.
Thus partitioned to its own NIC
Can be handled by 
- separate switches
- separate VLANs on the production switches

Admin interface
Many data centers have a specific network for administrative access.
Security measure 
- Services such as SSH can be bound only to the administrative interface and are therefore not accessible from the production network 
- Offers protection if a firewall gets breached or the server handles an internal application and doesn’t sit behind a firewall


### Programming for Multiple Networks
An application that just listens on a socket will listen for connection attempts on any interface.
Common mistake in language libraries.
Do: Specify which IP address we are opening the socket for:

```
ln, _ := net.Listen("tcp", ":8080") // incorrect
ln, _ := net.Listen("tcp", "spock.example.com:8080") // correct
```

The application must be told its own name or IP addresses. 
- development
-- Use getLocalHost() or equivalent methods in other language
- multihomed machine
-- Don't: getLocalHost() returns the IP address associated with the server’s internal hostname
-- can be any of the interfaces, depending on local naming conventions 
-- Do: Server applications that need to listen on sockets must add configurable properties to define to which interfaces the server should bind.

## Physical Hosts, Virtual Machines, and Containers

### Physical Hosts

Dev and production very similar in this regard.
- Run on multicore Intel or AMD x86 processor running in 64-bit mode.
- Similar clock speeds

Individual instances are not particularly powerful - vertical scaling prefered over horizontal.
Some applications will require specialized instances with more RAM (e.g. graph processing) or GPU for computation.


### Virtual Machines in the Data Center
Beware: unpredictable performance due to contention between many VMs on same physical host and oversubscription (CPU, RAM and network). 
Do: When designing applications to run in VMs, make sure they’re not sensitive to the loss or slowdown of any one host.

Watch out for:
- Distributed programming techniques that require synchronous responses from the whole cluster for work to proceed
- “Special” machines (e.g. cluster managers, lock managers), unless another machine can take over without reconfiguration
- Subtle dependency on request or event ordering — not by design, but it can creep in unexpectedly

Beware: Virtual machines make clock problems worse. Don’t trust the OS clock. 
Between two calls to examine the clock, the virtual machine can be suspended for an indefinite span of real time
Or migrated to a different physical host that has a clock skew relative to the original host.


Do: If external, human time is important, use an external source like a local NTP server.

### Containers in the Data Center
Widely used in DCs.
Provides process isolation and packaging of a virtual machine together with a developer-friendly build process.
Ensures that production matches QA.

Act a lot like virtual machines in the cloud.
Any individual container only has a short-lived identity.
Do not: Configure on a per-instance basis.
Beware: Can cause issues with older monitoring systems (e.g. Nagios) that need to be reconfigured and bounced every time a machine is added or removed.

Do: A container don’t have much local storage - application must rely on external storage for files, data, and maybe even cache.

The most challenging part of running containers in the data center is the network.
By default, a container doesn’t expose any ports (on its own virtual interface) on the host machine.
Do: You can selectively forward ports from the container to the host, but then you still have to connect them from one host to another

Do: Overlay network
Uses virtual LANs (VLANs) to create a virtual network among the containers.
The network has its own IP address space and does its own routing with software switches running on the hosts.
Within the network, control plane software manages containers, VLANs, IPs, and names.

Containers are operated by control plane software.
Declarative - you describe desired container load and the software figures out how to best use the physical hosts.


Do: configure control plane with the geographic distribution of the hosts, so it can allocate instances regionally for low latency and maintain availability should you lose a data center.
Do: Let control plane schedule container instances and manage their network settings (Kubernetes, Mesos, and Docker Swarm)

The whole container image moves from environment to environment. 
-- Don't: Harddcode things like production database credentials. 
-- Do: Supply credentials to the container - use password vaulting.
--- A 12-factor app handles this naturally 
--- If not using, consider injecting configuration on container start-up 
 
Do: externalize networking 
-- Don't: images should not contain hostnames or port numbers 
-- Setting needs to change dynamically while the container image stays the same 
-- Links between containers are all established by the control plane when starting them up

Containers are meant to start and stop rapidly 
-- Do: Avoid long startup/initialization sequences - aim for a total startup time of one second 
-- Production servers that take minutes to load reference data or to warm up caches are not suited for containers 

Hard to debug an application running inside a container. 
Do: Containerized applications need to send their telemetry out to a data collector

### Virtual Machines in the Cloud 

All else being equal, an individual virtual machine in the cloud has worse availability than an individual physical machine.
Runs on a physical host with an extra OS in the middle.
Can be started or stopped without notice by the management APIs (i.e. “control plane” software.)
Shares the physical host with other VM's and may contend for resources - Common in AWS that machines are randomly killed
Ephemeral machine identity - A machine ID and its IP address are only there as long as the machine keeps running 

Traditional application configurations keep hostnames or IP addresses in config files 
-- In AWS, a VM’s IP address changes on every boot 
--- If your application needs to keep those addresses in files, then you have to buy Elastic IP addresses 
--- Works well enough until you need a lot of them. A basic AWS account has a limit on how many addresses it can procure

A new VM should be able to start up and join whatever pool of workers handles load. 
-- Do: For HTTP requests, use autoscaling and load balancers (elastic or application).
-- Do: For asynchronous load, use competing consumers on a queue.

The default cloud VM network interface is one NIC with a private IP address. 
-- Limit to how much traffic a single NIC can support, based on the number of sockets available 
-- Socket numbers only range from 1 to 65535, so at best a single NIC can support about 64,000 connections 
-- Can be necessary to setup more production NICs to handle more simultaneous connections

Do: A dedicated NIC for monitoring and management traffic is a good idea 
-- Don't: have SSH ports available on front-end NICs for every server 
-- Do: Set up a single entrypoint (“bastion”/“jumphost” server) with strong logging on SSH connections - use the private network to get from there to other VMs

### Containers in the Cloud

Combine the challenges of containers and the cloud.
The containers have short-lived, ephemeral identities.
Connecting them means linking ports across different VMs, possibly in different zones or regions.
Designing individual services to run in this kind of deployment is similar to designing them to run in containers in DC.
Challenges mainly arise from building those containers into a whole system - containers push complexity out of the boxes and into the control plane.


#### Virtual LANs for Virtual Machines
A VLAN multiplexes Ethernet frames on a single wire but let the switch treat them like they arrived from separate networks.
The VLAN tag is a number from 1 to 4,094 that nestles into the physical routing portion of the header.

The OS that runs a NIC can create a virtual device assigned to a virtual LAN - all packets sent by that device will have that VLAN ID in them.
This means the virtual device must have its own IP address in a subnet assigned to that VLAN.

VXLAN is similar, but runs on “layer 3,” meaning it’s visible to IP on the host.
Uses 24 more bits in the IP header, so a physical network can have more than 16 million VXLANs.

Virtualization and containers increasingly rely on software switches (instead of hardware) to handle dynamic updates.
It will be common to see software switches running on the hosts, presenting a complete network environment to the containers that
-  Allows them to “believe” they’re on isolated networks
- Supports load-balancing via virtual IPs
- Uses a firewall as a gateway to the external network

For now our container systems have to provide their own loadbalancing and need to be told which IP addresses and ports their peers are on.

#### The 12 factor app
Succinct description of a cloud-native, scalable, deployable application.
Also a great checklist for application developers not running on the cloud.
Identify different potential impediments to deployment and recommended solutions.

Do:
Codebase -- Track one codebase in revision control. Deploy the same build to every environment.
Dependencies -- Explicitly declare and isolate dependencies.
Config -- Store config in the environment.
Backing services -- Treat backing services as attached resources.
Build, release, run -- Strictly separate build and run stages.
Processes -- Execute the app as one or more stateless processes.
Port binding -- Export services via port binding.
Concurrency -- Scale out via the process model.
Disposability -- Maximize robustness with fast startup and graceful shutdown.
Dev/prod parity -- Keep development, staging, and production as similar as possible.
Logs -- Treat logs as event streams.
Admin processes -- Run admin/management tasks as one-off processes.