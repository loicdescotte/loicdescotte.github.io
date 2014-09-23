---
layout: post
title: Playing with Twitter streams
tags:
 - Play Framework
 - Scala
 - Streaming
 - Twitter
 - Iteratee
---


I've updated a POC I made two years ago, about mixing and streaming some Twitter searches with Play Framework.

The new version uses the Twitter streaming API with OAuth authentication. The number of Twitter search queries is dynamic and the results are pushed through Server Sent Events.

## MixedTweets

We will see how to mix several searches from the Twitter streaming API and push the results to the browser in real time using SSE.

All the code is available in this [mini project](https://github.com/loicdescotte/MixedTweets-2014).


###Controller

We define a stream method in our Controller :

{% highlight scala %}
def stream(query: String) = Action {
    val queries = query.split(",")

    val streams = queries.map { query => 
      val twitterListener = new TwitterStreamListener(query, config)
      twitterListener.listenAndStream
    }

    val mixStreams = streams.reduce((s1,s2) => s1 interleave s2)

    val jsonMixStreams = mixStreams through upperCaseJson
    Ok.chunked(jsonMixStreams through EventSource()).as("text/event-stream")  
} 
{% endhighlight %}

For each query, we create a stream (an Enumerator in the Iteratee API). Then we can `reduce` the streams into only one mixed stream using the `interleave` method.
The two last lines will be explained later.

In the `TwitterStreamListener` class, we create an Enumerator from a Twitter search. We use Twitter4J to handle authentication and Twitter searches : 

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

`Concurrent.broadcast` is useful to feed an Enumerator via an input channel. The resulting Enumerator contains tuples of search queries and Twitter status.

#### Adapt the content with an enumeratee

An Enumeratee is a kind of adapter in the Iteratee API. We will use this to transform the results sent to the browser.
The results will be converted into JSON values, with upper case messages :

{% highlight scala %}
val upperCaseJson : Enumeratee[(String, TwitterStatus), JsValue] = Enumeratee.map[(String,TwitterStatus)] { case (searchQuery, status) =>
  Json.obj("message" -> s"$searchQuery : ${status.getText}", "author" -> status.getUser.getName)
}
{% endhighlight %}

Finally we can stream the result using SSE :

{% highlight scala %}
val jsonMixStreams = mixStreams through upperCaseJson
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
