
# Chapter 7: Making our own types and type classes 

## Defining a New Data Type

The *data* keyword is used to declare a type. 

```
data Shape = Circle Float Float Float | Rectangle Float Float Float Float
```

Type name and value constructors must start with uppercase.

## Shaping Up

Value constructors are actually functions that ultimately return a value of a data type

Note that Circle and Rectangle above are not types, but values of the type shape.
We can use Shape as a input/output type to a function, but we can't use Circle as a type argument.

We can pattern match against constructors 
```
area :: Shape -> Float
area (Circle _ _ r) = pi * r^2
area (Rectangle x1 y1 x2 y2) = ... 
```

To implement a type, we add deriving to the declaration 
```
data Shape = Circle ... | Rectangle ...
	deriving (Show)
```

As value constructors are functions, we can use map, partial application, etc.

### Improving Shape with the Point Data Type

We often use the same name for the data type and the value constructor when there's only one constructor. 

We can compose types together
```
data Point = Point Float Float deriving (Show)
data Shape = Circle Point Float | Rectangle Point Point deriving (Show)
````

Various examples of using the types p. 112

### Exporting Our Shapes in a Module

To export data types, write your type along with the functions you are exporting, using parentheses to specify the value constructors that you want to export,
separated by commas.

```
module Shapes
( Point(..)
, Shape(..)
, area
, nudge
, baseCircle
, baseRect
) where
```

( .. ) exports every value constructor for the type - this makes things easier for us if we later define new ones we want to export as they are automatically exported.

To prevent a value constructor from being created directly and restrict creation to a dedicated function, simply write the type with no parenthesis and then export the function.
Not exporting the value constructors of our data types makes them more abstract, since we’re hiding their implementation.
Whoever uses our module can’t pattern match against the value constructors. 
This prevents people from coding against internal implementation details of the module and allows us to freely change those as long as the functions that we export act the same.

For simpler data types, exporting the value constructors is perfectly fine.

### Record Syntax

Record types for objects 

```
data Car = Car { company :: String
, model :: String
, year :: Int
} deriving (Show)
```

Using this syntax automatically creates functions that look up fields in the data type.

Creating a record 
```
 Car {company="Ford", model="Mustang", year=1967}
 ```
 
 Use record syntax when a constructor has several fields and it’s not obvious which field is which
 
## Type Parameters

A value constructor produces a new value from its parameters. 
Similarly, **type** constructors can take types as parameters to produce new types

```
data Maybe a = Nothing | Just a
```

When passing a value to a type constructor we can force it into using another type than what type inference detects. For instance, numbers get the Num type. We may want to force it into an int instead 

```
Just 3 :: Maybe Int
```

### Should We Parameterize Our Car?

We usually use type parameters when the type that’s contained inside the data type’s various value constructors isn’t really that important for the type to work

it’s a very strong convention in Haskell to never add type class constraints in data declarations
```
data (Ord k) => Map k v =
```

 If we put the Ord k constraint in the data declaration for Map k v , we still need to put the constraint into functions that assume the keys in a map can be ordered.
  If we don’t, then we don’t need to put (Ord k) => in the type declarations of functions
that don’t care whether the keys can be ordered.

### Vector von Doom

An example of what was just discussed wrt. not putting the restraint in the declaration
```
data Vector a = Vector a a a deriving (Show)
vplus :: (Num a) => Vector a -> Vector a -> Vector a
(Vector i j k) `vplus` (Vector l m n) = Vector (i+l) (j+m) (k+n)
```

it’s very important to distinguish between the type construc-
tor and the value constructor. When declaring a data type, the part before
the = is the type constructor, and the constructors after it (possibly separated
by | characters) are value constructors.

## Derived Instances

Haskell type classes are often confused with OOP type classes where classes are a blueprint from which we create objects.
We don’t make data from Haskell type classes.

Instead, we first define our data type, and then we think about how it can act. 
If it can act like something that can be equated, we make it an instance of the Eq type class. If it can act like something that can be ordered, we make it an instance of the Ord type class.

### Equating People 

When we try to compare two values of a type that derives Eq (using == or /= ) Haskell checks if the value constructors match by comparing each pair of fields. Thus each field of a type that derives from Eq must additionally do so. 

### Show Me How to Read

The Show and Read type classes are for things that can be converted to or from strings, respectively.
When we derrive show we can call the show function with an instance of the type. 

Read is the opposite. We have a string that we want to convert to a type. We must explicitly provide which type we want. 
```
read mysteryDude :: Person
```

When reading using read on parameterized types, we must explicitly provide the type parameter type(s) as well 
```
read "Just 3" :: Maybe Int
```

## Order in the Court!
When comparing a two instances of a type deriving Ord, the instance that was made using the constructor defined first is considered smaller. 
Two values created with the same constructor are equal if they have no fields. 
If they have fields, the fields are compared. 

## Any Day of the Week

Creating enumerations 
A type where all the type’s value constructors are nullary (that is, they don’t
have any fields), can derive the Enum type class

```
data Day = Monday | Tuesday | Wednesday | Thursday | Friday | Saturday | Sunday
deriving (Eq, Ord, Show, Read, Bounded, Enum)
```

## Type Synonyms
Simply type aliases 

```
type String = [Char]
```

### Making Our Phonebook Prettier
Example of using type synomyms p.128

### Parameterizing Type Synonyms

Type synonyms can also be parameterized
```
type AssocList k v = [(k, v)]
```

we can partially apply type parameters and get new type constructors from them (examples have the same result)

```
type IntMap v = Map Int v
type IntMap = Map Int
```

### Go Left, Then Right

```
data Either a b = Left a | Right b deriving (Eq, Ord, Read, Show)
```
Note that when we call one of the constructors, the other type parameter is unbound (**a** below):
```
ghci> :t Right 'a'
Right 'a' :: Either a Char
```

Locker example of using Either type 
```
lockerLookup :: Int -> LockerMap -> Either String Code
lockerLookup lockerNumber map = case Map.lookup lockerNumber map of
	Nothing -> Left $ "Locker " ++ show lockerNumber ++ " doesn't exist!"
	Just (state, code) -> if state /= Taken
						  then Right code
						  else Left $ "Locker " ++ show lockerNumber ++ " is already taken!"
```

## Recursive Data Structures
in type definitions, types can have themselves as types in their fields

```
data List a = Empty | Cons a (List a) deriving (Show, Read, Eq, Ord)
```
### Improving Our List
We can use special characters to define functions to be automatically infix.  Infix constructors must begin with a colon tho.
```
infixr 5 :-: //  fixity declaration
data List a = Empty | a :-: (List a) deriving (Show, Read, Eq, Ord)
```
When we define functions as operators, we can give them a **fixity** that defines how tightly the operator binds and whether it’s left-associative or right-associative (infixl vs infixr). However, this is optional. 

Now,we can write out lists in our list type
```
3 :-: 4 :-: 5 :-: Empty
```

Pattern matching always works on constructor, which means we can match on the custom constructor we just defined, :-:,  and on :, the constructor for the built-in list type. 
The same goes for [] .
We can match for normal prefix constructors or stuff like 8 or 'a' , which are basically constructors for the numeric and character types.

```
infixr 5 ^++
(^++) :: List a -> List a -> List a
Empty ^++ ys = ys
(x :-: xs) ^++ ys = x :-: (xs ^++ ys)
```

### Let’s Plant a Tree
Sets and maps from Data.Set and Data.Map are implemented using balanced binary search trees. 

```
data Tree a = EmptyTree | Node a (Tree a) (Tree a) deriving (Show)
```
we’ll make a function that takes a tree and an element and inserts an element. 
We insert by comparing the new value to the tree’s root node.
- If smaller, we go left;
- if larger, we go right.

We repeat this until we reach an empty tree.

In an OOP language, you would modify the tree object by changing a pointer.
In Haskell, we  make a new subtree each time we decide to go left or right. In the end, the insertion function returns a completely new tree. 
Thus our function must return a tree whereas an OOP implementation would likely return void. 

```
singleton :: a -> Tree a
singleton x = Node x EmptyTree EmptyTree

treeInsert :: (Ord a) => a -> Tree a -> Tree a
treeInsert x EmptyTree = singleton x
treeInsert x (Node a left right)
	| x == a = Node x left right
	| x < a = Node a (treeInsert x left) right
	| x > a = Node a left (treeInsert x right)
```

Finding an element 

```
treeElem :: (Ord a) => a -> Tree a -> Bool
treeElem x EmptyTree = False
treeElem x (Node a left right)
	| x == a = True
	| x < a = treeElem x left
	| x > a = treeElem x right
```

We can use fold to create a tree from a list.
```
ghci> let nums = [8,6,4,1,7,3,5]
ghci> let numsTree = foldr treeInsert EmptyTree nums
```
## Type Classes 102
How to make your own type classes and creating type instances of them.
 
## Inside the Eq Type Class
Implementation of built-in Eq: 
```
class Eq a where
	(==) :: a -> a -> Bool
	(/=) :: a -> a -> Bool
	x == y = not (x /= y)
	x /= y = not (x == y)
```

 Note that it’s not mandatory to implement the function bodies themselves; only their type declarations.
If we do :t on a function defined in the type class, we get a type signature like 
```
(Eq a) => a -> a -> Bool 
```
### A Traffic Light Data Type
```
data TrafficLight = Red | Yellow | Green

....
instance Show TrafficLight where
show Red = "Red light"
show Yellow = "Yellow light"
show Green = "Green light"

....

instance Eq TrafficLight where
	Red == Red = True
	Green == Green = True
	Yellow == Yellow = True
_ == _ = False
```
*class* is for defining new type classes.
*instance* is for making our types instances of type classes.
*minimal complete definition*: Because of the way  /= is implemented based on == in the Eq class, when we overwrite == we automatically redefine /= as well. This means we only have to implement one of these. 

### Subclassing
Subclassing is just a class constraint on a class declaration
```
class (Eq a) => Ord a where
...

This is just like writing class Ord a where , but we require that our type must be an instance of Eq 

```
### Parameterized Types As Instances of Type Classes
Maybe is different from TrafficLight in that Maybe in itself isn’t a concrete type—it’s a type constructor that takes one type parameter to produce a concrete type (like Maybe Char ).

a is used as a concrete type in the Eq class definition because all the types in functions must be concrete.
To use Maybe, we must provide it with a type variable parameter (or an actual type like Int instead of M, if we didn't want to make the function generic) 
```
instance (Eq m) => Eq (Maybe m) where
	Just x == Just y = x == y
	Nothing == Nothing = True
	_ == _ = False
```

Most of the time, class constraints in class declarations are used for making a type class a subclass of another type class, and class constraints in instance declarations are used to express requirements about the contents of
some type.

When a type is used as a concrete in the type declarations (like the a in a -> a -> Bool ) when making instances, you need to supply type parameters and add parentheses so that you end up with a concrete type.


:info shows the type classes of an instance 

## A Yes-No Type Class
Implementing truthyness 

```
class YesNo a where
	yesno :: a -> Bool
	
instance YesNo Int where
	yesno 0 = False
	yesno _ = True
	
instance YesNo [a] where
	yesno [] = False
	yesno _ = True
	
instance YesNo Bool where
	yesno = id
...
```

Note that id is a built-in function that takes a parameter and returns identity. 

## The Functor Type Class
Functor type class - things that can be mapped over. 
Accepts a type constructor with a single type parameter. 

```
class Functor f where
	fmap :: (a -> b) -> f a -> f b
```

map is just an fmap that works only on lists.
```
map :: (a -> b) -> [a] -> [b]
```

 In previous definitions of type classes, the type variable that played the role of the type in the type class was a concrete type, like the a in (==) :: (Eq a) => a -> a -> Bool. Maybe Int is a concrete type, but Maybe is a type constructor that takes one type as the parameter.
  f is not a concrete type, but a type constructor that takes one type parameter. 
fmap takes a function from one type to another and a functor value applied with one type and returns a functor value applied with another type.

[a] is a concrete type (of a list with any type inside it); [] is a type constructor that takes one type and can produce types such as [Int]. Thus the list instance is implemented as 
```
instance Functor [] where
	fmap = map
```

with [] as the value of f, not [a].

### Maybe As a Functor
Wrapper that can either hold something or nothing can be functors; a list is essentially a wrapper.
Maybe satisfies this.
```
instance Functor Maybe where
	fmap f (Just x) = Just (f x)
	fmap f Nothing = Nothing
```

i.e. we apply f to whatever Maybe holds or return Nothing if Maybe holds Nothing.
We can use partial application:
```
ghci> fmap ( * 2) (Just 200)
Just 400
```
i.e. f is * with 2 bound already, which we applied to 200 to create Just(400). 
### Trees Are Functors, Too
 Mapping over an empty tree
will produce an empty tree. Mapping over a nonempty tree will produce a
tree consisting of our function applied every node. 

```
instance Functor Tree where
	fmap f EmptyTree = EmptyTree
	fmap f (Node x left right) = Node (f x) (fmap f left) (fmap f right)
```
Note that this means that we can have a tree that satisfies the binary search tree properties before application of f but will not have the properties after, for instance if you negate every value. 


### Either a As a Functor
```
instance Functor (Either a) where
	fmap f (Right x) = Right (f x)
	fmap f (Left x) = Left x
```
As the either constructor takes two type parameter and functor only accepts type constructors accepting a single, we must partially apply Either. 


## Kinds and Some Type-Foo
Type constructors take other types as parameters to eventually produce
concrete types. This behavior is similar to that of functions, which take
values as parameters to produce values. 

Types can be seen as labels that values carry so that we can reason about them. Types in themselves have labels called kinds; a type of a type
:k can be used to examine kinds 

:k Int 
* indicates that the type is a concrete type; a type that doesn’t take any type parameters.
Values can only have types that are concrete types.

:k Maybe 
*->* means the Maybe type constructor takes one concrete type (e.g. Int) and returns a concrete type (like Maybe Int).

To make Either a part of the Functor type class, we had to partially apply it, because Functor can only accept type constructors needing one type parameter, while Either takes two. 
I.e. Functor wants types of kind *->* , and with partial application we got from kind, * ->*->* to *->*.

What kind of data structure would you use to implement a set or map? Balanced BST 
What is the difference between a concrete type and a type constructor?  Maybe Int vs Maybe, [] vs. [a] - the latter takes a single type parameter.