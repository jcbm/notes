
# Ch 13 - Design for Deployment

## So Many Machines

*machine* - configurable operating system instance.
- the physical host
- on a virtual machine, container, or unikernel, the given unit

*Service* - a callable interface for others to use.\
Always made up of redundant copies of software running on multiple machines.

## The Fallacy of Planned Downtime

Fundamental premise of book is that version 1.0 is the beginning of the system’s life and many future deployments are to be expected (i.e. planned downtime).

We typically design for the state of the system after a release - assumes the whole system can be changed in an instant.
But the process of updating the system takes time.

A typical design requires that the system always sees itself in either the “before” or “after” state, never “during.”\
But users experience the system in the “during” state. Even so, we want to avoid disrupting their experiences.

We should design applications to account for the act of deployment and the time while the release takes place.\
Don’t just write for the end state and leave it up to operations to figure out how to get the stuff running
Treat deployment as a feature.

key concerns: automation, orchestration, and zero-downtime deployment.

## Automated Deployments
Focus of chapter is how to design applications so that they’re easy to deploy.

The build pipeline picks up after someone commits a change to version control (some like to build every commit to master, others require a particular tag to trigger a build).

The pipeline spans both development and operations activities.\
Starts exactly like CI with steps that cover development concerns like unit tests, static code analysis, and compilation.\
CI stops after publishing a test report and an archive.\
A build pipeline goes on to run a series of steps that culminate in a production deployment (deploy code into a trial environment (real or virtual, migration scripts, and integration tests).\
Each stage of a build pipeline is looking for reasons to reject the build

Don’t try to find the best tool, but instead pick one that suffices and get good with it - Jenkins, GoCD, Spinnaker, AWS Code Pipeline

At the end of the build pipeline, the build server interacts with a configuration management tool (ch. 8).
Many options that all share some attributes.
- declarative configuration - describe desired end-state
--  live in text files so they can be version-controlled
- the tool’s job is to figure out what actions are needed to make the machine match end state

Configuration management also means mapping a specific configuration onto a host or VM.\
Either manually by an operator or automatically by the system itself.\
With manual assignment, the operator tells the tool what each host or virtual machine must do.\
The tool then lays down the configurations for that role on that host.

With automatic role assignment the operator doesn’t pick roles for specific machines.\
Instead, he supplies a configuration that says, “Service X should be running with Y replicas across these locations.”\
Suited for a platform-as-a-service infrastructure. \
It must then deliver on that promise by running the correct number of instances of the service, but the operator doesn’t care which machines handle which services.

The platform combines the requested capacity with constraints.\
Finds hosts with enough CPU, RAM, and disk while avoiding co-locating instances on hosts.\
Because the services can be running on any number of different machines with different IP addresses, the platform must also configure the network for load balancing and traffic routing.

There are different strategies for packaging and delivering the machines.\
"Convergence"
- all installation is done after booting up a minimal image
-- A set of reusable, parameterizable scripts installs OS packages, creates users, makes directories, and writes files from templates
-- install the designated application build
I.e the scripts are a deliverable and the packaged application is a deliverable

The deployment tool must examine the current state of the machine and make a plan to match the declared desired state.\
To reach the desired state, files may be need to be copied, values substituted into templates, users created, network configured, and
more.

Every tool has a way to specify dependencies among the different steps.\
Directories must exist before copying files.\
User accounts must be created before files can be owned by them, and so on.

Under the *immutable infrastructure* approach the unit of packaging is a VM or container image.\
Fully built by the build pipeline and registered with the platform.\
If the image requires any extra configuration, it must be injected by the environment at startup time.

*immutable infrastructure* proponents argue that convergence never works.\
Suppose a machine has been around a while, a survivor of many deployments.\
Some resources may be in a state the configuration management tool just doesn’t know how to repair.\
There’s no way to get from the current state to the desired state.\
A more subtle issue is that parts of the machine state aren’t even included in your configuration recipes.\
These will be left untouched by the tool, but might be radically different than you expect.\
E.g. kernel parameters and TCP timeouts.

In immutable infrastructure, you always start with a basic OS image.\
Instead of trying to converge from an unknown state to the desired state, you always start from a known state: the master OS image.\
 This should succeed every time.\
 If not, at least testing and debugging the recipes is straightforward because you only have to account for one initial state rather than the stucco-like appearance of a long-lived machine.\
 When changes are needed, you update the automation scripts and build a new machine.\
 Then the outdatedmachine can simply be deleted.

Immutable infrastructure is closely aligned with infrastructure-as-a-service (IaaS), platform-as-a-service (PaaS), and automatic mapping.\
Convergence is more common in physical deployments and on long-lived virtual machines and manual mapping.\
*Immutable infrastructure is for cattle, convergence is for pets*

## Continuous Deployment
Between the time a developer commits code to the repository and the time it runs in production, code is a pure liability.\
Undeployed code is unfinished inventory with unknown bugs.\
It may break scaling or cause production downtime.\
It might be a great implementation of a feature nobody wants.\
Until you push it to production, you can’t be sure.

The idea of continuous deployment is to reduce that delay as much as possible to minimize the liability of undeployed code.\
A vicious cycle is at play between deployment size and risk, too.

As the time from check-in to production increases, more changes accumulate in the deployment.\
A bigger deployment with more change is definitely riskier.\
When those risks materialize, the most natural reaction is to add review steps as a way to mitigate future risks.\
But that will lengthen the commit-production delay, which further increase risk.

You should run the full build pipeline on every commit.\
The final stage of the build pipeline vary.\
Some teams trigger the production deployment automatically.\
Others have a “pause” stage, where a human must provide positive affirmation that the build is good\
What you choose depends greatly on your organization’s context:\
if the cost of moving slower exceeds the cost of an error in deployment, then you’ll lean toward automatic deployment to production.\
In a safety-critical or highly regulated environment, the cost of an error may be much larger than the cost of moving slowly relative to the competition.

## Phases of Deployment
Deployment spectrum: files - archives - whole machines\
Single files with no runtime process will always be faster than copying archive files and restarting application containers.\
Copying gigabyte-sized virtual machine images and booting an operating system is slowest.

The larger the grain, the longer it takes to apply and activate.\
We must account for this when rolling a deployment out to many machines.\
A 30-minute window wont do if every machine needs 60 minutes to restart.

As we roll out a new version, both the macroscopic and microscopic time scales are at play.\
The microscopic time scale applies to a single instance (host, virtual machine, or container).\
The macroscopic scale applies to the whole rollout.

At the microscopic level
- how long does it take to prepare for the switchover?
-- mutable infrastructure: copying files into place so you can quickly update a symbolic link or directory reference
-- immutable infrastructure: time needed to deploy a new image
- how long does it take to drain activity after you stop accepting new requests?
-- may be just a second or two for a stateless microservice
-- For something like a front-end server with sticky session attachment, it could be a long time—your session timeout plus your maximum session duration
--- Bear in mind you may not have an upper bound on how long a session can stay active - especially if you can’t distinguish bots and crawlers from humans!
--- Any blocked threads in your application will also block up the drain. Those stuck requests will look like valuable work but definitely are not
--- you can watch the load until enough has drained that you’re comfortable killing the process
--- you can pick a “good enough” time limit
--- The larger your scale, the more likely you’ll just want the time limit to make the whole process more predictable.
- how long does it take to apply the changes?
-- symlink update only is very quick
-- For disposable infrastructure, there’s no “apply the change”; it’s about bringing up a new instance on the new version
--- time span overlaps the “drain” period
-- if your deployment requires you to manually copy archives or edit configuration files, this can take a while (and its more error-prone)
- once you start the new release on a particular machine, how long is it before that instance is ready to receive load?
-- Not just your runtime’s startup time
-- Many applications aren’t ready to handle load until they have loaded caches, warmed up the JIT, established database connections, and so on
-- Request to a machine that isnt ready yet will result in server errors or very long response times

The *macroscopic* time frame encompass all the microscopic ones, plus some preparatory and cleanup work.\
Preparation involves all the things you can do without disturbing the current version of the application.\
During this time the old version is still running everywhere, but it’s safe to push out new content and assets (as long as they have new paths or URLs).\
when considering a deployment as a span of time, we can enlist the application to help with its own deployment.\
Allows the application to smooth over the things that normally calls for downtime: schema changes and protocol versions.

### Relational Database Schemata
Database changes typically lead to planned downtime, especially schema changes.\
Can be avoided with some thought.

You should not be running raw SQL scripts against an admin CLI.\
You should have programmatic control to roll your schema version forward (good for testing to roll it backward as well as forward, too).\
Migrations framework like Liquibase helps apply changes to the schema, but doesn’t automatically make those changes forward- and backward-compatible.

That’s when we have to break up the schema changes into expansion and cleanup phases.

Expansion phase:
Schema changes that are safe to apply before rolling out the code:
- Add a table
- Add views
- Add a nullable column to a table
- Add aliases or synonyms
- Add new stored procedures
- Add triggers
- Copy existing data into new tables or columns

The main criterion is that nothing here will be used by the current application.\
This is the reason for caution with database triggers.\
As long as those triggers are nonconditional and cannot throw an error, then it’s safe to add them.

Triggers allow us to create “shims.”, i.e. a bit of code that helps join the old and new versions of the application.\

Suppose you have decided to split a table.
In the preparation phase, you add the new table.\
Once the rollout begins, some instances will be reading and writing the new table - others will still be using the old table.\
It’s possible for an instance to write data into the old table just before it’s shut down.\
Whatever you copied into the new table during preparation won’t include that new entity, so it gets lost.

Shims solve this by bridging the old and new structures.\
An INSERT trigger on the old table can extract the proper fields and insert them into the new table.\
An UPDATE trigger on the new table can issue an update to the old table as well.\
*You typically need shims to handle insert, update, and delete in both directions.*\
Be careful not to create an infinite loop, where inserting into the old table triggers an insert into the new table, which triggers an insert into the old table, and so on.

Half a dozen shims for each change is a lot of work.\
That’s the price of batching up changes into a release.\
You can accomplish the same job with less effort by doing more, smaller releases.

You have to test shims on a realistic sample of data, e.g. a copy of production data.\
A lot of migrations fail in production because the test environment only had QA-friendly data.\
You need to test on all the weird data that has survived years of DBA actions, schema- and application-changes.\
Do not rely on what the application currently allows.

### Schemaless Databases
A schemaless database is only schemaless as far as the database engine cares.\
Your application still expects certain structure in the documents, values, or graph nodes returned by your database.

Will all the old documents work on the new version of your application? \
Chances are your application has evolved over time, and old versions of those documents might not even be readable now.

Your database may have a patchwork of documents, created using different application versions, with some that have been loaded, updated, and stored at different points in time.\
If you try to read one now, your application will raise an exception and fail to load it.\
Whatever that document used to be, it effectively no longer exists.

How to deal with it
- write your application so it can read any version ever created
-- With each new document version, add a new stage to the tail end of a “translation pipeline”
--  If the document format has been split at some point in the past, then the pipeline must split as well
-- A lot of work
--- All the version permutations must be covered by tests, which means keeping old documents around as seed data for tests
-  write a migration routine that you run across your entire database during deployment
--  works well in the early stages, while your data is small
-- Later it can take multiple hours
-- hours of downtime for migration is unacceptable - the application must be able to read the new document version and the old version while migration is ongoing
-- New instances will be able to read both old and new format. However, old instances only know the old format
--- best to roll out the application version before running the data migration
- "Trickle then batch"
-- prefered
-- Instead of migrating all at once, add code that checks for version and converts documents as they touched by the application
---  adds a bit of latency to each request, so it basically amortizes the batched migration time across many requests
--  For documents that don’t get touched for a long time:
--- After conversion code has run in production for a while, the most active documents are updated
--- You then run a batch migration on the remainder
---- safe to run concurrently with production, because no old instances are around
-- best of both worlds:
--- allows rapid rollout of the new application version, without downtime for data migration
--- Takes advantage of our ability to deploy code without disruption so that we can remove the migration test once it’s no longer needed
-- Main restriction: you really shouldn’t have two different, overlapping trickle migrations against the same document type
--- you may need to break up larger design changes into multiple releases.
--  can be used for any big migration that would normally take too long to execute during a deployment (also non-schemaless)

### Web Assets

Web UI's has multiple assets: images, style sheets, and JavaScript files.\
Front-end asset versions are very tightly coupled to back-end application changes.\
Vital to ensure that users receive assets that are compatible with the back-end instance they will interact with.

Major concerns:
- cache-busting
- versioning
- session affinity.

*Static assets should always have far-future cache expiration headers* (ten years is good).
- Helps the user, by allowing the user’s browser to cache as much as possible
- helps your system, by reducing redundant requests

However, when we deploy an application change, we need the browser to fetch the new version.\
“Cache busting” refers to any number of techniques to convince the browser—and all the intermediate proxies and cache servers—to fetch the new hotness.

Some cache busting libraries work by adding a query string to the URL, just enough to show a new version.  The server-side application emits HTML that
updates the URL:
```
<link rel="stylesheet" href="/styles/app.css?v=4bc60406"/>
```

Static assets are often served differently than application pages.
- better to incorporate the version number into the URL or the filename instead of into a query string
- allows you to have both the old and new versions sitting in different directories
-- you can get a quick view into the contents of a single version, since they’re all under the same top-level directory

```
<link rel="stylesheet" href="/a5019c6f/styles/app.css"/>
```

Some recommend you only use version numbers for cache busting, then use rewrite rules to strip out the version portion and have an unadorned path to look up for the actual file.\
This assumes a big bang deployment and an instantaneous switchover.\
Not suitable for deployments as described in this book.

If your application and your assets are coming from the same server you might encounter that the browser gets the main page from an updated instance, but gets load-balanced onto an old instance when it asks for a new asset.\
The old instance hasn’t been updated yet, so it lacks the new assets.

Solutions:
- Configure *session affinity*
-- all requests from the same user go to the same server
-- Anyone stuck on an old app keeps using the old assets. Same with new
- Deploy all the assets to every host before you begin activating the new code
--  means you’re not using the “immutable” deployment style, because you have to modify instances that are already running

Generally easier to just serve your static assets from a different cluster.

### Rollout
Rolling out code to machines varies depending on environment and choice of configuration management tool.

“convergence” style infrastructure with long-lived machines that get changes applied to them:\
We have to decide how many machines to update at a time.\
The goal is zero downtime, so enough machines have to be up and accepting requests to handle demand throughout the process.\
I.e. we can’t update all machines simultaneously.\
If we update one machine at a time, the rollout may take an unacceptably long time.
Instead, we typically look to update machines in batches.\
You may choose to divide your machines into equal-sized groups

To update a group
- Instruct it to stop accepting new requests
- Wait for load to drain from it
- Run the configuration management tool to update code and config
- Wait for green health checks on all machines in group
- Instruct group to start accepting requests

Repeat the process for other group
Your first group should be the “canary” group
- Pause there to evaluate the build before moving on to the next group
- Use traffic shaping at your load balancer to gradually ramp up traffic to the canary group while watching monitoring for anomalies in metrics
-- big spike in errors logged?
-- increase in latency?
-- increase in RAM utilization?
-- => Shut traffic off to that group and investigate before continuing the rollout

To stop traffic from going to a machine, we can remove it from the load balancer pool.\
Abrupt - may needlessly disrupt active requests

Better to have a robust health check on the machine:\
Every application and service should include an end-to-end “health check” route.\
The load balancer can check that route to see if the instance is accepting work.\
Also useful for monitoring and debugging.\
A good health check page reports
- the application version
- the runtime’s version
- the host’s IP address
- the status of connection pools, caches, and circuit breakers

A simple status change in the application can inform the load balancer not to send any new work to the machine.\
Existing requests will be allowed to complete.\
We can use the same flag when starting the service after pushing the code.\
Often considerable time elapses between when the service starts listening on a socket and when it’s really ready to do work.\
The service should start with the “available” flag set to false so the load balancer doesn’t send requests prematurely.

“immutable” infrastructure.\
To roll code out we spin up new machines on the new version of the code.\
Key decision is whether to spin them up in the existing cluster or to start a new cluster and switch over.\
If we start them up in the existing cluster, then as the new machines come up and get healthy, they will start taking load.\
This means that you need session stickiness, or a single caller could bounce back and forth from the old version on different requests.

Starting from a new cluster the new machines can be checked for health and well-being before switching the IP address over to the new pool.\
Here we dont need session stickiness, but the moment of switching the IP address may hurt unfinished requests.

With very frequent deployments, it's better to use the existing cluster.
- avoids interrupting open connections
- more palatable choice in a virtualized corporate data center, where the network is not as easy to reconfigure as in a cloud environment

No matter how you roll the code out, in-memory session data on the machines will be lost.\
You must make that transparent to users.\
In-memory session data should only be a local cache of information available elsewhere.\
Decouple the process lifetime from the session lifetime.

After every machine runs the new code, wait a bit and keep an eye on monitoring.\
When you’re sure the new changes are good.\
Once you’re done with that grace period it’s time to undo some of our temporary changes.

### Cleanup
Cleanup after database expansions and shims

Shims:
- Can just be deleted
-- Once every instance is on the new code, those triggers are no longer necessary
- Do the deletion as a new migration

“contraction,” /tightening down the schema:
- Drop old tables
- Drop old views
- Drop old columns
- Drop aliases and synonyms that are no longer used
- Drop stored procedures that are no longer called
- Apply NOT NULL constraints on the new columns
- Apply foreign key constraints

We can only add constraints after the rollout; the old application version wouldn’t know how to satisfy them, resulting in errors.\
This breaks our principle of undetectability.\
It might be easy for you to split up your schema changes this way.
Using a migrations framework makes it easier.
- keeps every individual change around as a version-controlled asset in the codebase
- can automatically apply any change sets that are in the codebase but not in the schema

The old style of schema change relied on a modeling tool or a DBA to create the whole schema at once.\
New revisions in the tool would create a single SQL file to apply all the changes at once.\
Splitting the changes into phases this way requires more effort.\
You must model the expansions explicitly, version the model, then model the contractions and version it again.

Whether you write migrations by hand or generate them from a tool, the time-ordered sequence of all schema changes is helpful to have.\
Provides a common way to test those changes in every environment.

For schemaless databases, the cleanup phase is another time to run *one-shots* where you delete documents or keys that are no longer used or remove elements of documents that aren’t needed any more.

This cleanup phase is also a great time to review your feature toggles:\
New feature toggles should have been set to “off” by default. You should review them to see what you want to enable.\
Are there any existing toggles that you no longer need? Schedule them for removal.

## Deploy Like the Pros

In modern software development, deployments are frequent and should be seamless.\
The boundary between operations and development is non-existent.

We must design our software to be deployable, just as we design software for production.\
Not just an added burden on development team. Designing for deployment gives you the ability to make large changes in small steps.\
Rests on a foundation of automated action and quality checking.

Your build pipeline should be able to apply all the accumulated wisdom of your architects, developers, designers, testers, and DBAs.\
That goes way beyond running tests during the build.\
E.g. forgetting an index on a foreign key constraint is a common omission that causes hours of downtime:
Why would such an omission reach production?\
If you start from the premise that your build pipeline should be able to catch all mechanical errors like that, then it’s obvious that you should start specifying your schema changes in something other than SQL DDL.\
Whether you use a home-grown DSL or an off-the-shelf migration library doesn’t matter that much.\
The main thing is to turn the schema changes into data so the build pipeline can inspect schema changes.\
Then it can reject every build that defines foreign key constraints without an index.\

*Let humans define the rules.\
Let the machines enforce them.*