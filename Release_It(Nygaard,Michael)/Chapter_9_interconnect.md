# Chapter 9 - Interconnect 

## Solutions at Different Scales

At Interconnect layer and above we need to consider what solution is *right* for your organization. In lower layers, production environment characteristics determines the solution. 

A large team/department with many of microservices benefits from using dynamic discovery service (e.g. Consul).\
The cost of running and operating Consul is easily amortized over the number of teams that benefit. 
The rate of change is going to be high enough to justify something highly dynamic. 
 
A small business with just a few developers are likely better off with direct DNS entries.\
Changes aren’t going to be as rapid and the developers can keep services up-to-date.
 
## DNS 

Best choice for small teams, particularly in a slowly changing infrastructure.\
When using dedicated physical machines and dedicated, long-lived virtual machines, IP addresses are stable enough for DNS to be useful.

### Service Discovery with DNS

“Service discovery” usually implies automated query and response - DNS is not so fancy. 
- The DNS name(s) must be retrieved from the service owners 
- “host” name put into a configuration file.

If a provider of a service only has a single DNS name it implies that the provider is responsible for load balancing and high availability.\
With multiple names, then it’s up to the caller to balance among them.

*Important to have a logical service name to call, rather than a physical hostname.\
Even a logical name that is just an alias to the underlying host is preferable.\
An alias only needs to be changed the name server’s database rather than in every consuming application.

### Load Balancing with DNS
DNS round-robin load balancing operates at the application layer (layer 7) of the OSI stack.\
Operates during address resolution; not during service request.

DNS server has a list of IPs associated with a service name.\
When client does a look-up, one of these is returned.\
The client therefore connects to one out of a pool of servers

Issues 
- all instances in the pool has to be directly “routable” from callers 
--  may sit behind a firewall, but their front-end IP addresses are visible and reachable from clients
-- DNS round-robin approach puts too much control in the client’s hands
--- When the client connects *directly* to one of the servers, it's not possible to redirect traffic if one particular instance is down 
-- The DNS server has no information about the health of the instances, so it can keep vending out IP addresses for downed instances 
- does not guarantee that the *load* is distributed evenly; only the initial connections 
-- Some clients may consume more resources than others, leading to unbalanced workloads
-- the DNS server has no way to know, so it just keeps sending every k'th request to the k'th instance, ignoring it might be overwhelmed 
- inappropriate for long-running enterprise systems
-- Java’s built-in classes caches the first IP address it receives from DNS, locking every future connection to the same instance, completely defeating load balancing

### Global Server Load Balancing with DNS
Despite the issues above, DNS excels in global server load balancing (GSLB).\
Used to route clients across multiple geographic locations.\
This can be for physical data centers of your own or for multiple regions in a cloud infrastructure.\
Most common in the context of external clients routing across the public Internet. 

Clients get the best performance by routing to a nearby location (“nearby” in network terms doesn’t always match physical geography).

Each location has one or more pools of load-balanced instances for the service.\
Each pool has an IP address that goes to the load balancer (load balancing with virtual IPs.) 
 
The job of GSLB is just to get the request to the virtual IP address for a particular pool.\
GSLB works via specialized DNS servers at each location. 
 
A GSLB server keeps track of the health and responsiveness of the pools (unlike normal DNS).\
If the pool is offline, or doesn’t have any healthy instance to serve the request, the GSLB server won’t give out the IP address of the pool.
 
Different GSLB servers can also give back different IP addresses for the same request.\
Useful to balance across several local pools, or to provide the closest point of presence for the client.

- the caller queries DNS for the address related to “price.example.com.”
- Multiple GSLB servers might respond. Each one returns a different address for “price.example.com.”
-- The European server returns 184.72.248.171,
-- The North American returns 151.101.116.113.
- A client in Europe will likely get the response from the european server first, i.e. 184.72.248.171 
- The client connects directly to 184.72.248.171 and is served by the load balancer 
- The load balancer directs traffic to the instances just as it normally would

Operates at two levels: 
- At the *global* level, it’s based on DNS and clever schemes for deciding which IP address to offer. 
-- After name resolution, it’s out of the picture. 
- The load balancer (“local traffic manager”) operates as a reverse proxy so the actual call and response pass through it

Requires that the caller can reach both the global traffic managers and the local traffic managers.\
Global traffic managers can apply a ton of intelligence to the routing decision.\
In a real deployment, each GSLB server would have all the pools configured but would prefer to send traffic nearby.\
Allows them to direct traffic to the more distant pool if that’s the only one available.\
Apply weighted distribution and a host of load-balancing algorithms can be used as part of a disaster recovery strategy or part of a rolling deployment process.


#### Availability of DNS
DNS is part of the invisible infrastructure
Any server resolved by DNS is unresolvable and thus unreachable if the DNS server goes down

The main emphasis for DNS servers should be *diversity*.
- Don’t host on the same infrastructure as your production systems
- Have more than one DNS provider with servers in different locations 
- Use a different DNS provider for your public status page
- Make sure there are no failure scenarios that leave you without at least one functioning DNS server

Remember This
- Use DNS to call services that don’t change often
- DNS round-robin offers a low-cost way to load-balance
- “Discovery” is a human process. DNS names are supplied in configuration
- DNS works well for global traffic management in coordination with local load balancers
- Diversity is crucial in DNS hosts. Don’t rely on the same infrastructure for DNS hosts and production services

## Load Balancing
Horizontally scalable farms of instances that implement request/reply semantics are the norm today.\
Helps with overall capacity and resilience; it introduces the need for load balancing.\
I.e. distributing requests across a pool of instances to serve all requests correctly in the shortest feasible time.\ 

**Active load balancing** 
A piece of hardware or software between the caller and provider instances\
Listens on one or more sockets across one or more IP addresses (“virtual IPs”/“VIPs.”)\ 
A single physical network port on a load balancer may have dozens of VIPs bound to it.\ 
Each of these VIPs maps to one or more “pools.”\
A pool defines the IP addresses of the underlying instances along with policy information:
- The load-balancing algorithm to use
- Health checks to perform on the instances
- The kind of stickiness, if any, to apply to client sessions
- What to do with incoming requests when no pool members are available

To a calling application, the load balancer should be transparent.\ 
If the client can tell there’s a load balancer involved, it’s probably broken.

The service provider instances sitting behind the proxy server need to generate URLs with the DNS name of the VIP rather than their own hostnames.\
Load balancers can be implemented in software or with hardware, each with advantages and disadvantages. 

#### Software Load Balancing
Low-cost.\
Uses an application to listen for requests and dole them out across the pool of instances - basically a reverse proxy server. 

*A normal proxy multiplexes multiple outgoing calls into a single source IP address.\
A reverse proxy server does the opposite: it demultiplexes calls coming into a single IP address and fans them out to multiple addresses.* 

Reverse proxy load balancers: Squid, HAProxy, Apache httpd, nginx 

Reverse proxy servers operate at the application layer.\
They aren’t fully transparent, but adapting to them is easy.\
Logging the source address of the request is useless, because it will represent only the proxy server.\
Well-behaved proxies will add the “X-Forwarded-For” header to incoming HTTP requests, so services can use a custom log format to record that.

Reverse proxy servers can also be configured to cache responses to reduce the load on the service instances.\
This provides some benefits in reducing the traffic on the internal network.\
If the service instances are the system's capacity bottleneck, then offloading this traffic improves the system’s overall capacity.\
If the load balancer itself is the constraint, then this has no effect.

The biggest reverse proxy server “cluster” in the world is Akamai.\
Works just like a caching proxy.\
Certain advantages over Squid and HAProxy, including a large number of servers located near the end users, but is otherwise logically equivalent.

Because the reverse proxy server is involved in every request, it can get burdened very quickly.\
Once you start contemplating a layer of load balancing in front of your reverse proxy servers, it’s time to look at other options.

### Hardware Load Balancing
Specialized network devices that serve a similar role to the reverse proxy server.\
These devices, such as F5’s Big-IP products, provide the same kind of interception and redirection capabilities as the reverse proxy software. 

Because they operate closer to the network, hardware load balancers provide better capacity and throughput.\
Hardware load balancers are application-aware and can provide switching at layers 4 through 7 of the OSI stack. 

This means they can load-balance *any* connection-oriented protocol, not just HTTP or FTP.\
They can also hand off traffic from one entire site to another, which is useful for diverting traffic to a failover site for disaster recovery.\
This works well in conjunction with global server load balancing.\
Main drawback is their price - in the five digits for a low-end configuration, six digits for high-end.

### Health Checks
One of the most important services a load balancer can provide is service health checks.\
The load balancer will not send traffic to an instance that fails a certain number of health checks.\
Both the frequency and number of failed checks are configurable per pool. 

### Stickiness
Load balancers can ensure that repeated requests go to the same instance. 
Useful when you have stateful services, like user session state, in an application server.\
Provides a better response time for the caller because necessary resources will already be in that instance’s memory.

A downside of sticky sessions is that they can prevent load from being distributed evenly across machines.\
A machine can run “hot” for a while if it happens to get several long-lived sessions.

Stickiness requires some way to determine how to group “repeated requests” into a logical session.
- attach a cookie to the outgoing response to the first request 
-- Subsequent requests are hashed to an instance based on the value of that cookie 
- assume that all incoming requests from a particular IP address are the same session
-- This approach will break 
--- if you have a reverse-proxy upstream of the load balancer 
--- when a large portion of your customer base reaches you through an outbound proxy in their network

### Partitioning Request Types
Load balancers can also do “content-based routing.”\
Uses parts of the URLs of incoming requests to route traffic to one pool or another.\
E.g. search requests may go to one set of instances, while use-signup requests go elsewhere.\
A large-scale data provider may direct long-running queries to a subset of machines and cluster fast queries onto a different set. 

Remember This
Load balancers are integral to the delivery of your service 
- cannot be treated as as just part of the network infrastructure.
- plays a part in availability, resilience, and scaling. 
- 
Because so many application attributes depend on them, it pays to incorporate load-balancing design as you build services and plan deployment.

If your organization treats load balancers as “those things over there” that some other team manages, then you might even think about implementing a layer of software
load balancing under your control, entirely behind the hardware load balancers in the network.
- Load balancing creates “virtual IPs” that map to pools of instances.
- Software load balancers work at the application layer. 
-- low cost and easy to operate.
- Hardware load balancers reach much higher scale than software load balancers. 
--  require direct network access and specific engineering skills
- Health checks are a vital part of load balancer configuration. 
-- ensures that requests can succeed, not just that the service is listening to a socket
- Session stickiness can help response time for stateful services
- Consider content-aware load balancing if your service can process workload more efficiently when it is partitioned

## Demand Control

Most of our services are either directly or indirectly exposed to the entire world that can crush our systems at any time.\
To protect our systems, we can either refuse work or scale out. 
 
### How Systems Fail
Every failing system starts with some queue filling up.
When analyzing request/reply workload, we need to consider the resources being consumed and the queues to get access to those resources.
Lets us decide where to cut off new requests.

Each request consumes a socket on each tier it passes through.\
While the request is active on an instance, it has one fewer ephemeral sockets available for new requests.\
The socket is even consumed for a little while after the request completes. 

There’s a relationship between the number of sockets available and the number of requests your service can handle per second.\
Little's law relates this to the duration of the requests. 
The faster your service ends requests, the more throughput it can handle. 
With systems under high levels of load it's natural to slow down, but that means fewer and fewer sockets are available to receive requests exactly when the most requests are coming in! (“going nonlinear”)

The next resource to consider is raw I/O bandwidth through the NICs.\
No matter how many virtual NICs your machine has, or how many sockets your instance has open, Ethernet is inherently a serial protocol.\
Takes time to shove packets through the wires.\
Any packet you want to send while the port is busy just has to queue up.\
Conversely, applications only receive packets when they are ready.\
Anything that arrives on the NIC in the meantime has to be buffered until the application calls some form of read on the socket.\
On both the transmit side and the receive side, a finite amount of RAM is allocated to these buffers.\
Any data that goes into those buffers has to work its way through the queue.\
When the write buffers are full, the TCP stack won’t accept any new writes and write calls will block.\
When the read buffers are full, the stack won’t accept any new incoming data and the connection will stall. (Eventually, that backs up into the sending application and the write call there also blocks.)\
When the application is under high load, it's most likely to be slow at reading from TCP buffers (nonlinear effect).

There’s  the “listen queue” on the server’s socket.\
TCP connection requests can get through the three-phase handshake but then have to wait for the application to accept the connection.\
When the application calls accept, the server’s TCP stack removes the connection from the listen queue and hands it over for reads and writes.\
If a connection request sits in that queue long enough, the client will eventually give up and abandon the connection.\
If the listen queue is full, clients that attempt to connect will work their way through a series of delayed retries and then ultimately give up.\
As requests from the outside world reach further into the system, they activate resources at every tier until the work can be retired.

A single request at the network edge may translate into a tree of service requests through many layers of internal structure.\
Each request adds transient load on a provider’s listen queue and persistent load on its sockets and NICs.\
Under high load those resources are held longer, which further extends response times for the new incoming work.\
At some point, the response time for one or more services extends past the caller’s timeout. \
The caller will stop waiting for a response on the original request and probably fire a retry at us (when it hurts the worst!).

### Preventing Disaster
The best thing to do under high load is turn away work we can’t complete in time (“load shedding”).

Shed load as early as possible to avoid tying up resources at several tiers before rejecting the request; a quick rejection is better than a slow timeout.\
*Load balancers near the network edge are the ideal place for this.\
A health check on the first tier of services can inform the load balancer when response times are too high (i.e., higher than the service’s SLA). 
 
The load balancer needs to be configured to send back an HTTP 503 response code when all instances fail their health checks.\
Quickly informs the caller to try later.

Services can measure their own response time to help with this.\
Can check their own operational state to see if requests will be answered in a timely fashion.\
E.g., monitoring the degree of contention for a connection pool allows a service to estimate wait times. 

Likewise, a service can check response times on its own dependencies.\ 
If those are too slow and are required, then the health check should show this service is unavailable.\
Provides back pressure through service tiers.

Services should have relatively short listen queues.\
Every request spends some time in the listen queue and some time in processing (“residence time” is the total of these).\
If a service needs to respond in 100 milliseconds or less, that’s the allowed residence time. 

Common mistake to only measure a services own processing time and ignore time spent in the listening queue.\
The service itself may think all is well while its consumers complain that it’s slow.\
The listen queue is serial while processing is multithreaded, so queuing time ultimately dominates processing time. 

The queuing math is complex
- Little’s law doesn’t apply when you hit boundaries and maximum queue length 
- You’ll need to know whether the service is exposed directly to the Internet — an infinite source of demand for all practical purposes — or whether
it’s internal, where the demand population is finite. 
- Neil Gunther’s “PDQ” toolkit is an accurate model

For a quick heuristic, take your maximum wait time divided by mean processing time and add one.\
Multiply that by the number of request handling threads you have and bump it up by 50 percent.\
Provides a reasonable starting point for your listen queue length.

Because clients retry TCP connections, it can be useful to run a “listen queue purge” when the service can’t keep up with demand.\
This is a kind of self-awareness that goes along with the idea of a “yellow alert” or “red alert” status.\
A listen queue purge is just a tight loop that accepts connections and then immediately responds with a canned rejection.\
E.g. 503 Try Again\r\n\r\n.

Remember This
- A service has to deal with Internet-scale load. 
- Either it directly handles requests from the world at large, or it serves some other piece of code that does. 
- Load and traffic patterns is outside our load, so our services need to protect themselves when the load gets too heavy
-- Reject work as close to the edge as possible. The further it penetrates your system, the more resources it ties up
-- Provide health checks that allow load balancers to protect your application code
-- Start rejecting work when your response time is going to provoke retries


#### TIME_WAIT and the Bogons
A recently closed socket remains in the TIME_WAIT state for a bit to ensure that any stray packets either time out or arrive (to be dropped). 

Suppose there were no such TIME_WAIT state.\
A server could close socket 32768 and then reallocate it to a new request.\
Meanwhile, a leftover delayed packet could arrive from the old connection. 

It might come with a sequence number that matches the server’s expectations.\
Now the TCP stream is out of sync.\
Such a packet is called a “bogon,”. TIME_WAIT is the antibogon protection.

Services that only deal with work inside a data center can set a very low TIME_WAIT to free up ephemeral sockets (tcp_tw_reuse kernel setting on linux)\
Be sure to reduce the machine’s TCP setting for the default “time to live” on packets accordingly. 

## Network Routing
Because machines in a data center usually have multiple network interfaces, questions will sometimes arise about which interfaces particular kinds of traffic should traverse.\
E.g., commonly machines have
- a front-end network interface connected to one VLAN for communication to the web servers
- a back-end network interface connected to a different VLAN for communication to the database servers. 

The server must be told which interface to use in order to reach a particular destination IP address.\
For nearby servers, the routes are likely based on the subnet addresses.\
In the example of the application server
- the back-end interface probably shares a subnet with the database server
- the front-end interface probably shares a subnet with the web servers. 

Routing is more complicated when distant services—perhaps third-party services—are involved.\
Modern OSs strive to make routing automatic and invisible.\
When a machine brings up its perceived primary NIC, it uses the main IP address for that NIC as its “default gateway.”\
This becomes the first entry in the routing table for the host.\
As the host gets cozier with its switches, they gossip about routes and the host updates its routing table.\
That table tells the operating system which NIC to use to reach a destination address or network.\
When an application sends a packet, the host checks the destination IP address against the routing table to see if it knows how to move that packet a hop closer to its destination.


Occasionally multiple routes seem plausible to the host but aren’t actually equivalent.\
Consider the case of a service provided by a close business partner.\
If the integration includes personally identifiable information (PII), then you might set up a VPN rather than send sensitive data straight over the public Internet.\
Depending on a ton of configuration options that are outside your control, both the VPN and the primary switch may advertise routes that could reach the destination address.

In the best case, you’ll discover this problem during testing because nothing is reaching the partner’s service.\
Your service won’t be able to open a socket and will get a “destination unreachable” response.\
If the host happens to receive route advertisements in the right order, it might send those sensitive packets over the VPN.\
If it gets them in the wrong order, it may try to send them over the front-end, i.e. the public—network. 

You should hope that the partner is better at networking and won’t accept connections.
Otherwise, that PII will be sent in cleartext over the public Internet. 
Your service will appear to be working normally so you won’t even know it’s happening.

One solution is *static* route definitions.\
Static routes are frowned upon, but sometimes they’re the only way.

Another increasingly common solution to routing is *software-defined networking*.\
Goes hand-in-hand with virtualized/container-based infrastructure.\
Containers and VMs use virtual IP addresses, VLAN tagging, and virtual switches to create a kind of “network on a network.”\
The packets still run over the same wires, but the host machine’s IP address is not involved.\
This lets the virtual switches operate independently of the physical ones. \
They can assign IPs from private pools, attach DNS names to those IPs to identify services, and dynamically create firewalls and subnets.

Getting these routing issues right requires paying attention to each and every integration point.\
Getting them wrong risks reduced availability or exposure of customer data. 

*For each connection to a remote system, keep a record in a spreadsheet or a database with the destination name, address, and desired route.*\
Somebody is going to need that information to write firewall rules.

## Discovering Services
There are two cases where service discovery is important 
- you have too many services for DNS management to be practical
- you have a highly dynamic environment 

Container-based environments usually satisfy both, but there's other cases.

“Service discovery” has two parts - dynamic addition and lookup.\
New service instances can announce themselves ready to receive a load.\
Replaces *statically* configured load balancer pools with dynamic pools. 
 
A caller needs to know at least one IP address to contact for a particular service.\
The lookup process can appear to be a simple DNS resolution for the caller, even if some super-dynamic service-aware server is supplying the DNS service.

Service discovery is itself a service - can fail or get overloaded.\
It’s a good idea for clients to cache results for a short time.

Use an existing service discovery solution - hard to implement correctly yourself.\
You can build a service discovery mechanism on top of a distributed data store (Apache ZooKeeper, etcd).\
I.e. wrap the low-level access with a library to make it both easier and more reliable to use these databases. 

In the terminology of the CAP theorem, ZooKeeper is a “CP” system.\
When faced with a network partition some nodes won’t answer queries or accept writes.\
Since clients need to be available, they must have a fallback to use other nodes or previously cached results.\
It’s not reasonable to expect every client to implement this behavior. 

Consul resembles ZooKeeper in that it operates as a distributed database.\
However, Consul’s architecture places it in the “AP” arena; remains available and risk stale information when a partition occurs.\
In addition to service discovery it also handles health checks.

Some other service discovery tools integrate directly with the control plane of PaaS platforms.\
E.g. when Docker Swarm starts containers to run service instances, it automatically registers them with the swarm’s dynamic DNS and load-balancing mechanism.

These tools have different considerations for each.\
They cover different scope and are subject to divergent behavior in failure cases. 
There’s no plug-and-play replaceability.\
Choosing one is not a simple matter, and replacing one will have wide-reaching consequences.\
Do your research and commit to solving implementation challenges with whichever tool you choose.

## Migratory Virtual IP Addresses
Suppose a server hosting a critical, but not natively clustered, application goes down.\
The cluster server on its failover node notices the lack of a regular heartbeat from the failed server.\
This cluster server concludes that the original server has failed.\
It starts up the application on the secondary server and mounts any required filesystems.\
Takes over the virtual IP address assigned to the clustered network interface.

"Virtual IP" is overloaded.\
Generally, it means an IP address that is not strictly tied to an Ethernet MAC address.\
Cluster servers use it to migrate ownership of the address between the members of the cluster.\
Load balancers use virtual IPs to multiplex many services (each with its own IP address) onto a smaller number of physical interfaces. 

Since load balancers typically come in pairs, the virtual IP (as in “service address”) can also be a virtual IP (as in “migrating address”).\
This kind of virtual IP address is just an IP address that can be moved from one NIC to another as needed.\
At any given time, exactly one server claims the IP address.\
When the address needs to be moved, the cluster server and the operating systems collaborate to do some funny stuff in the lower layers of the TCP/IP stack.\
Associate the IP address with a new MAC address (hardware address) and advertise the new route (ARP).

This kind of migratory IP address is often used for active/passive database clusters.\
Clients connect only using the DNS name for the virtual IP address, not to the hostnames of either node in the cluster.\
Ensures that no matter which node currently holds the IP address, the client can connect to the same name.

This approach can not migrate the in-memory state of the application.\
Any non-persistent state about interactions will be lost.\
For databases, this includes uncommitted transactions.\
Some database drivers (e.g. Oracle JDBC and ODBC drivers) will automatically re-execute queries that are aborted because of a failover.\
Updates, inserts, or stored procedure calls cannot be automatically repeated. 

Any application calling a database through a virtual IP should be prepared to get a SQLException when such a failover occurs.\
Generally, if your application calls any other service through a handoff virtual IP, it must be prepared for the possibility that the next TCP packet isn’t going to the same interface as the last packet.\
The application logic should be prepared to handle IOExceptions in strange places and must handle it differently than just a “destination unreachable” error.\
If possible, the application should retry its request against the new node (see p. 95, for some important safety limits on retries).