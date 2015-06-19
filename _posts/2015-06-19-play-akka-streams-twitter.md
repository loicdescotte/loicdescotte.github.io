---
layout: post
title: Playing with Akka Streams and Twitter
tags:
 - Play Framework
 - Scala
 - Reactive-Streams
 - Akka streams
---

You may have heard of [Reactive-Streams](http://www.reactive-streams.org/), a specification inspired by the [Reactive Manifesto](http://reactivemanifesto.org/).  
A lot of big actors are involved in this initiative, like Typesafe, Netflix, Pivotal, RedHat and Twitter.  
Its goal is to "provide a standard for asynchronous stream processing with non-blocking back pressure".  
In a few words, back pressure is the ability for a data producer to adapt the data transmission rate depending on the speed of the consumer, to avoid overwhelming it.

This article will focus on Akka Streams, which is an implementation of Reactive-Streams that relies on Akka actors andu provides an higher level API on top of them.

Play 3 will rely on Akka-Streams under the hood, and it will also allow to benefit from it to to handle data streams reactively, like we are doing today with Play 2 and the Iteratee API.  
Akka Streams is still tagged as "experimental", but the 1.0 version is close. With Play 2.4, it's already possible to use it, via Reactive-Streams to Iteratee and Iteratee to Reactive-Streams conversions.

## MixTWeets, updated

In a [previous post](http://loicdescotte.github.io/posts/mixtweets/), we've seen how to handle Twitter message streams reactively, how to merge several Twitter searches in a unique stream and push this into an EventSource (Server Sent Event) output with the Iteratee API.  
Now, we will do the same with Akka Streams and compare this two solutions.  

### How to define a source

In Akka Streams, data producers are called sources. It's the equivalent of the `Publisher` interface in the Reactive-Streams specification.  
We are using twitter4j and Twitter Streaming API to receive messages. For each new message, we need to push data into a source.  
To achieve this, we need a reference to an actor implementing the `ActorPublisher` trait. We also need to bind the source to this actor.  
Then, we can push some data into the actor and retrieve this data from the source.

{% highlight scala %}
def listenAndStream = {

    val (actorRef, publisher) =  Source.actorRef[TweetInfo](1000, OverflowStrategy.fail).toMat(Sink.publisher)(Keep.both).run()

    Logger.info(s"#start listener for $searchQuery")
 
    val statusListener = new StatusListener() {
 
      override def onStatus(status: TwitterStatus) = {   
       Logger.debug(status.getText)
       //push elements into a publisher
       actorRef ! TweetInfo(searchQuery, status.getText, status.getUser.getName)
      }
 
    }
 
    twitterStream.addListener(statusListener)
    twitterStream.filter(query)
    
    Source(publisher)
}
{% endhighlight %}


The first line creates an actor reference and a publisher. Now we can keep references to the actor and to the source. When we send elements to the actor (with the `!` method), it will dynamically add elements in the source.  We must choose a size for the actor message queue (1000) and an `OverflowStrategy` to handle cases where this size is exceeded.  

Note that we need to materialize the source before being able to use the actor behind it.  
The source is materialized into a publisher, and then we need to redefine a source from this publisher.  
This trick is helps us to solve a problem that is specific to our use case : we don't want to consume the source right now because we want to keep it for later, and merge it with other sources. Fortunately, this kind of use case should be simplified in future versions of Akka streams (an issue is opened in Github to address this).

In simpler cases, you can just declare a `Source`, a `Flow` (i.e. a bridge that takes an input a produces an output) and define a way to consume the data in the same time. Using directly a `Sink`, the source will be materialized so you can get a reference to an actor.  

Here is an example that just prints messages matching a pattern : 

{% highlight scala %}
val source = Source.actorRef[Message](1000, OverflowStrategy.fail)

val helloSource = source.filter(message => message.text.startsWith("Hello"))

val ref = Flow[Message].to(Sink.foreach(println)).runWith(helloSource)

ref ! Message("Hi there!")
ref ! Message("Hello there!")
{% endhighlight %}


If you need more customization, it's also possible to create a custom `ActorPublisher` to do this.  


In comparison, with Iteratee we would do things like that :

{% highlight scala %}
val (enum, channel) = Concurrent.broadcast[TweetInfo]

override def onStatus(status: TwitterStatus) = {  
  channel push TweetInfo(searchQuery, status.getText, status.getUser.getName)
}

{% endhighlight %}

The `(enum, channel)` tuple is quite similar to `(actorRef, publisher)`.  
We would also push new elements into the channel to feed the enumerator (i.e the data producer).

### How to transform and merge sources

As we're merging several Twitter searches, we have in result several sources. We will use a merge to have a single stream, containing all messages, so we can consume them more easily.

{% endhighlight %}
// get an Array of Source from an array of Twitter search queries
val streams = queries.map { query => 
    val twitterStreamListener = new TwitterStreamListener(query, config)
    twitterStreamListener.listenAndStream 
}

//merge streams in a single stream
val mergedStream = Source[TweetInfo]() { implicit builder =>

  val merge = builder.add(Merge[TweetInfo](streams.length))

  for (i <- 0 until streams.length) {
    builder.addEdge(builder.add(streams(i)), merge.in(i))
  }

  merge.out
}

//transform to JSON
val toJson = (tweet: TweetInfo) => Json.obj("message" -> s"${tweet.searchQuery} : ${tweet.message}", "author" -> s"${tweet.author}")

val jsonStream = mergedStream.map(tweets => toJson(tweets))
{% endhighlight %}

Note : At this stage, we could consume messages directly using a sink (an Akka Stream data consumer), for example `source.runWith(Sink.foreach(println))`

In comparison, with Iteratee would transform and merge streams like that :

{% highlight scala %}
// get an Array of Enumerator from an array of Twitter search query
val streams = queries.map { query => 
  val twitterListener = new TwitterStreamListener(query, config)
  twitterListener.listenAndStream
}

//merge streams in a single stream
val mixStreams = streams.reduce((s1,s2) => s1 interleave s2)

//transform to JSON
val toJson : Enumeratee[TweetInfo, JsValue] = Enumeratee.map[TweetInfo] { case tweet =>
  Json.obj("message" -> s"${tweet.searchQuery} : ${tweet.message}", "author" -> s"${tweet.author}")
}

val jsonMixStreams = mixStreams through toJson
{% endhighlight %}

The advantage of Akka Streams here is that we don't need to use an intermediate structure like Enumeratee, we can just use a good old `map` function.
Right now, the merge is really simpler with Iteratee. But there is already an issue opened in github (yes, another one) to be able to merge a stream of streams in a simple line, like `streams.flatten(FlattenStrategy.concat)`.


### Push to EventSource

To be compatible with Play controllers, we still need to use Enumerator and Iteratee objects.  
Then we can use conversions included in Play 2.4 :  

{% highlight scala %}
val jsonEumerator : Enumerator[JsValue] = sourceToEnumerator(jsonStream)
Ok.chunked(jsonEumerator through EventSource()).as("text/event-stream")  
{% endhighlight %}

That'all!  
You can see all the code (including details of the `sourceToEnumerator` method) in this [github project](http://github.com/loicdescotte/MixTweets-AkkaStreams).

You can create an application account to configure the Twitter API client on the [Twitter apps page](https://apps.twitter.com/).  
Then open your browser on `http://localhost:9000/liveTweets?query=java&query=ruby` (for example) to see the stream live!


In conclusion, I've tried to note a few good/bad points for each library : 

Akka-stream  
-----------

++ Very flexible, while the API is quite high level, we can easily go to a lower level with actors for more specific needs  
++ It's easy to defines flow and transformations on sources (need to use an Enumerattee to transform an input (Enumerator) in Iteratee API)  
-- A little more verbose / needs more boilerplate code for some basic stuffs. But Akka Streams is very young and will continue to provide new features quickly


Iteratee
--------

++ Very good for this use case (broadcast and channel are very handy, merge is very easy)
-- Low level / complex cases can be more difficult  

Both are very good stream processing libraries, with high level API and good capabilities to handle backpressure.

Note that I'm just beginning with Akka Streams, if you find better ideas to achieve this, please tell me :)  
