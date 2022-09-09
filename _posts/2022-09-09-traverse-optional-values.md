---
layout: post
title: How to traverse optional values in Scala, Kotlin and Java
tags:
 - Scala
 - Kotlin
 - Java
 - Stream
 - Option
 - FP
 - ZIO
---

Let's study a real life, quite simple problem : 

We have a list of optional values, and we want to produce an optional single output, made from present values (if there is any).
We will also join values using a separator character.

Exemple 1 : Using a `List(Some(A), None, Some(B))` should produce `Some(a,b)`.
  
Exemple 2 : Using `List(None)` should produce `None`.  

## Scala version

We can use the `flip` method from ZIO-prelude (or the sequence method from Cats) to turn a `List[Option[String]]` into a `Option[List[String]]`.

```scala
import zio.prelude._
val list: List[Option[String]] = List(Some("A"), None, Some("B"))
val flippedList: Option[List[String]] = list.filter(_.isDefined).flip // <- Option(List(A,B))
flippedList.map(_.mkString(",")) // <- Some(A,B)
```

Note that if we just wanted to produce a String (that may be empty), we could have done this to remove the Option wrapper and undefined values : 

```scala
val flattenList = List(Some("A"), None, Some("B")).flatten // <- List(A, B)
flattenList.mkString(",")
```

## Kotlin version

In Kotlin, we usually just use nullable types to replace Option. 

```kotlin
val list = listOf("a", null, "b")
val result = list.filterNotNull().joinToString(separator=",").ifBlank { null }
```

## Java version

In Java we will use the stream API, which is a bit more verbose but still working fine.
Stream::flatMap will remove all the undefined elements and remove the Optional wrapper (like Scala's flatten).

```java
List<Optional<String>> list = List.of(Optional.of("A"), Optional.empty(), Optional.of("B"));
     
var result = Optional.of(
         list.stream()
        .flatMap(Optional::stream)
        .collect(Collectors.joining( "," ))
).filter(Predicate.not(String::isEmpty));

```
