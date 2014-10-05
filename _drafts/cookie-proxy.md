---
layout: post
title: "Proxying Web Services"
description: ""
category: monads
tags: [monads, web services]
---
In this article I'll show how we can use a monad to automatically marshall cookies through a intermediate system. This will allow the intermediate system to proxy web service calls from a user through to a backend system. The monad will have two advantages, it will allow us to focus our attention on the request/response bodies, and it will allow us to chain together multiple back end web services requests to service a single front end request.

In order to act as a proxy for the end user when we make successive calls to backend web services we should collect any cookies from those responses. Further we should behave as if we are the end user, sending in the cookie values set in a response to subsequent requests to the same service. What we have is:

1. A process to happen (request to response),
2. To chain together these processess,
3. Some plumbing to happen when we chain together processes (cookies are handled).

What our monad will do is allow us to create the basic request processes, and to automatically handle chaining the requests together so that:

1. Subsequent calls contain the right cookies,
2. The final response sent back to the end user gets all the cookies set by each of the backend calls.

The code will:

1. Have reusable logic for marshalling cookies around,
2. Allow us to focus on the request values and response values (bodies) and tuck the cookie handling neatly behind the scenes.

Our final result will allow us to write code like:

    for {
        sku <- getSku(skuId)
        quantityToBuy = sku.price / amountToSpend
        addToCartResponse <- addToCart(skuId, quantityToBuy)
    } yield addToCartResponse
        

All call to our back end web services may return cookies in its response. For example, it might provide a new session cookie. The requires are:

1. The end user recieves any new/updated cookie values,
2. Subsequent calls to the back end services provide properly updated cookies.
    

Our final result will look like

    for {
      cart <- getCart
      firstItem = cart.items.headOption
      response <- firstItem match {
        case None => emptyCartResponse
        case Some(item) => removeItem(item)
      }
    } yield response

The response will contain any cookies returned from the `getCart` call, as well as (and overridden by) any cookies from the second `removeItem` call.


## Request and Response with Cookies

For convenience we'll use libraries from the Play Framework (Scala API) to implement our cookie proxying monad.

We'll create some basic classes, the first to represent a request with a typed body, and some cookie values:

    case class CRequest[+T](cookies: CookieValues, body: T)

`CookiesValues` is just an alias:
    
    type CookieValues = Map[String, String]

that respresents the key-value sequence provided in the `Cookie` header of a request.


Another similar class to represent a response with cookies:

    case class CResponse[+T](cookies: Seq[WSCookie], value: T)

`WSCookie` is `play.api.libs.ws.WSCookie`, which represents the data associated to a cookie in a `Set-Cookie` response header, including name, value, domain, whether it is secure only, etc.

### Strongly Typed Representations

You'll note we could have represented the request and response bodies as just `String`s (or perhaps `Option[String]` in the case of a request). By using the type parameter `T` we can express some constraints on our requests and responses. For example, we may expect that our response body is valid JSON. Then we could write `CResponse[JsValue]` using Play's JSON value type. We would likely have some code that takes the response body string, parses it as JSON, and throws an exception if it can not be parsed.

Taking this a little further we may expect that the response is not only JSON, but that it is in a particular form. We could use Play's JSON macros parse that form, for example:

    case class SkuResponse(skuId: Integer, name: String, price: BigDecimal)

    object SkuResponse {
        implicit val format = Json.format[SkuResponse]
    }

Now a web service response in this format could be typed `CResponse[SkuResponse]`.

In order to facilate operations on the body of a request or response we'll add `map` methods to the `CRequest` and `CResponse` classes.

    case class CRequest[+T](cookies: CookieValues, body: T) {
      def map[U](f: T => U) = CRequest(cookies, f(body))
    }

    case class CResponse[+T](cookies: Seq[WSCookie], value: T) {
      def map[U](f: T => U) = CResponse(cookies, f(value))
    }

`map` makes it easy to parse a raw string body (e.g. `rawResponse: CResponse[String]`) into a stongly typed response:

    val r: CResponse[SkuResponse] = rawResponse.map(Json.parse).map(_.as[SkuResponse])
    
Notice that the `map` methods just pass along the same cookie values untouched. The `CResponse[SkuResponse]` produced by the above line will carry the same cookies that the original `String` response had. Methods like `map` provide the ability to focus our code on the response body while having the cookies automatically carried along for the ride.


## CookieContext

To represent the current cookie state we'll need two things:

1. The original key-value cookie mapping provided by the initial user request,
2. The values for the `Set-Cookie` response headers.

    case class CookieContext(ambient: CookieValues, local: Seq[WSCookie]) {
      def cookieValues: CookieValues = {
        ambient ++ local.toValues
      }
    }

For an intermediate state between backend requests `cookieValues` method will provide the appropriate key-value cookie map obtained from the original provided cookies, updated with any `Set-Cookie` responses from preceeding backend requests. When we call a backend service the cookies we set in the request header will be obtained from this method.

## The WebCall class

Now we get to the class for our monad. `WebCall[T]` will represent a web service call that returns a response body of type `T`.

    class WebCall[T](val withCookies: CookieContext => Future[CResponse[T]])

A `WebCall[T]` instance is an asynchronous operation (hence the `Future`) that takes a `CookieContext` and produces a response-with-cookies that has a body of type `T`.


The API we are aiming to have would provide us with methods like:
    
    def get(url: String): WebCall[String]
    
    def post(url: String)(body: String): WebCall[String]
    



 




