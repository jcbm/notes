# Chapter 3 - Stabilize your system 

Enterprise software must be *cynical*
- expects bad things to happen and prepared for it
- doesn’t trust itself;  puts up internal barriers to protect itself from failures
- refuses to get too intimate with other systems to avoid getting hurt

Bad things that happen in the real world do not happen in development.

In development, all the tests are contrived by people who know what answer they expect to get.

The real world is unpredictable.


Poor stability carries significant real costs, particularly lost revenue due to downtime.

Loss of customers and reputation.

*Good stability is not costly.*

A highly stable design usually costs the same to implement as the unstable one.


When building the architecture, design, and even low-level implementation of a system, many decision points have high leverage over the system’s ultimate stability. 

Confronted with these leverage points, two paths might both satisfy the functional requirements (aiming for QA). One will have regular downtime, while the other will be stable. 

## Defining stability 
**Transaction** -  abstract unit of work processed by the system (not to be confused with db transaction).

**System** - the complete, interdependent set of hardware, applications, and services required to process transactions for users (from a single application to a multitier network of applications and servers).

**Impulse** - rapid shock to the system (i.e. black friday).

**Stress** - force applied to system over extended period of time (e.g. slow responses from credit card processor). 

**Strain** - odd/negative effects from stress.
- higher RAM usage on web servers, 
- excess I/O rates on the database server 

**Longevity** - System keeps processing transactions for a "long time" - "long" depends on time between code deployments 


Common type of transaction is “customer places order.” 

Spans several pages, often including external integrations (e.g. credit card verification).


Transactions are the reason that the system exists.
A single system can process just one type of transaction, making it a dedicated system.

A mixed workload is a combination of different transaction types processed by a system.

 
**Stability** - A robust system keeps processing transactions, even when transient impulses, persistent stresses, or component failures disrupt normal processing. 

Not just that individual servers or applications stay up and running but rather that the user can still get work done.


## Extending Your Life Span
*Memory leaks* and *data growth* are the main dangers to longevity.

Rarely caught during testing. 


Murphy's law: Whatever you dont test against will happen 

To catch longevity bugs an application must 
- Run long enough in the development environment 
- Be under continuous load 

Longevity bugs are not caught by load testing 
- Expensive, paid by hour 
- Runs for a specified amount of time

Finding longevity bugs
- Run your own longevity tests 
-- Dedicate a developer machine
-- Expose it to a continues moderate load 
-- Simulate inactive period during night to catch connection pool/firewall timeouts

If the economics don’t justify setting up a complete environment, at least try to test important parts while stubbing out the rest.

## Failure modes 
Sudden impulses and excessive strain can trigger catastrophic failure.
Sometimes, some parts of a system will fail before the rest does ("cracks in the system").

**Failure mode**: The initial trigger, how the failure spreads and the resulting damage

Any system has a number of failure modes. You must contain failures by creating safe failure modes that contain the damage and protect the rest of the system ("crackstoppers"). 

Particularly, you must decide what features of the system are indispensable and build in failure modes that protect these features from cracks.

*Failure to limit failure modes will result in unpredictable and dangerous ones to emerge.*
 
### Stopping Crack Propagation

#### Airplane Core Facilities example:
The crack started at the improper handling of the SQLException, but it could have been stopped at many other points: 

The pool was configured to block requesting threads when no resources were available and eventually tied up all request-handling threads.

Could  have been configured to
-  create more connections if it was exhausted.
-  block callers for a limited time only

A problem with one call in CF caused the calling applications on other hosts to fail. CF services built on EJBs that uses RMI which never times out by default.
- Client could have set a timeout on the RMI sockets
- Client could be implemented such that the blocked threads could be jettisoned, instead of having the request-handling thread make the external integration call 

CF severs could have been partitioned into more than one service group, as it would limit a problem in one service group to only affect a subset of CF users 


CF could’ve been built using *request/reply message queues*. Then the caller would know that a reply might never arrive and have to deal with that case as part of handling the protocol itself. 


The callers could have been searching for flights by looking for entries in a tuple space that matched the search criteria. 
CF would have to have kept the tuple space populated with flight records. The more tightly coupled the architecture, the greater the chance this coding error can propagate. 

Conversely, the less-coupled architectures act as shock absorbers, diminishing the effects of this error instead of amplifying them.

## Chain of Failure
Every system outage is caused by a chain of events; a small issue leads to another, which leads to another. 

A failure in one point or layer increases the probability of other failures. 

If the database gets slow, then the application servers are more likely to run out of memory.

 Because the layers are coupled, the events are not independent.

Fault --> Error --> Failure


**Fault** - A condition that creates an incorrect internal state in your software.

May be due to 
-- a latent bug that gets triggered 
-- an unchecked condition at a boundary or external interface.


**Error** - Visibly incorrect behavior. 


**Failure** - An unresponsive system. 


Triggering a fault opens the crack. 

Faults become errors, and errors provoke failures. That’s how the cracks propagate.


At each step in the chain of failure, the crack from a fault may accelerate, slow, or stop. 
A system with many degrees of coupling offers more pathways for cracks to propagate along, more opportunities for errors.

Tight coupling accelerates cracks.
Tight coupling of EJB calls allowed a resource exhaustion problem in CF to create larger problems in its callers. Coupling the request-handling threads to the external integration calls in those systems caused a remote problem to turn into downtime.


To prepare for every possible failure look at 
- every external call
- every I/O
- every use of resources, and every expected outcome 


Consider what are all the ways this can go wrong? Think about the different types of impulse and stress

What if 
- it can’t make the initial connection?
- it takes ten minutes to make the connection?
- it succesfully connects and then gets disconnected?
- it can make the connection but doesn’t get a response from the other end?
- it takes two minutes to respond to my query?
- 10,000 requests come in at the same time?
- the disk is full when the application tries to log the error message about the SQLException that happened because the network was bogged down with a worm?
- etc 

Too impractical for non-critical systems.  

### Philosophies to fault handling
Faults cannot be completely prevented and will happen. 
Faults must not be allowed to turn into errors keep faults from becoming errors. 
You have to decide for your system whether it’s better to risk failure or errors— even while you try to prevent failures and errors.


**Fault-tolerant**
- Catch exceptions
- Check error codes
- Generally keep faults from turning into errors

**Crash-and-recover**
Fault tolerance is futile.

No matter what faults you try to catch and recover from, something unexpected will 
occur.

Let the system crash so it can restart from a known good state 