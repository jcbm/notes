# From 0 to Millions: A Guide to Scaling Your App - Part 1 - https://substack.com/inbox/post/102989215

## Single server for entire application stack
RESTful API for mobile and web clients 
Static content are stored on the local disk of the server 
Database runs on the same host as the application server 
Sufficient for 100-1000s of users, depending on the application. 

## Scaling up for higher load involves improving hardware
- Expensive
- Only works until a certain limit where hardware will have to be replaced again 
- no failover/redundancy - if server goes down, application is completely unavailable

## Horizontal scaling - move db to own host to tweak AS and db seperately
the database workload requires a different set of CPU, memory, and disk capacity than the application server. By separating the two, they can be independently tuned for their specific conditions.
may make sense to move static files on a dedicated file server, Depending on the size of the static contents and the traffic volume

Adding application servers 
As server becomes overloaded, we add more instances and use a load balancer to distribute requests (we are still using the same db and file server for each) 
LB algos
-Round robin - each request goes to new server 
-Sticky session - requests from a given user always goes to the same server 

Avoid sticky sessions - can lead to uneven distribution of load, leading to some servers being overloaded, while others are underutilized

Serverless round robin - store user session data in db .
On every request, session data is retrieved and used when processing request.
Greater resilience in the event of a server failure, as a stateless server can be easily replaced.

Other Load balancer benefits:
- TLS termination - LB handles TLS termination, which simplifies the deployment of secure applications.
- Transparent failure - tracks the health of individual application servers and automatically takes each one in and out of service based on its health, making software deployment non-disruptive from the user’s perspective.
- Failover If one application server fails, the load balancer automatically directs traffic to the healthy servers, providing high availability and resilience.

## Primary-replicate for database bottleneck - handles several hundred thousand users, up to a million for simple applications 
Users experience that search in products is slow 
Massive number of reads, frequent writes 
Add 1 db for writes, multiple to serve reads 
 we deploy a database cluster with a single primary database to handle writes and a series of read replicas to handle reads. We can have different replicas handle different kinds of read queries to spread the load.
 => replication lag 
 Whether this slight inconsistency is acceptable is determined on a case-by-case basis, and it is a tradeoff for supporting an ever-increasing scale.
 Redirect small number of reads that must be accurate to write instance who is always consistent
 
 Handling fuzzy search
 allows for searching even if there are typos in the search term, which is useful for applications like online e-commerce
 Relational db has no support for fuzzy search 
 
 For these specific use cases, it is useful to replicate a subset of the dataset where such search capabilities are required for a full-text search engine like Elasticsearch.