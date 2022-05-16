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