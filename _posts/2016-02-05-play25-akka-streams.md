---
layout: post
title: Akka Streams integration in Play Framework 2.5
tags:
 - Play Framework
 - Scala
 - Reactive-Streams
 - Akka Streams
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
    }.limit(100)).as("application/json")
  }

  private def prefixAndAuthor = {
      // random message and author
  }
```

The `tick` method creates a simple source with a 1 second delay between 2 message emissions. We can transform this source to get a message per second, related to a keyword using `map`.
Finally we limit the feed to 100 messages and send it as a JSON stream.

## A tweet consumer

Now we need another Web application that will be plugged on our fake Twitter service to make several queries in parallel and mix the results in a single source. This source will be streamed to the browser using Server Sent Events.

```scala
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

  val sourceFuture = Future.sequence(sourceListFuture).map(Source(_).flatMapMerge(10, identity).map(_.toJson))
  sourceFuture.map { source =>
    //hack for SSE before EventSource builder is integrated in Play
    val sseSource = Source.single("event: message\n").concat(source.map(tweetInfo => s"data: $tweetInfo\n\n"))
    Ok.chunked(sseSource).as("text/event-stream")
  }
}
```


Play WS API `stream` method returns an Akka Streams Source of Akka ByteString via the `body` value of the response. The response itself is wrapped in a Future.
As the Twitter service may send several messages in a single chunk, we need to split the message on line breaks.
We can also imagine that the Twitter service could send chunks with truncated messages. In this case the messages need to be saved in a buffer until we reach a line break.
Fortunately, the `Framing` object does all the job for us. We just need to provide a separator (line break) and a max frame length for the source elements. This is a great improvement from Akka Streams 1, which was forcing developers to write a custom line parser.

Finally the sources can be merged using the `flatMapMerge` method and then be transformed to the new desired JSON format.  In this case we just add the query in the response to ease filtering on the client side.

Quite easy isn't it?

You can find all the sources of this example (including the frontend code) [here](http://github.com/loicdescotte/touiteur).
