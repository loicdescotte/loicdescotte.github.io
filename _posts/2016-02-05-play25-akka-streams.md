---
layout: post
title: Akka Streams integration in Play Framework 2.5
tags:
 - Play Framework
 - Scala
 - Reactive-Streams
 - Akka Streams
---

---

Update (February 10, 2016) : Using latest Play 2.5 SNAPSHOT to benefit from EventSource helper

---

From version 2.5, Play's default stream processing library is Akka Streams. Akka Streams is an implementation of the [Reactive Streams](http://www.reactive-streams.org/) standard.
With Play 2.4 it was possible to use Akka Streams via bindings to the Play Iteratee lib (see [this post](http://loicdescotte.github.io/posts/play-akka-streams-twitter)).
Now we don't need this Iteratee bindings anymore, we can use Akka Streams natively in Play controllers. Play 2.5 also comes with Akka Streams 2, which is easier to use and in many cases faster than the first version.

## A fake tweet API

In this example, we will create a fake Twitter service with Play. This service will stream messages using an Akka Streams Source :

```scala
  def timeline(keyword: String) = Action {
    val source = Source.tick(initialDelay = 0 second, interval = 1 second, tick = "tick")
    Ok.chunked(source.map { tick =>
      val (prefix, author) = prefixAndAuthor
      Json.obj("message" -> s"$prefix $keyword", "author" -> author).toString + "\n"
    }.limit(100))
  }

  private def prefixAndAuthor = {
      // random message and author
  }
```

The `tick` method creates a simple source with a 1 second delay between 2 message emissions. We can transform this source to get a Json message per second, related to a keyword using `map`. Finally we are limiting the feed to 100 elements.

## A tweet consumer

Now we need another Web application that will be plugged on our fake Twitter service to make several queries in parallel and mix the results in a single source. This source will be streamed to the browser using Server Sent Events.

```scala
case class TweetInfo(searchQuery: String, message: String, author: String) {
  def toJsonString = Json.stringify(Json.obj("message" -> s"${this.searchQuery} : ${this.message}", "author" -> s"${this.author}"))
}

def stream(query: String) = Action.async {
  val sourceListFuture = query.split(",").toList.map { query =>
    val futureTwitterResponse = WS.url(s"http://localhost:9000/timeline/$query").stream
    futureTwitterResponse.map { response =>
      response.body.via(Framing.delimiter(ByteString("\n"), maximumFrameLength = 100, allowTruncation = true).map(_.utf8String)).map { tweet =>
        val json = Json.parse(tweet)
        TweetInfo(query, (json \ "message").as[String], (json \ "author").as[String])
      }
    }
  }

  val sourceFuture = Future.sequence(sourceListFuture).map(Source(_).flatMapMerge(10, identity).map(_.toJsonString))
  sourceFuture.map { source =>
    Ok.chunked(source via EventSource.flow)
  }
}
```


Play WS API `stream` method returns an Akka Streams Source of Akka ByteString via the `body` value of the response. The response itself is wrapped in a Future.
As the Twitter service may send several messages in a single chunk, we need to split them on line breaks.
We can also imagine that the Twitter service could send chunks with truncated messages. In this case the messages need to be saved in a buffer until we reach a line break.
Fortunately, the `Framing` object does all the job for us. We just need to provide a separator (line break) and a max frame length for the source elements. This is a great improvement from Akka Streams 1, which was forcing developers to write a custom line parser.

Then the sources can be merged using the `flatMapMerge` method and be transformed into the new desired format. In this case we're just adding the query in the response to ease filtering on the client side. Finally, `EventSource.flow` method helps us to format the messages into the Server Sent Event format. And the stream can flow!  

Quite easy isn't it?

You can find all the sources of this example (including the frontend code) [here](http://github.com/loicdescotte/touiteur).
