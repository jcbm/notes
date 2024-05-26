
Ch 4 Stability Antipatterns

Modern systems are demanding with many users across the globe and demand for incredibly high availability.\
This calls for huge complex system that are often heavily connected inside and across organizations, often building on tight coupling.

With many moving parts and hidden, internal dependencies, high *interactive complexity arise*, causing operators to have incorrect mental models.\
With an incorrect mental model, the operator’s instinctive actions can trigger completely unexpected linkages and have results ranging from  ineffective to harmful.\
*hidden linkages often appear obvious during the post-mortem analysis, but are very difficult to anticipate.*

Tight coupling allows cracks in one part of the system to propagate across layers/system boundaries.\
Failure in one component causes load to be redistributed to peers and creates delays and stress to its callers.\
The increased stress makes it very likely that another system component will fail.\
This makes the next failure more likely, eventually resulting in total collapse.\
Tight coupling can appear
- within application code
- in calls between systems
- any place a resource has multiple consumers


In many failure modes, big systems fail faster than small systems.\
The size and the complexity of these systems in combination with tight coupling and hidden linkages can turn rapidly moving cracks into full-blown failures.

Antipatterns can wreck your system and have contributed to more than one system failure.\
Will *create*, *accelerate*, or *multiply* cracks in the system.

Faults can't be eliminated completely - it must be considered what happens after a fault takes place.

## Integration Points

Modern websites typically consist of some combination of browser client, mobile client, and API.

Either structured as "butterfly" or "spider" architectures.\
A butterfly has a central system with a lot of feeds and connections fanning into it on one side and a large fan out on the other side (=monolith)

Spiderweb has many services with many connections, sometimes in tiers  (=microservice).\
Every connection  is an integration point that can cause danger to your system.
The more we move toward a large number of smaller services, the more we integrate with SaaS providers, and the more we go API first, the worse this is going to get.

Integration points is the most common killer of systems.\
Every single feed presents a stability risk and can hang it, crash it, or generate other impulses at the worst possible time.\
All of these can hang:
- socket
- process
- pipe
- remote procedure call
- database calls

###  Socket-Based Protocols
Essentially all high-level integration protocols are based on sockets, except for named pipes and shared-memory IPC.\
Each protocol has its own failure modes, but they’re all susceptible to failures at the socket layer.

The simplest failure mode occurs when the remote system refuses connections.\
As this results in an exception in languages that have them or a magic return value in ones that don’t, it's easy to deal with.\
As the API makes it clear that connections don’t always work, programmers handle it.

It can take a long time to discover that you can't connect due to the way TCP/IP works:
- Connection starts when the caller sends a SYN packet to a port on the remote server
-- If nothing is listening on that port, the remote server immediately sends back a RST packet to indicate this
--- The calling application immediately gets an exception or a bad return value (fast failure)
-- Else, the remote server sends back a SYN/ACK packet indicating its willingness to accept the connection
-- The caller sends its own ACK in response

Imagine a remote application overloaded with connection requests until it can no longer service incoming connections.\
A “listen queue” defines how many pending connections (SYN incoming, but no SYN/ACK replied yet) are allowed by the network stack.\
Once that listen queue is full, further connection attempts are refused quickly.\
The listen queue is the worst place to be.\
While the socket is in that partially formed state, whichever thread called open() is blocked inside the OS kernel until the  remote application finally accepts the connection or until the connection attempt times out.\
Connection timeouts vary between OS'es, but they’re usually measured in minutes!\
The calling application’s thread could be blocked waiting for the remote server to respond for ten minutes.

when the caller can connect and send its request but the server takes a long time to read the request and send a response.\
The read() call will just block until the server gets around to responding.\
Often, the default is to block forever. *You have to set the socket timeout if you want to break out of the blocking call*.\
Be prepared to handle an exception when the timeout occurs.

Network failures can be fast or slow.\
Fast failures cause immediate exceptions in the calling code.

Slow failures, such as a dropped ACK, let threads block for minutes before throwing exceptions.\
The blocked thread can’t process other transactions, so overall capacity is reduced.\
If all threads end up getting blocked, the server is functionally down.

*A slow response is a lot worse than no response.*

## The 5 A.M. Problem

30 application server instances all hang within a 5 minute interval around 5AM every day\
From midnight to 5 a.m., only about 100 transactions per hour were of interest, but the numbers increased quickly once the East Coast started to wake up.
Restarting the servers would solve the issue.

A thread dump of a hanging server  showed it was up and running, but all request-handling threads were blocked inside the Oracle JDBC library, inside OCI calls.\
Ignoring threads that were just blocked trying to enter a synchronized method, it looked as if the active threads were all in low-level socket read or write calls.\
tcpdump/Wireshark showed very little activity - only a handful of packets were being sent from the application servers to the database servers, with no replies.\
nothing was coming from the database to the application servers, but the database was alive and healthy.\
no blocking locks, the run queue was at zero, and the I/O rates were trivial.

First priority was restoring service (collect data when possible, but not at the risk of breaking an SLA.)\
Deeper investigation would have to wait until the issue happened again.

A look at traffic on the databases’ network showed nothing.\
Decompilation of the application server’s resource pool class confirmed that one hypothesis was plausible:

An established TCP connection can exist for days without a single packet being sent by either side.\
As long as both computers have that socket state in memory, the “connection” is still valid.
the “connection” persists as long as the two computers at the endpoints think it does, even if routes change, and physical links are severed and reconnected.
Can be problematic when firewalls are involved.
- A firewall is essentially a specialized router
-- Routes packets from one set of physical ports to another
-- a set of access control lists define the rules about which connections it will allow

When the firewall sees an incoming SYN packet, it checks it against its rule base.\
A packet can be
- allowed (routed to the destination network)
- rejected (TCP reset packet sent back to origin)
- ignored (dropped with no response at all)

If the connection is allowed, then the firewall makes an entry in its own internal table like “192.0.2.98:32770 is connected to 192.168.1.199:80.”\
All future packets, in either direction, that match the endpoints of the connection are routed between the firewall’s networks.

The table of established connections inside the firewall is finite.\
Therefore, it does not allow infinite duration connections.\
The “last packet” time is stored for each connection.\
If too much time elapses without a packet, it drops the connection from its table
A firewall may inform the endpoints of this when they send a packet by responding with ICMP, but it's often disabled for security reasons.

For the problem it meant that any attempt to read or write from the socket on either end did not result in a TCP reset or an error due to a half-open socket.\
Instead, the TCP/IP stack sent the packet, waited for an ACK, didn’t get one, and retransmitted. The faithful stack tried and tried to reestablish contact, and that firewall just kept dropping the packets.\
A 2.6 series kernel, has its tcp_retries2 set to the default value of 15, which results in a twenty-minute timeout before the TCP/IP stack will inform the socket library that the connection is broken.

The application servers had a thirty-minute timeout.\
That application’s *write* to a socket could block for thirty minutes; *reading* from the socket could block forever.

The resource pool class used a LIFO strategy (i.e. stack).\
Overnight traffic volume was very light.\
A single database connection would get checked out of the pool, used, and checked back in.\
The next request would get the same connection, leaving the thirty-nine others to sit idle until traffic started to ramp up.\
They were idle well over the one-hour idle connection timeout configured into the firewall.

When traffic ramped up, all thirty-nine connections per application server would get locked up immediately.\
Even if the one connection was still being used to serve pages, sooner or later it would be checked out by a thread that ended up blocked on a connection from one of the other pools.\
Then the one good connection would be held by a blocked thread.

 Solutions
- The resource pool can test JDBC connections for validity before checking them out by executing a SQL query like “SELECT SYSDATE FROM DUAL.”
-- would also just make the request-handling thread hang
- Pool could be made to keep track of the idle time of the JDBC connection and discard any older than one hour. 
-- involves sending a packet to the database server to tell it that the session is being torn down

Oracle has a *dead connection detection* feature that can discover when clients have crashed.\
The database server sends a ping packet to the client at some periodic interval.
- If the client responds, then the database knows it’s still alive
- Else if the client fails to respond after a few retries, the database server assumes the client has crashed and frees up all the resources held by that connection

Client crashing wasn't the problem here, but the ping would reset the firewalls last packet time for the connection and keep the connection alive from its POV.

*Every problem can be solved at the level of abstraction where it manifests.*\
Sometimes the causes reverberate up and down the layers.\
You need to know how to drill through at least two layers of abstraction to find the “reality” at that level in order to understand problems.

### HTTP protocols

REST services serving JSON over HTTP are very common.

HTTP-based protocols use sockets, so they are vulnerable to all of the problems described previously.
Additional HTTP issues mainly relate to client libraries\
A provider may
- accept the TCP connection, but never respond to the HTTP request
- accept the connection, but not read the request
-- If the request body is large, it might fill up the provider’s TCP window, causing the caller’s TCP buffers to fill, which will cause the socket write to block. Even sending the request will never finish
- respond with a status the caller doesn’t know how to handle, e.g. “451 Resource censored.”
- respond with a content type the caller doesn’t expect or know how to handle, such as a generic web server 404 page in HTML instead of a JSON response
- claim to be sending JSON but actually sending plain text. Or something else

*Use a client library that allows fine-grained control over timeouts—including both the connection timeout and read timeout—and response handling.*

*Avoid client libraries that try to map responses directly into domain objects.* Treat a response as data until you’ve confirmed it meets your expectations.\
It’s just text in maps (also known as dictionaries) and lists until you decide what to extract.

### Vendor API Libraries

Usually, software vendors provide client API libraries that have a lot of problems and often have stability risks.\
Large variation in quality, style, and safety.

Main issue is the lack of control.\
If the code is not public, the best approach is to decompile the library, identify issues and report them as bugs.\
It might be necessary to do your own fix and recompile the code as a temporary solution.

The usual stability killer with vendor API libraries is all about blocking.
- internal resource pool
- socket read calls
- HTTP connection
- Java serialization

are all peppered with unsafe coding practices.

### Countering Integration Point Problems
The most effective stability patterns to combat integration point failures are *Circuit Breaker* and *Decoupling Middleware*.

Use test harnesses for each integration test to ensure that your software is *cynical*, i.e. can handle violations of form and function, such as badly formed headers or abruptly closed connections.\
A test harness is a simulator that provides controllable behavior.\
Having a test harness return hard-coded responses facilitates functional testing and provides isolation from the target system when you’re testing.\
Each such test harness should also allow you to simulate various kinds of system and network failures.

To test for *stability*, you need to flip all the switches on the harness while the system is under considerable load.\
Load should come from many instances (workstations or cloud instances) - requires much more than a handful of testers clicking around on their desktops.

### Remember This
Beware this necessary evil.\
Every integration point will eventually fail and you need to be prepared.

Prepare for the many forms of failure.
- Integration point failures take several forms, ranging from various network errors to semantic errors
- You will not get nice error responses delivered through the defined protocol
- various protocol violations
- slow responses
- outright hangs

Know when to open up abstractions.
- Debugging integration point failures usually requires peeling back a layer of abstraction
- Failures are often difficult to debug at the application layer because most of them violate the high-level protocols
- Packet sniffers and other network diagnostics can help

Failures propagate quickly.
Failure in a remote system quickly becomes your problem, usually as a cascading failure when your code isn’t defensive enough.

Apply patterns to avert integration point problems.
To avoid the dangers of integration points, apply
- Circuit Breaker
- Timeouts
- Decoupling Middleware
- Handshaking

## Chain Reactions

Horizontal scaling is the norm when scaling applications, i.e. load-balanced farms/clusters where each server runs the same applications.\
No single point of failure - multiplicity of machines provides fault tolerance through redundancy.\
A single machine or process can completely bonk while the remainder continues serving transactions.

Can still exhibit a load-related failure mode.
E.g. a concurrency bug that causes a race condition shows up more often under high load than low load. When one node in a load-balanced group fails, the other nodes
must pick up the slack, i.e. given n nodes, that each handle a load x, each node will now handle (x/n-1) more load in addition to the x load they already have.\
If the first server failed because of some load-related condition, such as a memory leak or intermittent race condition, the surviving nodes become more likely to fail.
For each one that fails, the chance increases for the rest.

A chain reaction occurs when an application has some defect — usually a resource leak or a load-related crash.\
As the cluster is a homogeneous layer, the defect is going to be in each of the servers; the only way you can eliminate the chain reaction is to fix the underlying defect.\
Splitting a layer into multiple pools (*Bulkhead pattern*) can help by splitting a single chain reaction into two separate chain reactions that occur at different rates.

A chain reaction failure in a layer can easily lead to a cascading failure in a calling layer.
Sometimes chain reactions are caused by blocked threads; happens when all the request-handling threads in an application get blocked and that application stops responding.\
 Incoming requests will get distributed out to the applicaions on other servers in the same layer, increasing their chance of failure.

### Remember This

Recognize that one server down jeopardizes the rest.
- the death of one server makes the others pick up the slack
- The increased load makes others fails, increasing load even more on the rest.
- A chain reaction will quickly bring an entire layer down

Other layers that depend on it must protect themselves, or they will go down in a cascading failure.

Hunt for resource leaks\
Typically chain reactions happens when your application has a memory leak.\
As one server runs out of memory and goes down, the other servers pick up the dead one’s burden.\
The increased traffic means they leak memory faster.

Hunt for obscure timing bugs\
Obscure race conditions can also be triggered by traffic.\
If one server goes down to a deadlock, the increased load on the others makes them more likely to hit the deadlock too.

Use Autoscaling.\
In the cloud, you should create health checks for every autoscaling group. The scaler shuts down instances that fail their health checks and start new ones.\
As long as the scaler can react faster than the chain reaction propagates, your service will be available.

Defend with Bulkheads.\
Partitioning servers with Bulkheads can prevent chain reactions from taking out the entire service—though they won’t help the callers of whichever partition does go down.\ 
Use Circuit Breaker on the calling side for that.


## Cascading failures

System failures start with a crack from some fundamental problem.
- a latent bug that some environmental factor triggers
- a memory leak
- some component just gets overloaded.

The crack can progress and even be amplified by some structural problems.\
A *cascading failure* occurs when a crack in one layer triggers a crack in a calling layer, e.g. when an entire database cluster goes dark, calling applications are going to experience problems.\
If the caller handles it badly, then the caller will also start to fail, resulting in a cascading failure.


Every enterprise or web system looks like a set of services grouped into distinct farms or clusters, arranged in layers.\
Outbound calls from one service funnel through a load balancer to reach the provider.\
Each service is like its own little stack of layers, which are then connected into layers of dependencies beyond that. Every dependency is a chance for a failure to cascade.\
Crucial services with a high *fan-in*, i.e. with many callers, spread their problems widely, so they are worth extra scrutiny.

Cascading failures need a mechanism to transmit the failure between layers.\
The failure “jumps the gap” when bad behavior in the calling layer gets triggered by the failure condition in the provider.\
Cascading failures *often result from resource pools that get drained* because of a failure in a lower layer.\
Integration points without timeouts are a surefire way to create cascading failures.\
The layer-jumping mechanism often takes the form of blocked threads, but can also be due to an overly aggressive thread that keeps calling a lower layer that fails (as opposed to using circuit breaker or other mechanisms).\
Speculative retries also allow failures to jump the gap. A slowdown in the provider will cause the caller to fire more speculative retry requests, tying up even more threads in the caller at a time when the provider is already responding slowly.

*While integration points are the number-one source of cracks, cascading failures are the number-one crack accelerator.\
Preventing cascading failuresis the very key to resilience. The most effective patterns to combat cascading failures are Circuit Breaker and Timeouts.*


### Remember This
Stop cracks from jumping the gap.
- A cascading failure occurs when cracks jump from one system or layer to another, usually because of insufficiently paranoid integration points
-- can also happen after a chain reaction in a lower layer
When you call other enterprise systems; make sure you can stay up when they go down.

Scrutinize resource pools.
- A cascading failure often results from a resource pool, such as a connection pool, that gets exhausted when none of its calls return
-- The threads that get the connections block forever; all other threads get blocked waiting for connections

Safe resource pools always limit the time a thread can wait to check out a resource.

Defend with Timeouts and Circuit Breaker.
A cascading failure happens after something else has already gone wrong.
- Circuit Breaker protects your system by avoiding calls out to the troubled integration point
- Using Timeouts ensures that you can come back from a call out to the troubled point

## Users

### Traffic
As traffic grows, it will eventually surpass your 4capacity. How does your system react to excessive demand?

“Capacity” is the maximum throughput your system can sustain under a given workload while maintaining acceptable performance.\
When a transaction takes too long to execute, it means that the demand on your system exceeds its capacity.

Internally, your system has some harder limits. Passing those creates cracks in the system, and cracks always propagate faster under stress.

If you are running in the cloud, autoscaling aids in dynamically adapting capacity, but watch out for the cost in buggy applications.

#### Heap Memory
One important limit is available memory, particularly in interpreted or managed programming languages.

Excess traffic can stress the memory system in several ways.

In web app backends, every user has a session.\
If using *memory-based* sessions, the session stays resident in memory for a certain length of time ("dead time") after the last request from the user after which the memory is freed.
Every additional user means more memory usage.

When remaining memory is limited, a number of things can happen.
- In the best case, the user gets an out-of-memory exception
-- If things are really bad, the logging system might not even be able to log the error
- If no memory is available to create the log event, then nothing gets logged (*you must use external monitoring in addition to log file scraping*)

A recoverable low-memory situation can rapidly turn into a serious stability problem.\
*You should try to keep as little in the in-memory session as possible.* I.e. never keep an entire set of search results in the session for pagination.\
Better to re-query the search engine for each new page of results.

For every bit of data you put in the session, consider that it might never be used again.\
It could spend the next thirty minutes uselessly taking up memory and putting your system at risk.

*Soft* references can be used to keep things in the session when memory is plentiful but automatically be more frugal when memory is tight.
- C#: System.WeakReference
- Java: java.lang.ref.SoftReference
- Python: weakref
- ...

A soft reference holds another large/expensive object ("the payload"), but only until the garbage collector needs to reclaim memory.\
When only soft references point to an object, the garbage collector is free to collect it when memory is low.
```
MagicBean hugeExpensiveResult = ...;
SoftReference ref = new SoftReference(hugeExpensiveResult);
session.setAttribute(EXPENSIVE_BEAN_HOLDER, ref);
```
GC decides when to reclaim softly reachable objects, how many of them to reclaim, and how many to spare.\
Generally weakly reachable objects will be reclaimed before an out-of-memory error occurs (see runtime's documentation).

Since the garbage collector may harvest the payload at any time, code that uses the payload object must be prepared to deal with a null.\
It can choose to
- recompute the expensive result
- redirect the user to some other activity
- take any other protective action

Weak references are a useful way to respond to changing memory conditions, but add complexity.\
*When you can, it’s best to just keep things out of the session*.

#### Off-Heap Memory, Off-Host Memory

Alternatively per-user memory can be farmed out; instead of keeping it in the heap (i.e. the address space of your server’s process) move it out to some other process.

Memcached is essentially an in-memory key-value store that you can put on a different machine/spread across several machines.\
Redis is a fast “data structure server”; somewhere between a cache and database; common to use Redis to hold session data instead of keeping it in memory/relational database.

*Trade-off between total addressable memory size and latency to access it*.\
This notion of memory hierarchy is ranked by size and distance.\
Registers are fastest and closest to the CPU, followed by cache, local memory, disk, tape, and so on.\
On one hand, networks have gotten fast enough that “someone else’s memory” can be faster to access than local disk. Your application is better off making a remote call to get a
value than reading it from storage.\
Local memory is however still faster than remote memory.

#### Sockets
Number of sockets on your server is another limit you can run into when traffic gets heavy.\
Every active request corresponds to an open socket.\
The operating system assigns inbound connections to an “ephemeral” port that represents the receiving side of the connection.

A port number is 16 bits long - can only go up to 65535.\
Different OSs use different port ranges for ephemeral sockets, but the IANA recommended range is 49152 to 65535. =>\
Server can at most have 16,383 connections open.\
Your machine is probably dedicated to only your service rather than handling, say, user logins.\
We can stretch that range to ports 1024–65535, for a maximum of 64,511 connections.

If there are only 64,511 ports available for connections, how can a single server have a million connections, which is common?\
The answer is virtual IP addresses.\
The OS binds additional IP addresses to the same network interface.\
Each IP address has its own range of port numbers; with 16 IP addresses we can handle that many connections.
This is not a trivial thing to tackle.\
Your application will probably need some changes to listen on multiple IP addresses and handle connections across them all without starving any of the listen queues.\
A million connections also need a lot of kernel buffers.

#### Closed Sockets
After your application code closes a socket, the TCP stack moves it through a couple of terminal states.
Including TIME_WAIT state; a delay period before the socket can be reused for a new connection.

It’s part of TCP’s defense against *bogons*; a wandering packet that got routed inefficiently and arrives late, possibly out of sequence, and after the connection is closed.
If a socket is reused too quickly, a bogon could arrive with the exact right combination of IP address, destination port number, and TCP sequence number to be accepted as legitimate data for the new connection.\
In essence a bit of data from the old connection would show up midstream in the new one.\
Bogons are a real, though minor, problem on the Internet at large.\
Within your data center or cloud infrastructure, though, they are less likely to be an issue.\
You can turn the TIME_WAIT interval down to get those ports back into use ASAP.

### Expensive to Serve
Some users are more demanding than others.\
For retail systems, with most activity on the site, and thus the users that create the most load are typically also the users that are most likely to buy something.\
Increasing the conversion rate might be good for the profit-and-loss statement, but it’s definitely hard on the systems.

Expensive users are not not a direct stability risk, but the increased stress they produce increases the likelihood of triggering cracks elsewhere in the system.

To be sure you can handle expensive users, you should test aggressively.\
*Identify your most expensive transactions and double or triple the proportion of those transactions.*\
If your retail system expects the typical 2 percent conversion rate, then your load tests should test for a 4, 6, or 10 percent conversion rate.\
Dont use the results to plan *capacity* for regular production traffic tho. By definition, these are the most expensive transactions.
The average stress on the system is guaranteed to be less than that.\
Build the system to handle nothing but the most expensive transactions and you will spend ten times too much on hardware.

### Unwanted users

(a lot left out - it's a bit all over the place)

Competitive intelligence companies agressively scrape company websites.\
Do not honor session cookies; if you are not using URL rewriting to track sessions, each new page request will create a new session.

Keeping out legitimate robots is easy through the use of the robots.txt file.\
By conventions, robots has to ask for the file and choose to respect your wishes.

Some sites choose to redirect robots and spiders, based on the user-agent header.\
Ideally, agents should be redirected to a static copy of the product catalog, or the site generates pages without prices (to be searchable by the big search engines but not reveal pricing.\
That way, you can personalize the prices, run trial offers, partition the country or the audience to conduct market tests, and so on.)\
In the worst case, the site sends the agent into a dead end so nothing is searchable.\
The robots most likely to respect robots.txt are the ones that might actually generate traffic (and revenue) for you, while the leeches ignore it completely.

Ways to deal with agressive scrapers:
Block it from your network.\
A CDN can do this, if using one.\
Otherwise, you can do it at the outer firewalls.

Some of the leeches send requests from legitimate IP addresses with real reverse DNS entries. ARIN is your friend here. Easy to block.\
Others mask their source addresses or make requests from dozens of different addresses. Some even change user-agent string between requests.\
*You may end up blocking quite a few subnets, so it’s a good idea to periodically expire old blocks to keep your firewalls performing well.*

Legal - Write some terms of use for your site that say users can view content only for personal or noncommercial purposes.\
When the screen scrapers start hitting your site, sic the lawyers on them.

Neither of these is a permanent solution. Consider it pest control - once you stop, the infestation will resume.

### Malicious Users

The overwhelming majority of malicious users are known as “script kiddies.”\
The primary risk to stability is the distributed denial-of-service (DDoS) attack. The attacker causes many computers, widely distributed across the Net, to start generating load on your site (e.g. typically a botnet)

Nearly all attacks target the applications rather than the network gear, forcing you to saturate your own outbound bandwidth, denying service to legitimate users and racking up huge bandwidth charges.

Session management is the most vulnerable point of a server-side web application.\
Application servers are particularly fragile when hit with a DDoS, so saturating the bandwidth might not even be the worst issue you have to deal with.

A specialized Circuit Breaker can help to limit the damage done by any particular host. This also helps protect you from the accidental traffic floods.

Network vendors all have products that detect and mitigate DDoS attacks.\
Proper configuring and monitoring of these products is essential.\
Shold be run in “learning” or “baseline” mode for at least a month to understand your normal, cyclic traffic patterns.

### Remember This
Users consume memory
- Each user’s session requires some memory
-- Minimize that memory to improve your capacity.
- Use a session only for caching so you can purge the session’s contents if memory gets tight

Users do weird, random things.
Users do things that you won’t predict.\
If there’s a weak spot in your application, they’ll find it through sheer numbers.\
Test scripts are useful for functional testing but too predictable for stability testing.\
Use fuzzing toolkits, property-based testing, or simulation testing.

Malicious users are out there.\
Become intimate with your network design; it helps avert attacks.\
Make sure your systems are easy to patch as you frequently need to.\
Keep your frameworks up-to-date, and keep yourself educated.

Users will gang up on you.\
Sometimes big user spikes can occur.\
Large mobs can trigger hangs, deadlocks, and obscure race conditions.\
Run special stress tests to hammer deep links or hot URLs.

## Blocked Threads
Managed runtime languages (e.g. C#, Java) very rarely crash crash.\
They get application errors, but it’s rare to see the kind of core dump that a C/C++ program would have.\
However, the interpreter can be running, and the application can still be totally deadlocked, doing nothing useful.\
The most common failure mode for applications built in these languages is navel-gazing — a happily running interpreter with every single thread sitting around waiting. Multithreading is complex.

You should supplement internal monitors (e.g. log file scraping, process monitoring, and port monitoring) with external monitoring.\
A mock client in another data center can run synthetic transactions on a regular basis. That client experiences the same view of the system that real users experience.\
If that client cannot process the synthetic transactions, then there is a problem, whether or not the server process is running.\
Metrics can also reveal problems quickly.\
Counters like “successful logins” or “failed credit cards” reveals problems long before an alert goes off.

Blocked threads can happen when you
- check resources out of a connection pool
- deal with caches/object registries
- make calls to external systems

It is normal for a thread to occasionally block whenever two (or more) threads try to access the same critical section at the same time.\
Assuming that the code was written by someone knowing multithreaded programming, then you can always guarantee that the threads will eventually unblock and continue.

Why blocked threads happen in production
- Error conditions and exceptions create too many permutations to test exhaustively
- Unexpected interactions can introduce problems in previously safe code
- Timing is crucial - probability that the app will hang goes up with the number of concurrent requests
- Developers never hit their application with 10,000 concurrent requests

These conditions mean that it’s very, very hard to find hangs during development; you can’t rely on “testing them out of the system.”\
The best way to improve your chances is to carefully craft your code.\
Use a small  set of primitives in known patterns, preferable a proven library.

You should not roll your own connection pool class. It is very difficult to make a reliable, safe, high-performance connection pool.

If you find yourself synchronizing methods on domain objects, you should probably rethink the design:\
Ensure that each thread can get its own copy of the object in question - if you are synchronizing the methods to ensure data integrity, then your application will break when it runs on more than one server.\
In-memory coherence doesn’t matter if there’s another server out there changing the data.\
Application scales better if request-handling threads never block each other.

A good way to avoid synchronization on domain objects is to *make your domain objects immutable*. Use them for querying and rendering.\
When you need to alter their state, do it by constructing and issuing a “command object.” (“Command Query Responsibility Separation”)
Avoids a large number of concurrency issues.

#### Spot the Blocking

Nothing in calling code tells you whether a calls is blocking or not

In Java, it’s possible for a subclass to declare a method synchronized that is unsynchronized in its superclass or interface definition.\
In C#, a subclass can annotate a method as synchronizing on “this.”\
These examples violate the Liskov substitution principle.

The Liskov substitution principle: any property true about objects of type T should also be true for objects of any subtype of T.\
I.e. a method without side effects in a base class should also be free of side effects in derived classes.\
A method that throws the exception E in base classes should throw only exceptions of type E (or subtypes of E) in derived classes.\
Java/C# generally does not allow violations of the substitution principle.

When subclasses add synchronization to methods, you cannot transparently replace an instance of the superclass with the synchronized subclass.\
Might seem like nitpicking, but it can be vitally important.

Example:

The basic implementation of the GlobalObjectCache interface is a simple object registry:

```
public synchronized Object get(String id) {
	Object obj = items.get(id);
	if(obj == null) {
		obj = create(id);
		items.put(id, obj);
	}
	return obj;
}
```

Only one thread may execute inside the method at a time.\
While one thread is executing this method, other callers will be blocked.

Part of the system needed to check the in-store availability of items by making expensive inventory availability queries to a remote system (taking a few seconds).\
The results were valid for at least fifteen minutes because of the way the inventory system worked.\
To spare the inventory system, the developer decided to cache the resulting Availability object, using a read-through cache.
- On hit, it returns the cached object 
- Else, it does the query, caches the result, and returns it

The developer extended of GlobalObjectCache, overriding the get() method to make the remote call.\
Functionally, RemoteAvailabilityCache was a nice piece of work, but under stress it had a nasty failure mode.\
The inventory system was undersized (see Unbalanced Capacities, p. 75) - when the front end got busy, the back end would be flooded with requests and eventually crashed. 
At that point any thread calling RemoteAvailabilityCache.get() would block, because one single thread was inside the create() call, waiting for a response that would never
come. 

Antipatterns combined to accelerate the growth of cracks:\
The conditions for failure were created by the blocking threads and the unbalanced capacities.\
The lack of timeouts in the integration points caused the failure in one layer to become a cascading failure.\
Ultimately, this combination of forces brought down the entire site.

No one designed this failure mode into the combined system, but no one designed it out either.


#### Use Caching, Carefully

The maximum memory usage of application-level caches should be configurable.\
Caches that do not limit memory consumption will eventually eat away at the memory available for the system.\
Causes GC to spend more and more time attempting to recover enough memory to process requests.\
By consuming memory needed for other tasks, the cache will actually cause a serious slowdown.

You must monitor hit rates for the cached items to see whether most items are being used from cache.\
If hit rates are very low, then the cache is not giving any performance gains - might even be slower than not using the cache.\
Caching an item is a bet that the cost of generating it once, plus the cost of hashing and lookups, is less than the cost of generating it every time it’s needed.\
If a particular cached object is used only once during the lifetime of a server, then caching it is useless.

Avoid caching things that are cheap to generate.\
Caches should be built using weak references to hold the cached item itself.

Any cache presents a risk of stale data and should have an invalidation strategy to remove items from cache when its source data changes.\
The chosen strategy can have a major impact on your system’s capacity.\
E.g. a point-to-point notification might work well when there are ten or twelve instances in your service.\
If there are thousands of instances, then point-to-point unicast is not effective and you need to look at either a message queue or some form of multicast notification.\
When invalidating, be careful to avoid the Database Dogpile.

### Libraries
Libraries are notorious sources of blocking threads.\
Many libraries that work as service clients do their own resource pooling inside the library.\
These often make request threads block forever when a problem occurs.\
Rarely allow you to configure their failure modes, e.g. what to do when all connections are tied up waiting for replies that’ll never come.

If it’s an open source library, then you may be able to find and fix such problems.\
You may even be able to search through the issue log to see if other people have already done the hard work for you.

If it’s vendor code, then you may need to exercise it yourself to see how it behaves under normal conditions and under stress.\
E.g. what does it do when all connections are exhausted?\
If it breaks easily, you need to protect your request-handling threads.\
If you can set timeouts, do so.\
If not, you might have to resort to some complex structure such as wrapping the library with a call that returns a future.\
Inside the call, you use a pool of your own worker threads. Then when the caller tries to execute the dangerous operation, one of the worker threads starts the real call.
- If the call makes it through the library in time, then the worker thread delivers its result to the future
- Otherwise, the request-handling thread abandons the call, even though the worker thread might eventually complete
-- Beware: Go too far down this path and you’ll find you’ve written a reactive wrapper around the entire client library

With vendor code, it may also be worth bugging them for a better client library.

A blocked thread is often found near an integration point and can quickly lead to chain reactions if the remote end of the integration fails.\
Blocked threads and slow responses can create a positive feedback loop, amplifying a minor problem into a total failure.

### Remember This

Blocked Threads antipattern is the proximate cause of most failures.\
Application failures nearly always relate to Blocked Threads in one way or another, including “gradual slowdown” and “hung server.”\
The Blocked Threads antipattern leads to Chain Reactions and Cascading Failures antipatterns.

Scrutinize resource pools.
- Blocked Threads usually happens around resource pools
-- particularly database connection pools

A deadlock in the database can cause connections to be lost forever; so can incorrect exception handling

Use proven primitives.\
Learn and apply safe primitives.\
Implementing your own producer/consumer queue is hard.\
Any library of concurrency utilities has more testing than your newborn queue.

Defend with Timeouts.\
You cannot prove that your code has no deadlocks in it, but you can make sure that no deadlock lasts forever.
- Avoid infinite waits in function calls
-- use a version that takes a timeout parameter.
- Always use timeouts, even though it means you need more error-handling code

Beware the code you cannot see.\
All manner of problems can lurk in the shadows of third-party code.\
Be very wary. Test it yourself.\
Whenever possible, acquire and investigate the code for surprises and failure modes.\
Prefer open source libraries to closed source for this reason.

## Self-Denial Attacks

Any situation in which the system—or the extended system that includes humans—conspires against itself.\
E.g. hype marketing creating too much load (like a secret coupon for a limited set of users that share it with friends).

In a horizontal layer that has some shared resources, it’s possible for a single rogue server to damage all the others.\
E.g. in an ATG-based infrastructure, one lock manager always handles distributed lock management to ensure cache coherency.

Any server that wants to update a RepositoryItem with distributed caching enabled must acquire the lock, update the item, release the lock, and then broadcast a cache invalidation for the item.\
Lock manager is a singular resource.\
As the site scales horizontally, the lock manager becomes a bottleneck and eventually a risk.\
If a popular item is inadvertently modified (e.g. due to a programming error), then you can end up with thousands of request-handling threads on hundreds of servers all serialized waiting for a write lock on one item.

### Avoiding Self-Denial
Use a “shared-nothing” architecture, i.e. each server can run without knowing anything about any other server.\
The machines don’t share databases, cluster managers, or any other resource.\
In practice there’s always some amount of contention and coordination among the servers, but we can sometimes approximate shared-nothing.

If not possible, apply decoupling middleware to reduce the impact of excessive demand.\
Or make the shared resource itself horizontally scalable through redundancy and a backside synchronization protocol.

You can design a fallback mode for the system to use when the shared resource is not available or not responding.\
E.g. if a lock manager that provides pessimistic locking is not available, the application can fall back to using optimistic locking.\
If you have a little time to prepare and are using hardware load balancing for traffic management, you can either set aside a portion of your infrastructure or provision new cloud resources to handle the promotion or traffic surge.\
Only works if the extraordinary traffic is directed at a portion of the system - even if the dedicated portion melts down, at least the rest of the system’s regular behavior is available.

Autoscaling can help when the traffic surge does arrive, but watch out for the lag time.\
Spinning up new virtual machines takes precious minutes.\
You should “pre-autoscale” by upping the configuration before the marketing event goes out.

As for the human-facilitated attacks, the keys are training, education, and communication.\
If you keep the lines of communication open, you might have a chance to protect the systems from the coming surge.

### Remember This
Keep the lines of communication open.
Self-denial attacks originate inside your own organization, when people cause self-inflicted wounds by creating their own flash mobs and traffic spikes.\
You can aid and abet these marketing efforts and protect your system at the same time, but only if you know what’s coming.
- Make sure nobody sends mass emails with deep links
- Send mass emails in waves to spread out the peak load
- Create static “landing zone” pages for the first click from these offers
- Watch out for embedded session IDs in URLs

Protect shared resources.
When traffic surges
- Programming errors
- unexpected scaling effects
- shared resources

create risks.\
Watch out for when increased front-end load causes exponentially increasing back-end processing.

Expect rapid redistribution of great offers.
Anybody who thinks they’ll release a special deal for limited distribution is asking for trouble.\
There’s no such thing as limited distribution.\
Even if you limit the number of times a fantastic deal can be redeemed, you’ll still get crushed with people hoping to make use of it.

## Scaling Effects

In the development environment, every application runs on one machine.\
In QA, pretty much every application looks like one or two machines.\
In production applications can be really, really small, medium, large, or humongous.\
Because the development and test environments rarely replicate production sizing, it can be hard to determine where scaling effects can happen.

#### Point-to-Point Communications
Each instance has to talk directly to every other instance.\
One of the worst places that scaling effects happen.

Total number of connections goes up as the square of the number of instances.\
With a hundred instances, and the O(n^2) scaling becomes quite painful.\
This is a multiplier effect driven by the number of application instances.\
Depending on the eventual size of your system, O(n^2) scaling might be acceptable.

Important to distinguish between point-to-point *inside* a service versus *between* services:\
The usual pattern between services is fan-in from external farm of machines to a load balancer in front of your machines.\
Here every service isnt calling other service.

It is unlikely you can build a test farm the same size as your production environment.\
This type of defect cannot be tested out, but must be designed out.

No universal “best” choice, only a suitable choice for a particular set of circumstances.\
If the application will only ever have two servers, then point-to-point communication is perfectly fine (assuming the communication is written so it won’t block when the other server dies).
As the number of servers grows, then a different communication strategy is needed.

Depending on your infrastructure, you can replace point-to-point with:
- UDP broadcasts
- TCP or UDP multicast
- Publish/subscribe messaging
- Message queues

Broadcasts
- aren’t bandwidth-efficient
- cause additional load on servers that aren’t interested in the messages, since the servers’ NIC gets the broadcast and must notify the TCP/IP stack

Multicasts
- more efficient, since they permit only the interested servers to receive the message

Publish/subscribe messaging
- better still, since a server can pick up a message even if it wasn’t listening at the precise moment the message was sent
- often associated with serious infrastructure cost

*“Do the simplest thing that will work.”*

### Shared Resources

Commonly seen in the form of a service-oriented architecture or “common services” project - some facility that all members of a horizontally scalable layer need to use.\
With some application servers, the shared resource will be a cluster manager or a lock manager.\
When the shared resource gets overloaded, it’ll become a capacity bottleneck.

When the shared resource is redundant and nonexclusive, e.g. it can service several of its consumers at once, then there’s no problem.\
If it saturates, you can add more, thus scaling the bottleneck.

The most scalable architecture is the shared-nothing architecture.\
Each server operates independently, without need for coordination or calls to any centralized services.\
Capacity scales linearly with the number of servers.

The issue with a shared-nothing architecture is that it might scale better at the cost of failover.

Consider session failover:
A user’s session resides in memory on an application server.\
When that server goes down, the next request from the user will be directed to another server.\
Transition should be invisible to the user, so the user’s session should be loaded into the new application server - requires coordination between the original application server and some other device.

The application server could send the user’s session to a session backup server after each page request.\
Maybe it serializes the session into a database table or shares its sessions with another designated application server.

There are numerous strategies for session failover - all involve getting the user’s session off the original server.\
Typically implies some level of shared resources.\
You can approximate a shared-nothing architecture by reducing the fan-in of shared resources, i.e., cutting down the number of servers calling on the shared resource.\
For session failover, you could do this by designating pairs of application servers that each act as the failover server for the other.

Often a shared resource will be allocated for exclusive use while a client is processing some unit of work.\
In these cases, the probability of contention scales with the number of transactions processed by the layer and the number of clients in that layer.

When the shared resource saturates, you get a connection backlog.\
When the backlog exceeds the listen queue, you get failed transactions.\
At that point, nearly anything can happen.\
It depends on what function the caller needs the shared resource to provide.

Particularly in cache managers (providing coherency for distributed caches), failed transactions lead to stale data or, worse, loss of data integrity.

### Remember This
Examine production versus QA environments to spot Scaling Effects.\
Reveal themelves when you move from small development and test environments to full-sized production environments.\
Patterns that work fine in small environments or one-to-one environments might slow down or fail completely when you move to production sizes.

Watch out for point-to-point communication.\
Point-to-point scales badly, since the number of connections increases as the square of the number of participants.\
Consider how large your system can grow while still using point-to-point connections — it might be sufficient.\
Once you’re dealing with tens of servers, you will probably need to replace it with some kind of one-to-many communication.

Watch out for shared resources.
Shared resources can be a bottleneck, a capacity constraint, and a threat to stability.\
If your system must use some sort of shared resource, stresstest it heavily.\
Make sure clients will keep working if the shared resource gets slow or locks up.

## Unbalanced Capacities
Whether your resources take months, weeks, or seconds to provision, you can end up with mismatched ratios between different layers.
Makes it possible for one tier/service to flood another with requests beyond its capacity - especially when dealing with rate-limited or throttled APIs.

It might be impractical to evenly match capacity in each system for a lot of reasons.\
I.e. one tier might generally send a limited number of requests to another, but then a spike may occur due to marketing.\
Waste of money to run the downstream tier at a ratio matching the upstream tier, just to be able to handle the rare spike.

Instead of building every service large enough to meet the potentially overwhelming demand from the frontend, you must build both callers and providers to be resilient in the face of a tsunami of requests.

For the caller:
- *Circuit Breaker* relieves the pressure on downstream services when responses get slow or connections get refused

For service providers,
- *Handshaking* and *Backpressure* to inform callers to throttle back on the requests
- Consider *Bulkheads* to reserve capacity for high-priority callers of critical services

### Drive Out Through Testing
Unbalanced capacities are rarely discovered in QA.\
QA for a system usually consists of just two servers, i.e. two servers represent the front-end system and two servers represent the back-end system (1:1 ratio).\
In production, the ratio could be ten to one or worse.

You can't duplicate production in QA, but you can apply a test harness.\
By mimicking a back-end system wilting under load, the test harness helps you verify that your front-end system degrades gracefully.\
If you provide a service, you probably expect a “normal” workload.\
I.e. you would expect that today’s distribution of demand and transaction types will closely match yesterday’s workload.\
If all else remains unchanged, then that’s a reasonable assumption.

Factors that can change the workload:
- marketing campaigns
- publicity
- new code releases in the front-end systems
- links on social media and link aggregators

As a service provider, you’re even further removed from the marketers who would deliberately cause these traffic changes. Surges in publicity are even less predictable.

Be ready for anything.
- use capacity modeling to make sure you’re at least in the ballpark. Three thousand threads calling into seventy-five threads is not in the ballpark
- Don’t just test your system with your usual workloads
-- Take the number of calls the front end could possibly make, double it, and direct it all against your most expensive transaction
-- If your system is resilient, it might slow down or even start to fail fast if it can’t process transactions within the allowed time, but it should recover once the load goes down
-- Crashing, hung threads, empty responses, or nonsense replies indicate your system won’t survive and could start a cascading failure
- If you can, use autoscaling to react to surging demand
-- Not perfect - suffers from lag and can just pass the problem down the line to an overloaded platform service
-- Impose a financial constraint on your autoscaling as a risk management measure

### Remember This
Examine server and thread counts.\
In development and QA, your system probably looks like one or two servers, and so do all the QA versions of the other systems you call.\
In production, the ratio might be more like ten to one instead of one to one.\
Check the ratio of front-end to back-end servers, along with the number of threads each side can handle in production compared to QA.

Observe near Scaling Effects and users.\
Unbalanced Capacities is a special case of Scaling Effects: one side of a relationship scales up much more than the other side.\
A change in traffic patterns—seasonal, market-driven, or publicity-driven—can cause a usually benign front-end system to suddenly flood a back-end system.

Virtualize QA and scale it up.\
Even if your production environment is a fixed size, don’t let your QA languish at a measly pair of servers.\
Try test cases where you scale the caller and provider to different ratios.\
You should be able to automate this all through your data center automation tools.

Stress both sides of the interface.
If you provide the back-end system, see what happens if it suddenly gets ten times the highest-ever demand, hitting the most expensive transaction.\
Does it fail completely? Does it slow down and recover?\
If you provide thefront-end system, see what happens if calls to the back end stop responding or get very slow.

## Dogpile

The steady-state load on a system can be significantly different than the startup or periodic load.\
Imagine a farm of app servers booting up.\
Each server
- needs to connect to a database and load some amount of reference or seed data
- starts with a cold cache and only gradually gets to a useful working set

Until then, most HTTP requests translate into one or more database queries.\
That means the transient load on the database is much higher when applications start up than after they’ve been running for a while.

When a bunch of servers impose this transient load all at once, it’s called a dogpile.\
A dogpile can occur when:
- booting up several servers, such as after a code upgrade and restart
- a cron job triggers at midnight (or on the hour for any hour, really)
- the configuration management system pushes out a change

Configuration management tools may allow you to configure a randomized “slew” that will cause servers to pull changes at slightly different times, dispersing the dogpile across several seconds.

Dogpiles can also occur when some external phenomenon causes a synchronized “pulse” of traffic (like pedestrians waiting for a light).\
Look out for places where many threads can get blocked waiting for one thread to complete.
When the logjam breaks, the newly freed threads will dogpile any other downstream system.

A pulse can develop during load tests, if the virtual user scripts have fixed time waits in them.\
*Every pause in a script should have a small random delta applied.*

### Remember This
Dogpiles force you to spend too much to handle peak demand.\
A dogpile concentrates demand.\
Requires a higher peak capacity than you’d need if you spread the surge out.

Use random clock slew to diffuse the demand.\
Don’t set all your cron jobs for midnight or any other on-the-hour time.\
Mix them up to spread the load out.

Use increasing backoff times to avoid pulsing.\
A fixed retry interval will concentrate demand from callers on that period.\
Use a backoff algorithm so different callers will be at different points in their backoff periods.

## Force Multiplier
Like a lever, automation allows administrators to make large movements with less effort - a force multiplier.

Reddit downtime story\
Service discovery failure

This situation arises mainly with “control plane” software.\
The “control plane” refers to software that exists to help manage the infrastructure and applications, unrelated to user functionality.\
I.e. 
- logging
- monitoring
- schedulers
- scalers
- load balancers
- configuration management

The common thread in these failures is that the automation is not being used to simply enact the will of a human administrator.\
Rather, the control plane senses the current state of the system, compares it to the desired state, and effects changes to bring the current state into the desired state.

In the Reddit failure, ZooKeeper held a representation of the desired state that was (temporarily) incorrect.\
In the case of the discovery service, the partitioned node was not able to correctly sense the current state.

A failure can also result when the “desired” state is computed incorrectly and may be impossible or impractical.\
E.g. a naive scheduler might try to run enough instances to drain a queue in a fixed amount of time. Depending on the individual jobs’ processing time, the number of instances might be “infinity.”

### Controls and Safeguards

Industrial robots have layers of safeguards to prevent damage to people, machines, and facilities.\
In particular, limiting devices and sensors detect when the robot is not operating in a “normal” condition

Similar safeguards can be implemented in control plane software:
- If observations report that more than 80 percent of the system is unavailable, it’s more likely to be a problem with the observer than the system
- Apply hysteresis. Start machines quickly, but shut them down slowly. 
-- Starting new machines is safer than shutting old ones off
- When the gap between expected state and observed state is large, signal for confirmation
-- Equivalent to a big yellow rotating warning lamp on an industrial robot
- Systems that consume resources should be stateful enough to detect if they’re trying to spin up infinity instances.
- Build in deceleration zones to account for momentum. 
-- Suppose control plane senses excess load every second, but it takes five minutes to start a virtual machine to handle the load
-- Must make sure not to start 300 virtual machines because the high load persists


### Remember This
Ask for help before causing havoc.\
Infrastructure management tools can make very large impacts very quickly.\
Build limiters and safeguards into them so they won’t destroy your whole system at once.

Beware of lag time and momentum.\
Actions initiated by automation take time.\
Usually longer than a monitoring interval, so make sure to account for some delay in the system’s response to the action.

Beware of illusions and superstitions.\
Control systems sense the environment, but they can be fooled. They compute an expected state and a “belief” about the current state. Either can be mistaken.


### Slow Responses

Generating a slow response is worse than refusing a connection or returning an error, particularly in the context of middle-layer services.

A quick failure allows the calling system to finish processing the transaction rapidly.\
Whether that is ultimately a success or a failure depends on the application logic.\
A slow response ties up resources in the calling system and the called system.\
Usually result from excessive demand.

When all available request handlers are already working, there’s no slack to accept new requests.\
Slow responses may also be a symptom of some underlying problem:\
Memory leaks often manifest via Slow Responses as the virtual machine works harder and harder to reclaim enough space to process a transaction.\
Shows as a high CPU utilization - all due to GC, not work on the transactions themselves.\
Can also result from network congestion - relatively rare inside a LAN but can definitely happen across a WAN, especially in chatty protocols.

More frequently, applications allow their sockets’ send buffers to get drained and receive buffers to fill up, causing a TCP stall.\
This usually happens in a hand-rolled, low-level socket protocol, in which the read() routine does not loop until the receive buffer is drained.

Slow responses tend to propagate upward from layer to layer in a gradual form of cascading failure.\
Your system should have the ability to monitor its own performance, so it can tell when it isn’t meeting its SLA.\
Suppose your system is a service provider that’s required to respond within one hundred milliseconds.\
When a moving average over the last twenty transactions exceeds one hundred milliseconds, your system could start refusing requests.

This could be at the application layer, in which the system would return an error response within the defined protocol.\
Or it could be at the connection layer, by refusing new socket connections.

Any such refusal to provide service must be well documented and expected by callers, so they can gracefully handle the errors.

### Remember This
Slow Responses trigger Cascading Failures.\
Upstream systems experiencing Slow Responses will themselves slow down and might be vulnerable to stability problems when the response times exceed their own timeouts.

For websites, Slow Responses cause more traffic - users waiting for pages frequently hit the Reload button

Consider Fail Fast.\
If your system tracks its responsiveness, then it can tell when it’s getting slow.\
Consider sending an immediate error response when the average response time exceeds the system’s allowed time (or at the very least, when the average response time exceeds the caller’s timeout!).

Hunt for memory leaks or resource contention.\
Contention for an inadequate supply of database connections produces Slow Responses.\
Slow Responses also aggravate that contention, leading to a self-reinforcing cycle.\
Memory leaks cause excessive effort in the garbage collector, resulting in Slow Responses.\
Inefficient low-level protocols can cause network stalls, also resulting in Slow Responses.

## Unbounded Result Sets
Design with skepticism to achieve resilience.\
Ask, “What can system X do to hurt me?” and then design a way to handle it.

Your application probably treats its database server with far too much trust.\
Often some code will send a query to the database and loop over the result set, processing each row.\
Processing a row typically means adding a new data object to a collection.

What happens when the database returns five million rows instead of the usual hundred?\
Unless your application explicitly limits the number of results it’s willing to process, it can end up exhausting its memory or spinning in a while loop long after the user loses interest.

Black monday story w. some good diagnostic info

Can occur when querying databases or calling services.\
Or when front-end applications call APIs.

Because datasets in development tend to be small, the application developers may never experience negative outcomes.\
After a system has been in production for a year, however, even a traversal such as “fetch customer’s orders” can return huge result sets.\
When that happens, the best, most loyal customers get the worst performance.

In the abstract, an unbounded result set occurs when the caller allows the other system to dictate terms.\
Failure in handshaking - in any API or protocol, the caller should always indicate how much of a response it’s prepared to accept.
- TCP uses the “window” header field
- Search engine APIs allow the caller to specify how many results to return and what the starting offset should be
- No standard SQL syntax to specify result set limits
-- ORMs support query parameters that can limit results returned from a query but do not usually limit results when following an association (such as container to contents)

Beware of any relationship that can accumulate unlimited children, such as orders to order lines or user profiles to site visits.\
Entities that keep an audit trail of changes are also suspect.\
Beware of the way that patterns of relationships can change from QA to production as well.\
Number of connections per user behaves like a power law distribution.\
If you test with bell-curve distributed relationships, you would never expect to load an entity that has a million times more relationships than the average - guaranteed to happen with a power law.

If handcrafting your own SQL, limit the number of rows to fetch:

Microsoft SQL Server
```
SELECT TOP 15 colspec FROM tablespec
```
Oracle (since 8i)
```
SELECT colspec FROM tablespec
WHERE rownum <= 15
```
MySQL and PostgreSQL
```
SELECT colspec FROM tablespec
LIMIT 15
```

Don't query for the full results but break out of the processing loop after reaching the maximum number of rows in the application code.
Provides added stability on the application server, it does so at the expense of wasted database capacity.

Unbounded result sets are a common cause of slow responses. 
Can result from violation of steady state.

### Remember This
Use realistic data volumes.\
Typical development and test data sets are too small to exhibit this problem.\
Use production-sized data sets to see what happens when your query returns a million rows that you turn into objects.\
Then you’ll also get better information from your performance testing when you use production-sized test data.

Paginate at the front end.\
Build pagination details into your service call. 
- request should include a parameter for
-- first item
-- the count
- reply should indicate (roughly) how many results there are

Don’t rely on the data producers.\
Even if you think a query will never have more than a handful of results, beware: it could change without warning because of some other part of the system. 
Unless your query selects exactly one row, it has the potential to return too many.\
Don’t rely on the data producers to create a limited amount of data.\
Sooner or later, they’ll go berserk and fill up a table for no reason.

Put limits into other application-level protocols.\
Service calls, RMI, DCOM, XML-RPC, and any other kind of request/reply call are vulnerable to returning huge collections of objects, thereby consuming too much memory.