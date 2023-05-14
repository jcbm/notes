# Chapter 13 - Evolving a Design and Keeping It Clean

It’s common for a domain model to start off clean and elegant and then as the requirements change, the model gets messy and the various subsystems become entangled and hard to test.

DDD is meant to be a continuous collaboration between developers, domain experts, and other stakeholders. 
If the requirements change, we must always start by reevaluating the domain model first, rather than just patching the implementation.

## Change 1: Adding Shipping Charges

The company wants to charge customers for shipping using a special calculation:

Shipping prices: 
Local states $5
Remote states $10
Another country $20

I.e. we must check the country. If US, we must determine if it is a local or remote state. Otherwise, we charge for another country. 

### Using Active Patterns to Simplify Business Logic
We can use Active Patterns to to turn conditional logic into a set of named choices that can be pattern-matched against to make the code more maintainable.
Seperation of concerns. 
We define a set of patterns to match each of our shipping categories

```
let (|UsLocalState|UsRemoteState|International|) address =
	if address.Country = "US" then
		match address.State with
		| "CA" | "OR" | "AZ" | "NV" ->
			UsLocalState
		| _ ->
			UsRemoteState
	else
		International
```	
Then we can use it in the shipping calculation

```
let calculateShippingCost validatedOrder =
	match validatedOrder.ShippingAddress with
	| UsLocalState -> 5.0
	| UsRemoteState -> 10.0
	| International -> 20.0
```

By separating the categorization from the business logic like this, the code becomes much clearer, and the names of the active pattern cases act as documentation as well.

The Active Pattern is only doing categorization, with no business logic.
If the categorization logic ever changes (such as having different states in “UsLocalState”), we only need to change the active pattern

### Creating a New Stage in the Workflow
We need to integrate our new shipping code with the order-placing workflow.
We could add it to the pricing, but we don't want to modify code that works or make it more complicated. 
Instead, we add a new stage to the workflow after it has been priced. 

PricedOrderWithShippingMethod example p 268

Where do we put the shipping cost in the order? Header or variant of PricedOrderLine? 

```
type PricedOrder = {
...
	ShippingInfo : ShippingInfo
	OrderTotal : Price
}
```
vs 
```
type PricedOrderLine =
	| Product of PricedOrderProductLine
	| ShippingInfo of ShippingInfo
```

The PricedOrderLine approach has the benefit that order total can be calculated from the list of order lines alone, with no additional logic needed to include fields from
the header.
The downside is that you could accidentally create multiple ShippingInfo lines and we must make sure to print the lines in the right order.

With the header approach you enforce a single shippingInfo value.

Final implementation p. 269

### Other Reasons to Add Stages to the Pipeline

Adding/removing stages is generally a good approach. As long as a stage is isolated from the other stages and conforms to the required types, you can be sure that you can add or remove it safely.
This approach makes it easy to add additional features 
- Logging, performance metrics, auditing (operational transparency)
- Authorization
- Dynamic addition/removal of stages from composition root based on configuration

## Change 2: Adding Support for VIP Customers
The business wants to add support for VIP customers that get special treatmen (free shipping, overnight delivery, etc)

We should not model the output of a business rule in the domain (such as adding a “free shipping” flag to the order). 
we should store the input to a business rule (“the customer is a VIP”) and then let the business rule work on that input. 
That way, if the business rules change, we don’t have to change our domain model.

#### Approaches:
Flag
```
type CustomerInfo = {
	...
	IsVip : bool
	...
}
```
Customer state
```
type CustomerStatus =
	| Normal of CustomerInfo
	| Vip of CustomerInfo
...

type Order = {
	...
	CustomerStatus : CustomerStatus
	...
}
```

There may be other customer statuses that are orthogonal to this, such as new versus returning customers, customers with loyalty cards, and so forth. I.e. a customer might have multiple statusus that we want to support and we cant do that in a single field. 

Thus we introduce a field specific for VIP status only:
```
type VipStatus =
	| Normal
	| Vip
type CustomerInfo = {
	...
	VipStatus : VipStatus
	...
}
```
Thus we can similarly add LoyaltyCardStatus, etc to CustomerInfo.
## Adding a New Input to the Workflow

When we update our CustomerInfo with the new VipStatus field, we will get a compile error as the field is never assigned to in existing code. 
This is a benefit to records over objects in OOP - all fields must be assigned on construction.

So we need to get VipStatus value from somewhere. We extend the UnvalidatedCustomerInfo with a similar field. The UnvalidatedCustomerInfo is itself created from a DTO, so we must add the field to the DTO as well. 
For these, we just use a string type with null if missing.

### Adding the Free Shipping Rule to the Workflow
Again, we'll just extend workflow with new stage.  

Function signature 
```
type FreeVipShipping =
	PricedOrderWithShippingMethod -> PricedOrderWithShippingMethod
```

Then we just implement it.

## Change 3: Adding Support for Promotion Codes

When placing an order, the customer can provide an optional promotion code.
If the code is present, certain products will be given different (lower) prices.
The order should show that a promotional discount was applied.

### Adding a Promotion Code to the Domain Model
We create a type (string alias) for the promotion code and add an option PromotionCode field to ValidatedOrder.
We get compiler errors as the field is not assigned previously and we must add a corresponding field to OrderDto and UnvalidatedOrder to transfer the value to ValidatedOrder.
Like for VIPStatus, this field is just a nullable string. 

### Changing the Pricing Logic
How can we accommodate the promotion code in our pricing logic?

 If the promotion code is present, we must provide a GetProductPrice function that
returns the prices associated with that promotion code
Otherwise, provide the original GetProductPrice function.

The original pricing function
```
type GetProductPrice = ProductCode -> Price
```
We are gonna create a new implementation for promotion code prices and then create a "factory" function that will return the appropriate pricing function, depending on whether a valid promotion code has been provided.

```
type GetPricingFunction = PromotionCode option -> GetProductPrice
```

However, ProductCode option is not the most clear type. Instead, we will create a new pricing type that either means that no promotion code is present or that one is. 
```
type PricingMethod =
	| Standard
	| Promotion of PromotionCode
```

We can then replace the PromotionCode field in ValidatedOrder with a PricingMethod field and the signature of GetPricingFunction becomes 
```
type GetPricingFunction = PricingMethod -> GetProductPrice
```

We then replace the original GetProductPrice step in the workflow with the GetPricingFunction and fix the resulting compiler errors.

### Implementing the GetPricingFunction
Code-heavy example p. 275
The GetPricingFunction is called with two dicts, one with the standard prices and one with the promotion prices. 
The original implementation simple looks up standard prices.

The promotion pricing function first tries to look up in the promotion prices and if that fails, it uses the standard prices.
 
Pattern matching is used to decide if the function for standard prices or promotion prices is returned.

### Documenting the Discount in an Order Line
How can we register in the order that a promotion was applied? 

Depends on whether downstream contexts needs to know about it. 
If not, we can simply add a “comment line” to the list of order lines with some text describing the discount.

We can define a new type of order line for this 
```
type CommentLine = CommentLine of string
type PricedOrderLine =
	| Product of PricedOrderProductLine
	| Comment of CommentLine
```
With PricedOrderProductLine corresponding to the old pricedOrderLine. 

Now we modify priceOrder such that whenever we use a promotion code, we add comment line
Example p. 277. 

### More Complicated Pricing Schemes

Pricing schemes tend to grow so much in complexity that they need their own BC. 
Signs that this might be necessary:

- A distinct vocabulary
- A dedicated team that manages prices
- Data specific to this context (previous purchases and usages of vouchers)
- Ability to be autonomous

### Evolving the Contract Between Bounded Contexts
With the addition of the CommentLine type in the OrderPlaced event that we're sending downstream, the downstream shipping must be aware of this, so we will have to change the OrderPlaced event.

When we add a new concept to the order-taking domain, do we really have to change the events and the DTOs and break the contract?
A good solution to this issue is to use “consumer-driven” contracts, i.e. downstream consumer decides contract. 

Let’s make a new type that provides only what the shipping context needs. All it needs is the list of products, the quantity for each one, and the shipping address. 

```
type ShippableOrderLine = {
	ProductCode : ProductCode
	Quantity : float
}
type ShippableOrderPlaced = {
	OrderId : OrderId
	ShippingAddress : Address
	ShipmentLines : ShippableOrderLine list
}
```

We then modify the PlaceOrderEvent output and return the ShippableOrderPlaced instead of the original more complex OrderPlaced. 

### Printing the Order
Now that we have simplified the event type for shipping, how can the shipping department print out the order?
The shipping department just needs something it can print, so we can precompute the information in the upstream context (i.e. order-taking) and provide a generated document (e.g. HTML or PDF) as a binary blob with the event or we could place the document in let shipping access it using the order id. 

## Change 4: Adding a Business Hours Constraint
Previous examples dealt with adding new data and behavior.
Let’s look at adding extending the workflow with new constraints.

The business has decided that the system should only be available during business hours.
We can create an adapter function for this that accepts any function as input and outputs a “wrapper” or “proxy” function that has exactly the same behavior but raises an error if called out of hours.

```
let isBusinessHour hour =
	hour >= 9 && hour <= 17

let businessHoursOnly getHour onError onSuccess =
	let hour = getHour()
	if isBusinessHour hour then
		onSuccess()
	else
		onError()
```

The original workflow takes an UnvalidatedOrder and returns a Result where the error type is PlaceOrderError.
Thus we extend PlaceOrderError with a new Error case.
```
type PlaceOrderError =
	| Validation of ValidationError
	...
	| OutsideBusinessHours
```

We can now wrap our original function 
```
let placeOrderInBusinessHours unvalidatedOrder =
	let onError() =
		Error OutsideBusinessHours
	let onSuccess() =
		placeOrder unvalidatedOrder
	let getHour() = DateTime.Now.Hour
	businessHoursOnly getHour onError onSuccess
```

We then replace placeOrder in the composition root with placeOrderInBusinessHours.

## Dealing with Additional Requirements Changes

To allow for free postage for VIPs inside the USA, we simply need to change the code in the freeVipShipping stage. 

To allow customers to split orders into multiple shipments, we need a new segment in the workflow. Then we need to change the model to turn the output of the workflow into a list of shipments to send to the shipping context as opposed to just a single one. 

To allow the customer to see shipment and payment status, we would need to look in multiple BCs. 
The best approach would probably be to create a new context (perhaps called “Customer Service”) that subscribes to events from the other BCs and update the state accordingly. 
Any queries about the status would then go directly to this context.