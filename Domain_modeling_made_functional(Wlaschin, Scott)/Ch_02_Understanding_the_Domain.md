# Chapter 2 - Understanding the domain

Note that a lot has been left out in the notes from this chapter as it is basically a practical example of a domain expert interview and it doesn't make sense to write it down verbatim

## Interview with a domain expert
With the commands/events approach (see chapter 1), we can have a series of short interviews, each focusing on only one workflow instead of having long meetings. 
In the first part of the interview, we want to keep it high level and focus only on the inputs and outputs of the workflow

To learn about a domain optimally you must think like an anthropologist, trying to study the user with a naive and open mind - resist the urge to jump to conclusions about anything about how the system is used. Good interviewing means doing lots of listening! 

### Understanding the Non-functional Requirements

Take a step back and discuss the context and scale of the workflow - do we need to design for massive scale or spiky traffic?
What about customer expectations? Experts or beginners? 

Typical requirements for a B2B application: needs like predictability, robust data handling, and an audit trail of everything that happens

### Understanding the Rest of the Workflow
More questions p. 27
Look out for hints for other bounded contexts and what our workflow needs from them 

### Thinking About Inputs and Outputs
The interview clearly identifies the input as an order form.

The output of a workflow should always be the events that it generates, the things that trigger actions in other bounded contexts. Thus a “completed order” or a “order acknowledgment” is a bad choice. 

The output of the workflow should be something like an “OrderPlaced” event, which is sent to the shipping and billing contexts.

## Fighting the impulse to do database-driven design
*Persistence ignorance*: Your first instinct might be to think about tables and the relationships between them (Order table, an OrderLine table,and Customer, etc). In DDD, you should not approach things this way. 
In domain-driven design we let the domain drive the design, not a database schema. You should work from the domain and model it without respect to any particular storage implementation. The concept of a “database” is not part of the ubiquitous language. The users do not care about how data is persisted.

Focus on modeling the domain accurately, without worrying about the representation of the data in a database.

## Fighting the impulse to do Class-driven-design
Similarly to the section above, you may introduce bias into the design if you think in terms of objects rather than the domain.

Letting classes drive the design can be just as bad as letting a database drive the design — you’re not listening to the requirements.

## Documenting the domain 
How should we record the identified requirements?
UML diagrams are often hard to work with and not detailed enough to capture the subtleties of the domain.

 For now, let’s just create a simple text-based language that captures the domain model
 
Workflows: document the inputs and outputs and then just use some simple pseudocode for the business logic
Data structures: use AND to mean that both parts are required, such as in Name AND Address .
 use OR to mean that either part is required, such as in Email OR PhoneNumber.
 
 Note that we are not trying to create a class hierarchy or database tables. We are just capturing the domain in a slightly structured way.
The advantage of this kind of text-based design is that it is understandable to non-programmers, which facilitates collaboration with domain experts.
## Diving deeper into the order-taking workflow
As developers, we tend to treat all requirements as equal. Businesses do not think that way. Making money (or saving money) is almost always the driver behind a development project. 
If you are in doubt as to what the most important priority is, follow the money! 

## Representing complexity in the domain model
As we have explored a single workflow, the domain model has become more complicated
Dealing with complexity early is good - we dont want surprises while implementing

### Representing Constraints

Start with the primitive values, e.g.
```
context: Order-Taking
data WidgetCode = string starting with "W" then 4 digits
data GizmoCode = string starting with "G" then 3 digits
data ProductCode = WidgetCode OR GizmoCode
...
```
I.e. model only allows one or the other.

 Is this too strict? When we're strict, we make things harder to change. But if there's no restrictions at all, there's no design. The right answer depends on the context.

It’s generally important to capture the design from the domain expert’s POV.
Validating the code formats is an important part of the validation process; the domain design should reflect this for self documentation. If we didn’t document the different kinds of product codes here as part of the model, we’d have to document them somewhere else.
If the requirements change, model is very easy to change; adding a new kind of product code only requires an extra line.

Just because the design is strict doesn’t mean that the implementation has to be. 
E.g., an automated version of the validation process might just flag a strange code for human approval rather than rejecting the whole order outright.

It is important to clarify and constrain nummeric quantities too, i.e. a expected max and min value. 

### Representing the Life Cycle of an Order

A simple model like 
```
data Order =
	CustomerInfo
	AND ShippingAddress
	AND BillingAddress
	AND list of OrderLines
	AND AmountToBill
```

doesnt capture the various stages in the order life cycle.
Not evident from the model above that the order can be unvalidated and validated or that it is only priced after validation.
To capture the phases in our domain model, both for documentation but also to make it clear that an unpriced order should not be sent to the shipping department, our we should have multiples model with  names corresponding to each phase: UnvalidatedOrder , ValidatedOrder, etc. 

This makes the design longer and more tedious to write out, but it makes everything clear and explicit.

Example p. 38

The model is now a lot more complicated than originally. However, this is reflects the complexity of how the business works. 
If our model wasn’t this complicated, we wouldn’t be capturing the requirements properly.

### Fleshing out the steps in the Workflow

the workflow can be broken down into smaller steps: validation, pricing, and so on
Each step has an input/output that we can add to each substep, we add details like dependencies, validation steps, etc 

```
substep "ValidateOrder" =
	input: UnvalidatedOrder
	output: ValidatedOrder OR ValidationError
	dependencies: CheckProductCodeExists, CheckAddressExists
	validate the customer name
	check that the shipping and billing address exist
	for each line:
	... //check x, y, z 
	
	if everything is OK, then:
		return ValidatedOrder
	else:
		return ValidationError
```


The documentation of the requirements now look a lot like actual code, but it can still be read and checked by a domain expert.