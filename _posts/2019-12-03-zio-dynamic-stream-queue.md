---
layout: post
title: How to add elements dynamically to a ZIO stream using a queue
tags:
 - Functional Programming
 - Scala
 - ZIO
 - Stream
---

Here is a small example of parallel pull/push into a ZIO queue. The data is consumed via a stream created from the queue.

<script src="https://gist.github.com/loicdescotte/05a34d3c39be327dcf1796a54aecf898.js"></script>

