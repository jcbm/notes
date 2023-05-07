
# Chapter 12: Persistence
## Pushing persistence to the edges 

We would ideally like all functions to be “pure,” which makes them easier to reason about and test.
When designing our workflows, we want to avoid any kind of I/O or persistence-related logic inside the workflow as this is not pure. 

This generally means separating workflows into two parts:
• A domain-centric part that contains the business logic only 
• An “edge” part that contains the I/O-related code

We sandwich the domain-centric part between reading and writing to the db. Our workflow is provided with the data from the db as a parameter and returns a choice result that we can match on later use to decide what to write to db (p.240). 

Composite functions that use I/O like this should of course be located at the
top level of the application—either in the “composition root” or in a controller.

### Making Decisions Based on Queries
Sometimes you need to look in the db during the workflow. Instead of poluting pure functions with database logic, we sandwich database-calling functions between the pure ones containing the business logic.

If there's two many layers alternating between I/O and pure functions, it might be better to split the workflow into shorter mini-workflows (p.140). 

### Where’s the Repository Pattern?

The Repository pattern is a nice way of hiding persistence in an object-oriented design that relies on mutability. 
When modeling everything as functions and pushing persistence to the edges, the Repository pattern is not needed.
This benefits maintainability as instead of having a single I/O interface with many methods, most of which we don’t need to use in a given workflow, we define a distinct function for each specific I/O access and use them only as needed.

## Command-query separation

Command-query separation is a design principle stating that code that returns data (“queries”) should not be mixed with code that updates data (“commands”).
Applied to FP:
- Functions that return data should not have side effects.
- Functions that have side effects (updating state) should not return data (should return unit)

Database operations
```
type InsertData = DbConnection -> Data -> Unit
type ReadData = DbConnection -> Query -> Data
type UpdateData = DbConnection -> Data -> Unit
type DeleteData = DbConnection -> Key -> Unit
```

DbConnection type is specific to a particular data store, so we’ll want to hide
this dependency from the callers using partial application 

```
type InsertData = Data -> Unit
...
```

we’re dealing with I/O with possible errors, so the signatures need to include some effects. It’s common to create an alias such asDataStoreResult or DbResult that includes the Result type and possibly Async

```
type DbError = ...
type DbResult<'a> = AsyncResult<'a,DbError>
type InsertData = Data -> DbResult<Unit>
...
```

### Command-Query Responsibility Segregation

It is tempting to re-use the same object types for reading and writing to/from db.

```
type SaveCustomer = Customer -> DbResult<Unit>
type LoadCustomer = CustomerId -> DbResult<Customer>
```

This is a bad idea. 
- Different data : 
-- Data returned by the query is often different than what is needed when writing;
--- A query might return denormalized data or calculated values, but these wouldn’t be used when writing data. 
--When creating a new record, fields such as generated IDs or versions wouldn’t be used, yet would be returned in a query. 
- Creates coupling: queries and commands tend to evolve independently and therefore shouldn’t be coupled. Over time you may need three or four different queries on the same data, with only one update command. If the query type and the command type are forced to be the same it can get awkward to work with.
 - Queries may return multiple different entitites: some queries may need to return multiple entities at once for performance reasons. For example, when you load an order, you may also want to load the customer data associated with that order, rather than making a
second trip to the database to get the customer. When you are
saving the order to the DB, you would use only the CustomerId rather than the entire customer.

Rather than trying to make one data type serve multiple purposes, it’s better to design each data type for a single case.  This separation of query types and
command types leads naturally to a design where they are segregated into
different modules so that they are truly decoupled and can evolve indepen-
dently. One module would be responsible for queries (known as the read
model) and the other for commands (the write model), hence command-query
responsibility segregation or CQRS.

if we wanted to have separate read and write models for a cus-
tomer, we might define a WriteModel.Customer type and a ReadModel.Customer type,

The data access functions would look like:
```
type SaveCustomer = WriteModel.Customer -> DbResult<Unit>
type LoadCustomer = CustomerId -> DbResult<ReadModel.Customer>
```
### CQRS and Database Segregation
When applying CQRS to databases  you would have two different data stores, one optimized for writing (no indexes, transactional, and so on), and one optimized for queries (denormalized, heavily
indexed, and so on).

This could be done in a single database. In a relational database, for example, the “write” model could simply be tables and the “read” model could be predefined views on
those tables. 
You could have physically distinct data stores, but you must consider whether the design benefits of separate data stores are worth it. Added cost of *eventual consistency* as the read store receives all writes with a delay. 

If you segregate the reads and writes, you then have the flexibility to use many distinct read stores, each of which is optimized for a certain domain. In particular, you can have a read store that
contains aggregated data from many bounded contexts, which is very useful for doing reports or analytics.

### Event sourcing 
CQRS is often associated with event sourcing. 
In event sourcing we persist every state change instead of just the current change, similar to version control. 
To restore the current state at the beginning of a workflow, all the previous events are replayed.
This approach lends itself well to auditing. 

## Bounded contexts must own their data storage 
To archieve isolation between bounded contexsts, so we are able to evolve them independently, we must adhere to 
- BC must own its own data storage and associated schemas
- BC can change them at any time without having to coordinate with other bounded contexts.
- No other system can directly access the data owned by the BC
- Other systems should either use the public API of the bounded context or use some kind of copy of the data store.

If two BCs accesses the same data store, we still couple the systems even tho they run completely independent code bases. 

At one extreme, each BC might have a physically distinct database or data store that is deployed completely separately from all the others. 
At the other extreme, all the data from all contexts could be stored in one physical database (making deployment easier) but use some kind of namespace mechanism to keep the data for each context logically separate.
### Working with data from multiple domains 

When doing reporting or business analytics we need to access data from multiple context, which we shouldnt do ([Bounded contexts must own their data storage](##bounded-contexts-must-own-their-data-storage)).
Instead we must treat those as their own domain and copy the data owned by the other BCs to here. 
This allosw the source systems and the reporting system to evolve independently, so each can be optimized for its own concerns. 

To get the data here, we can subscribe to events emited from other systems. This has the advantage that the Business Intelligence context is just another domain and does not require any special treatment in the design.
Alternatively, we can use ETL processing to copy the data. This is easier to implement initially, but it may impose extra maintenance since it’ll probably need to be altered when the source systems change their database schemas.

Within the Business Intelligence domain, there’s very little need for a formal domain model. It’s probably more important to develop a database that efficiently supports ad hoc queries and different access paths.

We can approach operations with a similar approach as described above and have a seperate domain for logging, metrics and other data for analysis and reporting. 
## Working with document databases

We use the techniques from [Chapter 11: Serialization](Ch_11_Serialization.md) to convert a domain object into a DTO and then into a JSON string (or other dataformat) and then store and load it through the API of the storage system

Example p.250

## Working with relational databases 
“impedance mismatch” - relational model in db is very different from code model.
Data models developed using functional programming principles tend to be more compatible with relational databases, as FP models do not mix data and behavior.

Tables in relational databases correspond nicely to collections of records in the functional model. And the db set-oriented operations (SELECT, WHERE) are similar to FP list-oriented operations(map, filter). 
The strategy is to use the serialization techniques ([Chapter 11: Serialization](Ch_11_Serialization.md)) to design record types that can be mapped directly to tables.
Example p.251 

However, relational databases only support primitives such as strings or ints, so we will have to unwrap our domain types.
Additionally, relational tables dont map nicely to choice types 

### Mapping Choice Types to Tables
If we think of choice types as a one-level inheritance hierarchy, then we can use [some of the approaches used to map object hierarchies to the relational model](http://www.agiledata.org/essays/mappingObjects.html).
Example p. 252

**"all cases in one table" approach (similar to approach discussed on p. 232)**
use just one table to store all the data from all the cases,
- we’ll need flags to indicate which case is being used and 
- there will have to be NULLable columns that are used for some of the cases.

We have used a bit field flag for each case rather than a single Tag VARCHAR field because it’s slightly more compact and easier to index.

**"each case has its own table" approach**
- Create child table for each case in addition to the main table.
- All tables share the same primary key.
- Main table stores the ID and some flags to indicate which case is active
- The child tables store the data for each case. 

In exchange for more complexity, we have better constraints in the database (such as NOT NULL columns in the child tables). 
Example p.253
This approach might be better when the data associated with cases are very large and have little in common. Most of the time the one table approach is better. 

### Mapping Nested Types to Tables
Handling nested types
-  If the inner type is an DDD Entity, with its own identity, it should be
stored in a separate table.
-  If the inner type is a DDD Value Object, without its own identity, it should
be stored “inline” with the parent data.

Example p.254

### Reading from a Relational Database
We generally avoid ORM in F# and instead work in raw SQL.
Typically with a F# SQL type provider as it has the benefit that it creates types that match the SQL queries or SQL commands at compile time;
If the SQL query is incorrect, you get a compile time error. 
If the SQL is correct, an F# record type gets generated that matches the output of the SQL code exactly.
Example p. 255

As we did with the serialization examples in the previous chapter, we should create a toDomain function. This one validates the fields in the database and then assembles them using a result expression, i.e. we must treat the database as untrusted and return a Result type. 

If database returns a null value the type provider will use an option type for the field, but our domain object constructor might only accept the raw type. We can use a helper function to unwrap such values 
```
let bindOption f xOpt =
	match xOpt with
	| Some x -> f x |> Result.map Some
	| None -> Ok None
```

Writing a custom toDomain function like this and working with all the Results is a bit complicated, but once written, we can be sure that we’ll never have unhandled errors.
Alternatively, if we’re sure that the database will never contain bad data and we’re willing to panic if it does, then we can throw exceptions for invalid data instead. 
Then we can change the code to use a panicOnError helper function (that converts error Results into exceptions), which in turn means that the output of the toDomain function is a plain Customer without being wrapped in a Result.
Example p.256

No matter what we do, we have three cases to handle: no record found, exactly one record found, and more than one record found. 
We need to decide which cases should be handled as part of the domain and which should never happen (and can be handled as a panic). 
Example p. 257

The benefit of this is that possible errors are explicitly handled and documented in in the code. 


Code can be made cleaner by parametizing everything variable into a general function where the table name, ID, records, and toDomain converter are all passed as parameters:

```
let convertSingleDbRecord tableName idValue records toDomain =
	match records with
	| [] ->
		let msg = sprintf "Not found. Table=%s Id=%A" tableName idValue
		Error msg 
	| [dbRecord] ->
		dbRecord
		|> toDomain
		|> Ok 
	| _ ->
		let msg = sprintf "Multiple records found. Table=%s Id=%A" tableName idValue
		raise (DatabaseError msg)
```

### Reading Choice Types from a Relational Database
Let’s say that we’re using the one-table approach to store ContactInfo records and we want to read a single ContactInfo by id 
- Define query as a type (p. 259)
- Create a domain function (p. 259)
- For nullable types, use Result.ofOption to convert the Option into a Result

Writing this type of code is tedious, but if you want to ensure the integrity of your domain (validate email addresses, order quantities, etc) and deal with nested choice types, ORM is not an option. 

###  Writing to a Relational Database
Same pattern as reading: 
- Convert the domain object to a DTO 
- Execute an insert or update command.

Simply let the SQL type provider generate a mutable type that represents the structure of a table, and then we just set the fields of that type.
Example p. 261

An alternative approach with more control is to use handwritten SQL statements. For example, to insert a new Contact , we first define a type representing a SQL INSERT statement
Example p. 261

### Transactions
Often when saving we need atomicity.
Some data stores support transactions between multiple calls as part of their API, i.e. by first calling BegingTransaction and Commit when finished (example p. 262).
Other data stores only support transactions as long as everything is done in a single connection. 

Sometimes, though, you are communicating with different services, and there’s no way to have a cross-service transaction.
Generally, you dont require transactions across different systems because the overhead and coordination cost is too heavy and slow.
It is better to assume that most of the time things go well and have a *reconciliation processes* to detect inconsistency and
*compensating transactions* to correct errors

Example p. 263