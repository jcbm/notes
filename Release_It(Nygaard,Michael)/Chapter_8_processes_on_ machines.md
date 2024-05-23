
# Chapter 8 - Processes on machines 

Definitions: 

Service
- Collection of processes across machines that work together to deliver a unit of functionality 
- May have processes from multiple executables (e.g., application code plus a database)
- One service may present a single IP address with load balancing behind the scenes or multiple IP addresses using the same DNS name

Instance
- An installation on a single machine (container, virtual, or physical) out of a load-balanced array of the same executable
- A service can be made of multiple different types of executables. With instances we refer to processes of the same executable, just running in multiple locations

Executable
- An artifact that a machine can launch as a process and created by a build process 
- In a compiled language, a binary
- in an interpreted language will include sources 
- Also covers shared libraries that need to be installed before execution

Process 
- An OS process running on a machine; the runtime image of an executable

Installation
- The executable
- any attendant directories
- configuration files
- other resources as they exist on a machine

Deployment
- The act of creating an installation on a machine
- Should be automated, with the deployment definition kept in source control


Deployment example:\
Transforming sources to binaries and binaries into deployments.\
Build process compiles the source code into binary executables that go into the package repository.\
As a build progresses through the pipeline, stages tag the build as having passed.\
If the build makes it all the way through, the tagged binary becomes an installation on each machine.

Runtime example: 
Each machine runs an instance of the same binary: our compiled service.\
Those instances sit behind an HAProxy load balancer with the address 10.10.128.19 bound to the DNS name loanrequest.example.com.

## Code 

### Building the Code
Great tools exist to build, house, and deploy code.\
Important rules to follow to ensure that you know exactly what goes into the code on the instance. 
- Use version control
-- Only for the code - doesn’t handle third-party libraries or dependencies very well

- Be able to build the system, run tests, and run at least a portion of the system locally 
-- build tools have to download dependencies from somewhere to the dev box 
--- The default would be to download libraries from the Internet - Convenient but unsafe 
--- One of those dependencies could be silently be replaced, either though a man-in-the-middle attack or by compromising the upstream repository
-- Even if you download dependencies from the Net to start with, you should move to a private repository ASAP 
--- Only put libraries into the repository when their digital signatures match published information from the upstream provider.
--- Handle plugins to the build system in the same way (e.g. Jenkins plugins)

- Don't do production builds from own machines 
-- Use a CI server, and have it put the binary into a safe repository that nobody else can write to

### Immutable and Disposable Infrastructure
Configuration management tools (Chef, Puppet, Ansible) are used to apply changes to running machines.\
Use scripts/playbooks/recipes to transition the machine from one state to another.\
After each set of changes, the machine will be fully described by the latest scripts

There's some problems with this approach.\
*Easy for side effects to creep in that are the result of, but not described by, the recipes:*\
E.g. suppose a Chef recipe uses RPM to install version 12.04 of a third-party package.\
That package has a post-install script that changes some TCP tuning parameters.\
Later Chef installs a newer version of the RPM, but the new RPM’s post-install changes a subset of the original parameters.\
Now the machine has a state that cannot be re-created by either the original or the new recipes.\
That state is the result of the *history* of the changes, not the latest script.

*Broken machines or scripts that only partially worked that leave the machine in an undefined state:*\
The configuration management tools put a lot of effort into converging unknown machine states into known machine states, but sometimes they fail.\
DevOps best practice says that it’s more reliable to always start from a known base image, apply a fixed set of changes, and then never attempt to patch or update that machine.\
Instead, when a change is needed, create a new image starting from the base again. (“immutable infrastructure.”, fig. p. 159 vs. p. 160)

Machines don’t change once they’ve been deployed.\
Take a container as an example. The container’s “file system” is a binary image from a repository. It holds the code that runs on the instance.\
To deploy new code, we don’t patch up the container; we build a new one, launch it and throw away the old one.
 *we can throw away the environment, piece by piece or as a whole, and start over.*
 
## Configuration
 
Production-class software has many configurable properties containing hostnames, port numbers, filesystem locations, ID numbers, magic keys, usernames, passwords, etc.\
If any of these properties are wrong, the system is broken.\
Even if the system seems to work most of the time, it could break at 1 a.m. when Daylight Saving Time kicks in.

*“Configuration” suffers from hidden linkages and high complexity* — factors leading to operator error.\
Puts the system at risk because configuration is part of the system’s user interface for the developers and operators who support it. 

**Configuration Files**
The configuration “starter kit” is a file or set of files the instance reads at startup.\
Configuration files may be buried deep in the directory structure of the codebase, possibly in multiple directories.
- basic application plumbing like API routes 
- variable environment-specific config

Because the same software runs on several instances, some configuration properties may need to vary per machine.\
Keep these properties in separate places so nobody ever has to ask, “Are those supposed to be different?”

Instance binaries should be constant across environments, but we likely need their properties to vary.\
That means the code should look outside the deployment directory to find per-environment configurations.\
These files include production database passwords and must be guarded appropriately.\
Keeping per-environment configuration out of the source tree is also avoids mistakenly commiting the production password to version control.

You should keep configurations in version control in another repo than the source code.\
Restrict access to a bare minimum of those that need to use this information.

### Configuration with Disposable Infrastructure
In image-based environments like EC2 or a container platform, configuration files can’t change per instance.\
Some of the instances will be there and gone so fast that it doesn’t make any sense to apply static configs.\
We nmust configure instances differently in this case.  

Configuration injection approaches
- at startup
- configuration service

Injecting configuration works by providing
- environment variables
- a text blob 

EC2 allows “user data” to be passed to a new virtual machine as a blob of text.\
To use the user data, some code in the image must already know how to read and parse it (properties format or JSON or YAML).

Heroku prefers environment variables.\
Thus the application code needs some awareness of its targeted deployment environment.

With a configuration service the instance code calls a known location to ask for its configuration.\
ZooKeeper and etcd are commonly used.

Beware:
Because this builds a hard dependency on the config service, any downtime is immediately a serious problem.\
Instances cannot start up when the config service is not available, but by definition we’re in an environment where instances start and stop frequently.

ZooKeeper and etcd—and any other configuration service, for that matter—are complex pieces of distributed systems software.\
They need a well-planned network topology to maximize availability, and must be managed very carefully for capacity.\
ZooKeeper is scalable but not elastic, and adding and removing nodes is disruptive.\
These services require a high degree of operational maturity and carry some noticeable overhead.\
It’s not worth introducing them to support just one application.

*Only use them as part of a broader strategy for your organization.
Most small teams are better off using injected config.*

## Transparency
To be able to monitor the state of our server machines, we must facilitate awareness by building transparency into our systems.

Transparency refers to the qualities that allow one to gain understanding of the system’s
- historical trends
- present conditions
- instantaneous state
- future projections 

Transparent systems communicate, and in communicating, they train their attendant humans.

In debugging the “Black Friday problem” we relied on component-level visibility into the system’s current behavior. 
It was possible because the system was intentionally designed with *enabling technologies*,  implemented with transparency and feedback in mind. 
Without it, we probably could’ve known that the site was slow, but not why. 

Debugging a transparent system is vastly easier, so transparent systems will mature faster than opaque ones.\
When making technical or architectural changes, you depend on data collected from the existing infrastructure.\
Good data enables good decision-making.\
In the absence of trusted data, decisions will be made for you based on arbitrary factors.

A system without transparency cannot survive long in production.
- If administrators don’t know what the system is doing, it can’t be tuned and optimized 
- If developers don’t know what works and doesn’t work in production, they can’t increase its reliability or resilience over time 
- If the business sponsors don’t know whether they’re making money on it, they won’t fund future work 

Without transparency, the system will drift into decay, functioning a bit worse with each release.\
Systems can mature well if, and only if, they have some degree of transparency.

### Designing for Transparency
Transparency arises from deliberate design and architecture.\
Must be built in from the beginning - later addition is hard/expensive.\
Visibility inside one application or server is not enough.\
Strictly local visibility leads to strictly local optimization.\
Retailer batch jobs example.\

Visibility into one application at a time can mask problems with scaling effects.\
E.g., observing cache flushes on one application server would not reveal that each server was knocking items out of all the other servers’ caches.\
Every time an item was displayed, it was accidentally being updated, therefore causing a cache invalidation notice to all other servers.\
Only with all the caches’ statistics on one page, is the problem obvious.\
Without that visibility, we would’ve added many servers to reach the necessary capacity—and each server would’ve made the problem worse.

In designing for transparency, keep a close eye on coupling.\
It’s easy for the monitoring framework to intrude on the internals of the system.\
The monitoring and reporting systems should be like an exoskeleton built around your system, not woven into it. 
In particular
- what metrics should trigger alerts
- where to set the thresholds
- how to “roll up” state variables into an overall system health status

Should be left outside of the instance itself.\
These are policy decisions that will change at a very different rate than the application code will.

### Enabling Technologies
A process running on an instance is totally opaque.\
Unless you’re running a debugger on it, it reveals little about itself.\ 
- might be working fine
- might be running on its very last thread,
- might be spinning in circles doing nothing 
Impossible to tell whether the process is alive or dead until you look at it.
The very first trick, then, is getting information out of the process. 

Enabling technologies reduce the process opacity. 
Can be classified as “white-box” or “black-box”.

Black box technology 
- sits outside the process, examining it through externally observable things 
- Can be implemented after the system is delivered, usually by operations 
- While  unknown to the system being observed, you can still do helpful things during development to facilitate the use of these tools
-- Good logging - Instances should log their health and events to a text file
-- Log-scrapers can collect these without disturbing the server process

White-box technology
- runs inside the process 
- often an agent delivered in a language-specific library
-- must be integrated during development
- have tighter coupling to the language and framework than black-box technologies
- often comes with an API that the application can call directly 
-- provides a great increase in transparency, because the application can emit very specific, relevant events and metrics 
-- comes at the cost of coupling to that provider (small price to pay for the degree of clarity it provides)


### Logging
Simple log files are one the most reliable, versatile information vehicles.\
Logging is a white-box technology - must be integrated pervasively into the source code.\
ubiquitous for a number of good reasons 
- reflect activity within an application and thus reveal its instantaneous behavior 
- persistent, so they can be examined to understand the system’s status—though that often requires some “digestion” to trace state transitions into current states
-avoids tight coupling to a particular monitoring tool or framework, then log files are the way to go
-- Every framework or tool that exists can scrape log files 
- valuable in development, where you likely dont have ops tools

Keys to successful logging:

### Log Locations
A logs directory shouldn't be under the application’s install directory.\
Log files can be large, grow rapidly and consume lots of I/O.\
For physical machines, it’s a good idea to keep them on a separate drive - lets the machine use more I/O bandwidth in parallel and reduces contention for the busy drives.

Even in a VM, it’s still a good idea to separate log files out from application code.\
The code directory needs to be locked down and have as little write permission as possible (ideally, none).\
Apps running in containers usually just emit messages on std. out, since the container itself can capture or redirect that.\
If you make the log file locations configurable, then administrators can set the right property to locate the files.\
Else, they’ll probably relocate the files anyway.\
On UNIX systems, symlinks are the most common workaround. 
This involves creating a symbolic link from the logs directory to the actual location of the files.\
There’s a small I/O penalty on each file open, but not much compared to the penalty of contention for a busy drive.\
Separate filesystems dedicated to logs can also be mounted directly underneath the installation directory.

### Logging Levels
The log file of a new system reveal whats “normal” means for that system.\
Some applications are noisy and generate a lot of errors in their logs.\
Some are quiet, reporting nothing during normal operation.\
In either case, the applications will train their humans on what’s healthy or normal.

Most developers implement logging as though they are the primary consumer of the log files.\
Operations will however spend far more time with these log files than developers will.\
*Logging should be aimed at production operations rather than development or testing.*

Anything logged at level “ERROR” or “SEVERE” should only be something that requires action on the part of operations.\
Not every exception needs to be logged as an error. A failed validation of user data is not relevant to ops.

*Log errors in business logic or user input as warnings (if at all)*.
Reserve “ERROR” for a serious system problem.
- A circuit breaker tripping to “open” is an error 
-- should not happen under normal circumstances
-- probably means action is required on the other end of the connection 
- Failure to connect to a database is an error
-- there’s a problem with either the network or the database server 

A NullPointerException isn’t automatically an error.

Debug logs should never be used in production.\
Burries real issues in noisy.\
Build process should be setup to automatically remove any configs that enable debug or trace log levels in case a commit enables by mistake. 

### Human Factors
Log files are human-readable.\
Constitute a human-computer interface and should be considered in terms of human factors.\
In a stressful situation, such as a Severity 1 incident, human misinterpretation of status information can prolong or aggravate the problem.\
Operators for the Three Mile Island reactor misinterpreted the meaning of coolant pressure and temperature values, leading them to take exactly the wrong action at every turn.\
Important to ensure that log files convey clear, accurate, and actionable information-\
If log files are a human interface, then they should also be written such that humans can recognize and interpret them as rapidly as possible.\
The format should be as readable as possible. 

### Voodoo Operations
Given a system on the verge of failure, administrators in operations have to proceed through observation, analysis, hypothesis, and action very quickly.\
If that action appears to resolve the issue, it becomes part of the lore, possibly even part of a documented knowledge base.\
Who says it was the right action, though? What if it’s just a coincidence?

"Reset required" example - A temporal connection, combined with an ambiguous, obscurely worded message, led the administrators to perform weekly database failovers during peak hours for six months.


### Final Notes on Logging
Messages should include an identifier that can be used to trace the steps of a transaction.
-user’s ID 
- a session ID
- a transaction ID
- an arbitrary number assigned when the request comes in 

When it’s time to read ten thousand lines of a log file (after an outage, for example), having a string to grep saves time.

Interesting state transitions should be logged, even if you plan to use SNMP traps or JMX notifications to inform monitoring about them.\
Logging the state transitions takes a few seconds of additional coding, but leaves options open downstream.\
The record of state transitions is also important during postmortem investigations.

### Instance Metrics
The instance itself won’t be able to tell much about overall system health, but it should emit metrics that can be centrally collected, analyzed, and visualized.\
May be as simple as periodically spitting a line of stats into a log file.\
The stronger your log-scraping tools are, the more attractive this option will be.\
Within a large organization, this is probably the best choice.

An ever-growing number of systems have outsourced their metrics collection to companies like New Relic and Datadog.\
In these cases, providers supply plugins to run with different applications and runtime environments.\
One for Python apps, one for Ruby apps, one for Oracle, one for Microsoft SQL Server, and so on.\
Small teams can get going much faster by using one of these services.\
Avoids having to devote time to the care and feeding of metrics infrastructure. 

### Health Checks
Metrics can be hard to interpret. 
It takes some time to learn what “normal” looks like in the metrics.\
For quick, easy summary information we can create a health check as part of the instance itself.\
A health check is just a page or API call that reveals the application’s internal view of its own health.\
It returns data for other systems to read (although that may just be nicely attributed HTML). 

Health checks should report at least:
- The host IP address(es)
- The version number of the runtime or interpreter (Python, JVM, .Net, Go, etc)
- The application version or commit ID
- Whether the instance is accepting work
- The status of connection pools, caches, and circuit breakers

Clients of the instance shouldn’t look at the health check directly; they should be using a load balancer to reach the service.\
The load balancer can use the health check to tell if a machine has crashed, but it can also use the health check for the “go live” transition, too.\
When the health check on a new instance goes from failing to passing, it means the app is done with its startup.


### Naming Configuration Properties
Property names should be clear to avoid “unforced errors.”
When you see a property called "hostname", how do you know which hostname to fill in?
It’s better to name the properties according to their function, not their nature. 
Don’t call it hostname just because it is a hostname - like naming a variable "integer" because it’s an integer. 
Name it "authenticationProvider" instead, so the admin knows to look for an LDAP or Active Directory host.