---
layout: post
title: How to compose Future and Option in Scala
tags:
 - Scala
 - Monads
 - Scalaz
 - Composition
---

In Scala, it's easy to compose options :

{% highlight scala %}
val oa: Option[String] = Some("a")
val ob: Option[String] = Some("b")

val oab : Option[String] = for{
  a <- oa
  b <- ob
} yield a+b
{% endhighlight %}

Note : you can read [my helper](https://gist.github.com/loicdescotte/4044169) about for comprehension translation to `map/flatMap`.

It's also easy to compose futures,  in a non blocking way :

{% highlight scala %}
def fa: Future[String] = Future("a")
def fb(a: String): Future[String] = Future(a+"b")

val fab : Future[String] = for{
  a <- fa
  ab <- fb(a)
} yield ab
{% endhighlight %}

But is it easy to compose several Future[Option[A]]? It's a common problem if you call sequentially several external resources, that can give you one or zero value.

{% highlight scala %}
def foa: Future[Option[String]] = Future(Some("a"))
def fob(a: String): Future[Option[String]] = Future(Some(a+"b"))

/* can't compose like this : type mismatch
val composedAB = for {
  optA <-foa
  a <- optA // option instead of Future
  ab <- fob(a)
}yield ab
*/
{% endhighlight %}

We can't compose easily `fob` and `foa`: types will mismatch because futures and options don't compose naturally.

Fortunately, there are some solutions.

## Handmade binding

You can solve this with pattern matching : 

{% highlight scala %}
val composedAB: Future[Option[String]] = foa.flatMap {
  case Some(a) => fob(a) //(*)
  case None => Future.successful(None)
}
{% endhighlight %}

(*) : In the for, call fob with the value inside the option and get a Future[Option[String]].
Doing this, you can compose result from `foa` and `fob` and keep consistent types.

## Using a custom FutureO monad

This is an idea from [Edofic's blog](http://www.edofic.com/posts/2014-03-07-practical-future-option.html). It allows more natural composition of Option and Future types.

You can create a custom `FutureO` monad that will compose well with other `FutureO` instances.

You just need to define a constructor and `flatMap` (plus `map` as it's used by for comprehension syntactic sugar) : 

{% highlight scala %}
case class FutureO[+A](future: Future[Option[A]]) extends AnyVal {
  def flatMap[B](f: A => FutureO[B])(implicit ec: ExecutionContext): FutureO[B] = {
    val newFuture = future.flatMap{
      case Some(a) => f(a).future
      case None => Future.successful(None)
    }
    FutureO(newFuture)
  }

  def map[B](f: A => B)(implicit ec: ExecutionContext): FutureO[B] = {
    FutureO(future.map(option => option map f))
  }
}
{% endhighlight %}

Then you can use a simple for comprehension form :

{% highlight scala %}
val composedAB2: Future[Option[String]] = (for {
  a <- FutureO(foa)
  ab <- FutureO(fob(a))
}yield ab).future
{% endhighlight %}

Or, with `flatMap` :

{% highlight scala %}
val noSugarComposedAB2 = FutureO(foa).flatMap(a => FutureO(fob(a))).future
{% endhighlight %}

### Using a Scalaz monad transformer

Finally, if you want to generalize this kind of usage, you should try Scalaz monad transformers, which work for a larger kind of types than options and futures.

{% highlight scala %}
import scalaz._
import Scalaz._
import scalaz.OptionT._

// We are importing a scalaz.Monad[scala.concurrent.Future] implicit transformer

val composedAB3: Future[Option[String]] = (for {
  a <- optionT(foa)
  ab <- optionT(fob(a))
}yield ab).run

{% endhighlight %}
