# System design interview - Chapter 1 - Scale from Zero to millions of users

## Initial setup for few users

Application server and db on same server 
Web browser and mobile client use DNS to retrieve server IP and send it requests over HTTP

## With a growth in users it becomes necessary to split db and application server so they can be scaled independently.

#### Choice of db 
#### Relational database 
Consists of structured rows and tables - **allows for joins**

#### Non-relation database 
- key-value stores
- graph stores
- column stores 
- document stores 

**Generally dont support joins** 

Relational databases work well for many purposes. *Non*-relational databases might be the right choice when:

- you need low latency

- data is unstructured and not relational 

- you only need to serialize/deserialize data (JSON, XML, YAML)

- you need to store a lot of data 


## With many concurrent users, the application server will slow down and users will fail to connect - we need to add more server resources.

### Vertical scaling vs horizontal scaling 
Vertical - adding more/stronger hardware to the server (cpu, ram, etc)

Horizontal - make more servers available.

#### Horizontal is preferable for big data systems:
- Vertical has an upper limit on how much you can upgrade it. 
- Vertical lacks failover and redundancy - a powerful machine is still a single point of failure - if server goes down, no users can access. 


#### Horizontal scaling requires a load balancer (LB) for failover and uniform load spread 
Register LB with DNS instead of an application server.

LB knows private internal IPs of app-servers in pool (private for security reasons). 

If a server goes down, LB will route all traffic around it. 
If traffic grows too high for the current server pool, it's simply to add more servers and register them with the load balancer:

*VM auto scalers* are used by cloud services to inform load balancer of whenever a new server is added or removed. 
Alternatively, the load balancer can be noticed directly by a new server when it is added or the information can be manually input in the LB by console.

Load balancers will typically send live checks. 

## Database replication - Extending failover and redundancy properties to database 
### Master for writes, slaves for reads
In db replication servers are assigned as masters and slaves. 

All updates go to the master (i.e. Insert, update, delete), all reads go to the slaves.


Applications generally read much more than they write, so slaves outnumber masters greatly. 

Updates are propagated from the master to every slave that will be eventually consistent.   
 

- Improves performance - queries can be performed in parallel, particularly reads. 
- Improves reliability - as data is replicated we can handle the complete loss of a database server (we must ensure servers are located in different geographically distant datacenters to be resistant to fires or natural disasters)  
- High availability - Website remains active if a database go down as we can just read data from elsewhere.

### Load balancing on failover
If all slaves are down, reads are exceptionally directed to the master server untill slaves have respawned. 

If the master goes down down, **a random slave is appointed master**. As slave is not necesarily up to date with the updates received by the late master, complex algorithms are used for data recovery so the new master holds the same data as the old master. (Alternatives like *multi-masters* and *circular replication*)

## Adding a cache - Improving response time

Temporary in-memory storage for the result of expensive database lookups or frequently
accessed data, which reduces database load and avoids calls. 

After receiving a request, a web server first checks if the cache has the apropriate response.
If not, the database is queried, the result is stored in cache, and sent to the client (*Read-through* cache)

Considerations
- Caches are **suitable for usage patterns with frequent reads and rare modifications** 
- **Apropriate expiration time must be chosen** - high enough to not read too frequently from db, but short enough that data dont get stale 
- Consistency is tricky - **data modification in db and cache are not in a single transaction**
- Multiple caches **must be placed across several datacenters to avoid single point of failure** 
- **Eviction policies** for when cache is full: LRU, LFU, FIFO

## Use CDNS for static content for global userbase 
Geographically disperved servers that cache content like images, video, css and js files 
Dynamic caching is also possible based on request path, query strings, cookies and request headers 

When a user visits  website, the CDN server closest
to the user will deliver static content. Intuitively, the further users are from CDN servers, the
slower the website loads

If CDN does not have a requested static resource, it must be retrieved from the application server and then stored along with a TTL value. 

### Considerations for using CDN
- Costly - you pay third-party for CDN services.
- **Only makes sense for frequently used assets** - Infrequently accessed assets don't benefit from CDN, so they should just be retried from the application server. 
- Apropriate cache expiry - same considerations as for caches above 
- Plan for failure - if CDN goes down it should be possible to retrieve assets directly from application server 
Handling of file file invalidation when a file has been updated on app server:
    - Use CDN API to invalidate object 
    - Use URL-parameter based versioning - if version of file exceeds the version found in cache, the latest is retrieved from app server

## Stateless web tier - simplifying automatic elastic scaling of app servers 

Sticky sessions between clients and application server instances make it hard to add and remove servers as clients cannot transfer to another server without losing the state that exists in the session on the current server 

When users have been authenticated to one server, all requests most be routed to this one or the user must login again (load balancer ensures this)

A stateless server keeps no state information. 

Moving session data from web tier to persistent storage means that all app servers use a shared data store which makes session data for each user avilable for all. 

The shared data store could be a relational database, Memcached/Redis, NoSQL,
etc. 

## Improved availability and user experience - multiple data centers 
Each DC will have a pool of webservers, associated databases and caches.
The DCs will use a shared state session data store. 
Load balancer is outside of DC and balances across these. 

DCs are used according to their geographical closeness to the user.
###Considerations:
- Redirection: Trafic must be directed to the correct datacenter. One solution is GeoDNS
- Synchronizing the users' data on failover - In a failover case where one user was connected to DC1 but is transfered to DC2, the users' data will not be found in the DC2 db/cache as the app serves within a datacenter only use the databases/caches within the same DC. One solution is to replicate data across multiple data centers (Netflix)  
- Good tooling: Test and deployment across multiple locations needs to good deployment tools. 

## Message queues

Durable, In-memory, asynchronous component for decouipling.

Producer services put messages on queue - consumer/subscriber services remove messages from queue 
Producer can put message on queue even tho consumer is unavailable. Consumer can read message even tho producer is unavailable

Producers and consumers can be scaled independently.

If a group of producers produce more messages than the consumers can keep up with (i.e. msgs accumulate on the queue) we can just add more consumers.

## Logging, metrics, automation
Logging: monitoring logs helps to identify errors and problems
- per server level 
- using tools to aggregate all logs to centralized service for easy search and viewing

Metrics: helps with business insights and the health status of the system. Useful metrics 
- Host level metrics: CPU, Memory, disk I/O, etc.
- Aggregated level metrics: performance of the entire database tier, cache
tier, etc.
- Key business metrics: daily active users, retention, revenue, etc.

Automation: Using tools to build, deploy and test to automatically handle size and complexity and speed up productivity. 

## Handling increasing data amounts - data tier scaling 

### Vertical vs. horizontal
Same principle as for application servers. 

#### Vertical downsides
- upper limit to hardware - even a maximally beefed up db server might not be enough with a lot of users 
- Risk of single point of failure 
- Costly 

### Horizontal 
Sharding separates large databases into smaller, more easily managed parts called shards.
User data is allocated to a database server based on user IDs. Anytime you access data, a hash function is used to find the corresponding shard. 

**Important to select a sharding key that uniformly distribute data.** 

#### Horizontal downsides

- **Resharding data** - when a single shard cannot hold more data or data was unevenly distributed, the sharding function muist be updated and data must be moved around. Often solved with *consistent hashing*. 
- Celebrity problem. **Shard hotspots that gets more load than others**, for instance shards that hold data for celebrity users that gets lots of reads. A single famous user might get their own shard or even multiple subshards. 
- **Join and denormalization**. Hard to perform joins when the data exists across multiple servers (what about other queries?). Commonly the database is denormalized so queries can be performed in a single table. 