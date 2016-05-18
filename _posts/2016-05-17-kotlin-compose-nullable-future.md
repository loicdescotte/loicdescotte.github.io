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

In a previous post, we've seen [how to compose Future and Options in Scala](http://loicdescotte.github.io/posts/scala-compose-option-future/). Let's try to do the same with Kotlin using nullables types and Java 8 completable futures.

This is how you can compose nullable functions :

```scala
  fun giveInt(x: Int):Int? = x+1

  fun giveInt2(x: Int):Int? = x+2

  fun combine(x: Int): Int? = giveInt(x)?.let { giveInt2(it) }

  combine(1) //4
```

`let` on applies if the first part is not null. Either a nullable is returned.


This is how you can compose futures (using Java 8 CompletableFuture API):

```scala
  fun giveInt(x: Int): CompletableFuture<Int> = CompletableFuture.supplyAsync({ x + 1 })

  fun giveInt2(x: Int): CompletableFuture<Int> = CompletableFuture.supplyAsync({ x + 2 })

  fun combine(x: Int): CompletableFuture<Int> =  giveInt(x).thenCompose({ giveInt2(it) })

  combine(1).get() //4
```

This is how you can compose futures of nullable types :

```scala
  fun giveInt(x: Int): CompletableFuture<Int?> = CompletableFuture.supplyAsync({ x + 1 })

  fun giveInt2(x: Int): CompletableFuture<Int?> = CompletableFuture.supplyAsync({ x + 2 })

  fun combine(x: Int): CompletableFuture<Int?> = giveInt(x).thenCompose({ it?.let { giveInt2(it) } })

  combine(1).get()
```

Finally even if it's not possible to use monad transformers as we would do in Scala, if think it is quite simple and readable!

This may be even simpler in the future using [Kotlin coroutines](https://github.com/Kotlin/kotlin-coroutines).
