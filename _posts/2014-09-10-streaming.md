---
layout: post
title: Streaming and proxying with Play
tags:
 - Play Framework
 - Scala
 - Streaming
---

You'll find bellow a few streaming examples with Play Framework, a kind of small streaming/proxying cheat sheet :)

This examples use the Iteratee API. The main elements from this API are the iteratees (data consumers) and the enumerators (data producers).
If you want to learn more about this API I recommend to read [this post](http://mandubian.com/2012/08/27/understanding-play2-iteratees-for-normal-humans/).

# Stream a file (and transform)

## Chunck by chunk

{% highlight scala %} 
def  transform = Action {

     import play.api.libs.iteratee._
    
     val fileStream: Enumerator[Array[Byte]] = {
         Enumerator.fromFile(new File("data.txt"))
     }
     
     val transfo = Enumeratee.map[Array[Byte]]{byteArray =>  
         val chunckedString = new String(byteArray)
         //add blablabla on each line
         val newChunk = chunckedString.replaceAll("\n", " blablabla\n")         
         //newChunk <-- stream text
         newChunk.getBytes //<-- stream file
     }
               
     Ok.chunked(fileStream.through(transfo))            
}
{% endhighlight %}

## Line by line

{% highlight scala %} 
def  transform = Action {

    import play.api.libs.iteratee._

    lazy val bufferedReader =  new BufferedReader(new InputStreamReader(new FileInputStream("data.txt")))
     
     val fileStream : Enumerator[String] = Enumerator.generateM[String] {
        scala.concurrent.Future{
          val line: String = bufferedReader.readLine()
          Option(line)  
        }
      }
     
     val transfo = Enumeratee.map[String]{line =>  
         val newLine = line + " blablabla" + "\n"
         newLine.getBytes
     }
               
    Ok.chunked(fileStream.through(transfo)) 
}
{% endhighlight %}

## HTTP Proxying

In this proxying example we will call an external service in streaming from a Play server, and stream the response to our HTTP clients.

{% highlight scala %}

def streamFromWS = Action.async { request =>

    import play.api.libs.iteratee._
    import scala.concurrent.Promise
    import play.api.mvc.ResponseHeader
    import play.api.mvc.Result
    import play.api.libs.ws.{ WSResponseHeaders, WS }
    import play.api.libs.iteratee.Concurrent.joined

    def consumer(promiseToFeed: Promise[(WSResponseHeaders, Enumerator[Array[Byte]])]) = { rs: WSResponseHeaders =>
      val (wsConsumer, stream) = joined[Array[Byte]]
      promiseToFeed.success((rs, stream))
      wsConsumer
    }

    val resultPromise = Promise[(WSResponseHeaders, Enumerator[Array[Byte]])]
    WS.url("http://dumps.wikimedia.org/simplewiki/latest/simplewiki-latest-pages-articles.xml.bz2").get(consumer(resultPromise)).map(_.run)
    
    //now you can do whatever you want with your enumerator : combine with another one, transform with an enumeratee, etc : 
    //val dataEnumeratorFuture = resultPromise.future.map(stream => stream._2)
    //dataEnumeratorFuture.map(Ok.chunked(_))
    
    //OR if you need to setup response headers
    resultPromise.future.map {
      case (rs, stream) =>
        Result(
          header = ResponseHeader(
            status = OK,
            headers = Map(
              CONTENT_LENGTH -> rs.headers.get("Content-Length").map(_.head).get,
              CONTENT_DISPOSITION -> s"""attachment; filename="wikipedia.xml.bz2"""",
              CONTENT_TYPE -> rs.headers.get("Content-Type").map(_.head).getOrElse("binary/octet-stream"))
          ),
          body = stream
        )
    }
}

{% endhighlight %}


This one deserves an explanation. `Concurrent.joined` creates an iteratee/enumerator tuple. According to the documentation, "When the enumerator is applied to an iteratee, the iteratee subsequently consumes whatever the iteratee in the pair is applied to. Consequently the enumerator is "one shot", applying it to subsequent iteratees will throw an exception.".

WS.url accepts a `WSResponseHeaders => Iteratee` function to consume the data from the remote service.
In the `consumer` method we feed the promise with the enumerator (stream) created from the WS result. Then we can stream a response from the enumerator.

(*) This example is inspired by an example from Yann Simon on Github 

----------------

Update : Since Play 2.3, WS provides a `getStream` method returning what we're trying to achieve, i.e. a `Future[(WSResponseHeaders, Enumerator[Array[Byte]])]`. (Thanks Martin for the comment).

----------------

If you want to mix several data sources, you can look at [this example](https://gist.github.com/loicdescotte/3266376), the post is a bit old but the principle remains the same.
