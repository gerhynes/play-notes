# Play Notes

Play is a Java and Scala web application framework which uses the MVC architecture and functional programming patterns.

It includes:
- an integrated HTTP server
- routing
- form handling
- CSRF protection
- i18n support

Play uses Akka and Akka HTTP under the hood for a stateless, non-blocking, event-driven architecture which provides horizontal and vertical scalability and efficient resource use.

Play can be integrated with ORM layers (with out of the box support for Anorm, JavaEbean, PlaySlick, and JPA) as well as with NoSQL data sources.

### Request-Response Cycle
A typical request-response cycle in a Play application goes like this:

- The internal HTTP server receives the request
- Play resolves the request using the routes file to map URIs to controller action methods
- The action method uses Twirl templates to render the page
- The server returns the response as a HTML page

### Application Structure
A Play application typically contains the following components:

- the `app` directory, containing Scala code and packages. This contains the `models`, `views` and `controllers` directories. Any `service` or `utils` directories would also go here
- the `public` directory, containing the `images`, `javascripts` and `stylesheets` directories 
- the `conf` directory, containing `application.conf`, the application configuration, and `routes`, the router's definition file

### Routes
The `routes` file in the `conf` directory defines all application routes. A route consists of a HTTP method, a path, and an action.

```scala
// conf/routes
GET     /                                   controllers.HomeController.index()  
GET     /explore                            controllers.HomeController.explore()  
GET     /tutorial                           controllers.HomeController.tutorial()

# Map static resources from the /public folder to the /assets URL path  
GET     /assets/*file               controllers.Assets.versioned(path="/public", file: Asset)
```

### Controllers
A controller class creates an Action, which is triggered when a request reaches a specific route.

```scala
// app/controllers/HomeController.scala
/**  
 * This controller creates an `Action` to handle HTTP requests to the  
 * application's home page. 
 */
@Singleton  
class HomeController @Inject()(cc: ControllerComponents) extends AbstractController(cc) {  
  
/**  
 * Create an Action to render an HTML page. 
 *  
 * The configuration in the `routes` file means that this method  
 * will be called when the application receives a `GET` request with  
 * a path of `/`.  
 */ 
 def index() = Action { implicit request: Request[AnyContent] =>  
		Ok(views.html.index())  
  }  
    
	def explore() = Action { implicit request: Request[AnyContent] =>  
		Ok(views.html.explore())  
  }  
    
  def tutorial() = Action { implicit request: Request[AnyContent] =>  
		Ok(views.html.tutorial())  
  }  
    
}
```

### Views
A view file contains a directive which calls a template. Here the `main` directive calls the main template and passes it the string parameter it expects.

```scala
// app/views/index.scala.html
@()  
  
@main("Welcome") {  
@defining(play.core.PlayVersion.current) { version =>  
  
<section id="content">
// More content
</section>
```

## REST APIs With Play
The `routes` file is suitable when the Play application is directly rendering HTML. For a REST API, Play has a more powerful routing DSL.

The `routes` file can be set up to direct requests to a dedicated Router.

```Scala
->     /v1/posts               v1.post.PostRouter
```

This Router examines the URL and extracts data to pass on to the controller.

```Scala
class PostRouter @Inject()(controller: PostController) extends SimpleRouter {
  val prefix = "/v1/posts"

  def link(id: PostId): String = {
    import com.netaporter.uri.dsl._
    val url = prefix / id.toString
    url.toString()
  }

  override def routes: Routes = {
    case GET(p"/") =>
      controller.index

    case POST(p"/") =>
      controller.process

    case GET(p"/$id") =>
      controller.show(id)
  }

}
```

Play’s routing DSL (technically "String Interpolation Routing DSL" / SIRD) uses a string interpolated extractor object to extract and use path parameters. There are also operators to extract query parameters and regular expressions, and custom extractors can be defined.

```scala
/posts/?sort=ascending&count=5

GET("/" ? q_?"sort=$sort" & q_?"count=${ int(count) }")
```

The Router has a Controller injected into it. This Controller handles processing a HTTP request into a HTTP response in the context of an Action. 

```Scala
class PostRouter @Inject()(controller: PostController) extends SimpleRouter {...}
```

A Controller extends `play.api.mvc.BaseController`, which contains utility methods and constants for working with HTTP. In particular, a Controller contains `Result` objects such as `Ok` and `Redirect`, and `HeaderNames` like `ACCEPT`.

Using the action, the Controller passes in a block of code that takes a `Request` as an implicit (any in-scope method that takes an implicit result as a parameter will use this request automatically). The block must return either a `Result` or a `Future[Result]` depending on whether the action was called as `action {}` or `action.async{}`.

```scala
import javax.inject.Inject
import play.api.mvc._

import scala.concurrent._

class MyController @Inject()(val controllerComponents: ControllerComponents) extends BaseController {

  def index1: Action[AnyContent] = Action { implicit request =>
    val r: Result = Ok("hello world")
    r
  }

  def asyncIndex: Action[AnyContent] = Action.async { implicit request =>
    val r: Future[Result] = Future.successful(Ok("hello world"))
    r
  }
}
```

Internally, it makes no difference whether you call `Result` or `Future[Result]`, since Play is non-blocking. However, if you’re already working with a `Future`, async makes it easier to pass that `Future` around.

For example, in the Play REST API example project, a GET request to `v1/posts/123` will be passed to the `show` method on the `PostController`. 

The id is passed in as a String, and the handler looks up and returns a `PostResource`. The `Ok()`directive sends back a `Result` with a status code of “200 OK”, containing a response body consisting of the `PostResource` serialized as JSON.

```scala
def show(id: String): Action[AnyContent] = PostAction.async { implicit request =>
  logger.trace(s"show: id = $id")
  postResourceHandler.lookup(id).map { post =>
    Ok(Json.toJson(post))
  }
}
```

### Processing Form Input
In this example a POST request is passed to the `process` method on the PostController. `process` calls `processJsonPost` .

Here, `form.bindFromRequest()` will map input from the HTTP request to a `play.api.data.Form`, and handle form validation and error reporting.

If the `PostFormInput` passes validation, it’s passed to the resource handler, using the `success` method. If the form processing fails, then the `failure` method is called and the `FormError` is returned in JSON format.

```scala
private val form: Form[PostFormInput] = {
  import play.api.data.Forms._

  Form(
    mapping(
      "title" -> nonEmptyText,
      "body" -> text
    )(PostFormInput.apply)(PostFormInput.unapply)
  )
}

def process: Action[AnyContent] = PostAction.async { implicit request =>
  logger.trace("process: ")
  processJsonPost()
}

private def processJsonPost[A]()(implicit request: PostRequest[A]): Future[Result] = {
  def failure(badForm: Form[PostFormInput]) = {
    Future.successful(BadRequest(badForm.errorsAsJson))
  }

  def success(input: PostFormInput) = {
    postResourceHandler.create(input).map { post =>
      Created(Json.toJson(post)).withHeaders(LOCATION -> post.link)
    }
  }

  form.bindFromRequest().fold(failure, success)
}
```

The form binds to the HTTP request using the names in the mapping – `title` and `body`  – to the `PostFormInput` case class:

```scala
case class PostFormInput(title: String, body: String)
```

### Using Actions
In a Controller, methods are connected to an Action (in this example, the `PostAction.async` method).

```scala
def index: Action[AnyContent] = PostAction.async { implicit request =>
  logger.trace("index: ")
  postResourceHandler.find.map { posts =>
    Ok(Json.toJson(posts))
  }
}
```

The `PostAction.async` method is a custom action builder that can handle `PostRequest`s. 

`PostAction` is involved in each action in the Controller. It mediates the processing of a request into a response, adds context to the request and enriches the response with headers and cookies. ActionBuilders are essential for handling authentication, authorization and monitoring functionality.

ActionBuilders work through a process called action composition. The ActionBuilder class has a method called `invokeBlock` that takes in a `Request` and a function (also known as a block, lambda or closure) that accepts a `Request` of a given type, and produces a `Future[Result]`.

```scala
class YourRequest[A](request: Request[A], val yourClass: YourClass) extends WrappedRequest(request)

class YourAction @Inject()(parsers: PlayBodyParsers)(implicit val executionContext: ExecutionContext) extends ActionBuilder[YourRequest, AnyContent] {

  type YourRequestBlock[A] = YourRequest[A] => Future[Result]

  override def parser: BodyParser[AnyContent] = parsers.defaultBodyParser

  override def invokeBlock[A](request: Request[A], block: YourRequestBlock[A]): Future[Result] = {
    block(new YourRequest[A](request, YourClass()))
  }
}
```

You create an `ActionBuilder[YourRequest, AnyContent]`, override `invokeBlock`, and then call the function with an instance of `YourRequest`.

Then, when you call `yourAction`, the request type is `YourRequest` and `request.yourClass` will be added automatically.

```scala
yourAction { request: YourRequest =>
  Ok(request.yourClass.toString)
}
```

You can compose action builders inside each other or create a custom `ActionBuilder` for each package you work with.

### Rendering Content as JSON
Play handles the work of converting resources to JSON using Play JSON. 

If the resource has a companion object that implicitly defines its JSON format, Play JSON will look this up when handling an instance of the class. This means that when the controller converts a class instance to JSON, no additional impots or setup are required.

```Scala
case class PostResource(id: String, link: String, title: String, body: String)  
  
object PostResource {  
/**  
* Mapping to read/write a PostResource out as a JSON value. 
*/ 
 implicit val format: Format[PostResource] = Json.format  
}
```

## Main Concepts
### Configuration API
Play provides a Scala wrapper, `Configuration`, around the Typesafe config library.

Usually, you'll access a `Configuration` object through dependency injection.

```scala
class MyController @Inject() (config: Configuration, c: ControllerComponents) extends AbstractController(c) {
  def getMyValue = Action {
    Ok(config.get[String]("myValue"))
  }
}
```

The `get` method is used to access a single value at a path in the configuration file.

```scala
// value1 = value
config.get[String]("value1")

// value2 = 8
config.get[Int]("value2")

// value3 = true
config.get[Boolean]("value3")

// listOfValues = ["value2", "value3"]
config.get[Seq[String]]("listOfValues")
```

`Configuration` accepts an implicit `ConfigLoader` but for common types like `String`, `Int` and `Seq[String]`, there are already loaders defined.

`Configuration` also supports validating against a set of valid values.

```scala
config.getAndValidate[String]("val1", Set("val2", "val3"))
```

You can define your own `ConfigLoader` to convert configuration into a custom type.

```scala
case class AppConfig(title: String, baseUri: URI)

object AppConfig {
  implicit val configLoader: ConfigLoader[AppConfig] = new ConfigLoader[AppConfig] {
    def load(rootConfig: Config, path: String): AppConfig = {
      val config = rootConfig.getConfig(path)
      AppConfig(
        title = config.getString("title"),
        baseUri = new URI(config.getString("baseUri"))
      )
    }
  }
}
```

Then you can use `config.get` as normal:

```scala
// app.config = {
//   title = "My App
//   baseUri = "https://example.com/"
// }
config.get[AppConfig]("app.config")
```

You can support optional configuration keys using the `getOptional[A]` method, which will return `null` if the key doesn't exist. But Play recommends setting optional keys to `null` in the configuration file and using `get[Option[A]]` instead. Reserve `getOptional[A]` for interfacing with libraries that use configuration in a non-standard way. 

### HTTP Programming
#### Actions, Controllers and Results
An Action is a function (of type `play.api.mvc.Request => play.api.mvc.Result`) that handles a request and generates a result to be sent to the client.

```scala
def echo = Action { request =>
  Ok("Got request [" + request + "]")
}
```
An Action returns a `play.api.mvc.Result` value, representing the HTTP response to send to the web client. Here `Ok` produces a 200 OK response with a `text/plain` response body.

In any controller that extends `BaseController`, the `Action` value is the default action builder. This action builder contains several helpers for creating `Action`s.

The simplest type of Action takes as an argument an expression block returning a result. This, however, doesn't give you access to the incoming request.

```scala
Action {
  Ok("Hello world")
}
```

To make use of the incoming request, use an Action builder that takes as an argument a function `Request => Result` .

```scala
Action { request =>
  Ok("Got request [" + request + "]")
}
```

It can be useful to mark the `request` parameter as `implicit` so it can be implicitly used by other APIs that need it.

```scala
Action { implicit request =>
  Ok("Got request [" + request + "]")
}
```

If your controller has additional methods, you can pass the implicit request to them from the action.

```scala
def action = Action { implicit request =>
  anotherMethod("Some string parameter")
  Ok("Got request [" + request + "]")
}

def anotherMethod(param: String)(implicit request: Request[_]) = {
  // do something that needs access to the request
}
```

You can also create an Action value by specifying an additional `BodyParser` argument. If you don't specify a `BodyParser` argument, the action builkder uses a default Any content body parser.

```scala
Action(parse.json) { implicit request =>
  Ok("Got request [" + request + "]")
}
```

#### Controllers Generate Actions
A controller is fundamentally an object that generates Action values. Controllers are usually defined as classes to take advantage of Dependency Injection.

The simplest use case for an action generator method is a method with no parameters that returns an `Action` value.

```scala
package controllers

import javax.inject.Inject
import play.api.mvc._

class Application @Inject() (cc: ControllerComponents) extends AbstractController(cc) {
	def index = Action {
		Ok("It works!")
	}
}
```

Action generator methods can have parameters and these parameters can be captured by the `Action` closure.

```scala
def hello(name: String) = Action {
  Ok("Hello " + name)
}
```

#### Results
Results are defined by `play.api.mvc.Result`. A relatively simple result from a controller method will be: a HTTP result with a status code, a set of headers, and a body.

```scala
import play.api.http.HttpEntity

def index = Action {
  Result(
    header = ResponseHeader(200, Map.empty),
    body = HttpEntity.Strict(ByteString("Hello world!"), Some("text/plain"))
  )
}
```

There are several helpers, such as `Ok`, to create common results.

```scala
def index = Action {
  Ok("Hello world!")
}
```

Here are several examples to create various results:

```scala
val ok           = Ok("Hello world!")
val notFound     = NotFound
val pageNotFound = NotFound(<h1>Page not found</h1>)
val badRequest   = BadRequest(views.html.form(formWithErrors))
val oops         = InternalServerError("Oops")
val anyStatus    = Status(488)("Strange response type")
```

All of these helpers can be found in the `play.api.mvc.Results` trait and companion object.

#### Redirects
Redirecting the browser to another URL is another kind of simple result. These results don't take a response body. Again, there are helpers to create redirect results.

```scala
def index = Action {
  Redirect("/user/home")
}
```

The default is to use a `303 SEE_OTHER` response type, but you can also set a more specific status code if you need one.

```scala
def index = Action {
  Redirect("/user/home", MOVED_PERMANENTLY)
}
```

#### Dummy Pages
You can use an empty `Action` implementation defined as `TODO`. The result is a standard "Not implemented yet" result page.

```scala
def index(name: String) = TODO
```

### HTTP Routing
The router is the component in charge of translating each incoming HTTP request into an Action.

A HTTP request is seen as an event by Play. The event contains two main pieces of information.

- the request path (`/clients/1`, `/photos/latest`), including the query string
- the HTTP method

The `routes` file defines the routes. This file is compiled, which means that route errors are shown directly in the browser during development.

#### Dependency Injection

Play's default routes generator creates a Router class that accepts controller instances in an `@Inject`-annotated constructor. This means the class can be used with dependency injection and can also be manually instantiated using the constructor.

#### The `routes` file

The `conf/routes` file lists all routes used by the application. Each route consists of a HTTP method, a URL pattern and a call to an `Action` generator.

```
GET   /users/:id          controllers.Users.show(id: Long)
```

You can tell the routes file to use a different router under a specific prefix by using “->” followed by the given prefix.

```
->      /api                        api.MyRouter
```

#### URI Patterns
The URI pattern can be static `/users/all` or dynamic `/users/:id`. The default matching strategy for a dynamic part is defined by the regular expression `[^/]+`, meaning that any dynamic part defined as `:id` will match exactly one URI path segment. Unlike other pattern types, path segments are automatically URI-decoded in the route (before being passed to the controller) and encoded in the reverse route.

The default matching strategy for a dynamic part is defined by the regular expression `[^/]+`, meaning that any dynamic part defined as `:id` will match exactly one URI path segment. Unlike other pattern types, path segments are automatically URI-decoded in the route, before being passed to your controller, and encoded in the reverse route.

If you want a dynamic part to capture more than one URI path segment, separated by forward slashes, you can define the dynamic part using the `*id` syntax, also known as a wildcard pattern, which uses the `.*` regular expression.

```
GET   /files/*name          controllers.Application.download(name)
```

Here, for a request like `GET /files/images/logo.png`, the `name` dynamic part will capture the `images/logo.png` value.

Note that dynamic parts spanning several `/` are not decoded by the router or encoded by the reverse router. It is your responsibility to validate the raw URI segment as you would for any user input. The reverse router simply does a string concatenation, so you will need to make sure the resulting path is valid, and does not, for example, contain multiple leading slashes or non-ASCII characters.

You can also define your own custom regular expression for the dynamic part, using the `$id<regex>` syntax.

```
GET   /items/$id<[0-9]+>    controllers.Items.show(id: Long)
```

Just like with wildcard routes, the parameter is not decoded by the router or encoded by the reverse router. You’re responsible for validating the input.

#### The Action Generator Method
The last part of the route must be a call to a method that returns an `Action`. This will usually be a controller action method.

If the action method has parameters, these will be searched for in the request URI and extracted from either the URI path itself or from the query string.

```routes
GET   /                     controllers.Application.homePage()

GET   /:page                controllers.Application.show(page)
```

```scala
def show(page: String) = Action {
  loadContentFromDatabase(page)
    .map { htmlContent =>
      Ok(htmlContent).as("text/html")
    }
    .getOrElse(NotFound)
}
```

Play supports the following Parameter Types:

-   String
-   Int
-   Long
-   Double
-   Float
-   Boolean
-   UUID
-   AnyVal wrappers for other supported types

For String parameters the type is optional. 

Play can transform the incoming parameter into a specific Scala type if you explicitly type the parameter.

```routes
GET   /clients/:id          controllers.Clients.show(id: Long)
```

```scala
def show(id: Long) = Action {
  Client
    .findById(id)
    .map { client =>
      Ok(views.html.Clients.display(client))
    }
    .getOrElse(NotFound)
}
```

If necessary, you can use a fixed value as a parameter.

```routes
# Extract the page parameter from the path, or fix the value
GET   /                     controllers.Application.show(page = "home")
GET   /:page                controllers.Application.show(page)
```

You can also provide a default value that will be used if no value is found in the incoming request.

```routes
# Pagination links, like /clients?page=3
GET   /clients              controllers.Clients.list(page: Int ?= 1)
```

You can also specify list parameters for repeated query string parameters.

```routes
# The item parameter is a list.
# /api/list-items?item=red&item=new&item=slippers
GET   /api/list-items      controllers.Api.listItems(item: List[String])

# or

# /api/list-int-items?item=1&item=42
GET   /api/list-int-item
```

#### Routing Priority
If multiple routes match the same request the first route in declaration order is used.

#### Reverse Routing
The router can also be used to generate a URL from within Scala. This lets you centralize all URI patterns in a single configuration file for easier refactoring.

For each controller used in the routes file, the router will generate a "reverse controller" in the `routes` package, having the same action methods, with the same signature, but returning a `play.api.mvc.Call` instead of a `play.api.mvc.Action`.

The `play.api.mvc.Call` defines a HTTP call, and provides both the HTTP method and the URI.

If you create a controller and map it in the `conf/routes` file, you can reverse the URL using the `routes` subpackage in the `controllers` package.

```scala
package controllers
  import javax.inject.Inject

  import play.api._
  import play.api.mvc._

  class Application @Inject() (cc: ControllerComponents) extends AbstractController(cc) {
    def hello(name: String) = Action {
      Ok("Hello " + name + "!")
    }
  }
```

If you map it in the `conf/routes` file:

```routes
# Hello action
GET   /hello/:name          controllers.Application.hello(name)
```

You can then reverse the URL to the `hello` action method, by using the `controllers.routes.Application` reverse controller:

```scala
// Redirect to /hello/Bob
def helloBob = Action {
  Redirect(routes.Application.hello("Bob"))
}
```

The reverse action method works by taking your parameters and substituting them back into the route pattern. In the case of path segments (such as `:id`), the value is encoded before the substitution is done. For regex and wildcard patterns, the string is substituted in raw form, since the value may span multiple segments.

#### Relative Routes
The routes returned by `play.mvc.Call` are always absolute. This can lead to problems when requests to your web application are rewritten by HTTP proxies, load balancers and API gateways.

Using a relative route can be useful when:
- Your app is hosted behind a web gateway that prefixes all routes with something other than what's configured in your `conf/routes` file, and roots your app at a route it's not expecting
- Your stylesheets are dynamically rendered and may end up getting served from different URLs by a CDN

To generate a relative route, you need the start route. This can be retrieved from the current `RequestHeader` or passed as a `String` parameter.

```scala
package controllers

import javax.inject._
import play.api.mvc._

@Singleton
class Relative @Inject() (cc: ControllerComponents) extends AbstractController(cc) {
  def helloview() = Action { implicit request =>
    Ok(views.html.hello("Bob"))
  }

  def hello(name: String) = Action {
    Ok(s"Hello $name!")
  }
}
```

```routes
GET     /foo/bar/hello              controllers.Relative.helloview
GET     /hello/:name                controllers.Relative.hello(name)
```

You can then define relative routes using the reverse router and include an additional call to `relative`.

```html
@(name: String)(implicit request: RequestHeader)

<h1>Hello @name</h1>

<a href="@routes.Relative.hello(name)">Absolute Link</a>
<a href="@routes.Relative.hello(name).relative">Relative Link</a>
```

**Note:** The `Request` passed from the controller is cast to a `RequestHeader` and is marked `implicit` in the view parameters. It is then passed implicitly to the call to `relative`.

When requesting `/something/somethingelse/hello`, the generated HTML will look like this:

```html
<!DOCTYPE html>
<html lang="en">
    <head>
        <title>Bob</title>
    </head>
    <body>
      <a href="/hello/Bob">Absolute Link</a>
      <a href="../../hello/Bob">Relative Link</a>
    </body>
</html>
```

#### The Default Controller
Play includes a `Default` controller with a couple of useful actions. These can be invoked directly from the `routes` file.

```routes
# Redirects to https://www.playframework.com/ with 303 See Other
GET   /about      controllers.Default.redirect(to = "https://www.playframework.com/")

# Responds with 404 Not Found
GET   /orders     controllers.Default.notFound

# Responds with 500 Internal Server Error
GET   /clients    controllers.Default.error

# Responds with 501 Not Implemented
GET   /posts      controllers.Default.todo
```

You can redirect to an external website, but also to another action.

#### Custom Routing
Play provides a DSL for defining embedded routers called the String Interpolating Routing DSL, or SIRD for short. This DSL has many uses, including embedding a light weight Play server, providing custom or more advanced routing capabilities to a regular Play application, and mocking REST services for testing.

### Manipulating HTTP Results
#### Changing the default `Content-Type`
The result content type is automatically inferred from the Scala value specified as the result body.

For example `Ok("Hello World")` will set the `Content-Type` header to `text/plain`.

You can use the `as` method to set a different `Content-Type` header: `Ok(<h1>Hello World</h1>).as(HTML)`.

#### Manipulating HTTP Headers
You cna add or update any HTTP header.

`Ok("Hello World").withHeaders(CACHE_CONTROL -> "max-age=3600", ETAG -> "xx")`

Setting a header will discard the previous value if it existed.

#### Setting and Discarding Cookies
You can add Cokkies to the HTTP response using the `withCookies` and `bakeCookies` methods.

`Ok("Hello World").withCookies(Cooke("theme", "dark")).bakeCookies()`

You can also discard exiting Cookies.

`result.discardCookies(DiscardingCookie("theme"))`

Setting and discarding Cookies can be done in the same response.

`result.withCookies(Cookie("theme", "dark")).discardingCookies(DiscardingCookie("discount"))`

#### Changing the Charset for Text-based HTTP Responses
Play handles the response charset automatically and uses `utf-8` by default. You can override it using the `play.api.mvc.Codec` class.

```scala
class Application @Inject() (cc: ControllerComponents) extends AbstractController(cc) {
  implicit val myCustomCharset = Codec.javaSupported("iso-8859-1")

  def index = Action {
    Ok(<h1>Hello World!</h1>).as(HTML)
  }
}
```

Here, because there's an implicit charset value in scope, it will be used by the `Ok` method to convert messages into `ISO-8859-1` encoded bytes and generate the `text/html; charset=iso-8859-1` Content-Type header.

#### Range Results
Play supports partial responses (a `206 Partial Content` response) if a satisfiable `Range` header is in the request. It also returns an `Accept-Ranges: bytes` for the delivered `Result`.

When the request `Range` is not satisfiable, for example, if the range in the request’s `Range` header field does not overlap the current extent of the selected resource, then a HTTP status `416 Range Not Satisfiable` is returned.