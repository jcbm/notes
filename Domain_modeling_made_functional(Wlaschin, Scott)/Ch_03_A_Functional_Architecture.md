# Chapter 3 - A functional architecture
Translating our domain understanding in a software architecture based on functional principles 

Too early in clarification process to fully define a final architecture. 
We need a rough idea, so we can start implementing the subset of the domain that we have understood, to determine how this should fit together with other components that we dont yet understand fully. These will then be implemented when we have understood those parts through more event storming and interviews.

Hierarchical [C4 approach](http://static.codingthearchitecture.com/c4.pdf) for software architecture levels:
- System context - entire system
- Containers - deployable units - website, web service, database, etc 
- Components - major building blocks in the code 
- Classes/modules - contains methods/functions 

(context contains containers containing components containing classes)
 
A good architecture succesfully defines boundaries between containers, components, and modules, such that when new requirements arise, the “cost of change” is minimized.
 
## Bounded contexts as autonomous software components
A BC is an autonomous subsystem with a well-defined boundary -  this could mean multiple things from an architecture POV.
- Single monolithic deployable - BC could be a module with a well-defined interface or a .NET assemble.
- SoA architecture - each BC could be a container. 
- Microservice architecture - more granular design with a standalone deployable container for each workflow.


It’s important chose the right boundaries, but this is hard to do at the beginning of a project, and boundaries will likely change as we learn more about the domain.
Good practice to build system as a monolith first as it is the easiest to refactor into a different structure/decoupled containers. 
Microservices as an initial architecture should be avoided:
- It is hard to create a truly decoupled microservice architecture.
- Going straight to microservices means that operations will be harder (["microservice premium"](https://www.martinfowler.com/bliki/MicroservicePremium.html)).  

## Communicating between bounded contexts
BCs communicate through events - the output of one is the input of another.
We should strive for decoupling in transmission of events to create fully autonomous components. 

The exact transmission mechanism depends on the architecture we choose.
- Microservices/agents - Queues provide buffered asynchronous communication; would be the first choice 
- Monolithic - we can use the same queuing approach internally, or use have a direct linkage through a function call between the upstream component
and the downstream component 

As long as we design the components to be decoupled, the choice is not important right now in the process 

The handler that translates events to commands can live at the boundary of the context or it can be done by a separate [router](http://www.enterpriseintegrationpatterns.com/patterns/messaging/MessageRouter.html) or [process manager](https://www.slideshare.net/BerndRuecker/long-running-processes-in-ddd) running as part of your infrastructure. 

### Transferring Data Between Bounded Contexts

Generally, an event for communication between contexts will contain all the data that the downstream components need to process it.
If the data is too large to be contained in the event, a reference to a data storage location can be transmitted instead.

We will not pass the domain objects between contexts; instead we will use [Data Transfer Objects](https://martinfowler.com/eaaCatalog/dataTransferObject.html) that will contain most of the same information, but it will be structured differently to suit its purpose (see chapter 11).

The top-level DTOs that are serialized are typically event DTOs, which in turn contain child DTOs, such as a DTO for Order , which in turn contains additional child DTOs (e.g. a list of DTOs representing OrderLines).
### Trust Boundaries and Validation
A BC has a "trust boundary". 
Inside the bounded context, everything will be trusted and valid
Outside the bounded context, we assume data might not be valid and trust nothing.

We add “gates” at the beginning and end of the workflow that act as interme diaries between the trusted domain and the untrusted outside world.

At the input gate, we will always validate the input to make sure that it conforms to the constraints of the domain model (see chapter 11).
At the output gate, we ensure that we don't leak information that could be a security issue or create accidental coupling. I.e. we intentionally "lose" information when we convert domain objects to DTOs. 

## Contracts between bounded contexts
We want to reduce coupling between BCs as much as possible, but we must have some when using a shared communication format. 
Events and DTOs define a contract between BCs - must agree on the format . 
Should upstream or downstream BC dictate contract? 

- **Shared Kernel** -  two contexts share a common domain design, so the teams involved must collaborate. Changing the definition of an event or a DTO must be done only in consultation with the owners of the other affected contexts
- **Customer/Supplier or Consumer Driven Contract** -  downstream context defines the contract that the upstream context must adhere to. The two domains can evolve independently, as long as the upstream context follows the contract
-**Conformist** - downstream context accepts the contract provided by the upstream context and adapts its own domain model to match

### Anti-Corruption Layers
When communicating with an external system, the interface that is available often does not match our domain model.
The interactions and data need to be transformed into something more suitable for use inside the BC .
The BC would become corrupted if we adapt our domain models to be compatible with this external model. 
The “input gate” often plays the role of the ACL — it prevents the internal, pure domain model from being “corrupted” by knowledge of the outside world.
ACL is different from performing validation or preventing data corruption; it's goal is to translate between the external and internal languages.

### A Context Map with Relationships
After we have defined the relationships between our BCs, we can define a context map that illustrates these. 
The map not only shows technical relationships, but also relationships between the owning teams and how they are expected to collaborate.
Organizational structure must be aligned with the architecture - inverse conway maneuver is one way to ensure this.
## Workflows within bounded context
The workflows that we identified during the discovery process will each be mapped to a single function that takes a single command object and creates a list of event objects.

Workflows can be **public** (i.e. exposed to other BCs) or **internal** to the BC. 
A workflow is always contained within a single bounded context and never crosses through multiple contexts

### Workflow Inputs and Outputs
The input to a workflow is always the data associated with a command.
The output is always a set of events to communicate to other contexts.

Example p. 51
Note that a workflow function does not publish events, but just returns them. Publishing is the concern of another function. 

### Avoid Domain Events Within a Bounded Context
In OOP, it is common to use a handler approach for events, where an event gets raised inside a workflow and subscribed event-listener(s) take care of it (e.g. *Observer* pattern).

In FP, we prefer to append a handler function at the end of the workflow. 
This is more explicit and there's no global event managers with mutable state which makes the code simpler to work with.
 
## Code structure withing a bounded context  
Traditional layered approach: 
- a core domain or business logic layer, 
- a database layer, 
- a services layer, 
- an API or user interface layer. 

A workflow will start at the top layer, work its way down to the database layer, and then return back to the top

Breaks the design principle “code that changes together belongs together.”
As  the layers are assembled “horizontally,” a change to the way that the workflow works means that you need to modify every layer.
A better way would be to use “vertical” slices, where each workflow contains all the code it needs; when the requirements change for a workflow, only the code in that particular vertical slice needs to change.

However, in this approach the layers are intermingled in a way that makes understanding and testing the code complicated.

### The Onion Architecture
[Onion Architecture](http://jeffreypalermo.com/blog/the-onion-architecture-part-1/): Domain code at the center with other aspects assembled around it.
Each layer can only depend on inner layers, not on layers further out.
Similar to [hexagonal](http://alistair.cockburn.us/Hexagonal+architecture)/[clean](https://8thlight.com/blog/uncle-bob/2012/08/13/the-clean-architecture.html) architecture.
To ensure that all dependencies point inward, we will use composition.

### Keep I/O at the Edges
To keep our functions pure, we must push I/O to the edges of the onion -  database must only be accessed at the start or end of a workflow, not inside the workflow.
Benefit of forcing us to separate different concerns: the core domain model is concerned only with business logic, while persistence and other I/O is an infrastructural concern.

Combines very nicely with the concept of persistence ignorance (chapter 2); if you can't access a database from your workflow, you can't model your domain around it.