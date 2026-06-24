---
layout: post
title: Hamsters is finally available for Scala 3 !
tags:
 - validation
 - minimal
 - functional-programming
 - error-handling
 - typeclass
 - monad-transformers
---

After a long wait, I'm happy to announce that [Hamsters](https://github.com/scala-hamsters/hamsters), the mini Scala utility library, has been ported to **Scala 3**!

The new **4.0.0** release is built for Scala 3 (3.3.x LTS) and cross-compiled for the JVM and Scala.js. The goal of the library hasn't changed: provide a small, focused set of functional utilities that stay approachable, even for people who are just getting started with functional programming.

To add it to your project:

```scala
libraryDependencies += "io.github.scala-hamsters" %% "hamsters" % "4.0.0"
```

Or for a Scala.js project:

```scala
libraryDependencies += "io.github.scala-hamsters" %%% "hamsters" % "4.0.0"
```

## A quick tour

Here is a reminder of a few things Hamsters can do for you.

### Validation and error handling

Accumulate several validation errors instead of failing on the first one:

```scala
import io.github.hamsters.Validation
import Validation._

val e1 = Valid(1)
val e2 = Invalid("error 1")
val e3 = Invalid("error 2")
val e4 = Valid("4")

Validation.run(e1, e2, e3)
// Invalid(List("error 1", "error 2"))

Validation.run(e1, e4)
// Valid((1, "4"))
```

You can also turn exceptions into values with `fromCatchable`:

```scala
def compute(x: Int): Int = 2 / x

fromCatchable(compute(1)) // Valid(2)
fromCatchable(compute(0)) // Invalid("/ by zero")
```

You can also extract just the successes or just the failures from a set of validations with `Validation.successes` and `Validation.failures`.

### Monad transformers

Composing `Future[Option[_]]` or `Future[Either[_, _]]` in a for-comprehension is painful by hand. Hamsters gives you `FutureOption` and `FutureEither` to flatten the layers:

```scala
import io.github.hamsters.FutureEither
import io.github.hamsters.MonadTransformers._

def fea: Future[Either[String, Int]] = Future(Right(1))
def feb(a: Int): Future[Either[String, Int]] = Future(Right(a + 2))

val composedAB: Future[Either[String, Int]] = for {
  a <- FutureEither(fea)
  ab <- FutureEither(feb(a))
} yield ab
```

## And more

Beyond validation and monad transformers, the library also ships:

- `mapN` to combine multiple values
- Lenses
- Memoization
- `NonEmptyList`
- Retry helpers
- Default values for options (`orEmpty`)
- Future squashing (nested type simplifications)
- And more

Have a look at the [README](https://github.com/scala-hamsters/hamsters) and the [docs](https://github.com/scala-hamsters/hamsters/tree/master/docs) for the full list. Contributions and feedback are, as always, very welcome!
