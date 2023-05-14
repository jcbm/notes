# Chapter 4 - Understanding types 

Converting informal requirements into compilable code.
Note: This chapter is generally specific to F#, not FP languages in general.

## Understanding Functions

A function can be seen as a blackbox with an input and an output.

### Type Signatures
The *let* keyword is used to define a function
Parameters are separated by spaces, without parentheses or commas
No *return* keyword; The last expression of the function is the output of the function.

You rarely need to explicitly declare what input/output types, as the compiler will generally be able to infer types automatically.
Functions consisting of more than one line are written with an indent.
 You can define subfunctions within a function
 
### Functions with Generic Types

Functions that can work with any type, will automatically be assigned a generic type by the compiler; A tick-then-letter is F#’s way of indicating a generic type, i.e. 'a.

## Types and Functions

A type is just the name given to the set of possible values that can be used as inputs or outputs of a function; e.g. the set of numbers in the range -32768 to +32767 can be given the label int16 . There is no special meaning or behavior to a type beyond that.

A function with the signature int16 -> string thus accepts all values in the range above and has an output in the set of all possible strings. 

In FP, functions are values, so we can use sets of functions as a type as well.
```
someInputType -> (Fruit -> Fruit)
```
“Values” vs. “Objects” vs. “Variables”
In FP we operate in values rather than variables or objects 
Value -  a *member of a type*, something that can be used as an input or an
output.

Functions can be used as values.
Values are immutable (in contrast to variables).
Values are just data with no behavior attached.

Object - an encapsulation of a data structure and its associated behavior (methods)
All operations that change the internal state must be provided by the object itself.

## Composition of Types 

In FP, composition is used to build new functions from simple functions and new types from smaller types

New types are built from smaller types in by ANDing or ORing. 

### “AND” Types

Records - each property has a value (product type).
To make fruit salad you need an apple *and* a banana *and* some cherries:

```
type FruitSalad = {
	Apple: AppleVariety
	Banana: BananaVariety
	Cherries: CherryVariety
}
```

### “OR” Types
Choice type (sum types, tagged unions, discriminated unions).
For a fruit snack you need an apple *or* a banana *or* some cherries:

```
type FruitSnack =
	| Apple of AppleVariety
	| Banana of BananaVariety
	| Cherries of CherryVariety
	
```
i.e.  a FruitSnack is either an AppleVariety (tagged with Apple) or a BananaVariety (tagged with Banana) or a CherryVariety (tagged with Cherries)

The varieties of fruit are themselves defined as OR types
```
type AppleVariety =
	| GoldenDelicious
	...
type BananaVariety =
	| Cavendish
	...
type CherryVariety =
	| Montmorency
	...
```

### Simple Types

We will often define a choice type with only one choice.
Easy way to create a “wrapper”—a type that contains a primitive as an inner value.

```
type ProductCode =
	| ProductCode of string
```
which simplifies to 
```
type ProductCode = ProductCode of string
```

### Algebraic Type Systems

In an algebraic type system is every compound type is composed from smaller types by ANDing or ORing them together

## Working with F# Types

The way that types are defined and the way that they are constructed are very similar.
### Records
To define a record type, we use curly braces and then name:type definitions for each field
The construction looks identical except that we use = in place of : to assign values to the names. 

```
type Person = {First:string; Last:string}
let aPerson = {First="Alex"; Last="Adams"}
```

To deconstruct a value of this type using pattern matching, we put the record before the equals.
```
let {First=first; Last=last} = aPerson
```

Dot-notation can be used to access the indivual fields of the record 
```
let first = aPerson.First
```
### Choice types 
To define a choice type, we use the vertical bar to sepa-
rate each choice, with each choice defined as caseLabel of type

```
type OrderQuantity =
	| UnitQuantity of int
	| KilogramQuantity of decimal
```
To construct a choice type, a case label is used as constructor function with the associated value passed as a parameter.
```
let anOrderQtyInUnits = UnitQuantity 10
```
UnitQuantity and KilogramQuantity are **not** types themselves, but just distinct cases of the OrderQuantity type. They both match the type OrderQuantity.

To deconstruct a choice type, we pattern match with a test for each case 

```
let printQuantity aOrderQty =
	match aOrderQty with
	| UnitQuantity uQty ->
		printfn "%i units" uQty
	| KilogramQuantity kgQty ->
		printfn "%g kg" kgQty
```

Note that matching a case lets us access the associated data (uQty or kgQty above).

## Building a Domain Model by Composing Types 
In a composable type system we can quickly create a complex model simply by mixing types together in different combinations.

Example p.68

In a functional model, there is no behavior directly associated with the types we define because.
To document the actions that can be taken, we can define types that represent functions.
```
type PayInvoice =
	UnpaidInvoice -> Payment -> PaidInvoice
```
meaning that PayInvoice accepts an UnpaidInvoice and a Payment and returns a PaidInvoice.

## Modeling Optional Values, Errors, and Collections

### Modeling Optional Values
Records and choice types are not allowed to be null in F#. 

To model missing/optional data, we use the built-in Option choice type:

```
type Option<'a> =
	| Some of 'a
	| None
```

We can then wrap a value in this type in our model 
``` 
type PersonalName = {
	FirstName : string
	MiddleInitial: string option // syntactic sugar for Option<string>
	LastName : string
}
```

### Modeling Errors
How do we implement that something can go wrong?
F# supports exceptions, but we’ll often want to explicitly document in the type signature the fact that a failure can happen.
The built-in Result type is the answer. 

```
type Result<'Success,'Failure> =
	| Ok of 'Success
	| Error of 'Failure
```

We then need to pattern match when we call a function that returns a Result.
The Ok case holds the value when the function succeeds; the Error case hold the error data when something goes wrong. 

We then get a signature with the type of the Ok case and the Error case.   
```
type PayInvoice = 
	UnpaidInvoice -> Payment -> Result<PaidInvoice,PaymentError>
```

PaymentError can then be fleshed out into a number of different scenarios 

```
type PaymentError =
	| CardTypeNotRecognized
	| PaymentRejected
	| PaymentProviderOffline
	...
```

### Modeling No Value at All
Every function must return something in FP
In F# we return () (*unit*) when there is no data to return

```
type SaveCustomer = Customer -> unit
```

Unit can also be parameter to a function that has no input, but returns something
```
type NextRandom = unit -> int
```

Unit in function signatures is often a sign of side-effects.
 
### Modeling Lists and Collections
Built-in collection types:
- list - immutable linked list
- array - fixed-sized mutable collection
- ResizeArray - variable size array (alias for C# List<T>)
- seq - lazy collection where each element is returned on demand (alias for C# IEnumerable<T>) 
- Map/Set - rarely used directly in the domain model 
 
We generally should only use the list type in domain modelling.

Like the Option type, we can suffix a type with it
```
type Order = {
	OrderId : OrderId
	Lines : OrderLine list // Syntactic sugar for List<OrderLine>
}
```

Creating a list:
```
let aList = [1; 2; 3]
```
Note that the seperators are semicolon, not comma.
Prepend to list:
```
let aNewList = 0 :: aList
```

We can pattern natch on lists in two ways.
Matching on number of elements: 
```
let printList1 aList =
	match aList with
	| [] ->
		...
	| [x] ->
		...
	| [x;y] -> 
		...
	| longerList -> 
		....
```
Matching with cons 
```
let printList2 aList =
	match aList with
	| [] ->
		...
	| first::rest ->
		...
```

## Organizing Types in Files and Projects
 F# has strict rules about the order of declarations. 
 A type **higher** in a file cannot reference a type **below** in a file.
 Similarly, a file earlier in the compilation order cannot reference a file later in the compilation order.
 
 Typically, you put all the domain types in one file, say Types.fs or Domain.fs and put the functions that depend on them later in the compilation order.
 
 ```
Common.Types.fs
Common.Functions.fs
OrderTaking.Types.fs
OrderTaking.Functions.fs
...
```

When creating a type file, you put all the simple types at the top and the dependent complex types further down.

If you for some reason need to have a simple type below a complex type that uses it, you can use the **rec** keyword at the module level, to allow types to reference each other anywhere in the module.
```
module rec Payments =
type Payment = {
	Amount: ...
	Method: PaymentMethod 
	}
type PaymentMethod =
	| Cash
	| Card of ..
```

However, it should be avoided.  It’s better to put the types in the correct dependency order to make it consistent with other F# code and to make it easier for other developers to read.