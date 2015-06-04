---
layout: post
title: Compile Time Dependency Injection with Play 2.4
tags:
 - Play Framework
 - Scala
 - Dependency injection
 - Compile time
---

Play Framework, in its new version, provides a lot of new stuff to handle dependency injection at runtime, using Guice.  
With Scala, I always prefer using compile time dependency injection, as it allows to see errors as soon as possible. I must also admit that I find compile time DI a lot more elegant at it needs no container or proxy at runtime!

Fortunately, Play team also added the ability to control the way routes, controllers and other components are binded together at compile time.

## Controller and service

In this example, we have a controller that take a `LinkService` dependency.  
We want to be able to mock this service in our controller tests.

{% highlight scala %}
class Application(linkService: LinkService) extends Controller {

  def findLinks(query: String) = Action.async {
    val links = linkService.findLinks(query)
    links.map(response => Ok(views.html.index(response)))
  }

}
{% endhighlight %}


The route to this controller is defined as follows : 

{% highlight scala %}
# Home page
GET     /                           controllers.Application.findLinks(query: String)
{% endhighlight %}


## How to wire this

To have a fully functional application we need to tell the Play router how to find our `LinkService` dependency and how to wire it into our controller.

This can be done by defining a custom ApplicationLoader :


{% highlight scala %}
class SimpleApplicationLoader extends ApplicationLoader {
  def load(context: Context) = {
    new ApplicationComponents(context).application
  }
}

object ControllerDependencies {
  lazy val logService = new LogService
  lazy val linkService = new LinkService(logService)
}

class ApplicationComponents(context: Context) extends BuiltInComponentsFromContext(context) {  
  lazy val applicationController = new controllers.Application(ControllerDependencies.linkService)
  lazy val assets = new controllers.Assets(httpErrorHandler)
  lazy val router = new Routes(httpErrorHandler, applicationController, assets)
}
{% endhighlight %}

To enable this configuration we need to add this line in application.conf :  

`play.application.loader=SimpleApplicationLoader`  

And this line in build.sbt :  

`routesGenerator := InjectedRoutesGenerator`

## How to test it

To test our controller, we can easily wire a mock linkService = 

{% highlight scala %}
class ControllerSpec extends Specification with Mockito {

  val mockLinkService = mock[LinkService]
  mockLinkService.findLinks("hello") returns Future("http://coucou.com")

  "Application" should {

    "display query results" in {

      val app = new Application(mockLinkService)
      val response = app.findLinks("hello")(FakeRequest())
      
      status(response) must equalTo(OK)
      contentType(response) must beSome("text/html")
      contentAsString(response) must contain("http://coucou.com")

    }
  }
}
{% endhighlight %}

To be able to do test the routes direclty with our real services, e.g. for integration tests, we need to define our own 'WithApplication' helper : 

{% highlight scala %}
class WithDepsApplication extends WithApplicationLoader(new SimpleApplicationLoader)
{% endhighlight %}

Then, we can use it in our tests : 

{% highlight scala %}
@RunWith(classOf[JUnitRunner])
class ApplicationSpec extends Specification {

  "Application" should {

    "render the index page" in new WithDepsApplication{
      val home = route(FakeRequest(GET, "/?query=hello")).get

      status(home) must equalTo(OK)
      contentType(home) must beSome.which(_ == "text/html")
      contentAsString(home) must contain ("Results :")
    }
  }
  
}{% endhighlight %}


You can find all the sources in [this Github project](https://github.com/loicdescotte/play24SimpleDI).

Note that you can also use macwire macros to wire automatically services, controllers and routes. A good example can be found [here](https://github.com/gmethvin/play-macwire-di).


Next time we will see another new feature of Play 2.4, its (experimental) integration with Akka-Streams!