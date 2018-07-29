---
layout: post
title: Smart filters in Scala using function currying and pipe operator
tags:
 - Functional Programming
 - Scala
 - Currying
 - Pipe
---

First, let's define a pipe operator to chain functions : 

```scala
def f(l: List[Int]) = l.filter(x => x >1)
def g(l: List[Int]) = l.filter(x => x <3)

// apply f then g
List(1,2,3) |> f |> g 
//result : List(2)
```

Now, let's define a filter function, taking predicates for a type T in a first parameter block and a list of T in a second parameter block : 

```scala
def filterByField[T](selectField: Item => T, predicates: Set[String])(items: List[Item]) = {
  items.filter { item =>
    predicates.exists(included => selectField(item).toString.matches(included))
  }
}
```

With the pipe operator, it's now really easy to chain filters in a very readable way :

```scala
case class Item(name: String, value: String)
val items: List[Item] = ???

val filteredItems = items |>
  filterByField(_.name, nameFilters) |>
  filterByField(_.value, valueFilters) // chain as many filters as you want...
```
Here is an implementation for `|>` (found on [Reddit](https://www.reddit.com/r/scala/comments/480nfm/operator_in_scala/d0got47/)) : 

```
implicit class AnyEx[T](val v: T) extends AnyVal {
  def |>[U](f: T ⇒ U): U = f(v)
}
```
