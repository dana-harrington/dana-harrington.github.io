---
layout: post
title: "Creating a Monadic Rendering Pipeline"
description: ""
category: monads
tags: [monads]
---
{% include JB/setup %}
In this article I'll describe how the task of translating a JSON document to an HTML web page can be modularized with the use of monads. The monadic abstraction will allow us to add a variety of powerful features to our rendering pipeline, including validation, asynchrony, and dependency accumulation. This approach will let us write code that broadly focuses on the main task at hand: transforming a JSON document to an HTML document, while giving us the ability to augment our rendering pipeline with additional useful features.

# Introduction
In this article I'll outline the solution to a cache issue that arose during the final pre-release stage of a major e-commerce that I've helped to develop. The site is written in Scala and the Play Framework. My goal in writing this article is to demonstrate how monads and functional programming techniques can be used to solve real world software issues. I'll do this by showing how a monadic approach allowed use to abstract process that renders JSON content into HTML pages. Abstracting the details of the process allowed us to add features such as validation and asynchrony to this rendering process without cluttering the code.

My target audience is the working Scala developer, they have a basic knowledge of Scala, including familiarity with for-yield syntax, however no deep knowledge of functional programming is assumed. 
My hope is to share with developer who are new to functional programming some of the reasons why monads are of interest in solving practical programming problems. At the risk of being pedantic I will go into detail on the basic structure of our application. My hope is that concrete approach will help to illustrate the benefits of the solution I discuss here.

The content here is based on a talk I gave at the Toronto Scala meet-up group in April, 2014.


# The Problem Domain

Let's get some of the basics out of the way. In our application these classes are defined as part of the Play Framework, but to keep things self contained we'll stub them out here:

```
class Html

object Html {
  def apply(html: String): Html = ???

  def apply(html: scala.xml.Elem): Html = ???
}

// A JSON value
class JsValue

// A HTTP response
case class Result(code: Int,
                  content: Html,
                  headers: Map[String, String])
```

In our application the primary source of content is Endeca. In the common case we will construct an Endeca query based on the current request and Endeca will respond with a JSON document ('view advice') that we will transform into an HTML response.

```
object EndecaService {
  def getAdvice(query: String): ViewAdvice = ???
}

class ViewAdvice(json: JsValue) {
  val templateName: String = ???

  def zone( name: String ): Seq[JsValue] = ???
}
```
	
The first item of interest in the view advice is the template name. A template determines the overall structure of the resulting web page. Each template has a number of 'zones' that can be filled with content. The second piece of information in the view advice is a mapping from zone name to a sequence of JSON values which will supply the content for that zone. Each such JSON value in the sequence will correspond to a Cartridge.

Each Cartridge renders out to a fragment of HTML. Each Template can be rendered to a Result (typically with a status code of 200 and an HTML body).

```
trait Renderable[T] {
  def render: T
}

trait Cartridge extends Renderable[Html]

trait Template extends Renderable[Result]
```

It will be useful to provide a couple sample implementations to work with. A simple cartridge that just contains some title text:

```
case class TitleCartridge( title: String ) extends Cartridge {
  def render = Html(<h1>title</h1>)
}
```

Our sample template implementation will contain four zones. The process of rendering a template to HTML will consist of rendering each of the cartriges in each of its zone, then combining those HTML fragments into a complete HTML document. In the application this article is based on actual production of HTML for the cartridges and template are handled by the Play view template system. The particulars of the assembling the HTML fragments into a complete document are straightforward and not of interest here.

```
class BellTemplate(header: Seq[Cartridge],
                   leftNav: Seq[Cartridge],
                   body: Seq[Cartridge],
                   footer: Seq[Cartridge]) extends Template {
  def render = {
    val headerContent: Seq[Html] = header.map(_.render)
    val leftNavContent: Seq[Html] = leftNav.map(_.render)
    val bodyContent: Seq[Html] = body.map(_.render)
    val footerContent: Seq[Html] = footer.map(_.render)
    val html = ???
    Result(200, html, Map())
  }
}	
```
	
The main logic of the application is then:

1. Recieve a request for a page,
2. Construct a query from the request and send it to Endeca,
3. Recieve view advice from Endeca,
4. Dispatch to the appropriate handler for the template,
5. For each zone in the template:
	* For each cartridge in the zone:
		* Dispatch to the appropriate cartridge handler for the cartridge type

Let's fill out the rest of the picture, starting with the controller code:

```
object SearchController {
  def search(term: String, refinements: Seq[String]) = {
    val query = ???

    val viewAdvice = EndecaService.getAdvice(query)
    val template = Template(viewAdvice)
    template.render
  }
}
```

The Template apply method dispatches to the appropriate template handler:

```
object Template {
  def getName(json: JsValue): String = ???

  def apply( viewAdvice: ViewAdvice ): Template = {
    val templateName = viewAdvice.templateName
    templateName match {
      case "Bell" => BellTemplate(viewAdvice)
      case _      => throw UnknownTemplateException(templateName)
    }
  }
}
```

The view advice emitted by Endeca might look something like this:

```
{ "@type": "Bell",
  "header": [
  	{ "@type": "Title",
  	  "title": "I Like Birds"
  	},
  	{ "@type": "Image",
  	  "url": "/images/pigeon.png",
  	  "alt-text": "Picture of a bird",
  	}
  ],
  "leftNav": [],
  "body": [
  	{ "@type": "RichText",
  	  "text": "Birds are great. I wish I could fly too."
  	}
  ],
  "footer": []
}
```

This page uses the "Bell" template, contains zones "header", "leftNav", "body" and "footer", and the following cartridges: "Title", "Image", and "RichText". 
	
# The Problem

The problem encountered, shortly before site launch, was the following. On rare occasion the Endeca assembler that generated the view advice we used to render the page would experience a failure. When this happened, the only evidence would be at the cartridge level. Instead of the regular cartridge payload there would be a JSON object with an error message. Since the expected data specifying the type of cartridge to the inserted and its particulars was not available the result would be a web page with a part missing.

The issues were transient in nature and fairly rare. It would be a relatively minor issue if not for the fact that the resulting pages were cached.

Once we've determined a page rendering is flawed we have a choice to make. Discard it and try again, or use it. We've made the business decision to serve up, but not cache, the result of partially invalid view advice. This is based on the observation that the defects tend to be fairly minor in most cases. Should the page be rendered in a way such that the customer is unable to use it we hope they would reload the page. By serving up the result of the flawed view advice rather than making a subsequent attempt we maintain the responsiveness of the site.

# Enter the Monads

The solution within the Play application would need to involve examining each cartridge level JSON object to determine whether the content represented a legitimate cartridge. If the entire contents of the view advice was validated as legitimate then the resulting rendered page was suitable for caching.

This solution would either require two passes through the view advice structure, or a single pass to both validate and parse the cartridge data. Further, since we were inspecting the cartridges to determine whether they contained any error responses it would also help to determine if there were business logic related issues specific to that cartridge so that they could be logged.

```
case class Validated[+A](value: A, isValid: Boolean) {
  def map[B](f: A => B): Validated[B] = 
    Validated(f(value), this.isValid)
  
  def flatMap[B](f: A => Validated[B]): Validated[B] = {
    val Validated(fA, fAIsValid) = f(value)
    Validated(fA, this.isValid && fAIsValid)
  }
```

A `Validated` value is a value along with a flag indicating whether it is "valid" or not. Do to the decision we've made to attempt to produce some result even if we encounter errors along the way our `Validate` type is a bit different than those in Play's JSON parsing, or Scalaz validation type. Our `Validated[X]` type will always contain a value of type `X`, while in Play and Scalaz a `JsResult[X]` or `Validation[E,X]` will only contain a value of type `X` if on a successful validation, otherwise they will provide details on the error encounter, but no result value.

Returning back to our `Validated` monad definition, `map` allows us to apply a validation agnostic function to a validated value, preserving the validity of the result.

The `flatMap` method is typically the most interesting part of a monad. In this case flatMap defines what it means to use the result of one validated computation as the input to another validated computation. Our definition determines that the result of such a computation is valid if and only if the input is valid and the computation itself is produces a valid result.

At the ground level, each cartridge will implement its own logic to determine whether its result is valid. From here the Validated monad will bubble up and combine these validity flags to determine the validity of the entire template result.

For example our `TitleCartridge` might be considered valid as long as it contains a `title` field:

```
object TitleCartridge {
  def fromJson( json: JsValue ): Validated[TitleCartridge] = {
    val getTitle: Option[String] = ???
    Validated( TitleCartridge(getTitle.getOrElse("")),
               isValid = getTitle.isDefined)
  }
}
```

From here we will use a bit of a trick to hopefully make the code a little more readable. We'll define a method on sequences of validated values that combines the validity flags to the sequence level:

```
object Validated {

  implicit class ValidateSeq[X](val sv: Seq[Validated[X]]) extends AnyVal {
    def validate = Validated(sv.map(_.value), sv.forall(_.isValid))
  }
}
``` 

This 'extension method' will let us write our template constructor as follows:

```
object BellTemplate {

  def apply(viewAdvice: ViewAdvice): Validated[BellTemplate] = {
    for {
      headZone <- viewAdvice.zone("head").map(Cartridge.fromJson).validate
      leftNavZone <- viewAdvice.zone("leftNav").map(Cartridge.fromJson).validate
      bodyZone <- viewAdvice.zone("body").map(Cartridge.fromJson).validate
      footerZone <- viewAdvice.zone("footer").map(Cartridge.fromJson).validate
    } yield new BellTemplate(headZone, leftNavZone, bodyZone, footerZone)
  }

}
```

The `validate` methods at the end of each line convert the sequence of validated cartridge (`Seq[Validated[Cartridge]]`) into a validated sequence of cartridges (`Validated[Seq[Cartridge]]`). The sequence will be valid if all the cartridges in it are valid.

We can use the implicit class trick again to define a render method on a validated template:

```
implicit class ValidatedTemplate(vt: Validated[Template]) extends Renderable[Result] {
  def render: Result =
    vt.map(_.render) match {
      case Validated(result, isCacheable@true) =>
        result
      case Validated(result, isCacheable@false) =>
        result.copy(headers = result.headers + ("Cache-control" -> "no-cache"))
    }
}
```

Our `render` method on validated templates not only renders the template to a `Result`, but adds no-cache headers incase our result was not valid.

Using this our controller code is essentially unchanged:

```
object SearchController {
  def search(term: String, refinements: Seq[String]): Result = {
    val query = ???

    val viewAdvice = EndecaService.getAdvice(query)
    val template: Validated[Template] = Template(viewAdvice)
    template.render
  }
}
```

The advantage of these implicit class methods is that they allow us to see the essential structure of the application, uncluttered by the details of fiddling around with validity flags. This will be even more so once we start adding additional complexity to those "plumbing" details.

# Asynchronously

The next augmentation we'll make to our rendering pipeline is to make it asynchronous. This will be useful in the case of cartridges which require further web service calls to resolve them. Fortunately, we'll see, that the hard work of is already mostly done. Now that we have a monadic rendering pipeline we only have to change the monad we're using to change the features of the pipeline.

In fact, if you already have a non-blocking application using `Future`s then you already have monadic code. 

The Validated monad takes care of some special handling that we needed in the template/cartridge parsing logic. In order to further augment our view advice parsing and rendering all that is required is that we use a different monad.

# Using Different Monads 

It would be a simple substitution to replace the Validated monad with the Option monad. Then each cartridge parse result would be possibly undefined, and the resulting template would be undefined if any cartridge in it was undefined. We could use this for an all-or-nothing handling of errors, rather than the 'success at any cost' policy of Validated.

Another common monad is the Future monad. This would change the pipeline to an asynchronous one. Each cartridge would be parsed asynchronously. This is where we will go next.

# Asynchronous Validated Pipeline

As business requirements evolved it became necessary to perform additional web service calls in order to interpret the cartridge data in the view advice. To avoid blocking threads in the Play application we want to use Futures to make these calls. We would like to combine the Validated logic with a asynchronous logic. Fortunately this turns out to be reasonable easy to achieve. 
	
# The ValidatedBuilder

While there is no general procedure to combine two monads together, many monads can be combined. In this case we will call our combined `Validated` and `Future` monad a `ValidatedBuilder`. There are two ways to combine these two monads. We could create a class equivalent to `Future[Validated[_]]`, or to `Validated[Future[_]]`. Both of these would have different semantics. The former would eventually give us a validated value, while the latter would provide the validity upfront, then asynchronous produce the result. In our case the former will be what we require, since we won't be able to determine whether the result of our computation is valid until we have tried to produce the result.

As a monad ValidatedBuilder will have `map` and `flatMap` methods. The `map` method is always easy to define, but the `flatMap` can be difficult or even impossible to define for arbitrarily chosen monads. Fortunately, in our case it is not too difficult to come up with a suitable definition for `flatMap`:

```
case class ValidatedBuilder[+X](asFuture: Future[Validated[X]]) {

  def map[Y](f: X => Y): ValidatedBuilder[Y] =
    new ValidatedBuilder[Y](asFuture.map(_.map(f)))

  def flatMap[Y](f: X => ValidatedBuilder[Y]): ValidatedBuilder[Y] = {
    new ValidatedBuilder(asFuture.flatMap {
      case Validated(va, aIsValid) =>
        val fva = f(va).asFuture
        fva map {
          case Validated(vb, bIsValid) =>
            Validated(vb, aIsValid && bIsValid)
        }
    })
  }
}
```

Since we already have a monadic pipeline the changes required to use the ValidatedBuilder will be quite simple. We have already defined the monad, with its `map` and `flatMap` methods. What remains is to:

 * Provide the means to construct ValidatedBuilder[Cartridge],
 * Define how to convert from Seq[ValidatedBuilder[Cartridge]] to ValidatedBuilder[Seq[Cartridge]] for use in parsing template zones,
 * Provide a method to render a ValidatedBuilder[Template]  
 
```
object ValidatedBuilder {
  implicit class ValidatedBuilderSeq[X](val sv: Seq[ValidatedBuilder[X]]) extends AnyVal {
    def validate: ValidatedBuilder[Seq[X]] = {
      val fsv = Future.sequence(sv.map(_.asFuture))
      new ValidatedBuilder(fsv.map(_.validate))
    }
  }
}
```

```
implicit class ValidatedBuilderTemplate(vt: ValidatedBuilder[Template]) extends Renderable[Result] {
  def render: Future[Result] =
    vt.map(_.render)
      .asFuture
      .map {
        case Validated(result, isCacheable@true ) =>
          result
        case Validated(result, isCacheable@false) =>
          result.copy(headers = result.headers + ("Cache-control" -> "no-cache"))
       }
}
```

# Where to Next?

* Validation
* Asynchronous processing
* Rendering Context (e.g. language, page type)
* Produce set of JavaScript module requirements

# End Material	
  
The use of a monad to abstract the rendering pipeline allows us to augment the set of computational features of that pipeline:
Read JSON to construct a web page
Read JSON to construct and validated a web page
Read JSON to asynchronous construct and validated a web page
Read JSON to asynchronous construct and validate a context dependent web page
If you've already implemented an asynchronous architecture using Futures the hard work is mostly done.

If you view a monad as a way to 'augment', (i.e. augment Int to Option[Int], or to List[Int]) then a monad is a type augmenter (parametrized type) with:
A way to take any regular (non-augmented) value to an augmented value
A way to apply a function on regular types to a function on augmented types (map)
A way to take a function that takes a regular type and produces an augmented type, and apply it to an augmented type.
Read a function f: A => M[B] as "an M-computation from A to B". For example, a function of type A => Future[B] is "an asynchronous computation from A to B". A function of type A => Option[B] is a "nullable computation from A to B". A function of type A => List[B] is a "list valued computation from A to (lists of) B".

In the notions of computation these are:
A way to inject regular values into our special domain
A way to inject regular functions (operations) into our special domain
A way to compose augmented computations (flatMap). I.e. a way to use the output of one computation as the input to the next.

# Augmented computational features

Return a validation,
Asynchronously,
With a context,
Returning a set of JavaScript module dependencies


