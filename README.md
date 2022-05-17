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


