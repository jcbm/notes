# Chapter 14 - Handling Versions

When adding features, you must be careful to not break consuming applications.

## Help Others Handle Your Versions

A consuming application has its own development team that operates on its own schedule.\ 
You can’t force those to match your release schedule - they shouldn’t have to make a new release at the same time as yours to make your changes work.
Trying to coordinate consumer and provider deployments doesn’t scale.


### Nonbreaking API Changes
TCP specification provides a guiding principle for building robust systems from disparate providers. 

Postel’s robustness principle: *“Be conservative in what you do, be liberal in what you accept from others.”*

To make compatible API changes, consider what makes for an *incompatible* change. 

An “API” is essentially a layered stack of agreements between pieces of software. 

Some of the agreements are extremely fundamental (e.g. NetBIOS is very rarely used in place of TCP/IP).

We can assume a certain amount of commonality: IP, TCP, UDP, and DNS 
(Multicast may be allowed within some boundaries in your network, but this should only be used within a closed set of hosts. Never expect it to be routed between different networks.) 

Firmly in “layer 7,” the application layer.\
The consumer and provider must share a number of additional agreements in order to communicate.\
We can think of these as agreements in the following situations:
- Connection handshaking and duration
- Request framing
- Content encoding
- Message syntax
- Message semantics
- Authorization and authentication

With the HTTP family (HTTP, HTTPS, HTTP/2) for connection handshaking and duration, you get some of the other agreements baked in.\
E.g. “Content-Type” and “Content-Length” headers help with request framing. (determining where, in the incoming stream of bytes, a request begins and ends.)\
Both parties get to negotiate content encoding in the header of the same name.

It is not enough to specify that your API accepts HTTP.\
The HTTP specification is vast.\
When we say our service accepts HTTP or HTTPS, it usually means that it accepts a subset of HTTP, with limitations on the accepted content types and verbs, and responds with a restricted set of status codes and cache control headers.

Maybe it allows conditional requests, maybe not.\
It almost certainly mishandles range requests.\
In short, the services we build agree to a subset of the standard.

When we view communication as a stack of layered agreements, it follows that a breaking change stems from *any unilateral break from a prior agreement*.
Changes that break agreements:
- Rejecting a network protocol that previously worked
- Rejecting request framing or content encoding that previously worked
- Rejecting request syntax that previously worked
- Rejecting request routing (whether URL or queue) that previously worked
- Adding required fields to the request
- Forbidding optional information in the request that was allowed before
- Removing information from the response that was previously guaranteed
- Requiring an increased level of authorization

Notice that requests and replies are handled differently.\
Postel’s Robustness Principle creates that asymmetry.\
You might also think of it in terms of covariant requests and contravariant responses, or the Liskov substitution principle. 

*We can always accept more than we accepted before, but we cannot accept less or require more.\
We can always return more than we returned before, but we cannot return less.*\
Changes that don’t do those things are safe. 

Okay to
- require less than before
- accept more optional information than before
- return more than before the change

In terms of sets of required and optional parameters, the following changes are always safe:
- Require a subset of the previously required parameters
- Accept a superset of the previously accepted parameters
- Return a superset of the previously returned values
- Enforce a subset of the previously required constraints on the parameters

If you have machine-readable specifications for your message formats, you should be able to verify these properties by analyzing the new specification relative to the old spec.

A problem with applying the Robustness Principle:\
Documentation may say our service accepts something different from what it really accepts.\
Suppose a service takes JSON payloads with a “url” field.\
The input is not validated as a URL, but just received as a string and stored in the database as a string.\
You want to add some validation to check that the value is a legitimate URL.\
The service now rejects requests that it previously accepted. 
That is a breaking change.

The documentation said to pass in a URL.\
Anything else is bad input and the behavior is undefined. It could do absolutely anything.\
However, as soon as the service went live, its implementation becomes the de facto specification.


Common to find gaps like these between the documented protocol and what the software actually expects. Can be large, even when you think you have a strong specification.\
Use *generative testing* techniques to find these gaps before releasing the software.
Once the protocol is live, you have to live with the implementation. The Robustness Principle says we have no choice but to keep accepting the input.

A similar situation arises when a caller passes acceptable input but the service does something unexpected with it.
- e.g. an edge case in an algorithm 
- e.g. someone passed in an empty collection instead of leaving the collection element out of the input 

Whatever the cause, some behavior just happens to work.\
Again, this isn’t part of the specification but an artifact of the implementation.\
You're stuck with the behavior, even if it was something you never intended to support.\
Once the service is public, a new version cannot reject requests that would’ve been accepted before.\
Anything else is a breaking change.

You should publish the message formats via something like Swagger/OpenAPI. 
- allows other services to consume yours by coding to the specification 
- allows you to apply generated tests that will push the boundaries of the specification
-- helps finding gaps between what your spec says and what you think it says, and between what the spec says and what your implementation does. (“inbound” testing")

You should run randomized, generative tests against services you consume.\
Use their specifications but your own tests to see if your understanding of the spec is correct.(“outbound” testing - exercise your dependencies to make them act the way you expect)

Contract tests:\
exercise parts of the provider’s contract that the consumer cares about.\
With a data format spec shared across teams, as the consuming group you should wrote FIT tests that illustrate every case in the specification.\
Run test suite against the staging system from the other team.\
The act of writing the tests alone uncovers a huge number of edge cases.\
Tests act as an early warning system if the provider changes.

### Breaking API Changes
If you can't avoid a breaking change, there are some things you can do to help consumers of your service.

Version your format:
Put a version number in your request and reply message formats.\ 
Any individual consumer is likely to support only one version at a time, so this is not for the consumer to automatically bridge versions.\
This version number helps with debugging when something goes wrong.

Evolving an API example: 

HTTP gives us several options to deal with changes in an API. None are great.

#### Version in URL
Add a version discriminator to the URL, either as a prefix or a query parameter.\
The most common approach.

Advantages:
- easy to route to the correct behavior 
- URLs can be shared, stored, and emailed without requiring any special handling
- You can query your logs to see how many consumers are using each version over time
- For the consumer, a quick glance will confirm which version they are using
Disadvantage: 
- Different representations of the same entity seem like different resources - goes against REST

#### Version in header
GET: Use the “Accept” header to indicate the desired version.\
PUT/POST: Use the “Content-Type” header to indicate the version being sent.\
E.g. use the media type “application/vnd.lendzit.loan-request.v1” and a new media type “application/vnd.lendzit.loan-request.v2” for our versions. 

If a client doesn't specify a version, it gets the default (the first non-deprecated version). 

Advantage: 
- Clients can upgrade without changing routes because any URLs stored in databases will continue to work 

Disadvantages: 
- The URL alone is no longer enough
- Generic media types like “application/json” and “text/xml” are no help at all 
- The client has to know that the special media types exists, and what the range of allowed media types are. Some frameworks support routing based on media type with varying degrees of difficulty

#### Version in custom header 

Use an application-specific custom header to indicate the desired version, e.g. “api-version.” 
Advantages: 
- Complete flexibility
- Orthogonal to the media type and URL 

Disadvantages: 
- You’ll need to write routing helpers for your specific framework
- Header is "secret knowledge" that must be shared with your consumers

#### Property in request body (only PUT/POST)
For PUT and POST only, add a field in the request body to indicate the intended version. 

Advantages:
- Easy to implement - No routing needed

Disadvantage: 
- Doesn’t cover all the cases we need

*Version in the URL is preferable*. The benefits outweigh the drawbacks. 
The URL by itself is enough - all the client needs to know.\
Intermediaries like caches, proxies, and load balancers don’t need any special (read: error-prone) configuration. Matching on URL patterns is simple for operations.\
Specifying custom headers or having the devices parse media types to direct traffic is much more likely to break. 

Particularly important when the next API revision also entails a language or framework change, where you'd really like to have the new version running on a separate cluster.


No matter the approach, as the provider, you must support both the old and the new versions for some period of time.\
When you roll out the new version, it must co-exist with the old.\
Allows consumers to upgrade as they are able.\
Be sure to run tests that mix calls to the old API version and the new API version on the same entities.\
You’ll often find that entities created with the new version cause internal server errors when accessed via the old API.

If you put version in the URLs, make sure to bump all the routes at the same time.\
Even if just one route has changed, don’t force your consumers to keep track of which version numbers go with which parts of your API.

When your service receives a request, it has to process it according to either the old or the new API.\
Preference is to handle this in the controller.\
Methods that handle the new API go directly to the most current version of the business logic.\
Methods that handle the old API get updated so they convert old objects to the current ones on requests and convert new objects to old ones on responses.

## Handle Others’ Versions
When receiving requests or messages, your application has no control over the format.\
The same goes for calling out to other services.\
The other endpoint can start rejecting your requests at any time.\
A new deployment could change the set of required parameters or apply new constraints. Always be defensive.
 
Suppose a consumer sends a request to a lending service (described p.268).\
The request body represents a requester and some loan information.\
On the server the request gets dispatched to a function with some data object arguments that represent the incoming request. 
 
We can't make any assumptions about the data objects having all the right information in the right fields.\
If we’re using an automatic mapping library, we only know that the fields have the right type (integer, string, date, and so on).\
With raw JSON, you don’t even have that guarantee. 

Some banks now want to offer a better rate for borrowers with good credit, but only for loans in certain categories (e.g. avoid mobile homes in Tornado Alley).\
So you add new fields to enable this. 
 
Requester gets a new numeric field for “creditScore.”\
Loan data gets a new field for “collateralCategory” and a new allowed value for the “riskAdjustments” list. 
 
A caller may provide all, some, or none of these new fields and values.\
In some rare cases, you might just respond with a “bad request” status and drop it.\
Most of the time, however, your function must be able to accept any combination of those fields. 

What should you do 
- if the loan request includes the collateral category—and it says “mobile home”— but the risk adjustments list is missing? 
- if the credit score is missing? 
-- Do you still send the application out to your financial partners? 
-- Are they going to do a credit score lookup or will they just return an error?

A similar problem exists with calls from your service to other services.\
Your suppliers can deploy a new version at any time, too. A request that worked before may fail now.

Another argument for the contract testing approach on p. 263.
 
A common mistake in integration tests is the desire to overspecify the call to the provider. 
- set up a request
- issue the request 
-- makes assertions about the response based on the data in the original request 

Verifies how the end-to-end loop works right now, but it doesn’t verify that the caller correctly conforms to the contract, nor that the caller can handle any response the supplier is allowed to send. 

Consequently, a new release in the provider can change the response in an unexpected way, and the consumer will break.\
With this style of testing, it can be hard to provoke the provider into giving back error responses.\
We often need to resort to special flags that mean “always throw an exception when I give you this parameter.”\
Sooner or later that test code will reach production by mistake.


Better to use a style of testing where each side check its own conformance to the specification with a two-part test.\
The first part checks that requests are created according to the provider’s requirements.\
The second part checks that the caller is prepared to handle responses from the provider.\
Neither of these actually invoke the external service - strictly about testing how well our code adheres to the contract. 

We exercised the contract test before with explicit contract tests that ensure the provider does what it claims to do.\
Separating the tests into these parts helps isolate breakdowns in communication.\
It also makes our code more robust because we no longer make unjustified assumptions about how the other party behaves.\
Even if your most trusted service provider claims to do zero-downtime deployments every time, don’t forget to protect your service (ch. 5)