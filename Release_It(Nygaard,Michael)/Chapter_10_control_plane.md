# Chapter 10 - Control plane 

All the software and services that run in the background to make production load successful.
- If production user data passes through it, it’s *production software*. 
- If its main job is to manage other software, it’s the *control plane*.

## How Much Is Right for You?
Every part is optional.\
You can do without every piece of it, if you’re willing to make some trade-offs.

E.g.  logging and monitoring is useful for 
- postmortem analysis
- incident recovery
- defect discovery

Without it, these will take longer or not be done.\
You wont be able to discover if your software is down.

The more sophisticated your control plane is, the more it costs to implement/operate - *Every piece represents ongoing operational cost*.

Fixed cost of dedicated people versus the variable cost of speeding up deployments, incident recovery, provisioning services, etc. 
-  If you’re small and the rate of change is low, you may find it’s not worth it 
 - If you can amortize the cost of a platform team across hundreds of services deployed hundreds of times per year, then it makes great sense

Cost equation isn’t static, new open-source operations tools are constantly released.

## Mechanical Advantage

*The multiplier on human effort that simple machines provide.*

High leverage allows a person to make large changes with less effort.\
Makes it easy to release new software to thousands of machines. But also amplifies errors (e.g. *Force multiplier* antipattern)

AWS outage example:\
S3 team member executed a command from an established playbook intended to remove a small number of servers for one of the S3 subsystems for billing.\
The command was entered incorrectly and a larger set of servers was removed than intended.

## System failure - not human error 

Not a human error - The administrative tools and playbooks allowed it to happen and amplified a minor error into enormous consequences; It's a system failure.\
“System” meaning the whole system — S3 plus the control plane software and human processes to manage it all.

Why did the tried and tested playbook fail? Did something change?
- Who executed it? Was there a “second set of eyes”?
- Had it been changed? Sometimes error-checking steps get relaxed over time
- What feedback did the underlying system provide? Feedback may have helped avert previous problems

We normally have postmortem reviews of incidents with bad outcomes.\
We look for causes and label any anomalies as either a root cause or a contributing factor. 

Often the same anomalies are present during “ordinary” operations, but only pay them proper attention with the benefit of hindsight after an outage.\
We also have many opportunities to learn from successful operations.\
Anomalies are present all the time, but most of the time they don’t cause outages. 

*You should do postmortems for successful changes.*
- What variations or anomalies happened? 
- Were there any “near misses”? 
-- Did someone type an incorrect command but catch it before executing? 
--- How did they they catch it?
--- What safety net could have helped them catch it or stop it from doing harm?

## Automation Goes Really Fast
The AWS post-mortem also discussed the tool used.\
Removal of capacity is a key operational practice, but the tool used allowed too much to be removed too quickly.\
The tool was changed to remove capacity more slowly and safeguards were added to prevent capacity from being removed when it will take any subsystem below its minimum required capacity level

Similiar to reddit autoscaling issue where the partially migrated zookeeper db claimed that reddit didnt need most of the servers it had running.

A common thread in these outages is that the automation is not being used simply to enact the will of a human administrator.\
The automated control plane senses the current state of the system, compares it to the desired state, and effects changes to bring the current state into the desired state.

It’s normal to shut down a few machines in a horizontally scaled solution and has little effect on capacity.\
Exact threshold depends on how much spare capacity you have for handling bursts.\
When the control plane is about to shut down more than 50 percent of total server capacity, it would be a good idea to prompt for human confirmation.\
Automation has no judgment. When it goes wrong, it tends to do so really, really quickly.  By the time a human perceives the problem, it’s a question of recovery rather than intervention. 

How can we allow human intervention without constantly needing human input?\
Use automation for the things humans are bad at: repetitive tasks and fast response.\
Use humans for the things automation is bad at: perceiving the whole situation at a higher level.


### Platform and Ecosystem

 ...

### Development Is Production

The tools, services, and environments that developers need to do their jobs should be treated with production-level SLAs.\
The development platform is the production environment for the job of creating software.

### System-Wide Transparency
How to assemble a picture of system-wide health from the individual instances’ information?\
When trying to understand the system as a whole, you should ask
- Are users receiving a good experience?
- Is the system creating the economic value we want?

“Is everything running?” is not relevant. We should expect a few instances to be down at all times and the system should be able to run fine despite this.\
Partially broken should be the normal state. 


### Real-User Monitoring
The best way to tell if users are receiving a good experience is to measure it directly - *real-user monitoring* (RUM).

Mobile and web app instrumentation can report their timing and failures up to a central service.\
This may call for a lot of infrastructure - consider a service like New Relic or Datadog.\
If you are at a scale where it makes sense to run it yourself, on-premise software such as AppDynamics or CA’s APM might make sense.\
Some of these allow you to watch network traffic at the edge of your system, recording HTTP sessions for analysis or playback.

These services have several advantages over the “DIY” approach. 
- rapid startup
-- No need to build infrastructure or configure monitoring software 
-- It is possible to get going with data collection in under an hour 
- Offer agents and connectors for a wide array of technology
--  Makes it much easier to integrate all your monitoring into one place
- more polished dashboards/visualizations than open-source alternatives

Downsides 
- Subscription fees
--  Usually increase with system scale
-- may come a time when the fees become unpalatable, but the switching cost of moving to your own infrastructure is equally unpalatable. 
- some companies dont want monitoring data traveling over the public internet.

 On-premise solutions offer easy integration and polished visualization, but lack the advantage of rapid startup and have scaling fees.

There's excellent open-source tools, but integrating the tools to your system and with eachother can be a challenge.\
The dashboards and visualization are also less polished and less user-friendly.\
While there's no monthly fees, open source still has costs in the form of labor and infrastructure.

“application performance management,” is one of the last areas of operations software that hasn’t been replaced by open-source.\
It’s not that important to choose the ideal solution - focus on adopting your chosen solution thoroughly.\
Don’t leave any “dead zones” in your system.

Real-user monitoring is most useful to understand in terms of the current state and recent history.\
Dashboards and graphs are the most common ways to visualize this.


## Economic Value
Most of the software we write for companies exists to create economic value. Ttransparency is where we can most directly perceive the linkage between our systems and our financial success. 

Bad user experience can harm the value of the system.\
It can also be harmed if the system cost is too high.\
(“top line” and “bottom line” effects.)

Transparency should show how the recent past, current state, and future state connect to revenue and costs.

The top line is income/revenue. Our system should be able to tell us if we’re making as much as we “should be” right now.\
I.e. are there performance bottlenecks that prevent us from signing up more new users?\
Is some crucial service returning errors that turn people off before they register? 

The specific needs vary according to your domain - you should watch:
- The steps of business processes 
-- Any steps with a rapid drop-off? 
-- Is some service in a revenue-generating process throwing exceptions in logs? If so, it’s probably reducing your top line
- Queue depths 
-- first indicator of performance degradation
-- A non-zero queue depth always means work takes longer to get through the process
-- For many business transactions, that queuing time directly hits your revenue

The bottom line is net profit/loss - top line minus costs. 
Cost comes from
- infrastructure, especially with autoscaled, elastic, pay-as- you-go services
-- Many horror stories about unchecked autoscaling costing them thousands of dollars due to unchecked demand
-- Or runaway automation spinning up too many resources
- operations
-- The harder your software is to operate, the more time it takes from people 
-- Any time spent responding to incidents is unplanned work that could have gone to raising the top line
- platforms/runtimes
-- Some languages are fast to code in but require more instances to handle a particular workload 
-- Bottom line can be improved by moving crucial services to technology with a smaller footprint or faster processing
-- make sure it’s a service that actually makes a difference 

Previous section adressed current state and recent past.\
Transparency tools should also help us consider the near future 
- Are there opportunities to increase the top line by improving performance or reducing queues?
- Are we going to hit a bottleneck that will prevent us from increasing the top line?
- Are there opportunities to increase the bottom line by optimizing services? Can we see places that are overscaled?
- Can slow-performing or large-footprint instances be replaced with more efficient ones?

When you relate monitoring, log collection, alerting, and dashboarding to economic value more than technical availability  you’ll find it easy to make decisions about what to monitor, how much data to collect, and how to represent it.


### The Risk of Fragmentation
Concerns
- business 
- technical 
-- Development 
-- Operations
 
Typically look at different measurements collected by different means.

Hard to plan when
- marketing uses tracking bugs on web pages
- sales uses conversions reported in a BI tool
- operations analyzes log files in Splunk
- development uses blind hope and intuition 
 
Could this crew ever agree on how the system is doing? 
 
You should integrate the information so all parties can see the same data through similar interfaces.

Different constituencies require different perspectives.\
These perspectives won’t all be served by the same views into the systems, but they should be served by the same information system overall.\
Just as the question, “How’s the weather?” means very different things to a gardener, a pilot, and a meteorologist, the question, “How’s it going?” means something decidedly distinct when coming from the CEO or the system administrator. 

Likewise, a bunch of CPU utilization graphs won’t mean a lot to the marketing team.\
Each “special interest group” in your company may have its own favorite dashboard, but everyone should be able to see how releases affect user engagement or conversion rate affects latency.

#### Logs and Stats

Log and metrics collectors gather and make sense of all data system-scale.
Collectors can either work in push or pull mode.
- Push 
-- the instance is pushing logs over the network, typically with the syslog protocol
-- helpful with containers, since they don’t have any long-lived identity and often have no local storage
- pull
-- the collector runs on a central machine and reaches out to all known hosts to remote-copy the logs 
-- services just write their logs to local files

After you've indexed the logs, you can search them for patterns, make trendline graphs, and raise alerts when bad things happen.\
Splunk dominates the log indexing space today, but ELK stack is also popular.


The story for metrics is much the same, except that the information isn’t always available in files.\
Some information can only be retrieved by running a program on the target machine to sample, say, network interface utilization and error rates.\
That’s why metrics collectors often come with additional tools to take measurements on the instances.

Metrics can be aggregated over time.\
Metrics databases typically keep fine-grained measurements for very recent samples, but then they aggregate them to larger and larger spans as the samples get older.\
For example, the error rate on a NIC may be available second by second for today, in one-minute granularity for the past seven days, and only as hourly aggregates before that. 
- saves on disk space
- makes queries across very large time spans possible

#### What to Expose

Ideally you would like to predict which metrics best reveals any problems and just watch those  - hard to guess and key metrics change over time as your system changes. 

You could spend infinite amounts of time exposing metrics for everything.\
A few heuristics to help decide which variables/metrics to expose. Some of these might need custom code to collect the data 
- Traffic indicators
-- Page requests
-- page requests total 
-- transaction counts
-- concurrent sessions
- For each type of business transaction
-- Number processed 
-- number aborted
-- dollar value 
-- transaction aging
-- conversion rate
-- completion rate
- Users
-- Demographics or classification
-- technographics
-- percentage of users who are registered
-- number of users
-- usage patterns
-- errors encountered,
-- successful logins
-- unsuccessful logins
- Resource pool health
-- Enabled state
-- total resources (as applied to connection pools, worker thread pools, and any other resource pools)
-- resources checked out
--high-water mark
-- # of resources created
-- # of resources destroyed
-- # of times checked out
-- # of threads blocked waiting for a resource
-- # of times a thread has blocked waiting
- Database connection health
-- # of SQLExceptions thrown
-- # of queries
-- average response time to queries
- Data consumption
- no. of entities or rows present
- footprint in memory and on disk
- Integration point health
-- State of circuit breaker
-- # of timeouts
-- # of requests
-- average response time
-- # of good responses
-- # of network errors
-- # of protocol errors
-- # of application errors 
-- actual IP address of the remote endpoint
-- current number of concurrent requests
-- concurrent request high-water mark
- Cache health
-- Items in cache
-- memory used by cache
-- cache hit rate
-- items flushed by garbage collector
-- configured upper limit 
-- time spent creating items

All of the counters should have a time component, e.g. “in the last n minutes” or “since the last reset.”

Each metric has some range of normal and acceptable values.\
E.g. be a tolerance around a target value or a threshold that should not be crossed.
“nominal” as long as it’s within that acceptable range. 
A second range will often be used to indicate a “caution” signal, warning that the parameter is approaching a threshold.
*For continuous metrics, a handy rule-of-thumb definition for nominal would be “the mean value for this time period plus or minus two standard deviations.”*
 
Most metrics have a traffic-driven component, so the time period that shows the most stable correlation will be the “hour of the week”—that is, 2 p.m. on Tuesday.\
The day of the month means little. 

In some industries— e.g. travel, floral, and sports—the most relevant measurement is counting backward from a holiday or event.\
For a retailer, the “day of week” pattern will be overlaid on a strong “week of year” cycle.\
No one right answer for all organizations.


## Configuration Services
Configuration services (ZooKeeper, etcd) are distributed databases that applications can use to coordinate their configuration.
Configuration in this sense is more than just the static parameters that an instance would keep in .properties files. 
It includes simple settings, e.g. hostnames, resource pool sizes, and timeouts. 
But also the arrangement of instances among themselves.

Can be used for orchestration, leader election (in the case of a cluster with a master node), or quorum-based consensus.
Scalable but not elastic - you can add and remove nodes, but response time will degrade as the nodes rebalance their data. 
Often requires an admin action to get the cluster to accept a new member or to indicate that an old member is gone for good.

CAP theorem applies to configuration services:
Any service may experience network outages.
- clients may be unable to reach the configuration service 
- nodes of the configuration service will be unable to reach each other (clients can still reach the nodes)

It must be safe for the clients to run with slightly outdated configurations. 
Alternatively, you're forced to shut down applications when the configuration service is partitioned.

Information can also flow from the clients to the configuration services, like version numbers (or commit SHAs) and node identifiers.\
Allows you to write a program or script to reconcile the actual state of the system with the expected state after a deployment.\
Be careful with this - configuration services can sustain high *read* volume but have to go through some consensus mechanism for every write.\
OK for relatively slowly changing configuration data, but there's a limit.

A few pointers about configuration services:
- Make sure instances can start without the configuration service
- Make sure your instances don’t stop working when configuration is unreachable
- Make sure that a partitioned configuration node doesn’t have the ability to shut down the world
- Replicate across geographic regions

## Provisioning and Deployment Services

A host of deployment tools represent “push” and “pull” methods.
- push
--  uses SSH or another agent so a central server can reach out to run scripts on the target machines. 
-- The machines may not know their own roles. The server assigns them.
- pull
-- rely more on the machines to know their own roles. 
-- Software on the machine reaches out to a configuration service to grab the latest bits for its role.
-- work especially well with elastic scaling. 
--- Elastically scaled virtual machines or containers have ephemeral identities
--- no point in having a push-based tool maintain a mapping from machine identity to role

With long-lived virtual machines or physical hosts, push-based tools can be simpler to set up and administer since they use commodity software like SSH rather than agents that require their own configuration and authentication techniques.

The deployment tool by itself should be augmented with a package repository.\
Production builds need to be run on a clean build server using libraries with known provenance.\
The build pipeline should tag the build as it passes various stages, especially verification (e.g. unit or integration tests).\
Repeatable builds are important so code that works on your machine also works in production.

Canary deployments are an important job of the build tooling.\
The “canary” is a small set of instances that get the new build first.\
For a period of time, the instances running the new build coexist with instances running the old build.\
If the canary instances behave oddly, or their metrics go south, then the build can't be rolled out. 

The purpose of the canary deployment is to reject a bad build before it reaches the users.\
The deployment tool needs to interact with another service to decide on placement.\
That placement service will determine how many instances of a service to run.\
It should be network-aware so it can place instances across network regions for availability.\
Typically, it’ll also drive the interconnect layer to set up IP addresses, VLANs, load balancers, and firewall rules.\
Even though a dedicated team will sustain and operate the platform, your software needs to include a description of its needs and wants for the platform to provide (e.g. a JSON/YAML file in the build artifacts.)

### Command and Control
Live control is only necessary if it takes your instances a long time to be ready to run.\
With instances that are fast to run, it would be simpler to just kill the instance and let the scheduler start a new one.\
E.g. instances running in containers that get their configuration from a configuration service.

Not every service is made of instances that start up so quickly.\
The JVM needs a “warm-up” period before the JIT really kicks in and makes it fast.\
Many services need to hold a lot of data in cache before they perform well enough which adds to the startup time.\
If the underlying infrastructure uses virtual machines instead of containers, then it can take several minutes to restart.

### Controls to Offer
In the cases above, you must consider ways to send control signals to running instances. 

Controls to plan for:
- Reset circuit breakers
- Adjust connection pool sizes and timeouts
- Disable specific outbound integrations
- Reload configuration
- Start/stop accepting load
- Feature toggles

Not every service will need all of these controls, but they can serve as a starting point.

Many services expose controls to update the database schema, or to delete all data and reseed it.\
Helpful in test environments but extremely hazardous in production. 

Another common dangerous control is the “flush cache” button.\
*An instance that flushes a cache will have really bad performance for the next several minutes.*\
May generate a dogpile on the underlying service or database.\
Certain services can’t respond until their working set is loaded into memory.

### Sending Commands
Once you’ve decided which controls to expose, you must decide how to convey the operator’s intention out to the instances themselves. 
- Offer an admin API over HTTP 
-- Simple
-- Instances of a service would listen on a specific port for these requests
-- needs to be a different port than ordinary traffic to make it unavailable for the general public!
-- Leaves the door open for higher levels of automation in the future.

If API is described in Open API format, then Swagger UI provides a UI for the API. Else cURL is fine. 

At larger scales, simple scripts to call the admin API may no longer suffice.\
It takes time to make the API call to each instance.\
Suppose each API call takes just a quarter-second to complete.\
Looping over 500 instances will then take two minutes, assuming that all the instances are up and responding properly.
More likely, whatever script loops over those API calls will stall out partway through because some instance doesn’t respond.

Better to to build a “command queue.”\
A shared message queue/pub/sub bus that all the instances listen to.\
The admin tool sends out a command that the instances then perform.\
Be careful -  makes it very easy to create a dogpile.\
Often a good idea to have each instance add a random bit of delay to spread them out a bit.\
It can also help to identify “waves” or “gangs” of instances.\
A a command may target “wave 1,” followed by “wave 2” and “wave 3” a few minutes later.

### Scriptable Interfaces

The best interface for long-term operation is the command line - GUIs are bad. 
Lets operators easily build a scaffolding of scripts, logging, and automated actions to keep your software happy.

### Remember This
It’s easy to get excited about control plane software - but keep the operating costs in mind.\
Choose the options that are appropriate for your team size and the scale of your workload.\
Anything you build must either be maintained or torn down.

Start with visibility. 
- Use logging, tracing, and metrics to create transparency
- Collect and index logs to look for general patterns
-- Also gets logs off of the machines for postmortem analysis when a machine or instance fails
- Use configuration, provisioning, and deployment services to gain leverage over larger or more dynamic systems
-- The more you move toward ephemeral machines, the more you need these

This pipeline to production is not just a set of development tools.\
It is the production environment that developers use to produce value.\ 
Treat it with the same care as you would any other production environment.\

Once the system is (somewhat) stabilized and problems are visible, build control mechanisms.\
Should give you more precise control than just reconfiguring and restarting instances.\
A large system deployed to long-lived machines benefits more from control mechanisms than a highly dynamic environment will.

#### The Platform Players
The solutions discussed have been “some assembly required.”\
I.e. you can adopt them incrementally and defer commitment.\
Optionality comes at a cost because you’ll need to devote time and resources to plumbing together different parts. 
E.g. 
- getting authentication and role-based authorization systems working together
- integrating the components’ monitoring to provide a unified view

At the other end of the integration spectrum, we have the platform players\
Kubernetes, Apache’s Mesos, CloudFoundry, Docker’s “Swarm Mode.”
- abstracts the underlying infrastructure and presents a friendlier programming model
- manages resources and schedules tasks, just across multiple computers
- offers assurance that its parts will all work together coherently

A distinguishing feature of the platforms versus the cloud providers is about location. 
With the platforms, the software is available to be installed at any location: 
- on your premises 
- in a hosting facility
- on top of a public cloud

Relatively easy for one team in a large organization to deploy its own monitoring framework.\
Not the case with the platforms. They require care and feeding in their own right.\
It is more likely that a big group within an organization will move to one of the prefab platforms.\
Individual teams likely don’t have the capacity or authority to build their own platforms. (It wouldn’t be cost-efficient anyway, because you need
to amortize the support cost across a larger number of teams to justify it.)

When these platforms work well, it can be an amazingly smooth experience to deploy services.\
A single command can bundle up a JAR file with its runtime, build a VM/container image, run it, and set up DNS for you.

If you are adopting one of these platforms, you should really embrace it.\
Don’t try to wrap the API or provide your own set of scripts.\
You’re investing a lot in the platform, so get the most you can out of it!


#### The Shopping List
Things you might need - apply a cost/benefit trade-off analysis.
- Log collection and search
- Metrics collection and visualization
- Deployment
- Configuration service
- Instance placement
- Instance and system visualization
- Scheduling
- IP, overlay network, firewall, and route management
- Autoscaler
- Alerting and notification