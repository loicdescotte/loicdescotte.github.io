---
layout: post
title: Memoization with ZIO
tags:
 - Scala
 - ZIO
---

## ZIO tip : How to memoize the result of an effectful computation : 

ZIO values are lazy and referentially transparent. So a `IO[E,A]` will be computed each time it needs to be evaluated.  
It is a very intersting property in functional programming, but sometimes for long and cachable computations, it can be useful to memoize a result. It can be done using `ZIO.memoize`.

Here is an example : 

```scala
val compute: Int => UIO[Int] = (x: Int) => ???
for {
      m <- ZIO.memoize(compute)
      a <- m(1)
      b <- m(1)
      c <- m(1)
      d <- m(2)
} yield(a, b, c, d)
```

As a result, `m(1)` result will be cached (only be computed 1 time).
