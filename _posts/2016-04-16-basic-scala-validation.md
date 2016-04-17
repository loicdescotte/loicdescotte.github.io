---
title: Basic validation with Scala
tags:
 - Validation
 - Scala
 - Either
---

In Scala you don't always need libraries like [Scalaz](https://github.com/scalaz/scalaz) or [Cats](https://github.com/typelevel/cats) to do some validation and functional error management. Don't get me wrong this libs are great, very powerful and very useful in a lot of cases.  
But they can also be frightening for beginners and overkill for simple use cases. Scalaz and Cats scope is widely larger than just validation, but let's focus on this use case for this small article. 

## Validation : get all errors

We will just rely on the scala standard [Either](http://www.scala-lang.org/api/2.11.8/#scala.util.Either) and build a mini API on top of it.  
We consider that the "good" side of Either is always Right, and that the error side is Left.

```scala
class Validation[L,R](eithers: List[Either[L,R]]){
  def errors: List[L] = eithers.collect{ case l : Left[L,_] => l.left.get}
  def dones: List[R] =  eithers.collect{ case r : Right[_,R] => r.right.get}
}

object Validation {
  def apply[L,R] (eithers: Either[L,R]*) = {
    new Validation(eithers.toList)
  }
}
```

Then it can be used this way : 

```scala
val e1: Either[String, Int] = Right(1)
val e2: Either[String, Int] = Left("nan")
val e3: Either[String, Int] = Left("nan2")

val validation = Validation(e1,e2, e3)
val errors = validation.errors //List[String] : List("nan", "nan2")
val dones = validation.dones //List[Int] : List(1)
```

## Stop at the first error

For this example we will just enhance a little the Either API to allow using Right at the default "good" side by default in for comprehension :

```scala
implicit class RightEither[L, R](e: Either[L, R]) {
  def map[R2](f: R => R2) = e.right.map(f)
  def flatMap[R2](f: R => Either[L, R2]) = e.right.flatMap(f)
}
```

Then we can do : 

```scala
val e1: Either[String, Int] = Right(1)
val e2: Either[String, Int] = Left("nan")
val e3: Either[String, Int] = Left("nan2")

for {
  v1 <- e1
  v2 <- e2
  v3 <- e3
} yield(s"$v1-$v2-$v3")  //Left("nan")
```

The monadic properties of Either allow to stop at the first error to get a result if everything is fine or a single error if it fails.


