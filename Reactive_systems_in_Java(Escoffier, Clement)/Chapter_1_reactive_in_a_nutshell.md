# Chapter 1 - Reactive in a nutshell


### What Do We Mean by Reactive?

*An application reacting to stimuli, such as user events, requests, and failures.*

Reactive is
- an approach to designing, implementing, and reasoning about your system in terms of *events and flows*. 
- all about building responsive, resilient, and elastic applications. 
- resource utilization through efficient management of resources and communication.


### Reactive Software Is Not New

The nature of software is to react to user inputs and operating system signals.
The DYSEAC (1950s) was already using hardware interrupts as an optimization to avoid waiting time in polling loops. 

Reacting to events implies being *event-driven*. 
Event-driven software receives and produces events. 
Received events determine the flow of the program. 

Event-driven is associated with asynchronicity: you don’t know when you are going to receive events.

### The Reactive Landscape

Reactive systems are motivated by the challenges of building distributed systems, but principles can also be applied to nondistributed systems. 
Reactive systems concept was introduced in "The Reactive Manifesto". 	
Reactive provides *a blueprint to ensure that no significant known concerns are overlooked while architecting and developing your system.* 

A reactive system is most importantly responsive. 
Must handle requests in a timely fashion, even under load or facing failures.
To achieve responsiveness, the manifesto proposes using *asynchronous message passing as the primary way to communicate between the system's components*. 


Using asynchronous message passing at the core of distributed systems has certain consequences.
The application must use *asynchronous code* and *nonblocking I/O*, the ability provided by the OS to enqueue I/O interactions without having to actively wait for the completion. 
The latter is essential to improve resource utilization, such as CPU and memory.

Many toolkits/frameworks - Quarkus, Eclipse Vert.x, Micronaut, Helidon, and Netty - are using nonblocking I/O for this very reason - doing more with limited resources.
Having a runtime leveraging nonblocking I/O is not enough to be reactive - you need must write asynchronous code embracing the nonblocking I/O mechanics.
Otherwise, there's no resource utilization benefits. 

Writing asynchronous code is a paradigm shift. 
From the traditional (imperative):
 do x; do y;  
To:
 on event(e) do x; on event(f) do y;. 

Callbacks are a very straightforward approaches to implementing such code. You register functions that are then invoked when events are received. 
Futures, promises, and coroutines are all based on callbacks and offers higher-level APIs. 

Reactive programming (ch. 5) uses *data streams*. You observe the data transiting in streams and react to it.
However, if you have a fast producer directly connected to a slow consumer, you may flood the consumer. 
That would go against the responsiveness and anti-fragile ideas promoted by Reactive. 
Reactive Streams proposes an asynchronous and nonblocking backpressure protocol where the consumer signals to the producer its availability. 
May not be applicable everywhere, as some data sources cannot be slowed down.

#Why Are Reactive Architectures So Well-Suited for Cloud Native Applications?

The cloud is a distributed system.
 Cloud applications face a high degree of uncertainty. 
- Provisioning of your application can be slow, fast, and even fail
- Communication disruptions are common, because of network failures or partitions 
- You may hit quota restrictions, resource shortages, and hardware failures 
- Services you depend on can be unavailable at times or moved to other locations

While the cloud provides outstanding facilities for the infrastructure layer, it covers only half of the story.
Your application also needs to be designed to be a part of a distributed system and the challenges these systems present. 
The reactive principles help to embrace the inherent uncertainty and challenges of distributed systems and cloud applications.
As microservices and serverless computing are becoming prominent architectural styles, the reactive principles become even more important - help ensure that you design your system on a solid foundation.
 
 ### Reactive Is Not a Silver Bullet
Reactive has pros and cons. 
Reactive architectures are well-suited for distributed and cloud applications but can be disastrous on more monolithic and computation-centric systems.
Reactive is a good candidate for a system that relies on
- remote communication
- event processing
- high efficiency
 
Reactive only brings unnecesary complexity to systems that
- uses mostly in-process interactions
- handles a few requests per day
- is computation-intensive 

Reactive places events at the core of your system. 
If you are used to the traditional synchronous and imperative way of building applications, becoming reactive can be challenging. 
Being asynchronous disrupts most traditional frameworks and we try to move away from Remote Procedure Call (RPC) and HTTP endpoints. 