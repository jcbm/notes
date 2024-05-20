# Chapter 5 - Stability patterns
Expect failures
Patterns provide architecture/design guidance to reduce/eliminate/mitigate the effects of cracks in the system


## Timeouts 
Networks are faulty - Responses are not guaranteed to arrive
The default wait is infinite - timeouts allow you to stop waiting eventually
Provides fault isolation and protects you from problems another service.
Apply to a general class of problems - help systems recover from unanticipated events.

Can be useful within a single service too
- *any* resource pool can be exhausted
- Pools typically block client threads when a resource is not available currently
-- *must have a timeout to ensure that calling threads eventually unblock, whether resources become available or not.*

Beware of
- commercial software client libraries - may not allow you to configure sockets.
- language-level synchronization or mutexes - always use the form that takes a timeout argument.

Language runtimes that use callbacks or reactive programming styles let you specify timeouts easily.

Handle pervasive timeouts by structuring long-running operations into a set of re-usable primitives. 
E. g. a typical db call can hang in multiple places 
- check out a database connection from a resource pool
- run a query
- turn the result set into objects 
- return the connection to the pool

To avoid duplicate error/timeout handling code each place this happens
- use a *generic gateway* to provide the template for connection handling, error handling, query execution, and result processing
- use a *query object* to represent the variable part
Calling code can then provide the essential logic.
Having this in a single class also makes it easier to apply Circuit Breaker pattern.

Often used with retries - if an operation times out, the software tries again. 
Should have some delay - fast retries are likely to fail again
- If the operation failed because of a significant problem, it’s likely to immediately fail again 
- Certain transient failures might be overcome with a retry (e.g., dropped packets over a WAN) 
-- Inside a data center the failure is probably because of something wrong with the other end of a connection 
- Problems on the network, or with other servers, tend to last for a while

Retrying an operation can increase your response time past the client's timeout and unnecessarily tie up its resources. 
If you cannot complete an operation because of a timeout, *you should return a result immediately to the waiting client*
- a failure
- a success
- a note that the work is queued for later execution

Queuing the work for a slow retry later makes a system more robust.
- ensures that once the remote server is healthy again, the overall system will recover. 
- work does not need to be lost completely just because part of the larger system isn’t functioning.  

Mail server example 

How fast is fast enough? It depends on your application and your users. 
For a service behind a web API, “fast enough” is probably between 10 and 100 milliseconds. Beyond that, you’ll start to lose capacity and customers.

Synergy with circuit breakers: A circuit breaker can be tripped to the “off” state if too many timeouts occur.

Timeout and Fail Fast both address latency problems. 
- Timeouts: protects your system from someone else’s failure
-- applies primarily to *outbound* requests. 
- Fail Fast: when you need to report why you won’t be able to process some transaction
-- applies to *incoming* requests

Aids with unbounded result sets by preventing the client from processing the entire result set, but there's better solutions - timeouts are only a stopgap.
 


## Circuit breaker
Allows a subsystem to fail without breaking the entire system. 
Once the subsystem is good again, the circuit breaker can be reset to restore full function to the system.
Differs from retries:  exist to prevent operations, not reexecute them.

Circumvents calls when the system is not healthy.
State machine: Closed, open and half-open states. 
In the normal “closed” state, the circuit breaker executes operations as usual (calls to another system/internal operations that are subject to timeout or other execution failure).
- If the call succeeds, nothing extraordinary happens 
- If it fails, however, the circuit breaker registers the failure

When the absolute number/frequency of failure exceeds a threshold, the circuit breaker trips and “opens” the circuit.
- Calls to the breaker fail immediately, without attempting to execute the real operation 
- After a suitable amount of time, the breaker goes into the “half-open” state where the next call to is allowed to execute the dangerous operation 
-- If it succeeds, it returns to the “closed” state
-- Else it returns to the open state until another timeout elapses

May be configured to track different types of failures separately. 
E.g. setting a lower threshold for “timeout calling remote system” failures than “connection refused” errors.

When in open state, incoming calls must be handled. 
- let calls fail immediately
-- E.g. by throwing an exception different than an ordinary timeout, so that the caller can provide useful feedback
- “fallback” strategy 
-- return the last good response or a cached value. 
- return a generic answer rather than a personalized one 
- call a secondary service when the primary is not available

A way to *automatically degrade functionality when the system is under stress*. 
Every fallback strategy can impact the business of the system. 
-- Essential to involve the system’s stakeholders when deciding how to handle calls made when the circuit is open.
-- should a retail system accept an order if it can’t confirm availability of the customer’s items?
-- if it can’t verify the customer’s credit card or shipping address?  


Implementation details to consider: 
What constitutes “too many failures”? 
Just adding up all the faults isn’t useful, i.e. five faults over five hours vs. in the last thirty seconds. 
Use fault *density* instead. 

Leaky Bucket pattern: 
Counter incremented every time there's a fault. 
A background thread/timer decrements the counter periodically (if greater than zero).
If the count exceeds a threshold, faults must be frequent.

The state of the circuit breakers in a system is important to operations. 
*State changes should be logged, and the current state should be exposed for querying and monitoring.* 
The frequency of state changes is a useful metric to chart over time; leading indicator of problems elsewhere in the enterprise. 

*Operations should have a way to directly trip or reset the circuit breaker.* 
The circuit breaker is a convenient place to gather metrics about call volumes and response times.

A circuit breaker should be built at the scope of a single process. I.e. a breaker state affects every thread in a process, but is not shared across multiple processes. 
Results in some loss of efficiency when multiple instances of the caller each independently discover that the provider is down. 
Using a shared state would introduce another out of-process communication and thus a new failure mode.
Even when just shared within a process, circuit breakers face all the multithreaded programming terrors. 
Be sure to avoid accidentally single-threading all calls to a remote system! 
Better to use an existing library. 

*Effective at guarding against integration points, cascading failures, unbalanced capacities, and slow responses.* 
Work so closely with timeouts that they often track timeout failures separately from execution failures.


## Bulkhead

Bulkheads are used in ships to contain damage; ship is partitioned into separate, watertight compartments that prevents water from moving from one section to another.
 By partitioning your systems, you can keep a failure in one part of the system from destroying everything. 
Typically in the form of redundancy. 
Examples
- In a group of independent servers, hardware failure in one does not affect the others  
- When multiple application instances run on a server and one crashes, the others will still be running 

*Redundant VMs* are not as robust as redundant physical machines. 
Most VM provisioning tools do not allow you to enforce physical isolation, so more than one VM may end up running on the same physical box.

Largest scale: a mission-critical service can be implemented as several independent farms of servers, with certain farms reserved for use by critical applications and others available for noncritical uses.
E.g. a ticketing system could provide dedicated servers for customer check-in. These would not be affected if other, shared servers are overwhelmed with “flight status” queries. 

Cloud: you should run instances in different divisions of the service (e.g., across zones and regions in AWS) - very large-grained chunks with strong partitioning between them.

Functions as a service: basically every function invocation runs in its own compartment.

Hidden coupling through common downstream service:
Services Foo and Bar both use the service Baz.
Makes each system vulnerable to the other.
If Foo gets crushed under user load, goes rogue because of some defect, or triggers a bug in Baz, Bar will also suffer. 

The coupling makes diagnosing problems (particularly performance-related) in Bar very difficult. 
Baz maintenance windows must be coordinated with both Foo and Bar.

Assuming Foo/Bar are critical systems with strict SLAs, it’d be safer to partition Baz into two pools.
Dedicating some capacity to each critical client removes most of the hidden linkage (they would likely still share a database and therefore subject to deadlocks across instances).

It would be better to preserve all capabilities. 
Assuming that failures will occur, you must consider how to minimize the damage caused by a failure. 
Hard to do and one rule cannot apply in every case - you must examine the impact to the business of each loss of capability and cross-reference impacts against the architecture of the systems. 
*The goal is to identify the natural boundaries that let you partition the system in a way that is both technically feasible and financially beneficial.*
Boundaries of this partitioning may be aligned with the callers, with functionality, or with the topology of the system.

With cloud-based systems and software-defined load balancers, bulkheads do not need to be permanent.
Using automation a cluster of VMs can be carved out and the load balancer can direct traffic from a particular consumer to that cluster.
- Similar to A/B testing, but a protective measure rather than an experiment
- Dynamic partitions can be made and destroyed as traffic patterns change

At smaller scale, *process binding* is an example of partitioning via bulkheads.
Binding a process to a core or group of cores ensures that the OS schedules that process’s threads only on the designated core or cores.
Regarded as a performance tweak - reduces the cache bashing that happens when processes migrate from one core to another
- If a process goes crazy and starts using all CPU cycles, it can drag down an entire host machine 
- If bound and thus limited to a single core, it can only use all available cycles on that one

You can partition the threads inside a single process, with separate thread groups dedicated to different functions. 
E.g. it’s often helpful to reserve a pool of request-handling threads for administrative use.
That way, even if all request-handling threads on the application server are hung, it can still respond to admin requests, e.g. to collect data for postmortem analysis or a request to shut down.
 
*Bulkheads are effective at maintaining service, or partial service, even in the face of failures.* 
Especially useful in SOA, where the loss of a single service could have repercussions throughout the enterprise. 

#### Remember This

Bulkheads partition capacity to preserve partial functionality when bad things happen
Pick a useful granularity
-- thread pools inside an application
-- CPUs in a server
-- servers in a cluster

Consider Bulkheads particularly with shared services models
- Failures in service-oriented or microservice architectures can propagate very quickly 
- If your service goes down because of a Chain Reaction, does the entire company come to a halt? Then use Bulkheads!

## Steady State
 
Keep people off production systems to the greatest extent possible.
Whenever a human touches a server is an opportunity for unforced errors.
Servers shouldnt be treated  as “pets” rather than “cattle”, inevitably leads to fiddling. 

The system should be able to run at least one release cycle without human intervention. 
The logical extreme on the “no fiddling” scale is immutable infrastructure. 

“One release cycle” may be tough if the system is deployed once a quarter.
But a microservice being continuously deployed from version control should be easy to stabilize for a release cycle.
Unless the system is crashing every day, the most common reason for log-ins will probably be cleaning up log files or purging data.

Any mechanism that accumulates resources 
- log files in the filesystem
- rows in the database
- caches in memory

is like a bucket that fills up at a certain rate, based on the accumulation of data.
Must be drained too or it will eventually overflow:
- servers go down
- databases get slow or throw errors
- response times increase

The Steady State pattern says that *for every mechanism that accumulates a resource, some other mechanism must recycle that resource*.

#### Data Purging
Computing resources are always finite; therefore, you cannot continually increase consumption without limit. 
Often overlooked issue in new software.
Some day your little database will grow up.  
In the worst case, it’ll start undermining the whole system. 
The most obvious symptom of data growth will be steadily increasing I/O rates on the database servers. 
You may also see increasing latency at constant loads.
Data purging is nasty, detail-oriented work.
Referential integrity constraints in a relational database help a lot. 
- Difficult to cleanly remove obsolete data without leaving orphaned rows. 
Important to ensure that applications still work once the data is gone - takes coding and testing.

General rules (depends on database/ sed):
RDBMS plus ORM tends to deal badly with dangling references, for example, whereas a document-oriented database won’t even notice.
As a consequence, data purging always gets left until after the first release is out. 
The rationale is, “We’ve got six months after launch to implement purging.” 
After launch, there are always emergency releases to fix critical defects or add “must-have” features from marketers tired of waiting for the software to be done. 
The first six months can slip away pretty quickly, but when that first release launches, a fuse is lit.

#### Log Files

Left unchecked log files on individual machines will eventually fill up their containing filesystem. 
Whether that’s a volume set aside for logs, the root disk, or the application installation directory, it means trouble. 
Jeopardizes stability due to the negative effects that can occur when the filesystem is full.

UNIX
- The last 5–10 percent (configurable) of space is reserved for root
- An application will get I/O errors when the filesystem is 90-95 percent full. 
- If the application is running as root, then it can consume the very last byte of space 

Windows
- an application can always use the very last byte 

For both, the OS will report errors back to the application.
In the best-case scenario, the logging filesystem is separate from any critical data storage (such as transactions), and the application code protects itself enough that users never realize anything is amiss. 
Less ideal, but still tolerable, is a nicely worded error message asking the users to have patience and try again later. 

It’s better to avoid filling up the filesystem in the first place
- Log file rotation requires minimal configuration 
- For legacy code, third-party code, or code that doesn’t use a logging framework
-- UNIX: logrotate utility  
-- Windows: try building logrotate under Cygwin, or write a your own script

Make sure that all log files will get rotated out and eventually purged, though, or you’ll eventually spend time fixing the tool that’s supposed to help you fix the system.

Log files on production systems have a terrible signal-to-noise ratio. 
Best to get them off the individual hosts as quickly as possible and to a centralized logging server, such as Logstash, where they can be indexed, searched, and monitored.

Sometimes logs needs to be stored for long (e.g. financial systems due to Sarbanes–Oxley Act of 2002): 
- Get logs off of production machines as quickly as possible
- Store them on a centralized server and monitor it closely for tampering

#### In-Memory Caching
A untended cache can quickly use up all of a server's memory. 
Low memory conditions are a threat to both stability and capacity. 

When building a cache, consider
- Is the space of keys finite or infinite?
- Does the cached items change?

If infinite keys, a cache size limits must be enforced and the cache needs some form of cache invalidation. 
The simplest mechanism is a *time-based cache flush*. 
Alternatively *least recently used (LRU)* or *working-set algorithms*.

Improper use of caching is the major cause of memory leaks, leading to daily server restarts and thus production server fiddling.
Sludge buildup is a major cause of slow responses - Steady State helps avoid that antipattern. 
Steady State encourages better operational discipline by limiting the need for system administrators to log on to the production servers.


#### Remember This
Avoid fiddling.
- Human intervention leads to problems - eliminate frequent need for it 
- System should run for at least a deployment cycle without manual disk cleanups or nightly restarts

Purge data with application logic
- DBAs can create scripts to purge data, but they don’t always know how the application behaves when data is removed 
- Maintaining logical integrity, especially if you use an ORM tool, requires the application to purge its own data

Limit caching
- In-memory caching speeds up applications, until it slows them down
- Limit the amount of memory a cache can consume

Roll the logs
- Don’t keep an unlimited amount of log files 
-- Configure log file rotation based on size
-- If you need to retain them for compliance, do it on a non- production server


## Fail Fast
A slow response is bad, but slow *failure* responses are worse.
Wastes system resources to burn cycles and clock time only to throw away the result because its an error.
Improves overall system stability by avoiding slow responses. 
Combined with timeouts, failing fast can help avert impending cascading failures and  helps maintain capacity when the system is under stress because of partial failures.


*If the system can determine in advance that it will fail at an operation, it’s always better to fail fast.* 
The caller then doesn’t have to tie up any of its capacity waiting and can get on with other work.
How can the system tell whether it will fail? There’s a large class of “resource unavailable” failures. 
E.g., when a load balancer gets a connection request, but none of the servers in its service pool is functioning, it should immediately refuse the connection. 
With some configurations the load balancer will queue the connection request for a while in the hopes that a server will become available in a short period of time. 
Violates the Fail Fast pattern.

An application/service can tell from an incoming request/message roughly what database connections and external integration points are needed. 
The service can quickly check out the connections it will need and verify the state of the circuit breakers around the integration points before it begins processing the request.
If any resources aren't available, the service can fail immediately, rather than getting partway through the work.

A web application can fail fast by performing basic parameter checking in the servlet/controller that receives the request, before talking to the database.
Argument for moving some parameter checking out of domain objects into something like a “Query object.”
 
When failing fast, be sure to report a system failure (resources not available) differently than an application failure (parameter violations or invalid state). 
A generic “error” message could cause an upstream system to trip a circuit breaker when some user entered bad data and hit Reload multiple times.


#### Remember This

Avoid Slow Responses and Fail Fast.
- If system cannot meet its SLA, inform callers quickly
- Dont make your problem their problem 
-- Don’t make them wait for an error message/until they time out 

Reserve resources, verify Integration Points early
- Make sure you’ll be able to complete the transaction before you start 
-- If critical resources aren’t available, e.g. a popped Circuit Breaker on a required callout, then don’t waste work by getting to that point 
-- unlikely to change between the beginning and the middle of the transaction

Use for input validation.
- Do basic user input validation even before you reserve resources
-- Don’t bother checking out a database connection, fetching domain objects, populating them, and calling validate() just to find out that a required parameter wasn’t entered

#### Let It Crash
Sometimes the best thing you can do to create *system-level* stability is to abandon *component-level* stability. 
“Let it crash” philosophy from Erlang.
- every possible error cant be prevented
-- Dimensions proliferate and the state space exponentiates
- no way to test everything/predict all the ways a system can break

*Assume that errors will happen.*
Typically we try to recover from errors, i.e. getting the system back into a known good state using exception handlers to fix the execution stack and try-finally
blocks or block-scoped resources to clean up memory leaks. 

But a program's cleanest state is right after startup. 
“Let it crash”: error recovery is difficult and unreliable - our goal should be to get back to that clean startup as rapidly as possible.

Approach requires certain system properties:
 
#### Limited Granularity
We must be able to contain a crash in a controlled way - a component must crash in isolation. 
The rest of the system must protect itself from a cascading failure.

In Erlang, the natural boundary is the actor.
The runtime system allows an actor to terminate without taking down the entire OS process. 

Other languages have libraries that impose the actor model on a runtime that has no idea what an actor is. 
Following the library’s rules for resource management and state isolation gives you the benefits of “let it crash.” 
Requires thorough code review to make sure everyone adheres to these rules. 

In a microservices architecture, granularity chould be a whole service instance. 
Depends mainly on how quickly it can be replaced with a clean instance.

### Fast Replacement
It must be possible to get into a clean state and resume normal operation as quickly as possible. 
Performance would degrade if too many instances are restarting at the same time. 
Worst case, we could have loss of service because all of our instances are busy restarting.

With *in-process* components like actors, the restart time is measured in microseconds. 
Callers are unlikely to really notice that kind of disruption.
Service instances startup time depends on how much of the “stack” has to be started up. 
E.g. 
- Running Go binaries in a container. Startup of a new container and a process in it is takes milliseconds. Crash the whole container.
- A NodeJS service running on a long-running virtual machine in AWS. Starting the NodeJS process takes milliseconds, but starting a new VM takes minutes. In this case, just crash the NodeJS process.
- An aging JavaEE application with an API pranged into the front end runs on virtual machines in a data center. Startup time is measured in minutes. “Let it crash” is not the right strategy.

#### Supervision
When an actor or a process is intentionally crashed, how does a new one get started?

Actor systems use a hierarchical tree of supervisors to manage restarts:
- The runtime notifies the supervisor when an actor terminates 
- The supervisor can decide to
-- restart the child actor
-- restart all of its children
-- crash itself 

If the supervisor crashes, the runtime terminates all its children and notifies the supervisor’s supervisor. 
Ultimately whole branches of the supervision tree can become restarted with a clean state. 

Note that the supervisor is *not a service consumer*. 
Managing the worker is different than requesting work. 
Systems suffer when they conflate the two.

Supervisors must track how often they restart child processes.
It may be necessary for the supervisor to crash itself if child restarts happen too frequently. 
Indicates that either the state isn’t sufficiently cleaned up or the whole system is in jeopardy and the supervisor is just masking the underlying problem.

With service instances in a PaaS environment, the platform itself decides to launch a replacement. 
In a virtualized environment with autoscaling, the autoscaler decides whether and where to launch a replacement. 
*Not the same as a supervisor because they lack discretion. * - always restarts the crashed instance, even if it is just going to crash again immediately.
No notion of hierarchical supervision.

#### Reintegration
After an actor/instance is restarted, the system must resume calling the newly restored provider. 

- If the instance was called directly, then callers should have circuit breakers to automatically reintegrate the instance
- If the instance is part of a load-balanced pool, then the instance must be able to join the pool to accept work
-- A PaaS will take care of this for containers
- With statically allocated virtual machines in a data center, the instance should be reintegrated when health checks from the load balancer begin to pass

##### Remember This
Crash components to save systems.
-  Counterintuitive to create system-level stability through component-level instability 
- May be the best way to get back to a known good state

Restart fast and reintegrate.
- The key to crashing well is getting back up quickly
-- You risk loss of service when too many components are bouncing
- Once a component is back up, it should be reintegrated automatically

Isolate components to crash independently.
- Use Circuit Breakers to isolate callers from components that crash 
- Use supervisors to determine what the span of restarts should be
- Design your supervision tree so that crashes are isolated and don’t affect unrelated functionality

Don’t crash monoliths
- Large processes with heavy runtimes/long startups  
- Applications that couple many features into a single process

### Handshaking
Underused technique that can be applied to great advantage in application-layer protocols.
*Effective way to stop cracks from jumping layers, as in the case of a cascading failure.*

Information exchanged between devices that regulate their communication 
- Serial protocols (e.g. EIA-232C) rely on the receiver to indicate when it’s ready to receive data 
- Analog modems used a form of handshaking to negotiate a speed and a signal encoding that both devices would agree upon 
- TCP uses a three-phase handshake to establish a socket connection 
-- also allows the receiver to signal the sender to stop sending data until the receiver is ready 

Ubiquitous in low-level communications protocols - rare at the application level.

HTTP-based protocols - XML-RPC or WS-I Basic - have few options available for handshaking.
Provides a response code of “503 Service Unavailable,” which is defined to indicate a temporary condition.

Most clients do not distinguish between different response codes. 
If the code is not a “200 OK,” “403 Authentication Required,” or “302 Found (redirect),” the client probably treats the response as a fatal error. 
Many clients even treat other 200 series codes as errors.

The protocols underlying RPC technology (CORBA, DCOM, Java RMI, etc) are equally bad at signaling their readiness to do business.

*Handshaking lets a server protect itself by throttling its own workload. *
Instead of being victim to whatever demands are made upon it, the server should have a way to reject incoming work.

*HTTP-based servers can aproximate handshaking by partnering the load balancer and the web/application servers.*
Load balancer pings a “health check” page on the web server periodically. 
The web server notifies it that it is busy by returning either an error page (e.g. HTTP response code 503 “Not Available” ) or an HTML page with an error message. 
The load balancer then knows not to send any additional work to that particular web server.

Only helps for web services and breaks down if all the web servers are too busy to serve another page.
When there's several services, each can provide a “health check” query for use by load balancers. 
The load balancer would then check the health of the server before directing a request to that instance. 
This provides good handshaking at a relatively small expense to the service.

*Handshaking is most valuable when unbalanced capacities are leading to slow responses.*
If the server can detect that it will not be able to meet its SLAs, then it should have a way to ask the caller to back off.
If the servers are behind a load balancer, then they have the binary on/off control of stopping responses to the load balancer, which would in turn take the unresponsive server out of the pool.
Pretty crude - best bet is to build handshaking into any custom protocols that you implement.

*Circuit Breaker is a stopgap you can use when calling services that cannot handshake.*
Instead of asking politely whether the server can handle the request, you just make the call and track whether it works.

#### Remember This
Create cooperative demand-control.
- Handshaking between client/server permits demand throttling to serviceable levels 
- Common application-level protocols do not perform handshaking

Consider health checks.
- Use health checks in clustered or load-balanced services as a way for instances to handshake with the load balancer

Build handshaking into your own low-level protocols so that the endpoints can each inform the other when they are not ready to accept work.

#### Test Harnesses
Distributed systems have failure modes that are difficult to trigger in development/QA environments. 
To test various components together, an “integration testing” environment is often used where the system is fully integrated to all the other systems it interacts with.

Issues: 
For greatest assurance, we’d like to test against the dependency versions that are in use when we release our system. 
This would constrain the entire company to testing only one new piece of software at a time. 

The interdependencies of today’s systems create such an interlocking web of systems that an integration testing environment really becomes unitary—one global integration test that duplicates the real production systems of the entire enterprise. 
Such a unitary environment would need change control similar to the actual production environments.
 
We would like to test that our own system handles unexpected behavior from dependencies.
Integration test environments can verify only what the system does when its dependencies are behaving nicely.
Even if a remote system can be provoked to return errors, they're still functioning within specifications. 
E.g. If the spec says, ”The system shall return an error code 14916 unless the request includes the date of the last telephone sanitization,” then the caller can force that error condition to occur.
Every system will however eventually operate outside of spec; vital to test the local system’s behavior when the remote system goes wonky. 
Unless the designers of the remote system built in modes that simulate out-of-spec failures that can occur in production, there will be behaviors that integration testing does not verify.

A test harness is superior to integration testing 
- allows you to test most failure modes 
- preserves/enhance system isolation to
-- avoid the version-locking problem
-- allow testing in many locations instead of the unitary enterprise-wide integration testing environment 

Use test harnesses to emulate the remote system for each integration point. 
Should be as nasty and vicious as real-world systems are - its purpose is to help the system under test become cynical.

Consider building a test harness that substitutes for the remote end of every web services call. 
As the remote call uses the network, the socket connection is susceptible to the following failures:

- It can be refused
- It can sit in a listen queue until the caller times out
- The remote end can 
-- reply with a SYN/ACK and then never send any data
-- send nothing but RESET packets
-- report a full receive window and never drain the data
- The connection can be established, but
-- the remote end never sends a byte of data
--  packets could be lost, causing retransmit delays
-- but the remote end never acknowledges
receiving a packet, causing endless retransmits.
- The service can
-- accept a request, send response headers (supposing HTTP), and never send the response body
-- send one byte of the response every thirty seconds.
-- send a response of HTML instead of the expected XML.
-- send megabytes when kilobytes are expected.
-  can refuse all authentication credentials

These failures fall into distinct problem categories: 
- network transport
- network protocol
- application protocol
 - application logic 

You can find failure modes in every layer of the OSI model. 
Not a good idea to add switches/flags to applications to enable them to simulate all of these failures. 
You dont want a risk of turning on a “simulated failure” in production, but also expensive and odd. 

Integration testing environments are only good for examining failures in the the application layer and doesn't even test all of those.
A test harness' exists specifically as a tool for testing - a real application wouldn’t be written to call low-level network APIs directly, but the test harness can be.
- Therefore, it’s able to send bytes too quickly, or very slowly 
- can set up extremely deep listen queues
- can bind to a socket and then never service a single connection attempt.

The test harness should act like a hacker, trying all kinds of bad behavior to break callers.

Different applications and protocols tend to misbehave in the same way.  
E.g. socket protocols (HTTP, RMI, or RPC)
- refusing connections
- connecting slowly
- accepting requests without reply 

For these, a single test harness can simulate many types of bad network behavior. 

Use different port numbers to indicate different kinds of misbehavior. 
- Port 10200 would accept connections but never reply 
- Port 10201 gets a connection and a reply, but the reply will be copied from /dev/random 
- Port 10202 will open a connection, then drop it immediately, etc 
- etc 

Prevents having to change modes on the test harness and a single test harness can break many applications.
Helps with functional testing in the development environment by letting multiple developers hit the test harness from their workstations (they should also be able to run their own instances of the test harness).

Bear in mind that your test harness might kill apllications. It should log requests, in case your application dies with no indication of the cause.

A test harness that injects faults will unearth many hidden dependencies.
Injecting latency in requests will uncover many more.
Reordering TCP packets will uncover more again. 
The only limit is your imagination.

The test harness can be designed like an application server with pluggable behavior for tests related to the real application.
A single framework for the test harness can be subclassed to implement any application-level protocol, or any perversion of the application-level protocol, necessary. 

Broadly speaking, a test harness leads toward “chaos engineering.

#### Remember This
Emulate out-of-spec failures.
- Real applications lets you test only those errors that the real application can deliberately produce 
- A test harness lets you simulate all sorts of messy, real-world failure modes

Stress the caller.
The test harness can produce slow responses, no responses, or garbage responses. Then you can see how your application reacts.

Leverage shared harnesses for common failures.
- You don’t necessarily need a separate test harness for each integration point
- A “killer” server can listen to several ports, creating different failure modes depending on which port you connect to

Supplement, don’t replace, other testing methods.
- The Test Harness pattern augments other testing methods. 
- Unit tests, acceptance tests, penetration tests, and so on help verify functional behavior 
- A test harness helps verify *nonfunctional* behavior while maintaining isolation from the remote systems

### Decoupling Middleware
The tools that inhabits the space of integrating systems that weren't meant to work together. 
Rebranded as "enterprise application integration" - occupies the interstices between enterprise systems. 
"The connective tissue that bridges gaps between different islands of automation". 

Often described as “plumbing”
Inherently messy - must work with different business processes/technologies, and even different definitions of the same logical concept. 
Service-oriented architectures are currently stealing attention from the less glamorous, but more necessary, job of middleware.

Done well, *middleware simultaneously integrates and decouples systems.* 
**Integrates** by passing data and events back and forth between the systems.
**Decouples** by letting participating systems remove specific knowledge of and calls to the other systems. 

Any kind of synchronous call-and-response or request/reply method forces the calling system to stop what it’s doing and wait. 
Temporal coupling: The calling system and the receiving system must both be active at the same time. 
Remote procedure calls (RPCs), HTTP, XML-RPC, RMI, CORBA, DCOM, and any other analog of local method calls. 

Tightly coupled middleware amplifies shocks to the system. 
*Synchronous calls are particularly vicious amplifiers that facilitate cascading failures.*

Less tightly coupled forms of middleware allow the calling and receiving systems to process messages in different places and at different times.
IBM MQseries and other queue-based or publish/subscribe messaging systems, system-to-system messaging via SMTP or SMS.  
*As the requesting system doesn’t just sit around waiting for a reply, this form of middleware cannot produce a cascading failure*. 
Messaging systems used to be very expensive infrastructure, but now solid open source tools exist.

The main advantage of synchronous (tightly coupled) middleware lies in its simplicity. 
Suppose a customer’s proposed credit card purchase needs to be authorized. 
If implemented using RPC/XML-RPC, the application can clearly decide whether to proceed with the next step of the checkout process or send the user back to the payment methods page. 
By comparison, if the system just sends a message asking for credit card authorization, without waiting for a reply, then it must somehow decide what to do if the authorization request ultimately fails or remains unanswered. 

Designing asynchronous processes is inherently harder.
The process must deal with exception queues, late responses, callbacks (computer-to-computer, human-to-human), and assumptions.
These decisions even involve the business sponsors of the calling system, who will occasionally have to decide what the acceptable level of financial risk is.
You can apply most of the patterns in this chapter without greatly affecting the implementation cost of the system. 

Middleware decisions are hard to undo. The move from synchronous request/reply to asynchronous communication necessitates very different design. 
Switching cost must be considered.

#### Remember This
 Decide at the last responsible moment
-- Other stability patterns can be implemented without large-scale changes to the design or architecture 
- Decoupling middleware is an architecture decision - ripples into every part of the system 
- nearly irreversible decisions that should be made early rather than late

Avoid many failure modes through total decoupling.
- The more you decouple individual servers, layers, and applications, the fewer problems you will observe with Integration Points, Cascading Failures, Slow Responses, and Blocked Threads.
- Decoupled applications are more adaptable, since you can change any of the participants independently of the others

Have a large mental toolbox of architectures
- Not every system needs to look like a three-tier application with relational database 
- Learn many architectural styles, and select the best architecture for the problem at hand

### Shed Load
Services, microservices, websites, and open APIs have no control over their demand. 
In theory, any device on the internet could send a request at any moment.  
No matter how strong your load balancers or how fast you can scale, the world can always make more load than you can handle.

At the network level, TCP deals with floods of connection attempts via a listen queue. 
- Every port has a queue for every incomplete connection  
- The application decides when to accept the connections 
- When the queue is full, new connection attempts are rejected with an ICMP RST (reset) packet

TCP still has issues:
Services often fall over before the connection queue fills up - typically due to contention for a pooled resource. 
Threads start to slow down, waiting for a resource.
Once they have the resource, they run slower because too much RAM and CPU are used by all the extra threads. 
Sometimes this gets exacerbated by other resource pools that are also exhausted.
The net result is lengthening response times until callers start timing out.
To an outside observer, there’s no difference between “extremely slow” and “down.”

Services should model TCP’s approach: 
When load is too high, new requests for work should be refused (Fail Fast)
A service should monitor its own performance relative to its SLA to determine what “load is too high” means.  
When requests take longer than the SLA, it’s time to shed some load. 
Failing that, you may use a semaphore in your application and only allow a certain number of concurrent requests in the system. 
A queue between accepting connections and processing them would have a similar effect at the expense of both complexity and latency.

If using a load balancer, individual instances can use a 503 status code on their health check pages to tell the load balancer to back off for a while (see Handshaking)

Inside the boundaries of a system or enterprise, it’s more efficient to use back pressure to create a balanced throughput of requests across synchronously coupled services. 
Shed load as a secondary measure in these cases.

#### Remember This
You can’t out-scale the world.
- No matter how large your infrastructure or how fast you can scale it, the world has more devices than you can support 
- If your service is exposed to uncontrolled demand, then you need to be able to shed load when the world goes crazy on you

Avoid slow responses using Shed Load.
- Creating slow responses is being a bad citizen
- Keep your response times under control rather than getting so slow that callers time out

Use load balancers as shock absorbers.
- Individual instances can report HTTP 503 to get some breathing room
- Load balancers are good at recycling connections very quickly

### Create Back Pressure
Every performance problem starts with a queue backing up somewhere
- socket listen queue 
- OS run queue
- the databases I/O queue
- ....

An unbounded queue can consume all available memory. 
As the queue grows, the time it takes for a piece of work to get all the way through it grows too (Little’s law). 
As a queue’s length goes toward infinity, so does response time. 
*Unbounded queues should be avoided*.

For a bounded queue, we must decide what to do when it’s full and a producer tries to add an element. 
- Pretend to accept the new item but actually drop it
- Actually accept the new item and drop something else from the queue
- Refuse the item
- Block the producer until there is room in the queue

In some use cases, dropping the item may be the best option. 
For data whose value decreases rapidly with age, dropping the oldest item in the queue can be the best option.

Blocking the producer is a kind of flow control. 
It allows the queue to apply “back pressure” upstream. 
Presumably that back pressure propagates all the way to the ultimate client, who will be throttled down in speed until the queue releases.

TCP has extra fields in each packet to create back pressure.
Once the window is full, senders are not allowed to send anything until released. 
Back pressure from the TCP window can cause the sender to fill up its transmit buffers, causing subsequent calls to tell the socket to block.

Back pressure can lead to blocked threads. 
*It’s important to distinguish back pressure due to a temporary condition from back pressure because a consumer is just broken.*
The Back Pressure pattern works best with asynchronous calls and programming. 
An Rx frameworks can help here, as can actors or channels.

*Back pressure only helps manage load when the pool of consumers is finite.*
That’s because the “upstream” is so diverse that there’s no systemic effect on all of them:

Suppose your system provides an API for user-created “tags” at a specific location, used by native and web apps.
Internally, there’s a certain rate at which you can create and index new tags.
That’s going to be limited by your storage and indexing technology. 
When the rate of “create tag” calls exceeds the storage engine’s limit, the calls get slower and slower. 
Without back pressure, this would lead to a progressive slowdown until the API seems to be offline.
By ususing a blocking queue for “create tag” calls  we can create back pressure.

Assume each API server is allowed 100 simultaneous calls to the storage engine. 
When the 101st call arrives at the API server, the calling thread blocks until there is an open slot in the queue (=back pressure) 
The API server cannot make calls any faster than it is allowed.
A flat limit of 100 calls per server is very crude - as a result one API server may have blocked threads while another has free slots available.

It would be better letting the API servers make as many calls as they want and put the blocking on the receiver’s end.
The off-the-shelf storage engine would then be wrapped with a service to receive calls, measure response times, and adjust its internal queue size to maximize throughput and protect the engine.

At some point, though, the API server still has a thread waiting on a call. 
Blocked threads can quickly result in downtime.
At the edge of your system boundary, blocked threads will frustrate a user or provoke a retry loop. 
As such, *back pressure works best within a system boundary.* 
*At the edges, you also need load shedding and asynchronous calls.*

In our example, *the API server should accept calls on one thread pool and issue the outbound call to storage on another set of threads.* 
When the outbound call blocks, the request-handling thread can then time out, unblock, and respond with HTTP 503.
Alternatively, it could drop a “create tag” command in a queue for later indexing, where HTTP 202 would be more appropriate.

A consumer inside your system boundary will experience back pressure as a performance problem or as timeouts. 
Indicates a real performance problem—the consumers collectively generated more load than the provider can handle! 
The provider is not necessarily to blame, though. 
It might have enough capacity for “normal” traffic, but one consumer went nuts and started eating Cincinnati. 
It could be due to an attack of self- denial or just organic changes in traffic patterns.

When Back Pressure kicks in, monitoring needs to know about it to be able to tell whether it’s a random fluctuation or a trend.

#### Remember This
Back Pressure creates safety by slowing down consumers.
- Consumers will experience slowdowns
- The only alternative is to let them crash the provider

Apply Back Pressure within a system boundary
- Across boundaries, look at load shedding instead. 
-- Especially true when the Internet at large is your user base
- Queues must be finite for response times to be finite

You only have a few options when a queue is full - all unpleasant: 
- drop data
- refuse work
- block.
-- Consumers must be careful not to block forever.

### Governor
Return to reddit outage:  
Reddit’s configuration management system restarted a part of its infrastructure management that scales server instances up and down. 
This was in the middle of a ZooKeeper migration, so the autoscaler read a partial configuration and decided to shut down nearly every machine instance in Reddit.

The flip side of that coin is a job scheduler that spins up too many compute instances in order to process a queue before a deadline.
The work still can’t get done fast enough and the cloud provider’s invoice that month is huge.
Automation has no judgment - when it goes wrong, it tends to go wrong really quickly. 
By the time a human perceives the problem, it’s a question of recovery rather than intervention. 
How can we allow human intervention without putting a human in the loop for everything? 

*Use automation for things humans are bad at: repetitive tasks and fast response.* 
*Use humans for what automation is bad at: perceiving the whole situation at a higher level.*


A governor limits the speed of a steam engine. Even if the source of power could drive it faster, the governor prevents it from running at unsafe RPMs.

We can create governors to slow the rate of actions. Reddit did this with its autoscaler by adding logic that says it can only shut down a certain percentage of instances at a time.
A governor is
- stateful and time-aware
- knows what actions have been taken over a period of time
- should be asymmetric 
-- Most actions have a “safe” direction and an “unsafe” one.
-- Shutting down instances is unsafe.
-- Deleting data is unsafe.
-- Blocking client IP addresses is unsafe.

Often tension between definitions of “safe.” 
- Shutting down instances is unsafe for availability
- spinning up instances is unsafe for cost. 
These forces don’t cancel each other out - define a U-shaped curve where going too far in either direction is bad. 

That means actions may also be safe within a defined range but unsafe outside the range.
Your AWS budget may allow for a thousand EC2 instances, but if the autoscaler starts heading toward two thousand, then it needs to slow down. 
You can think about this U-shaped curve as defining the response curve for the governor.
Inside the safe zone, the actions are fast. Outside the range, the governor applies increasing resistance.

The whole point of a governor is to slow things down enough for humans to get involved. 
That means connecting to monitoring both to alert humans that there’s a situation and to give them enough visibility to understand what’s happening.

#### Remember This

Slow things down to allow intervention.
- Often things go wrong due to automation tools pushing the throttle to its limit 
- Humans are better at situational thinking, so we need to create opportunities for us to intervene

Apply resistance in the unsafe direction.
- Some actions are inherently unsafe.
-- Shutting down, deleting, blocking things are all likely to interrupt service
- Automation makes them go fast, so you should apply a Governor to provide humans with time to intervene
 
Consider a response curve.
- Actions may be safe within a defined range
- Outside safe they should encounter increasing “resistance” by slowing down the rate by which they can occur 