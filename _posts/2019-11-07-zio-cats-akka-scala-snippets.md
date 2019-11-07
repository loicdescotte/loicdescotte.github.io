---
layout: post
title: Scala snippets (ZIO, Cats, Slinky, Akka Streams)
tags:
 - Functional Programming
 - Scala
 - ZIO
 - Akka Streams
 - React
 - Slinky
 - Cats
---

I write this snippets as memory aid, but maybe it can be useful for someone else :)

## Change delimiter in a ZIO Stream

Example, group/split by lines : 

<script src="https://gist.github.com/loicdescotte/1551b102c7baecb77b3a0b94b512f965.js"></script>

## Accumulate errors with Either and Cats 

Based on Cats documentation but with Error ADT and NonEmptyChain

<script src="https://gist.github.com/loicdescotte/52aac795c74dfca1cd622d6736abe906.js"></script>

##  Slinky React Hooks example 

<script src="https://gist.github.com/loicdescotte/09fb8bdd0807434e5e3b577d52d61ac6.js"></script>

## Akka Streams : Consume a source with two sinks and get the result of one sink 

<script src="https://gist.github.com/loicdescotte/e1d236001fadc7a2d1fae88098a2c5b5.js"></script>

## Akka Streams and Play Framework : add elements to a source dynamically, using a queue

<script src="https://gist.github.com/loicdescotte/5f3ed7a56b8d3fa8eb2982e9e97dcb36.js"></script>

