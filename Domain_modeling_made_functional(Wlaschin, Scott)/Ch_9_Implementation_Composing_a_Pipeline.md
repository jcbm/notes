
# Chapter 9: Implementation: Composing a pipeline 
Implementating the design from chapter 7.

The workflow can be thought of as a series of document transformations — a pipeline — with each step in the pipeline designed as a section of “pipe.

Stages in pipeline: 
Incoming UnvalidatedOrder must be converted to ValidatedOrder, returning an error if the validation fails:
- The ValidatedOrder must be turned into a PricedOrder by adding some extra information.
- The PricedOrder must be used to create an acknowledgment letter that is sent
- Create a set of events representing what happened and return them.

Using F# pipes, the code would look something like 
```
let placeOrder unvalidatedOrder =
	unvalidatedOrder
	|> validateOrder
	|> priceOrder
	|> acknowledgeOrder
	|> createEvents
```

First, implement each step in the pipeline as a stand-alone function, making sure that it is stateless and without side effects so it can be tested and reasoned about independently.

Next, compose these smaller functions into a single larger one.

 Some functions have extra parameters that aren’t part of the data pipeline but are needed for the implementation—we called these “dependencies.”

We explicitly indicated “effects” such as error handling by using a wrapper type like Result in the function signatures. But that means that functions with effects in their output cannot be directly connected to functions that just have unwrapped plain data as their input.(see chapter 10)

## Working with simple types
To implement simple types, follow the outline for implementing constrained types (p. 104):
For each simple type, we need:
- A create function that constructs the type from a primitive, e.g. OrderId.create will create an OrderId from a string, or raise an error if the string is the wrong format.
- A value function to extract the inner primitive value

We typically put these helper functions in the same file as the simple type
```
module Domain =
type OrderId = private OrderId of string
module OrderId =
	let create str =
...
	let value (OrderId str) = 
		str
```
## Using function types to guide the implementation
In our previous modelling, we defined a function type for each step of the workflow. 
We could simply define each function, e.g. validateOrder, with no reference to the abstract validate order type defined earlier. However, it to make it clear that we are implementing a specific function type, it is better to write the function as a value (i.e. no parameters), annotate it with the function type, and implement the body as a lambda

```
type MyFunctionSignature = Param1 -> Param2 -> Result
let myFunc: MyFunctionSignature =
	fun param1 param2 ->
...
```

This ensures that the parameters and return value is assigned the correct type and thus you get an error if using it incorrectly whereas with type inference, the compiler might assume incorrect types when used incorrectly. 

## Implementing the validation steps
The validation step takes an unvalidated order and transforms it into a proper, fully validated domain object.

The steps to create a ValidatedOrder from an UnvalidatedOrder:
 - Create an OrderId domain type from the OrderId string in the unvalidated order.
- Create a CustomerInfo domain type from the corresponding UnvalidatedCustomerInfo field in the unvalidated order.
- Create an Address domain type from the corresponding ShippingAddress field in the unvalidated order which is an UnvalidatedAddress .
- Do the same for BillingAddress and all the other properties.
Finally, when we have all the individual values, we can put them together in a record. 

We are using some helper functions to construct a domain type from an unvalidated type, such as toCustomerInfo
and toAddress. For example, toAddress will convert an UnvalidatedAddress type to the domain type Address or raise an error if the primitive values in the unvalidated address don’t meet the constraints (i.e. non-null and less than 50 characters long).

To convert any non-domain type to a domain type: for each field of the domain type, get the corresponding field of the non-domain type and use an appropriate helper function to convert the field into a domain type.

### Creating a Valid, Checked Address
Some validations require calls to external services, like toAdress. In that case we pass the service as a parameter and otherwise do like above. This corresponds to dependency injection and allows us to provide a mock service in tests. 

### Creating the Order Lines 
To validate the list of orderlines we must be able to transform a single line. Like above, we transform property by property. We use List.map to validate the full list 

p. 169 - example of "lifting" to make the function return the same type from two different cases

### Creating function adapters - predicate function to passthrough function 
In a pipeline we might need to call a validation function that checks something and if the check is true we pant to pass on the validated object to the next function. However, our function passes on the boolean, not the validated object. 

```
let toProductCode (checkProductCodeExists:CheckProductCodeExists) productCode =
	productCode
	|> ProductCode.create
	|> checkProductCodeExists
	...
```

The solution is a wrapping function that we can make completely generic and re-use for many purposes
```
let predicateToPassthru errorMsg f x =
	if f x then
		x
	else
		failwith errorMsg
```

Applied to the problem above 

```
let toProductCode (checkProductCodeExists:CheckProductCodeExists) productCode =
	let checkProduct productCode =
		let errorMsg = sprintf "Invalid: %A" productCode
		predicateToPassthru errorMsg checkProductCodeExists productCode

	productCode
	|> ProductCode.create
	|> checkProduct
```

## Implementing the rest of the steps
Nothing new in this section. Just applying the above stuff to new things. 
 Use failwith "not implemented" when implementing validation steps that you are not sure about yet. 
### Implementating the acknowledgment step 

The key step here is that we have to return a result bases on whether the acknowledgment was succesfully sent. We do this by calling the sending function as the match expression 

```
let acknowledgeOrder : AcknowledgeOrder =
.....
	match sendAcknowledgment acknowledgment with
	| Sent ->
	  let event = {
		OrderId = pricedOrder.OrderId
		EmailAddress = pricedOrder.CustomerInfo.EmailAddress
		}
	  Some event
	| NotSent ->
	  None
```
### Creating the Events
We have 3 events to return. 

```
type PricedOrder = ...
type OrderPlaced = PricedOrder
	type OrderAcknowledgment = {
	EmailAddress : EmailAddress
	Letter : HtmlString
}
type BillableOrderPlaced = {
	OrderId : OrderId
	BillingAddress: Address
	AmountToBill : BillingAmount
}
```
The OrderPlaced and AcknowledgmentSent creation has been described above, so we only need to create and additional BillableOrderPlaced event here.  The billing event should only be sent when the billable amount is greater than zero, so the createBillingEvent function returns Option of BillableOrderPlaced. 

How do we make a list that contains all the 3 different order types? We must both handle that they have different types and that some types are Optional and one isn't. OrderPlaced is mandatory, while the two others are option types.

Using the “lowest common multiple” approach (p. 158), i.e. we define a common type and we must then lift each value to the common type (p. 176 for full example).

```
type PlaceOrderEvent =
	| OrderPlaced of OrderPlaced
	| BillableOrderPlaced of BillableOrderPlaced
	| AcknowledgmentSent of OrderAcknowledgmentSent

... 
let event1 = pricedOrder |> PlaceOrderEvent.OrderPlaced
let event2Opt = acknowledgmentEventOpt |> Option.map PlaceOrderEvent.AcknowledgmentSent
let event3Opt = pricedOrder |> createBillingEvent |> Option.map PlaceOrderEvent.BillableOrderPlaced
``` 

We turn each of these into a list individually and flatten these into one single list 
```
let listOfOption opt =
	match opt with
	| Some x -> [x]
	| None -> []
	
...
let event1 = pricedOrder |> PlaceOrderEvent.OrderPlaced |> List.singleton
let event2Opt = acknowledgmentEventOpt |> Option.map PlaceOrderEvent.AcknowledgmentSent |> listOfOption
let event3Opt = pricedOrder |> createBillingEvent |> Option.map PlaceOrderEvent.BillableOrderPlaced |> listOfOption

[
yield! events1
yield! events2
yield! events3
]
```




## Composing the pipeline steps together
We want to compose a final pipeline that looks like this from all the work done above 
```
fun unvalidatedOrder ->
	unvalidatedOrder
	|> validateOrder
	|> priceOrder
	|> acknowledgeOrder
	|> createEvents
```
	
However, for this to work every function must accept a single parameter of the type that is the output of the previous function in the pipeline. 
The validateOrder function we have implemented does not simply take a single parameter but also needs a CheckProductCodeExists and a CheckAdressExists dependency. 
Similarly, the PriceOrder function takes a validatedOrder and a GetProductPrice function and AcknowledgeOrder takes a sendAcknowledgment function in addition to the pricedOrder. 

To solve this, we could use a monad-based solution, but for now we use partial application instead, where we bake in the missing dependencies to the functions before we compose the workflow. 

```
let validateOrder =
	validateOrder checkProductCodeExists checkAddressExists
```

We use shadowing to call our baked-in function the same as the original unapplied function. Alternatively, a tick can be used to show that it is a variant (let validateOrder' ...). 
We do the same thing for PriceOrder and AcknowledgeOrder. 

```
let placeOrder : PlaceOrderWorkflow =

	let validateOrder =
		validateOrder checkProductCodeExists checkAddressExists
	let priceOrder =
		priceOrder getProductPrice
	let acknowledgeOrder =
		acknowledgeOrder createAcknowledgmentLetter sendAcknowledgment

	fun unvalidatedOrder ->
		unvalidatedOrder
		|> validateOrder
		|> priceOrder
		|> acknowledgeOrder
		|> createEvents
```

This wont compile as the output of acknowledgeOrder is the AcknowledgeOrder event, not the pricedOrder. We could write an adapter function or we can take a imperative style approach

```
fun unvalidatedOrder ->
	let validatedOrder =
		unvalidatedOrder
		|> validateOrder checkProductCodeExists checkAddressExists
	let pricedOrder =
		validatedOrder
		|> priceOrder getProductPrice
	let acknowledgmentOption =	
		pricedOrder
		|> acknowledgeOrder createAcknowledgmentLetter sendAcknowledgment
	let events =
		createEvents pricedOrder acknowledgmentOption
	events
```

## Injecting dependencies
We dont do DI in functional programming as dependencies then become implicit and we emphasize doing things explicitly. 

We could use advanced techniques like “Reader Monad” and the “Free Monad", but instead, we pass all the dependencies to the top-level function, which then passes them into the inner functions, which in turn passes them down to their inner functions, and so on.

 In OOP this top-level function is generally called the composition root. We should chose the composition root somewhere before calling the placeOrder workflow because setting up the services will typically involve accessing configuration, and so on that we don't want this function to do. Instead we provide all of these as parameters to function. This makes the entire workflow is easily testable because as the dependencies can be mocked easily.
The composition root function should be as close as possible to the application’s entry point—the *main* function for console apps or the *OnStartup/Application_Start* handler for long-running apps such as web services.

Suave example p. 182

### Too many dependencies
Providing every dependency through the top level function potentially leads to many unnecessary parameters in the function call hierachy until they reach the function where they are needed.
What if we need to add a new parameter to a helper function? Do we need to modify all of the pipeline then? 
A much better approach is to set up the low-level functions outside the top-level function and then just pass in a prebuilt child function with all of its dependencies already baked in.

This approach of reducing parameters by passing in “prebuilt” helper functions is a common technique that helps to hide complexity. When one function is
passed into another function, the “interface”—the function type—should be as minimal as possible, with all dependencies hidden.

Alternatively, you could group the dependencies into a single record structure, say, and pass that around as one parameter.
## Testing dependencies
Explicitly setting up the functions with dependencies as parameters makes them easy to test without any mocking libraries.

Setting up a mock dependency is as easy as implementing a function with the right signature and providing it the function under test. 
```
let checkProductCodeExists productCode =
	true
...
let result = validateOrder checkProductCodeExists checkAddressExists
...
``` 
More extensive test example p. 185

### Testing tools 
- FsUnit wraps NUnit/ XUnit with F#-friendly syntax.
- Unquote shows all the values leading up to a test failure (“unrolling the
stack,” as it were).
- FsCheck for "Property-based” testing (as opposed to example-based)
- Expecto is a lightweight F# testing framework that uses standard functions as test fixtures instead of requiring special attributes like [<Test>] .


## The assembled pipeline 
Revisiting the steps to assembling the pipeline

- Put all the code that implements a particular workflow in the same
module, named after the workflow (e.g. PlaceOrderWorkflow.fs).
- Put all type definitions at the top of the file.
- Next, put the implementations for each step.
- At the very bottom, assemble the steps into the main workflow function.

The types of the workflow that are “public” (part of the contract with the caller of the workflow) will be defined elsewhere, perhaps in an API module, so we only need to include in this file the types that represent the internal stages of the workflow. 
