# Chapter 4 - Encoding and evolution


## Mixed versions of code and data models for an application may potentially be active at the same time

### Databases may contain heterogenous data

- Relational database: rigid models -  All data conforms to the current schema and must be migrated if the schema changes or have null value defaults for new fields 
- Schemaless/schema on read: flexible models - Does not enforce a schema - data written at different times with different fields can coexist

When changing the model, application code must change too to accomodate new fields when reading or writing to db, typically both client side and server side.**

### Old and new versions of code may run at the same time server side
Server-side update is often performed as a rolling update where some servers will be running on the old code while others will be running the new untill the update is completed

### Clients might be running different versions if it requires the user to execute the update 

To ensure smooth operations, compatibility must be both backward and forward. 
- Backward - new code can read data written by old code 
- Forward - older code can read data written by new code 

Backward is simple - you know the old model, so you can explicitly handle it in the new code.
Forward requires older code to be implemented to ignore unknown future additions.

Data are represented both in memory as programming language datatypes and data structures and when written out from the application is some serialized/encoded format. 

Programming languages come with functionality to encode in-memory data into byte sequences (for instance [ObjectOutputStream](https://docs.oracle.com/javase/7/docs/api/java/io/ObjectOutputStream.html) in Java) 

### Language specific-formats are problematic for multiple reasons

#### Locks you into language 
A Java serialized file for instance can't be read by another language like C# or python. If you have made the decision to serialize data in a language-based format, you are forced to work with the language as long as you store and process data using its format. 
 
#### Security issues
A serialized file can represent any object which means the decoding process can instantiate arbitrary classes. Running arbitrary code is dangerous (see [OWASP guide](https://cheatsheetseries.owasp.org/cheatsheets/Deserialization_Cheat_Sheet.html) 
 
#### No support for versioning
Not designed with forward / backward compatibility in mind which can lead to problems. 
 
 Bad performance - not designed with performance and size in mind. 
 
### XML, JSON and CSV are widely supported and generally a good choice, but have subtle issues 
 
#### Ambiguity in number handling 
 XML/CSV cant distinguish between a number and a string of digits
 JSON can't distinguish integers and floating-point numbers and lack a precision, which leads to problems with large numbers.
 
#### No support for binary data without encoding (XML/JSON)
As a workaround the data is encoded in Base64 with a schema that specifies this. 
 
#### Optional schemas in XML/JSON
interpretation of data must be hardcoded in application when not used.  
#### CSV lacks schemas so the application must define the meaning of each row and column and changes to data format must be handled manually. 
 
 JSON and XML take up a lot of space by using text encoding and named elements - in many applications this is neglible, but for big data system changing the format to something more compact can have a big impact 
 There has been made binary encodings of json and xml - some extend the datatypes with support for binary strings and distinguishing ints and floats, but otherwise preserve the JSON/XML data model, i.e. with no schema prescribed the data must still include the names of the fields. 
 
 JSON take up 1 byte pr. character. 
 
 ### Thrift and Protocol Buffers - enforced schema binary encodings based on tag numbers for fields
Independently created encodings, but very similar

Built around numbered schema definitions with field names, type and whether they are mandatory. The actual data leaves out field names, but refer to the fields in the schema by numbers. Whether a field is mandatory does not appear from the encoded data either. It is only for runtime validation using the schema.
 Each is associated with code generation tools that can generate classes from a schema definition (like JAXB in Java) 
 
Thrift consists of two encoding types:
 - BinaryProtocol
 - CompactProtocol
 
#### BinaryProtocol
 
A field in the encoding is specified in hexadecimal from a fixed set of values that specify type, field tag (i.e. what field they correspond to in the schema definition), length (if relevant for type) and end of struct. Values are encoded in UTF-8.
 
#### CompactProtocol

Takes up even less space than binary protocol.

 Same fundamental structure as binary protocol, but places field type and tag number in a single byte and uses variable length integer using the top bit of each byte to indicate if there's additional bytes for the number. 
 
#### BinaryProtocol vs. CompactProtocol
 The binary protocol is fairly simple and therefore requires less computations.
 
 The compact protocol needs less bytes to send the same data at the cost of additional processing
 
 #### ProtocolBuffer (often shortened to just Protobuf) 

 Protocol Buffer encoding is very similar to CompactProtocol encoding and takes up roughly the same space.
 
 ### Benefits of number-tag data schemas for evolving data 
 Freedom to change name of fields, as they only exist in the schema - existing data wont break as it refers to the old schema with a number only, not the fields' name.
 
 Fields with new tag numbers can be safely added:
- Forward compatibility: When adding new fields with new tag numbers to a schema, old application code using an outdated schema can simply ignore encoded fields with unknown tag numbers and use the datatype annotation to compute the next field to consider in the data. Changing existing tag numbers will break old code.
- Backward compatibility: New fields cannot be required if backward compatibility is desired. As long as we only add new fields with new numbers, code using the new schema can work with code that sends data in the old format. However, if a new field is mandatory the code will fail, as old-format data wont have it.

#### Optional fields can be safely removed
Only optional fields can be removed, as code using the old schema will not allow new data that does not have a mandatory field.
 A removed tag number cannot be reused in schema, as code with the old schema will interpret the field incorrectly, and old data will similarly be parsed incorrectly by code with the new schema.
 Changing data types is possible but truncation is a danger. Increasing precision of an int from 32bit to 64bit is safe for new code - old data will just be padded - but old code expecting 32bit data will truncate 64bit values that don't fit in 32 bits when receiving new data (i.e. backward compatibility, but not forward)

In protocol buffer schemas there is no list type, only a repeated marker that indicates that a field tag appears multiple times in a record. As a consequence, an optional field can easily be changed to a list. New code will read old data as a list with one or no elements. Old code reading new data use only the last element of the list.

### Apache Avro - schema-based binary encoding without tag numbers - writing schema is always available to readers 
 
Also schema-based binary format, but different and even more compact than Thrift and Protocol Buffer. 

Encoding is essentially just a concation of values with no indication of fields or types.

Strings have a length prefix.

ints use variable length encoding like CompactProtocol.

Thus parsing is very dependent on the schema, and use the schema to read the values one by one with the proper data type length and interpretation.

As a consequence, since data reveals no information about itself, reading and writing code must agree on the schema or reading will fail. To handle this, Avro operates with a concept of a writers schema and a readers schema, used to respectively read and write the data. These are allowed to differ, as the reader application has both schema and the Avro library will works as an adapter between them. 

The writers schema can not be included with the data as this would obviously bloat the data. Instead there's different other ways to get the writer schema to the reader efficiently (https://avro.apache.org/docs/current/#schemas).

The order of common fields in the schemas does not matter, as Avro can figure out that they're the same field from their names. 
If data has fields that are not in the readers schema, it is ignored. 
If the readers schema expects fields that are not found in the writers schema, the readers schema will set those fields to default values. 

#### Schema evolution 
Forward compatibility: writer can have a new schema and reader an old - new fields in data will be ignored, missing fields due to deletetion in new schema will be replaced with default values. 
Backward compatibility: reader can have a new schema and writer an old - new field missing in old data will be replaced with default values, deleted fields present in old data will be ignored  
 
Fields are not required to have a default value. To have forward compatibility we can only delete fields with default values. To have backward compatibility we can only add fields with default values. 
Datatypes can be changed if Avro can convert the type. 

Avro lacks explicit indicators of optionality of fields, but union types can be used to set null as a valid value, which allows null to be used as default value, making a field optional.

Field name changing is only backward compatible: To change the name of a field, a readers schema can contain aliases that can be matched to old names in the writers schema, i.e. we can translate an old writer name to a new reader name.
Adding branches to union types is only backward compatible. 

Allows for dynamically generated schemas 

Optional code generation 

#### The Merits of Schemas

- Omitted field names make them more compact than binary JSON variants
- Because the scema
required for decoding it works as an always up to date documentation of fields (manual documentation is not so trustworthy).
- By keeping a database of schemas it is possible to check forward and backward compatibility of schema changes before deployment.
- In statically typed programming languages schema code generation allows for type checking at compile time.
- Schema evolution enables the same kind of flexibility as schemaless/
schema-on-read JSON databases provide while providing better guarantees about your data and nice tooling.


### Dataflow through databases 
When applications in different versions are running at the same time, we might have an active database schema with a new field used by the new application code, but not the old. 
When the old code reads from the database it might read the object without the new field and update one of the old fields and re-write the object to db, effectively erasing the unknown value (note that this scenario is only relevant to schemaless dbs). 

#### Different values written at different times

Rewriting data into a new schema is computationally expensive. Relational databases allow simple schema changes like adding a new column with null as a default which avoids rewriting of existing data; when an old row is read the database provides null as the missing column's value. 

The document database Espresso uses Avro for storage, which allows it to use Avro’s schema evolution rules. This makes the entire database appear as if it was encoded with a single schema, even though stored data might actually be encoded with a wide variety of historical schemas. 

#### Archival storage
When taking a backup of a datbase, all the data is typically migrated to the latest schema, as you're copying the data anyway. 
Avro object container files are suitable for the purpose or [Parquet](https://parquet.apache.org/) for a column-oriented format suitable for analytics.

### Dataflow Through Services: REST and RPC

In service-oriented/microservices architecture services are independently deployable and evolvable. 
Each service should be owned by one team that should be able to release new versions of the service frequently, independent of other teams. 
In other words, we should expect old and new versions of servers and clients to be running at the same time, and so the data encoding
used by servers and clients must be compatible across versions of the service API.

REST is a design philosophy that builds upon the principles of HTTP and emphasizes simple data formats, using URLs for identifying
resources and using HTTP features for cache control, authentication, and content type negotiation.

SOAP is an XML-based protocol for making network API requests. It aims to be independent from
HTTP and avoids using most HTTP features. 
Provides its own related standards (the web service framework, known as WS-*)
SOAP API is described using the XML-based WSDL language which enables generation of local client classes that call into the server (similar to RPC).

#### RPC 

RPC model builds on a ["location transparency"](https://en.wikipedia.org/wiki/Location_transparency) philosophy where calling a remote service looks the same as calling a local method. 
However, it behaves very differently:
- A network request can suffer from all kinds of problems, get lost, be slow, etc. 
- A local function will either return a value or throw an exception. If a network request times out, we don't get a response and the client wont know what happened.
- A request might actually get processed by the server, but the response gets lost. Resending can lead to unwanted behavior if the call is not idempotent. 
- Unpredictable performance due to variable network latency 
- When calling a local method with an object it can easily be resolved from local memory. To pass the same object over the network, we need to resolve the data associated with the object and encode it. 
- Client and server does not have to use same language - RPC framework have to convert between datatypes which can lead to unexpected behavior if the types are implemented differently 

While RPC is an old technology with the problems above, it is still relevant. 

- Thrift and Avro support RPC.
- Protocol Buffer is used for gRPC.
- [Finagle](https://twitter.github.io/finagle/) uses Thrift. 
- [Rest.li](https://linkedin.github.io/rest.li/) uses JSON over http 

However, these are very explicit about the nature of RPC calls and how they are not a local call; futures are used to encapsulate asynchronous calls that may fail. 
gRPC supports streams where a call consists of a series of requests and responses, as opoosed to a single message in each direction. 

#### RPC vs REST 
The performance of binary encoding RPC protocols is superior to something like JSON over REST . REST APIs are easier to experiement with and debug as you can just deploy your service and call it without having to generate code or install anything, has wide support in popular programming languages and many specialised tools exist for it. REST is widely used for public APIs, where RPC is more typically used internally between services owned by the same organization within the same datacenter. 

#### Data encoding and evolution for RPC

it is reasonable to assume that all the servers will be updated first,
and all the clients second. i.e. you only need backward compatibility on requests,
and forward compatibility on responses

The backward and forward compatibility properties of an RPC scheme are inherited
from whatever encoding it uses

Thrift, gRPC (Protocol Buffers), and Avro RPC can be evolved according to the
compatibility rules of the respective encoding format.
SOAP builds on XML schemas - these can be evolved, but has subtle pitfalls.
RESTful APIs typically use JSON without a formally specified schema
for responses, and JSON or URI-encoded/form-encoded request parameters for
requests. Adding optional request parameters and adding new fields to response
objects are usually considered changes that maintain compatibility.

Upgrading clients is hard in RPC. Often used for communication across organizational boundaries, so the provider of a service often has no
control over its clients and cannot force them to upgrade. 

As a consequence, RPC compatibility needs to be maintained for a long time. If a compatibility-breaking
change is required, the service provider often ends up maintaining multiple versions
of the service API side by side.

No standard for how to implement API versioning (i.e., how a client can
indicate which version of the API it wants to use). In RESTful APIs you typically provide a version number in the URL or in the HTTP Accept header.
If API keys are used to identify a particular client, another option is to store
a client’s requested API version on the server.

### Asynchronous message passing 
Somewhere between RPC and databases - client request process with low
latency, but message goes via message broker intermediary instead of directly to the receiver. 

One process sends a message to a named queue or topic, and the broker ensures that the message is delivered to one or more
consumers of or subscribers to that queue or topic. There can be many producers and
many consumers on the same topic.

**One-way**: sender does not get a reply. It just put a message on a queue and moves on. However, a consumer may publish
messages to another topic or to a reply queue that is consumed by the sender of the original message.

**Encoding agnostic**: Message brokers typically don’t enforce any particular data model—a message is just
a sequence of bytes with some metadata. If the
encoding is backward and forward compatible, you have the greatest flexibility to
change publishers and consumers independently and deploy them in any order.

If a consumer republishes messages to another topic there is a risk that unknown fields get lost in the same way described earlier for databases. 


Advantages over direct calls:
- Improves system reliability: If the recipient is unavailable or overloaded, broker acts as a buffer
- No lost messages: It can automatically redeliver messages to a process that has crashed
• Decoupling of sender and receiver: Sender doesnt need to know the IP address and port number of the
recipient (simplifies cloud deployment where virtual machines often come and go).
- Broadcast: It allows one message to be sent to several recipients.
- Logically decouples sender and receiver:  the sender just publishes messages and doesn’t care who consumes them

### Distributed actor frameworks

Programming model for concurrency in a single process.

Logic is encapsulated in actors instead of working directly with threads. 
An actor has a local state that cannot be acessed by other actors.
The only way actors can interact eachothers state is by asynchronous message passing. 

Distributed actor frameworks are used to scale an application over multiple nodes, where the same message passing is used both locally and over the network (transparently encoded into a byte sequence and decoded by the receiver)

Essentially integrates a message broker and the actor programming model into a single framework.  
If you want to perform rolling updates you still have to worry about forward
and backward compatibility as messages may be sent from a node running the new
version to a node running the old version, and vice versa.

**Location transparency**: Actor model assumes that messages will get lost, even within a single process. 

#### Actor frameworks:
- [Akka](https://akka.io/) - builds on Java serialization that lacks forward/backward compatibility, but it can be replaced with protobuf, so rolling updates become possible 
- [Orleans](https://dotnet.github.io/orleans/) - custom encoding format with no support for rolling upgrades. Deployment requires deployment of a new cluster and switching all traffic from the old to the new. Custom serialization can be used (will it enable rolling updates?)
- [Erlang OTP](https://erlang.org/doc/) - designed for high availability, but hard to make changes to schemas and rolling updates are hard to do right. A JSON-like eksperimental maps datatype may change this. 