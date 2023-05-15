# Chapter 1: Introducing Domain-Driven Design

“Garbage in, garbage out” rule:
Think of software development as a pipeline with an input (requirements) and an output (the final deliverable). If the input is bad (unclear requirements or a bad design), then no amount of coding can create a good output.

Domain-driven design is particularly useful for business and enterprise software, where developers have to collaborate with other nontechnical teams; not suited for systems software, games, etc 

## The Importance of a Shared Model

We must understand a problem correctly before we try to solve it or the solution will not be useful.
It’s the developers’ understanding, not the domain experts’, that gets released to production

Using written specifications or requirements documents to try to capture all the details of a problem is common.
This approach often creates distance between the people who understand the problem best and the people who will implement it.
 
 "Telephone game" - Domain experts -> Business analysts -> Requirements doc -> Architect -> Design document -> Development team - message is increasingly distorted from original
 
Better to eliminate the intermediaries and encourage the domain experts to be intimately involved with the development process, introducing a feedback loop between the development team and the domain expert,
Process can still result in distortion and loss of important subtleties if the developer fails to translate domain expert's mental model to code. If the code doesn’t quite correspond to the concepts in the domain, then future developers working on the codebase without input from a domain expert can easily misunderstand what’s needed and introduce errors.

The DDD aproach:
Domain experts, the development team, other stakeholders, and the source code itself all share the same model.
No translation from the domain expert’s requirements to the code; the code is designed to reflect the shared mental model directly.

Benefits:
- Faster time to market.
- Less waste. 
- Easier maintenance and evolution. 

## Understanding the Domain Through Business Events

The starting point for almost all of the business processes we want to model.
Focus on business events rather than data structures.
A typical business process as a series of data or document transformations. The value of the business is created in this process of transformation.
Important to understand how these transformations work and relate to each other.

**Domain Events** - when something happens triggering data to become transformed manually by a user or an automated process (receiving a mail, some condition becomes true, or a scheduled cronjob) 

Always in past sense 

### Using Event Storming to Discover the Domain

Bring together a mix of people who understands different parts of the domain for a facilitated workshop.
The idea is to get all the attendees to participate in posting what they know and asking questions about what they don’t know.
 Events are put on post it notes and put on the wall, allowing everyone to chime in with their knowledge of how this leads to new events 
 
 ### Discovering the Domain: An Order-Taking System
 
 Example of applying "focus on business events” - p. 8
 
### Expanding the Events to the Edges

Follow the chain of events out as far as you can, to the boundaries of the system. To start, you might ask if any events occur before the leftmost event. Similarly, you can ask about the rightmost.

Terminology:
 A *scenario* describes a goal that a customer (or other user) wants to achieve
 A *use case* is a more detailed version of a scenario, which describes in general terms
the user interactions and other steps that the user needs to take to accomplish
a goal.

*Business process* describes a goal that the business (rather than an individual
user) wants to achieve.

*Workflow* is a detailed description of part of a business process. It lists
the exact steps that an employee (or software component) needs to do to
accomplish a business goal or subgoal.

### Documenting Commands
Identifying what tries to/makes the events happen - commands.
if a command succeeds, it will initiate a workflow resulting an a corresponding Domain Event(s).

Command: “Place an order”; Domain Event: “Order placed.”

An event triggers a command, which initiates some business workflow. The output of the workflow is some more events. Those events may then trigger new commands.
Not all events are associated with a command; events can also be triggered by, say, a scheduler or monitoring system.

## Partitioning the Domain into Subdomains
Second guideline: “Partition the problem domain into smaller subdomains.”
A business has separate departments for the various aspects of a process - i.e. "order-taking" consists of taking the order, shipping, and billing - handled by different units.  We can follow that same separation in our design. 
Each of these areas corresponds to a domain. 
Sub-domains: Within a domain might be areas that are distinctive as well. 

When partitioning a domain into smaller parts we must be careful not to create to clear boundaries; the real world is fuzzy, e.g. there's some overlap between order-taking, shipping and billing. Someone in one of these departments must know a little about how the others work. 

## Creating a Solution Using Bounded Contexts

We must limit solution to only capture the information that is relevant to solving a particular problem; not all of the domain.
“Problem space” and “solution space,” must be treated as two different things.
To build the solution we will create a model of the problem domain limited to the aspects of the domain that are relevant and re-create them in our solution space

The domains and subdomains in the problem space are mapped to bounded contexts in the solution space.
A  bounded context is a small domain model in its own right.

A domain in the problem space does not always have a one-to-one relationship to a context in the solution space.
A single domain may be broken into multiple bounded contexts—or multiple domains in the problem space are modeled by only one bounded context. This is common integrating with legacy systems.

It is important that each bounded context has a clear responsibility when partitioning the domain. 
When we implement the model, a bounded context will correspond exactly to some kind of software component (a DLL, standalone, service, a namespace, etc). 

### Getting the Contexts Right

One of the most critical challenges of a domain-driven design is to determine the right context boundaries.

Not an exact science - Guidelines:
- Listen to the domain experts.
- Pay attention to existing team and department boundaries.
- Focus on the “bounded” part of a bounded context - be aware of scope creep
- Design for autonomy
- Design for friction-free business workflows

### Creating Context Maps
Context Maps describe high-level interactions between contexts 
Describes system as a whole - In complex designs you'd create a series of smaller maps, each focusing on specific subsystems.
Directed relationships - downstream/upstream 

### Focusing on the Most Important Bounded Contexts
Some domains are more important than others. These are the core domains — provide a business advantage, bring in the money.
Others are required, but not core - *supportive* domains. 
Or *Generic* domains, if they are not unique to the business.

p.21 for discussion.

Important to prioritize and not to attempt to implement all bounded contexts at the same time;
Focus on bounded contexts that add the most value and expand from there.

### Creating a Ubiquitous Language
Ubiquitous Language - The set of concepts and vocabulary that is shared between everyone on the team.
Defines the shared mental model for the business domain.
Should be used everywhere in the project.
Defined by collaboration between everyone on the team.
Always a work in progress; as the design evolves, new terms and concepts are discovered.

Often a single Ubiquitous Language cannot cover all domains and contexts.
Each context will have a “dialect” of the Ubiquitous Language, and a word can mean different things in different dialects (e.g. "order" has slightly different meaning in shipping and billing as they focus on different aspects).

## Summarizing the Concepts of Domain-Driven Design
*Domain* - area of knowledge associated with the problem we are trying to solve, or which a “domain expert” is expert in

*Domain Model* - a set of simplifications that represent the aspects of a domain that are relevant to a particular problem. The domain model is part of the solution space, while the domain that it represents is part of the problem space.

*Ubiquitous Language* - a set of concepts and vocabulary that is associated with the domain; shared by both team members and source code

*Bounded context*  
- a subsystem in the solution space with clear boundaries that distinguish it from other subsystems
- often corresponds to a subdomain in the problem space. 
- has its own set of concepts and vocabulary; its own dialect of the Ubiquitous Language.

*Context Map* - a high-level diagram showing a the relationships between a set of bounded contexts.

*Domain Event* - a record of something that happened in the system; often triggers additional activity

*Command*
- a request for some process to happen
- triggered by a person or another event
- If the process succeeds, the state of the system changes and one or more Domain Events occur 