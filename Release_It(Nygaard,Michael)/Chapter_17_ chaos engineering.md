
# Chapter 17 Chaos engineering

## Breaking Things to Make Them Better
“the discipline of experimenting on a distributed system in order to build confidence in the system’s capability to withstand turbulent conditions in production.”

Deals with distributed systems, frequently large-scale.

Staging or QA environments doesnt tell much about the behavior of large-scale production systems.

Scale matters:

- Different ratios of instances can cause qualitatively different behavior in production 
- Congested networks behave in a qualitatively different way than uncongested ones
- Systems that work fine in a low-latency, low-loss network may break badly in a congested network 
 
Not practically or finansially feasible to replicate production in QA.
 
Many problems only reveal themselves in the whole system (e.g. excessive retries leading to timeouts, cascading failures, dogpiles, slow responses, and single points of failure).
 
Can’t be simulated in a nonproduction environment because of the scale problem. 
 
We can’t gain confidence by testing components in isolation.

Safety is not a composable property; Two services may each be safe on their own, but the composition of them isn’t. 
 
Chaos engineering emphasizes the whole-system perspective.

Deals with emergent properties that can’t be observed in the individual components.

## Antecedents of Chaos Engineering

Resilience engineering offers a rich area to explore for new directions in chaos.

In the absence of other forces, we will optimize a system for maximum gain. 

We’ll push throughput up to the limit of what the machines and network can bear. 

The system will be maximally utilized and maximally profitable until when a disruption occurs.

Highly efficient systems handle disruption badly. 

They tend to break all at once.

We need to optimize our systems for availability and tolerance to disruption in a hostile, turbulent world rather than aiming for throughput in an idealized environment.

Another thread that led to chaos engineering has to do with the challenge of measuring events that don’t happen. 
“You don’t know how much you depend on your IT staff until they go on vacation.”

“Volkswagen microbus” paradox: 
You learn how to fix the things that often break. 

You don’t learn how to fix the things that rarely break. 

Thus when they do break, the situation is likely to be more dire. 

*We want a continuous low level of breakage to make sure our system can handle the big things.*

Expect that disorder will occur, but make sure there’s enough of it during normal operation that our systems aren’t flummoxed when it does occur. 
Use chaos engineering the way a weightlifter uses iron: to create tolerable levels of stress and breakage to increase the strength of the system over time.

## The Simian Army

Netflix’s “Chaos Monkey.”: 
- Occasionally picks an autoscaling cluster
- kills one of its instances 
- The cluster should recover automatically
-- If it doesn’t, then there’s a problem and the team that owns the service has to fix it


how can  you make sure that every deployment of every cluster stays robust when hidden coupling is so easy to introduce?
The company’s choice was not an “either/or” between making components more robust versus making the whole system more robust. It was an “and.”
They would use stability patterns to make individual instances more likely to survive. 

There’s no amount of code you can put into an instance that keeps AWS from terminating the instance! 

Instances in AWS get terminated just often enough to be a big problem as you scale, but not so often that every deployment of every service would get tested. 
Basically, Netflix needed failures to happen more often so that they became totally routine. (“If something hurts, do it more often.”).

Netflix has made an open source “Simian Army” with a number of different monkeys. 

Experience shows that every new kind of monkey it creates improves its overall availability. 

### Opt In or Opt Out?
Chaos is an *opt-out* process at Netflix.

Every service in production is subject to Chaos Monkey. 

A service owner can get a waiver, but it requires sign-off. 

Exempt services go in a database that Chaos Monkey consults.

Engineering management reviews the list periodically and prods service owners to fix their stuff.

Other companies use an opt-in approach.

*Adoption rates are much lower in opt-in environments than in opt-out.*

May be the only feasible approach for a mature, entrenched architecture. 

There may simply be too much fragility to start running chaos tests everywhere.

*When you’re adding chaos to an organization, consider starting with opting-in.*

Creates much less resistance and allow you to publicize some success stories before moving to an opt-out model.
If you start with opt-out, people might not understand what they’re opting out from or realize how serious it could be if they fail to opt-out.

## Adopting Your Own Monkey
Chaos Monkey makes many subtle issues visible.

### Prerequisites
Chaos engineering efforts should not be able to kill your company or your customers.

Netflix: If something fails, a customer will simply try pressing play again.

*If every single request in your system is irreplaceably valuable, then chaos engineering is not for you.* 
The whole point of chaos engineering is to disrupt things in order to learn how the system breaks.

You want a way to limit the exposure of a chaos test “blast radius”: the magnitude of bad experiences both in terms of the sheer number of customers affected and the degree to which they’re disrupted. 

To constrain the blast radius, you often want to pick “victims” based on a set of criteria. 
E.g. “every 10,000th request will fail” when you get started, but you’ll soon need more sophisticated selections and controls.

You’ll need a way to track a user and a request through the tiers of your system, and a way to tell if the whole request was ultimately successful or not. 
- If the request succeeds, then you’ve uncovered some redundancy or robustness in the system
-- The trace reveals where the redundancy saves the request. 
- If the request fails, the trace will shows where it happened

You also have to know what “healthy” looks like, and from what perspective.

Can monitoring tell when failure rates go from 0.01 percent to 0.02 percent for users in Europe but not in South America? 

Be wary that measurements may fail when things get weird, especially if monitoring shares the same network infrastructure as production traffic. 

*“If you have a wall full of green dashboards, that means your monitoring tools aren’t good enough.” *
There’s always something weird going on.

Make sure you have a recovery plan. 
The system may not automatically return to a healthy state when you turn off the chaos. 

You will need to know what to restart, disconnect, or otherwise clean up when the test is done.

### Designing the Experiment
After you have met the prerequisites, you need to design the experiment, beginning with a hypothesis.

The hypothesis behind Chaos Monkey was, “Clustered services should be unaffected by instance failures.”. 
Another hypothesis could be, “The application is responsive even under high latency conditions.”

Think about the hypothesis in terms of invariants that you expect the system to uphold even under turbulent conditions.
 
Focus on externally observable behavior, not internals. 

There should be some healthy steady state that the system maintains as a whole.

Once you have a hypothesis, check to see if you can even tell if the steady state holds now. 

You might need to go back and tweak measurements. 

Look for blind spots 
- hidden delay in network switches 
- lost trace between legacy applications

Think about what evidence would cause you to reject the hypothesis. 

Is a non-zero failure rate on a request type sufficient? Maybe not. 

If that request starts outside your organization, you probably have some failures due to external network conditions (e.g. aborted connections on mobile devices).
Use actual statistics to argue about the significance of observations.

#### Injecting Chaos
Apply your knowledge of the system to inject chaos. 

You know the structure of the system well enough to guess where you can use *injections* 
- kill an instance
- add latency
- make a service call fail 
 

Chaos Monkey only kills instances, the most basic and crude kind of injection. 

It will find weaknesses in your system, but it’s not the end of the story.

Latency Monkey adds latency to calls. Detects additional kinds of weaknesses. 
- some services just time out and report errors when they should have a useful fallback. 
- some services have undetected race conditions that only become apparent when responses arrive in a different order than usual.

When you have deep trees of service calls, your system may be vulnerable to loss of a whole service.
 
Netflix uses failure injection testing (FIT) to inject more subtle failures.

Can tag a request at the inbound edge (at an API gateway, for example) with a cookie that says, “Down the line, this request is going to fail when service G calls
service H.” 

Then at the call site where G would issue the request to H, it looks at the cookie, sees that this call is marked as a failure, and reports it as failed, without even making the request. (Netflix uses a common framework for all its outbound service calls)

Which instances, connections, and calls are interesting enough to inject a fault? 
Where should we inject that fault?

#### Targeting Chaos
Chaos Monkey picks a cluster at random, picks an instance at random, and kills it. 
Fine as an initial approach - most software has so many problems that shooting at random targets will uncover something alarming.

Once the easy stuff is fixed, you’ll start to see that this is a search problem.
You’re looking for faults that lead to failures; many faults won’t.


When you inject faults into service-to-service calls, you’re searching for the crucial calls.
However, we have to search in many dimensions.

Suppose there’s a process that runs once a week.

A fault during one part of that process causes bad data in the database. 

Later, when using the data for an API response, the service throws an exception and returns a 500 response code. 

Unlikely to be found by random chance. 

Randomness works well at the beginning because the search space for faults is densely populated. 

As you progress, the search space becomes more sparse, but not uniform. 

Some services, some network segments, and some combinations of state and request will still have latent killer bugs. 

Imagine exhaustively searching in a 2^n dimensional space, where n is the number of calls from service to service. 
With x services, there could be 2^2^x possible faults to inject

Thus we need a way to devise more targeted injections when we get past the initial stage. 

A top-level request generates a whole tree of calls that support it. 

Kick out one of the supports, and the request may succeed or it may fail. 

*It’s important to study all the times when faults happen without failures. 

The system did something to keep that fault from becoming a failure. 

We should learn from those happy outcomes, just as we learn from the negative ones.*

As humans, we apply our knowledge of the system together with abductive reasoning and pattern matching.

### Automate and Repeat
Aim for boring - chaos monkey is succesful when nothing happens because the system just keeps running as usual.

Assuming we did find a vulnerability, things probably got at least a little exciting in the recovery stages. 

Once you find a weakness. 
- fix that specific instance of weakness 
- see what other parts of your system are vulnerable to the same class of problem

With a known class of vulnerability, it’s time to automate testing.

Along with automation comes moderation. 

If the new injection kills instances, it probably shouldn’t kill the last instance in a cluster. 

If the injection simulates a request failure between service G to service H, then it isn’t meaningful to simultaneously fail requests from G to every fallback it uses when H isn’t working!

Companies with dedicated chaos engineering teams are all building platforms that let them decide how much chaos to apply, when, to whom, and which
services are off-limits. 

Ensures that one poor customer doesn’t get flagged for all the experiments at once! 

E.g. Netflix  “Chaos Automation Platform” (ChAP).

The platform decides what injections to apply and when, but it usually leaves the “how” up to some existing tool. 

Ansible is a popular - doesn’t require a special agent on the targeted nodes. 

The platform also needs to report its tests to monitoring systems, so you can correlate the test events with changes in production behavior.


## Disaster Simulations


People can similarly to servers become unavailable due to illness or family emergencies, quit, etc. 

Natural disasters can make a building or an entire city inaccessible. 

High-reliability organizations use drills and simulations to find the same kind of systemic weaknesses in their human side as in the software side.

In the large, this may be a “business continuity” exercise, where a large portion of the whole company is involved. It’s possible to run these at smaller scales.

Basically, you plan a time where some number of people are designated as “incapacitated.” 
Then you see if you can continue business as usual.

“zombie apocalypse simulation.”:
Randomly select 50 percent of your people and tell them they are counted as zombies for the day - must stay away from work and not respond to communication attempts.

The first few times you run this simulation, you’ll immediately discover some key processes that can’t be done when people are out. 

Maybe there’s a system that requires a particular role that only one person has. 

Or another person holds the crucial information about how to configure a virtual switch. 

During the simulation, record these as issues.

After the simulation, review the issues, like you would conduct a outage postmortem. 

Decide how to correct for the gaps by improving documentation, changing roles, or even automating a formerly manual process.

### Cunning Malevolent Intelligence

Start by collecting traces of normal workload. 

Workload is subject to the usual daily stresses of production operations only (not chaos).

The traces are used to build a database of inferences about what services a request type needs (a graph). 

Graph algorithms can be used to find links to cut with an experimentation platform.  

Once a link is cut, we may find that the request continues to succeed. 

Maybe there’s a secondary service, so we can see a new call that wasn’t previously active. 

It is then added to the database.

Or there may not be a secondary call, so we learn that the link we cut wasn’t that crucial after all.

A few iterations of this process can drastically narrow down the search space - reduces the time needed to run productive chaos tests.