---
layout: post
title: Akka Streams, Play Framework and queues
tags:
 - Play Framework
 - Scala
 - Reactive-Streams
 - Akka Streams
 - Server Sent Events
 - Queue
---

In the previous post, we've seen how to stream, transform and combine data using Akka Streams on Play Framework.
Now let's see how to use a queue in which we can post new data dynamically. This can be very useful if you want to plug your application on a third party API that is sending events periodically and that is not using Akka Streams or Reactive Streams to emit data.

This is the common way to use an Akka Streams SourceQueue :

```
val bufferSize = 10

val queue = (bufferSize, OverflowStrategy.fail)
  .to(Sink.foreach(println))
  .run()

for (i ← 1 to 3) {
  queue.offer(i)
}
```

But our case is a little more complex. Play controller needs a Source to be able to stream a response using the chunked method.
As Play uses its own Akka Stream Sink (i.e. a reactive data consumer) under the hood, we can't materialize the source queue ourselves using a Sink because the source would be consumed before it's used by the chunked method.

The solution is to use `mapMaterializedValue` on the source returned by `Source.queue` to get a future of its queue materialization.

Here is an helper method to do it (thanks to the Akka team who gave me this hint on github) :

```scala
 //T is the source type, here String
 //M is the materialization type, here a SourceQueue[String]
 def peekMatValue[T, M](src: Source[T, M]): (Source[T, M], Future[M]) = {
   val p = Promise[M]
   val s = src.mapMaterializedValue { m =>
     p.trySuccess(m)
     m
   }
   (s, p.future)
 }
```

We will use an Akka scheduler to simulate an API that would produce data periodically. To begin, let's declare the actor that will post in our queue :

```scala

  val Tick = "tick"

  class TickActor(queue: SourceQueue[String]) extends Actor {
    def receive = {
      case Tick => queue.offer("tack")
    }
  }
```

Note that this method can materialize any kind of source, not only a queue source.

Now we can use this helper with a queue source and start a scheduler to post data into the queue using the TickActor :

```scala
def queueAction = Action {

  val (queueSource, futureQueue) = peekMatValue(Source.queue[String](10, OverflowStrategy.fail))

  futureQueue.map { queue =>

    val tickActor = actorSystem.actorOf(Props(new TickActor(queue)))
    val tickSchedule =
      actorSystem.scheduler.schedule(0 milliseconds,
        1 second,
        tickActor,
        Tick)

      queue.watchCompletion().map{ done =>
        Logger.debug("Client disconnected")
        tickSchedule.cancel
        Logger.debug("Scheduler canceled")
      }
  }

  Ok.chunked(queueSource via EventSource.flow)

}
````


If you close your browser tab, the source queue will be canceled automatically.
As we catch the `watchCompletion` event, we can also stop the Akka scheduler. Then you will see the following logs in your console :

```
[debug] application - queue source : tack
[debug] application - queue source : tack
[debug] application - queue source : tack
[debug] application - Client disconnected
[debug] application - Scheduler canceled
```

You can see the full code of this example [here](https://gist.github.com/loicdescotte/3914f3fd6513cb85ea1638b60b444f9d).
