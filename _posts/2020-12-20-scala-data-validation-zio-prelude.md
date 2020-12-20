---
layout: post
title: Data Validation in Scala with ZIO Prelude
tags:
 - Scala
 - Scala 3
 - ZIO
 - Prelude
 - data
 - validation
 - scalaz
 - cats
---

[ZIO Validation](https://github.com/zio/zio-prelude) is a new library made to help functional programming with algebraic constructions in Scala.  
It aims to be simpler and to fit better to the Scala language than Cats and Scalaz.  

Here is a simple data validation example (using Scala 3 indentation-based syntax) : 

<script src="https://gist.github.com/loicdescotte/7b99d4375615df989ec42d45a4b354f2.js"></script>


As you can see it is really easy to mix monadic error handling (stop at first error) with Either (or Option) and validation of multiple errors (parallel validation instead of stpping at first error) with the ZIO Prelude Validation data type.  

P.S. : I'd like to thank the Univalence team for their youtube channel and their unboxing video of ZIO Prelude.
