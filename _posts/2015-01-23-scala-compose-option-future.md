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

It's also easy to compose futures,  in a non blocking way :

{% highlight scala %}
def fa: Future[String] = Future("a")
def fb: Future[String] = Future("b")

val fab : Future[String] = for{
  a <- fa
  b <- fb
} yield a+b
{% endhighlight %}

But is it easy to compose several Future[Option[A]]? It's a common problem if you call sequentially several external resources, that can give you one or zero value.

{% highlight scala %}
def foa: Future[Option[String]] = Future(Some("a"))
def fob(a: String): Future[Option[String]] = Future(Some(a+"b"))

/* can't compose like this : type mismatch
val composedAB = for {
  optA <-foa
  a <- optA // option instead of Future
  b <- fob(a)
}yield b
*/
{% endhighlight %}

We can't compose easily `fob` and `foa`: types will mismatch because futures and options don't compose naturally.

Fortunately, there are some solutions.

## Handmade binding

You can solve this with pattern matching : 

{% highlight scala %}
val composedAB: Future[Option[String]] = foa.flatMap {
  case Some(a) => fob(a) //(*)
  case None => Future(None)
}
{% endhighlight %}

(*) : In the for, call fob with the value inside the option and get a Future[Option[String]].
Doing this, you can compose result from `foa` and `fob` and keep consistent types.

## Using a custom FutureO monad

This is an idea and an implementation from [Edofic's blog](http://www.edofic.com/posts/2014-03-07-practical-future-option.html).

You can create a custom `FutureO` monad that will compose well with other `FutureO` instances.

You just need to define a constructor and `flatMap` (plus `map` if you want to use for comprehension syntactic sugar) : 

{% highlight scala %}
case class FutureO[+A](future: Future[Option[A]]) extends AnyVal {
  def flatMap[B](f: A => FutureO[B])
                (implicit ec: ExecutionContext): FutureO[B] = {
    FutureO {
      future.flatMap { optA =>
        optA.map { a =>
          f(a).future
        } getOrElse Future.successful(None)
      }
    }
  }

  def map[B](f: A => B)(implicit ec: ExecutionContext): FutureO[B] = {
    FutureO(future.map(_ map f))
  }
}
{% endhighlight %}

Then you can use a simple for comprehension form :

{% highlight scala %}
val composedAB2: Future[Option[String]] = (for {
  a <- FutureO(foa)
  b <- FutureO(fob(a))
}yield b).future
{% endhighlight %}

Or, with `flatMap` :

{% highlight scala %}
val noSugarComposedAB2 = FutureO(foa).flatMap(a => FutureO(fob(a)))
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
  b <- optionT(fob(a))
}yield b).run

{% endhighlight %}