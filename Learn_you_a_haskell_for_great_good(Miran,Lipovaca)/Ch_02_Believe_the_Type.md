# Chapter 2: Believe the type 

Every expression’s type is known at compile time and Haskell has type inference.
Everything in Haskell has a type, so the compiler can reason quite a lot about your program before compiling it.


## Explicit Type Declaration

:t displays the type of any valid expression 

You should explicitly provide the type of a function, unless very short, as in 

```
removeNonUppercase :: [Char] -> [Char]
```
## Common Haskell Types

Int 
- bounded integer type  
- determined by the size of a machine word on your machine. 
- -2<sup>63</sup> - 2<sup>63</sup>-1 on a 64 bit CPU

Integer - unbounded integer type 

Float - single precision floating point number 

Double 
- double-precision floating point number 
- use twice as many bits to represent numbers 

Bool - True or False 

Char 
- Unicode character
- Single quotes
- A list of Char is a string

Tuple 
- type depends on both length and the type of components 
- The empty tuple is also a type - () 

# Type Variables
Type variables is a form of generics that allow for any type to be used in place of the variable.
For instance, the type signature of the head function is [a] -> a. 
All types start with uppercase; a, however, does not as it is a type variable.

Polymorphic functions: Functions that use type variables.

# Type Classes 101
A type class is an interface that defines some behavior;
an instance of a type class supports and implements that behavior.

When using :t on functions that only accepts a type class, everything before => is called a class constraint of which there can be multiple.

Eq
- Support for equality testing
- Instances must implement == and /=

Ord
- Support for ordering
- Must implement <, <=, >, >=
- Compare function accepts two Ord instances and returns an Ordering

Show
- Instances can be represented as strings
- show function

Read 
- "Opposite" of show 
- read function takes a string and returns an instance of Read typed value 
- If the type of the argument to read is not clear from the context, read fails
- *Type annotations* must instead be used to tell the compiler what type the expression should be (read "5" :: Int")

Enum
- Instances are sequentially ordered types 
- Main advantage is that we can use its values in list ranges 
- succ and pred functions can be used to get the successor and predecessor in the sequence 

Bounded 
- Instances have upper and lower bound 
- Can be checked with minBound/maxBound
- Tuples where all the components are bounded are themselves bounded 

Num 
- Instances can act like numbers 
- Show and Eq are required

Floating
- Instances are Float and Double types 
- Functions that take instances need to represent their results as floating point numbers 
- sin, cos, sqrt

Integral 
- Subset of num - num includes all numbers (including real), instances here are only integers
- fromIntegral function is useful when we need integral and floating-point types to work together - turns integer into more general number representation
### Some Final Notes on Type Classes
A type can be an instance of many type classes.
A type class can have many types as instances. 

Sometimes a type can only be a an instance of one type class, if it is an instance of another.
For instance, it being an instance of Eq is a prerequisite for being an instance of Ord.  