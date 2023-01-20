---
layout: post
title: How to define a flex String decoder with ZIO JSON
tags:
 - JSON
 - Scala
 - ZIO
---

I am using ZIO JSON to decode responses from a Web service and I've faced an issue : some objects have fields that can sometimes be strings and sometime be integers.

This can be solved with decoders `orElse` method like this : 

```scala
case class SoftwareVersion(
                            name: String,
                            // ...
                          )

case class SoftwareVersionWithIntName(
                            name: Int,
                            // ...
                          )

implicit val softwareVersionDecoder = DeriveJsonDecoder.gen[SoftwareVersion].orElse(
  DeriveJsonDecoder.gen[SoftwareVersionWithIntName].map(decoded =>
    SoftwareVersion(decoded.name.toString)
  )
)
```

It is working but if there is a lot of changing fields, it may lead to a lot of cases to manage.

To make it work at any level with the minimum amount of code, you can hack the String decoder itself : 

```scala
implicit val flexStringDecoder: JsonDecoder[String] = JsonDecoder[String].orElse(
  JsonDecoder[Int].map(decoded => decoded.toString)
)
```
And voilà! 

The same trick should also work with Circe.
