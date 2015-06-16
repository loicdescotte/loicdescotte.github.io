---
layout: post
title: Playing with Twitter streams
tags:
 - Play Framework
 - Scala
 - Streaming
 - Twitter
 - Iteratee
 - Real time Web
---

I've updated a POC I made two years ago, about mixing and streaming some Twitter searches with Play Framework.

The new version handles Twitter authentication through OAuth. The number of parallel Twitter search queries is dynamic and the results are pushed to the browser in real time through Server Sent Events.
Instead of using the Play Web Service API to call the Twitter REST API, this version uses Twitter4J and the Twitter Streaming API (the connection stays open to retrieve new arriving Tweets).

## MixedTweets github project

All the code is available in this [mini project](https://github.com/loicdescotte/MixedTweets-2014).
The old version using Play WS API is available [here](https://github.com/loicdescotte/Play2-MixedTweets).

Let's see what the code looks like.

### Controller

We define a stream method in our Controller :

{% highlight scala %}
def stream(query: String) = Action {
  val queries = query.split(",")

  val streams = queries.map { query => 
    val twitterListener = new TwitterStreamListener(query, config)
    twitterListener.listenAndStream
  }

  val mixStreams = streams.reduce((s1,s2) => s1 interleave s2)

  val jsonMixStreams = mixStreams through toJson
  Ok.chunked(jsonMixStreams through EventSource()).as("text/event-stream")  
} 
{% endhighlight %}

For each query, we create a stream (an Enumerator in the Iteratee API). Then we can `reduce` the streams into only one mixed stream using the `interleave` method.
The last two lines will be explained later.

In the `TwitterStreamListener` class, we create an Enumerator from a Twitter search. We use Twitter4J to handle authentication and Twitter searches (via the Twitter streaming API) : 

{% highlight scala %}
class TwitterStreamListener(searchQuery: String, config: Configuration) {
 
  val query = new FilterQuery(0, Array(), Array(searchQuery))
 
  val twitterStream = new TwitterStreamFactory(config).getInstance
 
  def listenAndStream = {
    Logger.info(s"#start listener for $searchQuery")
 
    val (enum, channel) = Concurrent.broadcast[(String, TwitterStatus)]
 
    val statusListener = new StatusListener() {
 
      override def onStatus(status: TwitterStatus) = {      
        Logger.debug(status.getText)  
        channel push (searchQuery, status)
      }

    //...
 
    }
 
    twitterStream.addListener(statusListener)
    twitterStream.filter(query)
    enum
  }
 
}
{% endhighlight %}

`Concurrent.broadcast` is useful to feed an Enumerator via an input channel.
When a new message arrives, we push the Twitter status into the channel (see `onStatus` method).
The resulting Enumerator contains tuples of search queries and Twitter status.

### Adapt the content with an Enumeratee

An Enumeratee is a kind of adapter in the Iteratee API. We will use it to transform the results sent to the browser.
The results will be converted into JSON values :

{% highlight scala %}
val toJson : Enumeratee[(String, TwitterStatus), JsValue] = Enumeratee.map[(String,TwitterStatus)] { case (searchQuery, status) =>
  Json.obj("message" -> s"$searchQuery : ${status.getText}", "author" -> status.getUser.getName)
}
{% endhighlight %}

Finally we can stream the result over SSE using an `EventSource` Enumeratee, with a `text/event-stream` Content-Type :

{% highlight scala %}
val jsonMixStreams = mixStreams through toJson
Ok.chunked(jsonMixStreams through EventSource()).as("text/event-stream")  
{% endhighlight %}

Now we just have to make the stream alive on the browser (see `index.scala.html`).

Note : To configure the Twitter API client you need to declare an application on the [Twitter apps page](https://apps.twitter.com/).
Then use your credentials as follows : 

{% highlight scala %}
val cb = new ConfigurationBuilder()
cb.setDebugEnabled(true)
  .setOAuthConsumerKey("xxx")
  .setOAuthConsumerSecret("xxx")
  .setOAuthAccessToken("xxx")
  .setOAuthAccessTokenSecret("xxx")

val config = cb.build 
{% endhighlight %}


Enjoy!!
