---
layout: post
title: Some ZIO tips
tags:
 - Scala
 - ZIO
 - Generic
 - Cache
 - Query
 - Micro service
 - Type classes
---

To start this new year, let's see some ZIO tips snippets!

## How to cache queries with ZIO and ZQuery

Let's say we have some employees and departments data.  
For each employee I want departement info, but I don't want to query multiple time for the same departement.

<script src="https://gist.github.com/loicdescotte/1cc5f2a00506138a64efe3534214f6d7.js"></script>

## How to write a generic micro service with Scala 3 and ZIO

Here, I want to write a service that would be able to read, transform and write any type of data, as soon as the data types and the transformers are defined.

<script src="https://gist.github.com/loicdescotte/8ab10c13b7c63920ec5637b5c695368b.js"></script>

To learn more about the transformer pattern, you can read [this post](./type-classes-in-scala-3/) about type classes.

Happy new year everyone!
