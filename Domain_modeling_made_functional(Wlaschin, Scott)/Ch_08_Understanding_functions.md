
# Chapter 8 - Understanding Functions
## Functions, Functions, Everywhere

In functional programming (FP), functions are used everywhere, for everything.

Contrast between FP and OOP: 
Consider a large program that is assembled from smaller pieces.
-  OOP: pieces would be classes and objects.
-  FP: pieces would be functions.

Parameterizing the program / reducing coupling between components:
-   OOP: interfaces and dependency injection.
-   FP: parameterize with functions.

Applying the “don’t repeat yourself” principle / reuse code between components:
- OOP: inheritance or a technique like the Decorator pattern.
-  FP: put all the reusable code into functions and glue them together using composition.

## Functions Are Things

As functions are standalone objects in FP, they can both be used as parameters and return values of other functions. 
higher-order functions - functions that take a function as input or creates one as output. 

### Treating Functions as Things in F#
Assigning a function to a variable: 
```
let plus3 x = x + 3
let square = (fun x -> x * x) // anonynmous function / lambda expression
let addThree = plus3 

let listOfFunctions = // list of functions
[addThree; times2; square]

for fn in listOfFunctions do
	let result = fn 100 // Applying each function to 100
	printfn "If 100 is the input, the output is %i" result
```
### Functions as Input
```
let evalWith5ThenAdd2 fn =
fn(5) + 2

let add1 x = x + 1 
evalWith5ThenAdd2 add
```
### Functions as Output

Commonly used to bake in parameters
```
let add1 x = x + 1
let add2 x = x + 2
let add3 x = x + 3
```
Instead of duplicating code like  above, we can define a function that creates a function that adds some x to whatever input 
```
let adderGenerator numberToAdd =
fun x -> numberToAdd + x
```
i.e. add1 = adderGenerator 1 returns a new partially applied function 

### Currying  
Using partial application to make every function a single parameter function 
We can use function-outputting functions to turn any function that takes an multiparameter function into one that only takes a single one.  

Any two-parameter function with signature 'a -> 'b -> 'c can be
interpreted as a one-parameter function that takes an 'a and returns a function
('b -> 'c) or ('b -> 'c -> 'd -> 'e...) for functions with more parameters

### Partial Application

We can call a function with only a subset of the arguments to bake them in and then we can apply the baked in function on various input to generate various values 
```
let sayGreeting greeting name =
	printfn "%s %s" greeting name

let sayHello = sayGreeting "Hello"

let sayGoodbye = sayGreeting "Goodbye"
```

Then we can say hello and goodbye to whatever input. 

## Total Functions
We would like functions to have an output for every input as it makes the behavior explicit, as all effects are documented in the type signature 
Throwing an exceptions deviates from the type signature - it is not evident that an int -> int function can fail.
One way to ensure that we can return something for every input is to restrict the possible valid input with a private type with a smart constructor that restricts the input 
```
type NonZeroInteger = private NonZeroInteger of int
... 
 
let twelveDividedBy (NonZeroInteger n) = ...
 ```
 Signature then becomes 
 ```
 twelveDividedBy : NonZeroInteger -> int
 ```
  which makes the requirements for the input very obvious 
 
 Alternatively, instead of restricting the input, we can use an option type for the output and return None for invalid inputs, which again makes the behavior obvious through the signature.  
 
 ## Composition
 Used for information hiding
 We can create new functions from existing functions by "glueing" them together 
 A function 'a -> 'b can be glued together with a function 'b -> c to create the new function a -> c. The outside world does not know about how 'a is transformed to 'b internally and it doesnt have to. 
 
 ### Composition of Functions in F#
 Any two functions can be glued together as long as the output type of
the first one is the same as the input type of the second one.

Piping - similar to linux piping - feed a value into the first function, take the output of the first function and feed it into
the next function, and so on.
```
let add1 x = x + 1 
let square x = x * x 
let add1ThenSquare x =
	x |> add1 |> square
```
	
### Building an Entire Application from Functions

A functional application is essentially simple functions that are composed into more advanced components like a service.
The services can then be glued together into more advanced workflows. 
The workflows can then be composed together in parallel into an application where a controller/dispatcher selects which workflow is appropriate, based on the input.

### Challenges in Composing Functions
Sometimes we need to compose functions that don't match up, i.e. 'a -> 'b and 'c -> d.
More commonly, we might have function that outputs Option<'a> and a function that accepts the 'a type, and similar scenarious with lists, Result types, async, etc. 

Typically we take a "lowest common multiple approach":
A function X: int -> int can be joined with a function Y: Option<int> int by converting the output of A to option 
5 |> X |> Some |> Y