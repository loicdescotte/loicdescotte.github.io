---
layout: post
title: Scala and generic types erasure
tags:
 - Scala
 - Generics
---

Just a little tip or reminder about generic types erasure in Scala.

{% highlight scala %}
scala> def displayGeneric[T](arg: T)(implicit m: reflect.Manifest[T]) = println(s"${arg.toString} : ${m.toString}")
displayGeneric: [T](arg: T)(implicit m: scala.reflect.Manifest[T])Unit

scala> displayGeneric(1)
1 : Int
{% endhighlight %}

You can retrieve type information and get over erasure using an implicit Manifest. It is also very useful if you need to access the T type in your method code.
Example from the Scaladoc : 

{% highlight scala %}
def arr[T] = new Array[T](0)                          // does not compile
def arr[T](implicit m: Manifest[T]) = new Array[T](0) // compiles
def arr[T: Manifest] = new Array[T](0)                // shorthand for the preceding
{% endhighlight %}
