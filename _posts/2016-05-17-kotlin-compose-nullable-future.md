---
layout: post
title: How to compose Future and Nullable in Kotlin
tags:
 - Kotlin
 - Nullable
 - Future
 - Completable Future
 - Java 8
---

In a previous post, we've seen [how to compose Future and Option in Scala](http://loicdescotte.github.io/posts/scala-compose-option-future/). Let's try to do the same with Kotlin using nullables types and Java 8 completable futures.

This is how you can compose nullable functions :

```scala
  fun giveInt(x: Int):Int? = x+1

  fun giveInt2(x: Int):Int? = x+2

  fun combine(x: Int): Int? = giveInt(x)?.let(::giveInt2)

  combine(1) //4
```
The result of `giveInt` is passed to `giveInt2`. But `let` only applies if the first part is not null. Either a nullable is returned.


This is how you can compose futures (using Java 8 CompletableFuture API):

```scala
  fun giveInt(x: Int): CompletableFuture<Int> = CompletableFuture.supplyAsync({ x + 1 })

  fun giveInt2(x: Int): CompletableFuture<Int> = CompletableFuture.supplyAsync({ x + 2 })

  fun combine(x: Int): CompletableFuture<Int> =  giveInt(x).thenCompose(::giveInt2)

  combine(1).get() //4
```

This is how you can compose futures of nullable types :

```scala
  fun giveInt(x: Int): CompletableFuture<Int?> = CompletableFuture.supplyAsync({ x + 1 })

  fun giveInt2(x: Int): CompletableFuture<Int?> = CompletableFuture.supplyAsync({ x + 2 })

  fun combine(x: Int): CompletableFuture<Int?> = giveInt(x).thenCompose({ it?.let(::giveInt2) })

  combine(1).get() //4
```

And this how we would have done it with Scala and with the [Hamsters](https://github.com/scala-hamsters/hamsters) library : 

```scala
def giveInt(x: Int): Future[Option[Int]] = Future.successful(Some(x+1))

def giveInt2(x: Int): Future[Option[Int]] = Future.successful(Some(x+2))

def combine(x: Int): Future[Option[Int]] = for {
  y <- FutureOption(giveInt(x))
  z <- FutureOption(giveInt2(y))
} yield z

Await.result(combine(1), 1 second) //Some(4)
```

Finally even if it's not possible to use monad transformers as we would do in Scala, if think Kotlin is giving out of the box a very simple and readable solution.

And this may be even simpler in the future using [Kotlin coroutines](https://github.com/Kotlin/kotlin-coroutines).
