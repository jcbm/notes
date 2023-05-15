# Chapter 7: Modeling workflows as pipelines

Business processes can typically be thought of as a series of document transformations. 
We can model the workflow in the same way.
ValidateOrder, PriceOrder, etc

## The Workflow Input

The input to a workflow should always be a domain object.
Thus ours start with the UnvalidatedOrder (created from some DTO)

### Commands as Input

the real input for the workflow is not actually the order form but the command (p. 13).

For order-placing workflow, we call the command PlaceOrder.
The command should contain everything that the workflow needs to process the request, which in this case is the UnvalidatedOrder.
Additionally, who created the command, the timestamp, and other metadata for logging and auditing (metadata)

### Sharing Common Structures Using Generics

As commands will typically have the same metadata, we should create a general command object. 
The FP approach for something like a shared base class would be to use generics
```
type Command<'data> = {
	Data : 'data
	Timestamp: DateTime
	UserId: string
}
``` 

We would then declare an actual type like 
```
type PlaceOrder = Command<UnvalidatedOrder>
```

### Combining Multiple Commands in One Type

Sometimes all commands for a BC are sent on the same input channel (e.g. a message queue) so we need to combine them into one data structure that can be serialized. 

We will use a choice type for this where each case is associated with a command type. 

```
type OrderTakingCommand =
	| Place of PlaceOrder
	| Change of ChangeOrder
	| Cancel of CancelOrder
```

These are mapped to a DTO and serialized/deserialized on the input channel using a routing/dispatching input stage at the edge of the BC (infrastructure ring on p. 53)

## Modeling an Order as a Set of States

An order isn't just just a static document; it transitions through a series of different states as it is processed.
How do we keep track of which state an order is currently in? 
We could add boolean flags (IsValidated, IsPriced, etc) to the order type, but it is not a good solution:
- states are implicit and lots of conditional code in order to be handled
- Some states have data that is not needed in other states; putting them all in one record complicates the design
- Not clear which fields go with which flags

It is far better to create a new type for each state. These types are already evident from the pseudo-code domain documentation described in earlier chapters.

Finally, we can create a top-level type that’s a choice between all the states:
```
type Order =
	| Unvalidated of UnvalidatedOrder
	| Validated of ValidatedOrder
	| Priced of PricedOrder
```

This is the object that represents the order in any state of the workflow and the type that can be persisted to storage or sent to other contexts.

### Adding New State Types as Requirements Change
When we use seperate types for each state, we can easily add new states without modifying existing code. 
E.g. If we need to add refunds, we can just add a RefundOrder state. 

## State Machines

Various examples of state machines p. 125 

### Why Use State Machines?
- Each state can have different allowable behavior.
-- By representing states with types we can use the type system to ensure that business rules are complied with
- All states are explicitly documented 
- Forces you to think about every possibility that could occur 

### How to Implement Simple State Machines in F# 

Create a type for each state containing only the data relevant to that state. The entire set of states can then be represented by a choice type with a case for each state.
Shopping cart example p. 127

We then create a command handler that matches on the choice type and outputs the new state, e.g. addItem for shopping carts.

## Modeling Each Step in the Workflow with Types

How to model each step of the order-placing workflow with the state machine approach.

### The Validation Step
First, we look at the validating an UnvalidatedOrder to a ValidOrder.
We translate the documentation from previously (p. 40) to code. 

We must check that the product code exists. For this we define a simple function type 
```
type CheckProductCodeExists =
	ProductCode -> bool
```

Next we must check that the address exists - we want to express through the type system that an address has not been checked yet. For now we just wrap an UnvalidatedAddress.

```
type CheckedAddress = CheckedAddress of UnvalidatedAddress
```

Our address checking function will then look like 
```
type CheckAddressExists =
UnvalidatedAddress -> Result<CheckedAddress,AddressValidationError>
```

Our final ValidateOrder then looks like 
```
type ValidateOrder =
	CheckProductCodeExists 
		> CheckAddressExists 
		-> UnvalidatedOrder 
		-> Result<ValidatedOrder,ValidationError>
```

When we use a result type as in CheckAddressExists, it ripples out to the calling flow and thus ValidateOrder will return a Result type as well.

### The Pricing Step 

Again, we look at the documentation from earlier. 

```
substep "PriceOrder" =
	input: ValidatedOrder
	output: PricedOrder
	dependencies: GetProductPrice
```

becomes 

```
type PriceOrder =
	GetProductPrice
		-> ValidatedOrder 
		-> PricedOrder
```

GetProductPrice will take a product code and return the price for that product. 
Instead of provoding PriceOrder itself with an object representing the product catalog, we abstract it away by using a function.
Nothing can go wrong here, so there's no need to return Result value.

### The Acknowledge Order Step
This step creates an acknowledgement letter and sends it to the customer. 

We will assume it is some kind of HTML
```
type HtmlString =
	HtmlString of string

type OrderAcknowledgment = {
	EmailAddress : EmailAddress
	Letter : HtmlString
}
```

The logic that generates the letter HTML is placed outside the workflow in a service function that we will provide as a dependency

```
type CreateOrderAcknowledgmentLetter =
	PricedOrder -> HtmlString
```

We also need to handle the sending. 
Initially, this would seem like a good function signature:
```
type SendOrderAcknowledgment =
	OrderAcknowledgment -> unit
```

However, we want to communicate to the overall Order-placing workflow if the acknowledgement was succesfully sent as we need to trigger an OrderAcknowledgmentSent event. 
We could use a boolean instead of unit, but their meaning is not clear. It is better to define a custom type 

```
type SendResult = Sent | NotSent
type SendOrderAcknowledgment =
	OrderAcknowledgment -> SendResult
```
The reason why we don't return Option<OrderAcknowledgmentSent> from this function directly is that it creates coupling between the service and our domain. 

The final workflow is then 
```
type AcknowledgeOrder =
	CreateOrderAcknowledgmentLetter 
		-> SendOrderAcknowledgment 
		-> PricedOrder // input
		-> OrderAcknowledgmentSent option
```
The OrderAcknowledgmentSent return value simply contains the order id and EmailAddress. It is optional as None is returned if sending fails.  

### Creating the Events to Return
In addition to the OrderAcknowledgmentSent event above, we need to create an OrderPlaced (shipping) and BillableOrderPlaced event (billing).
Defining the types is simple:
OrderPlaced is just an alias type PricedOrder 
BillableOrderPlaced is a record containing the order id, who to bill and how much.

How should we collectively return these events?
We don't want to create a record with a field for each, as we probably want to create more events and thus we must change the record type.
It is better to put them in a single list together. Thus we need to make the events have the same type as a list is restricted to a single type.
```
type PlaceOrderEvent =
	| OrderPlaced of OrderPlaced
	| BillableOrderPlaced of BillableOrderPlaced
	| AcknowledgmentSent of OrderAcknowledgmentSent
```

In the final step of the workflow, we create this list 
```
type CreateEvents =
	PricedOrder -> PlaceOrderEvent list
```

## Documenting effects 

We want to document effects in the type signature.

### Effects in the Validation Step

Check if the dependencies could return an error and whether its a remote call. 
For CheckProductCodeExists, we assume that  a local cached copy of the product catalog is available.

CheckAddressExists calls a remote service, so it has both Async and Result effects. 

These can be combined into a single type 

```
type AsyncResult<'success,'failure> = Async<Result<'success,'failure>>
```
We can then improve the method signature of CheckAddressExists

```
type CheckAddressExists =
	UnvalidatedAddress -> AsyncResult<CheckedAddress,AddressValidationError>
```

This ripples up to the containing step 
```
type ValidateOrder =
CheckProductCodeExists 
	-> CheckAddressExists
	-> UnvalidatedOrder 
	-> AsyncResult<ValidatedOrder,ValidationError list>
```

### Effects in the Pricing Step
The dependency GetProductPrice is assumed to be local catalog - no issues here.
PriceOrder step should in itself be able to return an error in case the AmountToBill is unrealistically large or negative.

```
type PricingError = PricingError of string
type PriceOrder =
	GetProductPrice
		-> ValidatedOrder 
		-> Result<PricedOrder,PricingError> 
```

## Effects in the Acknowledge Step

The dependency CreateOrderAcknowledgmentLetter is also assumed to be cached locally and wont throw an error.
SendOrderAcknowledgment does I/O, so there will be an Async effect. However, if there is an error we will ignore it, so we wont use the result type. 

```
type SendOrderAcknowledgment =
	OrderAcknowledgment -> Async<SendResult>
```

Again, this ripples up to the parent function 

```
type AcknowledgeOrder =
	CreateOrderAcknowledgmentLetter
	-> SendOrderAcknowledgment
	-> PricedOrder
	-> Async<OrderAcknowledgmentSent option>
```

## Composing the Workflow from the Steps

We have created each step in isolation and now we want to join them together into a workflow.
This means that the input of one step must match the output of the preceding step. 
We see that from the way we have defined our steps above, this is not the case. 
We need to make the functions fit to each other - a common challenge when doing type-driven design.


## Are Dependencies Part of the Design?

When implementing the individual steps, calls to other contexts were done through functions provided as dependencies. 
You can argue that how any process performs its job should be hidden - callers should not have to know what else a function needs to collaborate with. It should only know input and output.

We're gonna take a more nuanced approach:
- Public functions exposed through the API must hide dependencies from callers 
- Internal functions are explicit about dependencies

This means the top-level workflow reduces to 

```
type PlaceOrderWorkflow =
	PlaceOrder
		-> AsyncResult<PlaceOrderEvent list,PlaceOrderError>
```
For each internal step, the dependencies should be made explicit. This documents what a step actually needs. If the dependencies for a step change, then we can alter the function definition for that step, which in turn will force us to change the implementation.

## The Complete Pipeline

How are we gonna structure our workflow filewise? 
 
We will put the types for the public API in a single file - DomainApi.fs (p. 138)

### The Internal Steps

The types for the internal steps are put in a seperate file - PlaceOrderWorkflow.fs
We will later augment this file with the implementations. 

## Long-Running Workflows

We assume that calls to external services are fast, but what if they weren't?
we would then need to persist the state before calling the service, wait for the service to finish and re-load the state from storage and go the next step of the workflow. 
We would need to do this in each step of the workflow, essentially making each step a mini-workflow.

These kinds of long-running workflows are refered to as *Sagas*. 

In a more complicated system, with a high number of events, states and transitions, you may need to use a dedicated component, a Process Manager, to handle incoming messages and the determine the next action based on the current state.

