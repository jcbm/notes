# Chapter 6 - Integrity and consistency in the domain

As discussed in Chapter 3: trust boundaries, we must make sure that any data in a domain is valid and consistent. We want to be able to trust all data inside the BC and avoid having to write defensive code in any other place but the edges. 

Integrity/validity - a piece of data follows the correct business rules
Consistency - different parts of the domain model agree about facts

Examples p. 103

## The Integrity of Simple Values

We don't want to write in comments what the constraints of types are, we want to enforce these programmatically.

To do so, we use private constructors 

```
type UnitQuantity = private UnitQuantity of int
```

This prevents us from creating the type outside the containing module and 

```
let unitQty = UnitQuantity 1
```
will not compile.


We will then create a submodule with the exact same name as the type with a *create* function that accepts an int and returns a Result type.
This create function will enforce the constraints. 
```
module UnitQuantity =
	let create qty =
		if qty < 1 then	
			Error "UnitQuantity can not be negative"
		else if qty > 1000 then
			Error "UnitQuantity can not be more than 1000"
		else
			Ok (UnitQuantity qty)
```

As we can no longer pattern match on the type when using a private constructor, you normally also define a *value* function that unwraps the type:
```
let value (UnitQuantity qty) = qty
```
Pattern matching: 
```
match unitQtyResult with
	| Error msg ->
		...
	| Ok uQty ->
		let innerValue = UnitQuantity.value uQty
	printfn "innerValue is %i" innerValue
```

You can reduce repetition when using many constrained types by using a helper module that contains the common code for the constructors - See Domain.SimpleTypes.fs.
Signature files could be used as an alternative to private constructors.

## Units of Measure

For numeric types, we can additionally use units of measure to document requirements and ensure type-safety. 

```
[<Measure>]
type kg
[<Measure>]
type m

let fiveKilos = 5.0<kg>
let fiveMeters = 5.0<m>
```

Even though the raw value is the same, these are not interchangable.
We get a compiler error if we try to treat them as the same type: 

```
fiveKilos = fiveMeters
```
or 
```
let listOfWeights = [
fiveKilos
fiveMeters
]
```

We can augment our types with these to ensure that they are only initialized with the right unit
```
type KilogramQuantity = KilogramQuantity of decimal<kg>
```

Units of measure has no effect on run-time performance, they're only a form of annotation used by the compiler. 

## Enforcing Invariants with the Type System
An invariant is a condition that is always true. 
How can we enforce that a list always contains at least one element?

Define a custom list type and list operations like add and remove or [use a library](https://fsprojects.github.io/FSharpx.Collections/).
```
type NonEmptyList<'a> = {
	First: 'a
	Rest: 'a list
}
```

To express that an order must always have at least one OrderLine, we simply apply the NonEmptyList type: 
```
type Order = {
...
OrderLines : NonEmptyList<OrderLine>
...
}
```
Thus the requirement is documented and enforced using only the type system. 

## Capturing Business Rules in the Type System
Design guideline: “Make illegal states unrepresentable."

Assume that we store customer email adresses that must be treated differently depending on whether they're verified or not. 
- You should only send verification emails to unverified email addresses 
- You should only send password-reset emails to verified email addresses

In OOP, we often use a flag to tell the difference 
```
type CustomerEmail = {
	EmailAddress : EmailAddress
	IsVerified : bool
}
```
Bad because it is not clear when or why the flag should be set or unset - that is up to the developer to ensure. 

When domain experts talk about “verified” and “unverified” emails, you should model them
as separate things.

Naively, we could do 
```
type CustomerEmail =
	| Unverified of EmailAddress
	| Verified of EmailAddress
```
However, this still allows developers to accidentally create the wrong case, e.g. calling the Verified constructor with an unverified email adress. 

Instead, we will create a dedicated VerifiedEmailAdress type with its own private constructor and only let the verification service create verified email adresses.
```
type CustomerEmail =
	| Unverified of EmailAddress
	| Verified of VerifiedEmailAddress
```
Thus, a customer email adress can only be constructed using the Unverified case.
The only way to create the Verified case is by creating the VerifiedEmailAddress which is only possible through the verification service.

The approach with a dedicated VerifiedEmailAddress type documents the domain better than having a simplistic EmailAddress that tries to serve two roles. 
Instead we now have two distinct types with different rules around them which is what our business rules describe above.

This also means we can put limits in function signatures, so we can ensure on compile-time that functions that should operate on only verified email adresses cannot incorrectly be called with an unverified email address.
``` 
type SendPasswordResetEmail = VerifiedEmailAddress ->
```

How can we ensure that a customer must have at least an email adress or a postal address? 

Naive approach 
```
type Contact = {
Name: Name
Email: EmailContactInfo
Address: PostalContactInfo
}
```
This suggests that both are needed. 
With Option we can indicate that something is optional:
```
type Contact = {
Name: Name
Email: EmailContactInfo option
Address: PostalContactInfo option
}
```
However, we allow a scenario where neither email or address has been provided which is not the business rule. This means we would have to check on run-time to ensure this case doesn't occur.

Again, the optimal solution is to create dedicated types. 

```
type BothContactMethods = {
	Email: EmailContactInfo
	Address : PostalContactInfo
}
type ContactInfo =
	| EmailOnly of EmailContactInfo
	| AddrOnly of PostalContactInfo
	| EmailAndAddr of BothContactMethods
	
type Contact = {
	Name: Name
	ContactInfo : ContactInfo
}
```
The type system ensures on compile-time that we can only be in the states that the business rules dictate and it is explicit what contact info a customer can provide.


### Making Illegal States Unrepresentable in Our Domain
Similarly to email adresses above, the domain operates with validated and unvalidated postal addresses. For these we can similarly create two distinct types and restrict creation to an address validation service. 

Since an unvalidated order has an unvalidated address and a validated order has a validated, we must also split the Order type into an UnvalidatedOrder and a ValidatedOrder with each their type of address.

Again enforcing rigidly that you can only do things according to the business rules; you cannot send a ValidatedOrder with an unvalidated postal address. We have to go through the postol address validation service if we want to create a ValidatedOrder.

## Consistency
Consistency places a large burden on the design and can be costly.
During requirements gathering, a product owner will often ask for a level of consistency that is undesirable and impractical. 
The need for consistency can often be avoided or delayed.

Consistency and atomicity of persistence go hand in hand: It makes no sense to ensure at the application level that an order is
internally consistent if the order is not going to be persisted atomically.

### Consistency Within a Single Aggregate
An aggregate acts both as a consistency boundary and as a unit of persistence.

If we require that the total amount for an order should be the sum of the individual lines, we can either calculate the sum from the order lines when needed (application code or SQL query) or we can compute it as a seperate value and persist it with the order. 
If we insist on storing the total as a seperate value, we need to ensure that it stays consistent with the order lines. 

The only component that “knows” how to preserve consistency is the top-level Order; the aggregate that enforces a consistency boundary. Thus we should do all updates at the order level.
This means that an update to an order line should affect the order as a whole.

To update an order line price (see p.114 for code): 
- iterate over all the lines until we find one matching the id
- create a new record with the updated price (using *with* to copy the record)
- recreate the list of order lines, replacing the old record with the new 
- create a new order with the new order lines list and compute the total again 

if we save this order to a database, we must ensure that the order header and the order lines are
all inserted or updated in a single transaction.

### Consistency Between Different Contexts
How can we coordinate consistency across multiple isolated BCs? 
We must keep each bounded context isolated and decoupled.

When order is placed in the Order BC, an invoice must be created in the Billing BC. 
What if we succesfully create the order but invoice creation fails? 

We can try an approach based on calling Billing context's API

Ask billing context to create invoice
If successfully created:
	create order in order-taking context

However, the final order creation might fail and now billing is out of sync.  
In theory, two-phase commit can be used to keep seperate systems in sync. 
However, there's many issues with 2-PC.

Instead we will take an error-tolerant approach: 
Occasionally, things will go wrong, but dealing with rare errors is a cheaper solution than keeping everything in sync.

Instead of requiring an invoice be created immediately, we just send a message to billing domain and continue with the rest of the order processing.

What do we do if no invoice is created?
- **Nothing**
-- Customer gets stuff for free.
-- Adequate solution if errors are rare and the costs are small 
- **Detect loss and resend**
-- Typical reconciliation process: compare the two sets of data, and if they don’t match up, fix the error
- **Compensating action**
-- Undo previous action or fix error
-- For the case above, cancel the order and ask the customer to return the  items 
-- Other times a compensating action might be used to do things such as correct mistakes in an order or issue refunds.

If we strive for consistency, we must implement the second or third option.
Only ensures *eventual* consistency

### Consistency Between Aggregates in the Same Context
How can we keep aggregates in sync within a single BC? 

In general, you should only update one aggregate per transaction.
If more than one aggregate is involved, we should use messages and eventual consistency as described above, even within the same BC.
Sometimes it it might be worth including all affected entities in the transaction tho. E.g. the workflow is considered by the business
to be a single transaction.

Money transfer example:
If we need to transfer X dollars from account A to account B, A must be decreased with X, B must be increased with X.  

If the accounts are represented by an Account aggregate, then we would be updating two different aggregates in the same transaction.
Provides a clue that you can refactor to get deeper insights into the domain. 
A transaction would likely be identified with an id, which implies that it’s a DDD Entity in its own right.

```
type MoneyTransfer = {
	Id: MoneyTransferId
	ToAccount : AccountId
	FromAccount : AccountId
	Amount: Money
}
```

You shouldn’t reuse aggregates if it doesn’t make sense to do so. If you need to define a new aggregate for a single use case, go ahead.

### Multiple Aggregates Acting on the Same Data

How can we ensure that constraints are enforced consistently if we have multiple aggregates that act on the same data
Often, when we express constraints with types, they can easily be shared between multiple aggregates.  
The requirement that an account balance never be below zero could be modeled with a NonNegativeMoney type or using a shared validation functions

Advantage of FP over OOP: validation functions are not attached to objects or rely on global state