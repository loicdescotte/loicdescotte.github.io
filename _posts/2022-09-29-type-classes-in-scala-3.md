---
layout: post
title: Type classes and extension methods in Scala 3
tags:
 - Scala
 - FP
 - Type classes
 - Extension methods
 - DDD
---

## Type classes in Scala

A type class is a type system construct that supports "ad hoc polymorphism". It defines features than can be added to any type A if an instance of the type class for the A type is provided.  

### Type class definition

Let's define a type class with an "add" method : 

```scala
trait Addable[A]:
  extension (a1: A) def add(a2: A): A
```

The `extension` keyword means that we will augment the capabilties of a type A with an add method, that takes another A to produce a sum (also of type A).

### Instances

We also need instances to use this type class with concrete types. We will start to define instance to add integers and strings : 

```scala
given Addable[Int] with
  extension (i1: Int) def add(i2: Int) = i1 + i2

given Addable[String] with
  extension (s1: String) def add(s2: String) = s"$s1 . $s2"
```

Then we can use the extension method, directly on any integer or string : 

```scala
println(
  "Hello".add("how are you?")
)

// This prints: Hello . how are you?
```

We can add this capability to any data type, without changing the type directly. 
If you know Domain Driven Design (DDD) you know it is important to be able to keep the domain pure. Type class are very useful for this in a lot of cases, like JSON serialization for example.

It is also possible to add given instances to data type you can't change (from libs/external modules for example)

For example, if we have a Color type, define like that :

```scala
  enum Color :
  case Red 
  case Green
  case Blue
  case Purple
  case Yellow
  case Unknown
```

Without changing the Color type, we can add new capabilities to it :

```scala
given Addable[Color] with
  extension (c1: Color) def add(c2: Color) = 

   // dump color addition example
   (c1, c2) match {
     case (Color.Red, Color.Green) => Color.Yellow
     case (Color.Red, Color.Blue) => Color.Purple
     //... etc.
     case _=> Color.Unknown
   }

println(
  Color.Red.add(Color.Blue) 
)

// This prints: Purple
```

## More powerful functions

Finally, we can write methods that require a type with some given instances to work make operations using the augmented capabilities of this type : 

```scala
def addAll[A: Addable](l: List[A]) = l.reduceLeft(_.add(_))

println(addAll(List("Coucou", "comment", "ça", "va")))
   
// This prints : Coucou . comment . ça . va
```


