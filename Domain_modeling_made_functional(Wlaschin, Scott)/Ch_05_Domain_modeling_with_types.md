
# Ch. 5 - Domain modeling with types 

## Seeing Patterns in a Domain Model
Patterns of a typical domain:

 **Simple values** 
  Basic building blocks represented by primitive types *strings*, *ints* that are not actually strings or integers but has a deeper meaning in the domain 
  A domain expert thinks in terms of OrderId and ProductCode, not ints. 

**Combinations of values with AND** 
 Groups of closely linked data.
Typically documents or subcomponents of a document: names, addresses, orders, etc

**Choices with OR** 
Things that represent a choice in our domain - phone number *or* email address

**Workflows**
Business processes that have inputs and outputs.

### Modeling Simple Values w. “simple types”
Multiple things can be represented by an int, but they are not interchangeable and not though about as an int in the domain -  e.g. OrderId and ProductCode.
To make it clear that these types are distinct, create a “wrapper type” - a “single-case” union type
```
type CustomerId = CustomerId of int
let customerId = CustomerId 42
```
The label of the case is the same as the name of the type so you can use the same name for constructing and deconstructing it

Ensures at compile time that two simple types cannot be used in place of each other or compared by mistake.

**Using the wrapped types**
```
// construct
let customerId = CustomerId 42
// deconstruct
let (CustomerId innerValue) = customerId
```
### Avoiding Performance Issues with Simple Types
Wrapping simple types comes at a cost in both memory and efficiency which can become an issue in high performance applications.

Alternatives:

**Type aliases** 
```
type UnitQuantity = int
```
use type aliases instead of simple types to document the
domain. This has no overhead, but it does mean a loss of type-safety.

Structs 
```
[<Struct>]
type UnitQuantity = UnitQuantity of int
```
use a value type (a struct) rather than a reference
type. You’ll still have overhead from the wrapper, but when you store them
in arrays the memory usage will be contiguous and thus more cache-friendly.

Define collection of primitives rather than having a collection of simple types 
type UnitQuantities = UnitQuantities of int[]
works efficiently with raw data while preserving type-safety at a
high level.

## Modeling Complex Data

### Modeling with Record Types
Domain data structures  built from AND relationships translate directly to F# record structures: 
```
type Order = {
CustomerInfo : CustomerInfo
ShippingAddress : ShippingAddress
BillingAddress : BillingAddress
OrderLines : OrderLine list
AmountToBill : ...
}
```

### Modeling Unknown Types
in the early stages of the design process you'll only know the names of types that you need to model but not their internal structure. Untill then, you can use a placeholder type based on the exception type exn 
```
type Undefined = exn
```
Your models will then compile, but you can't actually do something with them before you replace the type with a real one 

### Modeling with Choice Types
Values that are represented as choices model directly to choice types 
```
type OrderQuantity =
| Unit of UnitQuantity
| Kilogram of KilogramQuantity
```
### Modeling Workflows with Functions
Workflows and processes translate to function types like 
```
type ValidateOrder = UnvalidatedOrder-> ValidatedOrder
```
Working with Complex Inputs and Outputs
A function has a single input and single output, but workflows might have multiple 

**Outputs**

**AND**  
If a workflow has an outputA *and* an outputB , then we can create a
record type for both
```
type PlaceOrderEvents = {
AcknowledgmentSent : AcknowledgmentSent
OrderPlaced : OrderPlaced
BillableOrderPlaced : BillableOrderPlaced
}
```

**OR**
If a workflow has an outputA *or* an outputB, a choice type is a natural choice 
```
type EnvelopeContents = EnvelopeContents of string
type CategorizedMail =
| Quote of QuoteForm
| Order of OrderForm
type CategorizeInboundMail = EnvelopeContents -> CategorizedMail
```
**Inputs**

**OR**  
When a workflow has a choice of different inputs, see above for output 

**AND**
When all inputs are required you can either pass them all individually to a function or group them in a record and pass it as a single parameter.

If the parameter is a dependency rather than a real input we want to use a separate parameter approach which allows for dependency injection 

A record type makes it clear when all inputs are always required and are strongly connected with each other.

### Documenting Effects in the Function Signature
To indicate that validation might fail, we can improve the method signature by returning a result 
```
type ValidateOrder = UnvalidatedOrder -> ValidatedOrder 
```
vs 
```
type ValidateOrder =
UnvalidatedOrder -> Result<ValidatedOrder,ValidationError list> 
ValidationError = {
FieldName : string
ErrorDescription : string
}
```
The type signature makes it clear that the function can fail and  callers will have to handle this scenario. 
Similarly, we can wrap types in async when there's async effects:
```
type ValidateOrder =
UnvalidatedOrder -> Async<Result<ValidatedOrder,ValidationError list>>
```
This can become a bit ugly, so we can create a generic type alias
```
type ValidationResponse<'a> = Async<Result<'a,ValidationError list>> 
```
The final function signature will then look like 
```
type ValidateOrder =
UnvalidatedOrder -> ValidationResponse<ValidatedOrder>
```
## A Question of Identity: Value Objects
In DDD data-types are classified based on whether they have a persistent identity or not.

*Entities* - objects with a persistent identity 
*Value Objects* - objects without a persistent identity 

Often data objects have no identity — they’re
interchangeable. For example, one instance of a WidgetCode with value “W1234”
corresponds to any other WidgetCode with value “W1234."

### Implementing Equality for Value Objects
Two value objects are the same if their fields have the same value. 

F# provides structural equality out the box - two record values (of the same type) are equal if all their
fields are equal. Two choice types are equal if they have the same choice
case and the data associated with that case is also equal.

## A Question of Identity: Entities
When entities change their identity remains the same, i.e. you remain the same person even though your address changes. 

Entities are often documents of some kind: orders,
quotes, invoices, customer profiles, product sheets, and so on. They have a
life cycle and are transformed from one state to another by various business
processes.

#### Context matters 
Whether an object is a value object or an entity can depend on context.  During manufacturing of a phone it is an entity it is given a serial number. From a sellers point of view all phones of a given model are equivalent and thus a value object, but once they're owned by someone they are again an entity. Even though you change the battery, it is still the same phone. 

### Identifiers for Entities
The stable identity of an entity is provided by an ID; each entity has a unique ID that doesn't change.  
The identifier may either be provided
by the real-world domain itself or we’ll need to create
an artificial identifier ourselves using techniques such as UUIDs, an auto-
incrementing database table, or an ID-generating service. 


### Adding Identifiers to Data Definitions
**Records (AND types)** 
For records, an identifier is simple another field.
**Choice (OR types)** 
We can put the ID in each case or outside it - inside is best

Outside:
```
type InvoiceInfo =
| Unpaid of UnpaidInvoiceInfo
| Paid of PaidInvoiceInfo
```

```
type Invoice = {
InvoiceId : InvoiceId 
InvoiceInfo : InvoiceInfo
}
```
Inside :
```
type UnpaidInvoice = {
InvoiceId : InvoiceId 
...
}
type PaidInvoice = {
InvoiceId : InvoiceId 
...
}
// top level invoice type
type Invoice =
| Unpaid of UnpaidInvoice
| Paid of PaidInvoice
```
Inside is better as we will have it available when we do pattern matching on the invoice. 

### Implementing Equality for Entities
While equality for value objects must compare all fields, for entity objects only the identifier matters. 

We can override equality (p.92), but it is often preferable to simply disallow equality testing on the object altogether using the NoEquality type annotation on the record type definition. This forces us to be explicit and directly compare the id properties to test for equality. 
```
[<NoEquality; NoComparison>]
type Contact = {
ContactId : ContactId
PhoneNumber : PhoneNumber
EmailAddress: EmailAddress
}
```
If multiple properties together defines the identity (like having ids for multiple things in a single record), they can be grouped into a single synthethic property (p.93)

### Immutability and Identity
F# uses immutability by default. 
Value objects are inherently immutable - if a value object is changed, it is a different object. 
Entities are expected to change. To handle this with immutable values, we use copying where the copy has the changed field set.
```
let initialPerson = {PersonId=PersonId 42; Name="Joseph"}
let updatedPerson = {initialPerson with Name="Joe"}
```
This means that we can't pass a record to function and change it as a side effect. We must explicitly take a record as input and have another one as output. 

## Aggregates

An aggregate is a collection of related objects that are treated as a single compo-
nent both to ensure consistency in the domain and to be used as an atomic unit
in data transactions. Other Entities should only reference the aggregate by its
identifier, which is the ID of the “top-level” member of the aggregate, known as
the “root.”

An order is an obvious entity. 
An orderline should generally be considered too - it is still the same order line, even though the quantity or price has changed over time.

If an orderline changes, the associated order should change too, semantically. And this is enforced by immutability:
- Make a copy of the line to change, 
- Make a copy of all lines and replace the changed line, 
- Insert the new list of order lines into a copy of the order object 

I.e. a change in one order line results in a new order line collection in a new order record

### Aggregates Enforce Consistency and Invariants
Aggreagates have an import role in ensuring consistency:  when one part of the aggregate is updated, other parts might also need to be updated to ensure consistency.

For instance, the aggregate root might contain a total price of the contained items. Thus if we change the quantity or price of an item, the total price must reflect this. 

The aggregate is also where any invariants are enforced. Say that you have
a rule that every order has at least one order line. Then if you try to delete
multiple order lines, the aggregate ensures there is an error when there’s only
one line left.

### Aggregate References
When modelling aggregates, one should consider if they make sense based on ripple effects. 
If a customer is associated with an order, should any changes to the customer change the order? Not likely. 
In that case, it is better to simply have the id of the customer as a property in the order record instead of the complete customer object and we will then load the customer information from the db when relevant. 

### The basic unit of persistence/transmission
Aggregates are  the basic unit
of persistence. Whole aggregates should be used whenever objects are read/written to db. 
Each database transaction should work with a single aggregate and not include multiple aggregates or cross aggregate boundaries.
When serializing an object, you should always send whole aggregates, not parts of them.

## Putting it all together - p. 98