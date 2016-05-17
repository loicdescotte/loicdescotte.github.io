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

```kotlin
  fun giveInt(x: Int):Int? = x+1

  fun giveInt2(x: Int):Int? = x+2

  fun combine(x: Int): Int? = giveInt(x)?.let { giveInt2(it) }

  combine(1) //4
```

This is how you can compose futures :

```kotlin
fun giveInt(x: Int): CompletableFuture<Int> = CompletableFuture.supplyAsync({ x + 1 })

    fun giveInt2(x: Int): CompletableFuture<Int> = CompletableFuture.supplyAsync({ x + 2 })

    fun combine(x: Int): CompletableFuture<Int> =  giveInt(x).thenCompose({ giveInt2(it) })

    combine(1).get() //4
```

This is how you can compose futures of nullable types :

```kotlin
  fun giveInt(x: Int): CompletableFuture<Int?> = CompletableFuture.supplyAsync({ x + 1 })

  fun giveInt2(x: Int): CompletableFuture<Int?> = CompletableFuture.supplyAsync({ x + 2 })

  fun combine(x: Int): CompletableFuture<Int?> = giveInt(x).thenCompose({ it?.let { giveInt2(it) } })

  combine(1).get()
```

See also Kotlin async API.

## Bonus : How to flatten a List of options (or nullables) in Scala, Java 8 and Kotlin
