
# Chapter 16 - Adaptation

## Convex returns
Rapid adaptation works when there’s a convex relationship between effort and return.

## Process and Organization
Optimizing decision cycles - time it takes to go through cycle is the key constraint on your company’s ability to absorb or create change.

Agile/lean development methods remove delay from the “act” part. 

DevOps helps remove additional delay from “act” + offers tools to improve “observe.” 

“Deciding” still has a lot of room for improvement - The timer starts when the initial observations are made, not when the story is ready to work on. 

#### The Danger of Thrashing
Organization changes direction without taking the time to receive, process, and incorporate feedback => constantly shifting development priorities/an unending series of crises.

Be careful not to shorten development cycle time so much that it’s faster than how quickly you get feedback from the environment.

How to avoid:
Create a steady cadence of delivery and feedback.

If one runs faster than the other, use extra time to find ways to speed up the other process. 

E.g. if development moves faster than feedback, build an experimentation platform to help speed up observation and decisions.


#### Platform Team
Modern platform is programmable - infrastructure, operations and programs.
 
Calls for team that views application development as its customer. 

Should provide API and command-line provisioning for the common capabilities that applications need, as well as Control Plane things. 
E.g. things that individual teams could build themselves, but aren’t valuable in isolation.

Implement *mechanisms* that allow others to do the real provisioning. 
E.g. provide an API that lets you install your monitoring rules into the monitoring service provided by the platform, but application developers setup the actual rules 

##### The Fallacy of the “DevOps Team”
Antipattern: Larger enterprises typically have a dedicated "DevOps team" between development and operations with the goal of moving faster and automating releases into production. 

DevOps is not just deployment automation - a cultural transformation, a shift from ticket- and blame-driven operations with throw-it-over-the-wall releases to one based on open sharing of information and skills, data-driven decision- making about architecture and design, and common values about production availability and responsiveness. 
Isolating these ideas to a single team undermines the whole point.

When a company creates a DevOps team, it typically intends to create 
- a platform team or a tools team
- a team to promote the adoption of DevOps by others. 

### Painless Releases
Releases should be a basic routine task.

Frequent releases results in user delight and business value. 

Forces you to get good at doing releases/deployments.

With more frequent releases, the better we can improve the release process 

Frequent releases with incremental functionality allows your company to outpace competitors and set the agenda in the marketplace.

In many companies making a release is not basic routine, cost too much and introduce a lot of risk.
 
To enable frequent releases, it is necessary to reduce the effort needed, remove people from the process, and make it more automated and standardized.

Patterns that help with ensuring frequent stable deployments:
 *“Canary Deploy”* pushes the new code to just one instance - released to remaining machines if all looks good.  
 *“Blue/Green Deploy,”* - Leaves time to test new code out before exposing it to customers.
 - Divide machines into two pools - one serving clients and one inactive.
 -  Release new code to inactive.  
- Shift production traffic over to inactive pool when all is good (simple with software-controlled load balancer) 

For really large environments, traffic might be too heavy for a small pool of machines to handle. 
Deploying in waves lets you manage how fast you expose customers to the new code.

Benefits of patterns 
- act as governors (p. 123) to limit the rate of dangerous actions.
- limit the number of customers who might be exposed to a bug 
-- restricts the time a bug might be visible 
-- restricts the number of people who can reach the new code 
- helps reduce the impact and cost of anything that got past the unit tests

### Service Extinction
Evolutionary architecture attempts to capture the adaptive power of incremental change, similarly to how evolution progresses species.

Makes an organization antifragile by allowing independent change and variation in small grains.

 Small units of technology/business capability can succeed or fail on their own.

When two competing services are tested against each, one will probably outperform the other - Options:
1. Keep running both - attendant development and operational expenses.
2. Decrease funding from the successful one to spend more on making the unsuccessful one better.
3. Retool the unsuccessful one to work in a different area where it isn’t competing with the other. 
4. Delete the unsuccessful one

#1 or #2 is common.
 
#3 is better - preserves some value.

#4 should be seriously considered. 
- Extinction is the most important part of evolution 
- Kill the service, delete the code, and reassign the team. 
-- Frees up capacity to work on higher value efforts 
-- Reduces dependencies
--- vital to the long-term health of the organization. 
-- Kill services in small grains to preserve the larger entity.

### Team-Scale Autonomy
Two-pizza team: teams should be sized no bigger than you can feed with two large pizzas.  

Not just about minimizing team size, although that has benefit for communication, but empowering teams. 

Each team member has to cover more than one discipline.  A two-pizza team is not possible with dedicated roles for everything.

Has a goal of reducing external dependencies. 

Dependencies across teams create timing and queuing problems. 

Anytime you need to wait for others to do their work before you can do yours, everyone gets slowed down. 

*Two-pizza is about having a small group that can be self-sufficient and push things all the way through to production.*

Reducing team size requires a lot of tooling and infrastructure support. 
Specialized hardware like firewalls, load balancers, and SANs must have APIs wrapped around them so each team can manage its own configuration without wreaking havoc on everyone else. 

The platform team must enable and facilitate team-scale autonomy.

#### No Coordinated Deployments
If you need to update both the provider and caller of an service interface at the same time, it’s a warning sign that those services are strongly coupled.

If you are the service provider, you are responsible. 
- If possible, rework the interface to be backward-compatible. (p. 263) 
- If not, consider treating the new interface as a new route in your API 
-- Leave the old one in place - remove it sometime after your consumers have updated

### Beware Efficiency
“Efficiency” sounds like it's only positive. 

Can go wrong in two crucial ways that hurt your adaptability.

Efficiency sometimes translates to “fully utilized.” 
E.g. every developer develops and every designer designs close to 100 percent of the time.

Has opposite effect - Keep the people busy all the time and your overall pace slows to a crawl.

A better view of efficiency looks at the process from the point of view of the *work instead of the workers*.
- Efficient value stream has a short cycle time and high throughput 
- Better for the bottom line than high utilization 

When you make a value stream more efficient, you also make it more specialized to today’s tasks - can make it harder to change for the future.
*Efficiency comes at the cost of flexibility.*:
- Car manufactorer example 
- Two-person sailboat vs. container-ship  
- Builds with Visual Studio out of TFS can't simply be switched to Jenkins and Git 
- Build pipelines can't just be ported from one company to another 

All the hidden connections that make things efficient also make it harder to adapt.

Be aware of these pitfalls whenever you build automation and tie into your infrastructure/platform. 

*Shell scripts are crude, but they work everywhere*

A fully automated build pipeline that delivers containers straight into Kubernetes every time you make a commit and that shows commit tags on the monitoring dashboard will let you move a lot faster at the cost of making some serious commitments.

Before you make big commitments, try to find out what might be coming down the road in organization. 

#### Summary
Adaptability requires conscious effort - Big Ball of Mud is the natural order of software. 
Without close attention, dependencies proliferate and coupling draws disparate systems into one brittle whole.

## System Architecture
“Form follows failure.” 
Changes in design are motivated more by the things early designs do poorly than those things they do well.
Each new attempt differs from predecessor in its attempts to correct flaws.

The fledgling system must do some things right, or it would not have been launched, and it might do other things as well as the designers could conceive.
Other features might work as built but not as intended, or they might be more difficult than they should be. 
In essence, there are gaps and protrusions between the shape of the system and the solution space it’s meant to occupy.


#### Evolutionary Architecture
*“support incremental, guided change as a first principle across multiple dimensions.”*

Many of basic architecture styles inhibit incremental, guided change. 
E.g. layered architecture where layers are separated to allow technology to change on either side of the boundary. 

Layers enforce vertical isolation, but *encourage horizontal coupling* which is much more likely to be a hindrance. 

Having a couple of gigantic domain classes that rule the world are common. 
Nothing can change without touching one of those and any change ripples through the codebase and calls for testing everything.

Rotating the barriers 90 degrees results in something like a *component-based architecture.* 

Instead of trying to isolate the domain layer from the database, we isolate components from each other.

Components can only have narrow, formal interfaces between each other.

Like microservice instances that happen to run in the same process.

Each component owns its whole stack - from database to UI/API. 

Creating a few of these component-oriented stacks results in a “self-contained system” that allows incremental guided change along the dimensions of “business requirements” and “interface technology.” 

Architecture styles that lend themselves to evolutionary architecture:
- Microservices
- Microkernel and plugins
- Event-based

Every architecture style has trade-offs and will be good in certain dimensions and weak in others. 

Startup vs. established enterprice

#### Bad Layering
The problem with layers is that any common change goes across several of them. 
e.g. a commit with new files like “Foo,” “FooController,” “FooFragment,” “FooMapper,” “FooDTO”.

Happens when one layer’s way of breaking down the problem space dominates the other layers.
 
The domain dominates, so when a new concept enters the domain, it has shadows and reflections in the other layers.

Layers can change independently if each layer expressed the fundamental concepts of that layer.
- “Foo” is not a persistence concept, but “Table” and “Row” are.
- “Form” is a GUI concept.

The boundary between each layer should be a matter of translating concepts.

In the UI, a domain object should be atomized into its constituent attributes and constraints. 

In persistence, it should be atomized into rows in one or more tables (for a relational DB) or one or more linked documents.

What appears as a class in one layer should be mere data to every other layer.

#### A Note on Microservices
*Microservices are a technological solution to an organizational problem.*
 
As an organization grows, the number of communication pathways grows exponentially. 

A growing piece of software shows a similar dramatic growth in the the number of possible dependencies within it.

Most classes has a few dependencies, but a couple has hundreds or thousands. 

Any particular change is likely to encounter one of those and incur a large risk of “action at a distance.” 

Makes developers hesitant to touch the problem classes - necessary refactoring pressure is ignored and the problem gets worse. 

Eventually, the software degrades to a Big Ball of Mud.

The need for extensive testing grows with the software and the team size. 
Unforeseen consequences multiply. 

Developers need a longer ramp-up period before they can work safely in the codebase. 

Microservices try to break the paralysis by curtailing the size of any piece of software. 
Should be no bigger than what fits in one developer’s head.


Microservices are great when you are scaling up your organization. 
But when you need to downsize, services can get orphaned. 

Even if they get adopted into a good home, it’s easy to get overloaded when you have twice as many services as developers.

Don’t pursue microservices just because the Silicon Valley unicorns are doing it.

Make sure they address a real problem you’re likely to suffer. 
Otherwise, the operational overhead and debugging difficulty of microservices will outweigh your benefits.

### Loose Clustering
Systems should exhibit loose clustering. 
- Allows clusters to scale independently. 
- Allows instances to appear, fail, recover, and disappear as the platform allows and as traffic demands.

In a *loose* cluster, the loss of an individual instance is insignificant, i.e. individual servers are identical (or any differentiated roles are present in more than one instance)

If a service needs a unique role, it should use leader election.

Bringing members up/down - independent/order shouldnt matter

Coupling prevents independent change - Instances in a cluster shouldn’t have any knowledge of individual instances of another cluster (knows virtual IP address/DNS name for service as a whole) 
 
Cluster members should not be configured to know the identities of other members.
- makes it harder to add/remove members. 
- encourages point-to-point communication - a capacity killer.

Cluster members should *discover* who their colleagues are. 

#### Explicit Context
Suppose your service receives a JSON body in a request:
{"item": "029292934"}

Our service can’t do much with this apparent identifier:
1. Pass it through as a token to other services (including returning it to the same caller in the future)
2. Look it up by calling another service
3. Look it up in our own database


Case 1 
- treats “itemID” as a token
-- don’t care about the internal structure 
- would be a mistake to convert it from string to numeric. 
-- imposes a restriction that doesn’t add any value and will probably need to be disruptively changed in the future

Case 2/3 
- treats “itemID” as something that can be resolved for more information 
- Serious problem: bare string doesn’t tell us who has the authoritative information 
-- If the answer isn’t in our own database, what service do we call? 
--- Implicit dependency: your service must already know who to call
--- limits you to working with just the one service provider 
--- If you need to support items from two different “universes,” it’s going to be very disruptive

Suppose the JSON looked like this:
{"itemID": "https://example.com/policies/029292934"}

- Still works if we just want to use it as an opaque token to pass forward 
-- We can treat it as *some* Unicode string, ignoring it's semantics
- Still works if we need to resolve it to get more information
-- Now our service doesn’t have to bake in knowledge of the solitary authority - we just use the URL
-- Easy to support more than one of them

Using a full URL makes integration testing easier 
- no need for “test” versions of the other services 
- Our own test harnesses can be supplied and URLs to those can be used instead of the production authorities


### Create Options

Sydney Opera house vs. Winchester house. 

Former is the coherent final realization of a vision. Not intended for modification. 

Continous change is the vision for the latter. 

Architecture debt - Stairways lead to ceilings. Windows look into rooms next door, but but allows for change.

A flat wall creates an option. 
A future owner can exercise that option to add a room/hallway/stair to nowhere.

*Modular systems inherently have more options than monolithic ones.* 
E.g. like building a PC. The graphics card is a module that you can substitute or replace.

Design Rules [BC00] identify six “modular operators.” in the context of computer hardware that applies equally to distributed service-based systems - every module boundary gives you an option to apply these operators in the future. 

**Splitting**
- Breaks a design into modules / a module into submodules.

Shipping service example: 

*The key with splitting is that the interface to the original module is unchanged. Before splitting, it handles the whole thing itself.
Afterward, it delegates work to the new modules but supports the same interface.*

**Substituting**
- Replacing one module with another.
- Original module and the substitute must share a common interface.
- Often has subtle bugs 

**Augmenting and Excluding**
Augmenting is adding a module to a system. 

Excluding is removing one. 

Example w. 3-tier application w. UI/API

**Inversion**
Takes functionality distributed in several modules and raising it up higher in the system. 

Takes a good solution to a general problem, extracts it, and makes it into a first-class concern.

Example w. A/B tests 

Inversion can be powerful. 

It creates a new dimension for variation and can reveal a business opportunity.

**Porting**
Repurposing a module from a different system. 

Any time we use a service created by a different project or system, we’re “porting” that service to our system

Risks adding coupling.

Results in a new dependency, and if the road map of that service diverges from our needs, then we must make a substitution. 

In the meantime, though, we may still benefit from using it.

Analogous to porting C sources from one operating system to another. 

The calling sequences may look the same but have subtle differences that cause errors. 

The new consumer must be careful to exercise the module thoroughly via the same interface that will be used in production. 

That doesn’t mean the new caller has to replicate all the unit and integration tests that the module itself runs. 

It’s more that the caller should make sure its own calls work as expected.


Modules can also be ported through *instantiation.* 

Not commonly considered, but nothing says that a service’s code can only run in a single cluster. 

If we need to fork the code and deploy a new instance, that’s also a way to bring the service into our system.

These operators can create any arbitrarily complex structure of modules. 

The economic value of the system increases with the number of options/boundaries where you can apply them.

Use operators as thinking tools. 

- When you look at a set of features, think of three different ways to split them into modules
- Think of how you can make modules that allow exclusion or augmentation
- See where an inversion might be lurking
 
## Information Architecture
Information architecture is how we structure the data and metadata we use to describe the things that matter to our systems.

A set of models that capture some facets of reality. 

You chose which facets to model, what to leave out, and how concrete to be.

Current state vs. event:
The act of changing a database is a momentary operation that has no long-lived reality of its own. 

Others mainly care about the event - preserved as a journal or log.

The notion of the current state is really to say, “What’s the cumulative effect of everything that’s ever happened?”

Each of these embeds a way of modeling the world.

Each paradigm defines what you can and cannot express. 

None of them are the whole reality, but each of them can represent some knowledge about reality.

When building systems you must decide what facets of reality matter to your system, how you are going to represent those, and how that representation can survive over time. 

You also have to decide what concepts will remain local to an application or service, and what concepts can be shared between them.

Sharing concepts increases expressive power, but it also creates coupling that can hinder change.


### Messages, Events, and Commands


Ways "event" is used to describe an action 
- Event Notification
-- A fire-and-forget, one-way announcement. 
-- No response is expected/used
- Event-carried state transfer
-- Replicates entities/parts of entities so other systems can do their work
- Event sourcing
-- When all changes are recorded as events that describe the change

Command-query responsibility segregation (CQRS) is not an event, but events are often found on the “command” side.

Apache Kafka - a persistent event bus: 

Mix of a message queue and a distributed log. 

*Events stay in the log forever.* 

With event sourcing, the events themselves are the authoritative record. 

Since it can be slow to walk through every event in history to figure out the value of attribute A on entity E, *views* are used to answer such queries fast.

With an event journal, several views can each project things in a different way. 
However, the event journal is the only "truth". 

The others are caches, optimized to answer a particular kind of question.
Views may even store their current state in a database of their own.

Versioning can be a challenge with events, especially once you have years’ worth of them. 

Avoid 
- closed formats like serialized objects
- frameworks that require code generation based on a schema 
- anything that requires you to write a class per message type or use annotation-based mapping 

Do
- Use open formats like JSON or self-describing messages

Treat the messages like data instead of objects to make it easier to support very old formats.

Apply versioning principles (Chapter 14, page 263).

In a sense, a message sender is communicating with a future interface.  
A message reader is receiving a call from the distant past. 

Using messages brings complexity. 

Business requirements are expressed in an inherently synchronous way. 
Requires some creative thinking to transform them to be asynchronous.

### Services Control Their Identifiers
Suppose you work for an online retailer and you need to build a “catalog” service. 
How should we identify which catalog goes with which user?
1:1 owner to catalog approach - owner ID is included in the request.
Problems:
- The catalog service must couple to one particular authority for users. 
-- I.e. the caller and the provider have to participate in the same authentication/authorization protocol
--- Protocol only works inside organization - makes it hard to work with external partners
-- increases the barrier to use of the new service
- One owner can only have one catalog 
-- If a consuming application needs more catalogs, it has to create multiple identities in the authority service (e.g. multiple account IDs in the AD)

1:* owner to catalog approach - The user provides a catalog ID with requests.

The catalog service issues an identifier for that specific catalog.

The catalog service acts like a little standalone SaaS business. 

With many customers that use catalogs however they need.

Some users will will change their catalogs all the time.

Other are just building a catalog for a one-time promotion.  

Different users may even have different ownership models.

You will probably need to ensure that callers are allowed to access a particular catalog. 
Especially when openening the service up to business partners. 

A “policy proxy” can map from a client ID (internal or external) to a catalog ID. 

This way, questions of ownership and access control are handled in a centrally controlled location, not the catalog service.

*Services should issue their own identifiers. 
Let the caller keep track of ownership. 
This makes the service useful in many more contexts.*

### URL Dualism
A URL is a reference to a representation of a value. 

You can exchange the URL for that representation by resolving it, like dereferencing a pointer. 

You can pass the URL around as an identifier. 

A program may receive a URL, store it as a text string, and pass it along without ever attempting to resolve it. 

Or your program might store the URL as an identifier for some thing or person, to be returned later when a caller presents the same URL.

*This dualism can be used to break a lot of dependencies that otherwise seem impossible.*

Retailer example 

It might seem expensive to resolve every URL to a source system on every call. That’s fine; use a HTTP cache to reduce latency.

The front end can now even use services that didn’t even exist when it was created. 

The new service just have to returns a useful representation of an item.

The item details doesnt even have to be served by a dynamic, database-backed service. 

You can just publish static JSON, HTML, or XML documents to a file server.
Item representations even have to come from inside your own company. 

The item URL could point to an outbound API gateway that proxies a request to a supplier or partner - a variation of “Explicit Context.” (p. 306.) 

We use URLs because they carry along the context we need to fetch the underlying representation. 

Has much more flexibility than plugging item ID numbers into a URL template string for a service call.

However, don’t go making requests to any arbitrary URL passed in to you by an external user. 

You need to encrypt URLs that you send out to users. 

That way you can verify that whatever you receive back is something you generated.

### Embrace Plurality
One of the basic EAP's is the “Single System of Record.” 

Any particular concept should originate in exactly one system, and that system will be the enterprise-wide authority on entities within that concept.

Hard getting all parts of the enterprise to agree on what those concepts actually are.

E.g. customer
- a company with which we have a contractual relationship.
- someone entitled to call our support line.
- a person who owes us money or has paid us money in the past.
- someone that might buy some- thing someday in the future.

A customer is *all* of these things. 

Being a “customer” isn’t the defining trait of a person or company, it's one facet of that entity.

It’s about how your organization relates to that entity.
- To sales team, someone who might someday sign another contract.
- To support organization, someone who is allowed to raise a ticket. 
- To your accounting group, a customer is defined by a commercial relationship. 

Each of those groups is interested in different attributes of the customer. 

Each applies a different life cycle to the idea of what a customer is. 

Your support team doesn’t want its “search by name” results cluttered up with every prospect your sales team ever pursued. 

Even the question, “Who is allowed to create a customer instance?” will vary.

“Dark matter” issue:

A system of record must pick a model for its entities. 

Anything that doesn’t fit the model can’t be represented there.

Either it’ll go into a different (possibly covert) database or it just won’t be represented anywhere.

*Instead of creating a single system of record for any given concept, we should think in terms of federated zones of authority.
We allow different systems to own their own data, but we emphasize interchange via common formats and representations. 
"Duck-typing" for the enterprise: If you can exchange a URL for a representation that you can use like a customer, then as far as you care, it is a customer service, whether the data came from a database or a static file.*

### Avoid Concept Leakage
Music retailer pricing example
 
From individual track prices to priced in groups 

Tracks are extended with "price point" external key that references a row with "amount" 

Affects all other downstream systems that would need to receive a feed of the price points.

Until this time, items had prices. 

The basic customer-visible concepts of category, product, and item were very well established.

The internal hierarchy of department, class, and subclass were also well understood.

Essentially every system that received item data also received these other concepts.

But would they all need to receive the “price point” data as well?

Introducing price point as a global concept across the retailer’s entire constellation of systems was a massive change. 

The ripple effect would be felt for years. 

Very hard to coordinate all the releases needed to introduce that concept. 

But it looked like that was required because every other system certainly needed to know what price to display, charge, or account for on the tracks.

Not a concept that other systems needed for their own purposes. 

Only needed because the item data was now incomplete thanks to an upstream data model change.

That was a concept leaking out across the enterprise. 

Price point was a concept the upstream system needed for leverage. 
It was a way to let the humans deal with complexity in that product master database. 

To every system downstream it was incidental complexity. 

The retailer would’ve been just as well served if the upstream system flattened out the price attribute onto the items when it published them.

*There’s no such thing as a natural data model, there are only choices we make about how to represent things, relationships, and change over time.
We need to be careful about exposing internal concepts to other systems.* 
Creates semantic and operational coupling that hinders future change.

Summary
- We don’t capture reality, we only *model* aspects of it
- There’s no such thing as a “natural” data model, only choices that we make 
- Every paradigm for modeling data makes some statements easy, others difficult, and others impossible
- Important to make deliberate choices about when to use relational, document, graph, key-value, or temporal databases
- You always need to think about whether we should record the new state or the change that caused the new state
- Use and abuse of identifiers causes lots of unnecessary coupling between systems. 
-- Invert the relationship by making services issue identifiers rather than receiving an “owner ID.” 
-- take advantage of the dual nature of URLs to both act like an opaque token or an address we can dereference to get an entity
- Be careful about exposing concepts to other systems - may force them to deal with more structure and logic than they need