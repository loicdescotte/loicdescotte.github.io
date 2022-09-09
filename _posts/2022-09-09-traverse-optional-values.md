---
layout: post
title: How to traverse optional values in Scala, Kotlin and Java
tags:
 - Scala
 - Option
 - FP
 - ZIO
---

Let's study a real life, quite simple problem : 

We have a list of optional values, and we want to produce an optional single output, made from present values (if there is any).
We will also join values using a separator character.

## Scala version

We can use the `flip` method from ZIO-prelude (or the sequence method from Cats) to turn a `List[Option[String]]` into a `Option[List[String]]`.

```scala
import zio.prelude._
val l: List[Option[String]] = List(Some("A"), None, Some("B"))
val lf: Option[List[String]] = l.filter(_.isDefined).flip // <- Option(List(A,B))
lf.map(_.mkString(",")) // <- Some(A,B)
```

Note that if we just wanted to produce a List, we could have done this to remove the Option wrapper and undefined values : 

```scala
List(Some("A"), None, Some("B")).flatten // <- List(A, B)
```

## Kotlin version

In Kotlin, we usually just use nullable types to replace Option. 

```kotlin
val l = listOf("a", null, "b")
val result = l.filterNotNull().joinToString(separator=",").ifBlank { null }
```

## Java version

In Java we will use the stream API, which a bit more verbose but still working fine.
Stream::flatMap will remove all the undefined elements and remove (like Scala's flatten) the Optional wrapper.

```java
  List<Optional<String>> l = List.of(Optional.of("A"), Optional.empty(), Optional.of("B"));
     
       var result = Optional.of(
                l.stream()
               .flatMap(Optional::stream)
               .collect(Collectors.joining( "," ))
       ).filter(Predicate.not(String::isEmpty));

```
