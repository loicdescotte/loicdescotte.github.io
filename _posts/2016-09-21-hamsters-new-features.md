---
layout: post
title: A small update on Scala Hamsters
tags:
 - Functional Programming
 - Scala
 - Hamsters
---

I've just published Hamsters 1.0.7, with a few new features.  

Now you can automatically catch exceptions into a KO object : 

```scala
import io.github.hamsters.Validation._

def compute(x: Int):Int = 2/x

fromCatchable(compute(1)) //OK(2)
fromCatchable(compute(0)) //KO("/ by zero")

fromCatchable(compute(0), (t: Throwable) => t.getClass.getSimpleName) //KO("ArithmeticException")
```

To see how to use the OK/KO monad, see the [Readme file](https://github.com/scala-hamsters/hamsters/blob/master/README.md).

Another new feature allows to ask for a specifc type and retrieve an option on an Union type :

```scala
import io.github.hamsters.{Union3, Union3Type}

//json element can contain a String, a Int or a Double
val jsonUnion = new Union3Type[String, Int, Double]
import jsonUnion._

def jsonElement(x: Int): Union3[String, Int, Double] = {
  if(x == 0) "0"
  else if (x % 2 == 0) 1
  else 2.0
}

jsonElement(0).get[String] // Some("0")
jsonElement(1).getOrElse("not found") // get String value or "not found" if get[String] is undefined
```
