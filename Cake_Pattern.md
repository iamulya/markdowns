Scala is a very rich and deep language that gives you several ways of doing DI solely based on language constructs, but nothing prevents you from using existing Java DI frameworks, if that is preferred.

Using the Cake Pattern

The current strategy we are using is based on the so-called Cake Pattern. This pattern is first explained in Martin Oderskys’ paper scalable component abstractions (which is an excellent paper that is highly recommended) as the way he and his team structured the Scala compiler. But rather than trying to explain the pattern and how it can be used to implement DI in plain English let’s take a look at some (naive) sample code (loosely based on our production code).

First, let’s create a UserRepository (DAO) implementation.

// a dummy service that is not persisting anything
// solely prints out info
class UserRepository {
  def authenticate(user: User): User = {
    println("authenticating user: " + user)
    user
   }
  def create(user: User) = println("creating user: " + user)
  def delete(user: User) = println("deleting user: " + user)
}
Here we could have split up the implementation in a trait interface and its implementation, but in order to keep things simple I didn’t see the need.

Now let’s create a user service (also a dummy one, merely redirecting to our repository).

class UserService {
  def authenticate(username: String, password: String): User =
    userRepository.authenticate(username, password)

  def create(username: String, password: String) =
    userRepository.create(new User(username, password))

  def delete(user: User) = All is statically typed.
    userRepository.delete(user)
}
Here you can see that we are referencing an instance of the UserRepository. This is the dependency that we would like to have injected for us.

Ok. Now the interesting stuff starts. Let’s first wrap the UserRepository in an enclosing trait and instantiate the user repository there.

trait UserRepositoryComponent {
  val userRepository = new UserRepository
  class UserRepository {
    def authenticate(user: User): User = {
      println("authenticating user: " + user)
      user
    }
    def create(user: User) = println("creating user: " + user)
    def delete(user: User) = println("deleting user: " + user)
  }
}
This simply creates a component namespace for our repository. Why? Stay with me and I’ll show you how to make use of this namespace in a second.

Now let’s look at the UserService, the user of the repository. In order to declare that we would like to have the userRepository instance injected in the UserService we will first do what we did with the repository above; wrap the it in an enclosing (namespace) trait and use a so-called self-type annotation to declare our need for the UserRepository service. Sounds more complicated than it is. Let’s look at the code.

// using self-type annotation declaring the dependencies this
// component requires, in our case the UserRepositoryComponent
trait UserServiceComponent { this: UserRepositoryComponent =>
  val userService = new UserService
  class UserService {
    def authenticate(username: String, password: String): User =
      userRepository.authenticate(username, password)
    def create(username: String, password: String) =
      userRepository.create(new User(username, password))
    def delete(user: User) = userRepository.delete(user)
  }
}
The self-type annotation we are talking about is this code snippet:

this: UserRepositoryComponent =>
If you need to declare more than one dependency then you can compose the annotations like this:

this: Foo with Bar with Baz =>
Ok. Now we have declared the UserRepository dependency. What is left is the actual wiring.

In order to do that the only thing we need to do is to merge/join the different namespaces into one single application (or module) namespace. This is done by creating a “module” object composed of all our components. When we do that all wiring is happening automatically.

object ComponentRegistry extends
  UserServiceComponent with
  UserRepositoryComponent
One of the beauties here is that all wiring is statically typed. For example, if we have a dependency declaration missing, if it is misspelled or something else is screwed up then we get a compilation error. This also makes it very fast.

Another beauty is that everything is immutable (all dependencies are declared as val).

In order to use the application we only need to get the “top-level” component from the registry, and all other dependencies are wired for us (similar to how Guice/Spring works).

val userService = ComponentRegistry.userService
...
val user = userService.authenticate(..)
So far so good?

Well, no. This sucks.

We have strong coupling between the service implementation and its creation, the wiring configuration is scattered all over our code base; utterly inflexible.

Let’s fix it.

Instead of instantiating the services in their enclosing component trait, let’s change it to an abstract member field.

trait UserRepositoryComponent {
  val userRepository: UserRepository

  class UserRepository {
    ...
  }
}
trait UserServiceComponent {
  this: UserRepositoryComponent =>

  val userService: UserService

  class UserService {
    ...
  }
}
Now, we can move the instantiation (and configuration) of the services to the ComponentRegistry module.

object ComponentRegistry extends
  UserServiceComponent with
  UserRepositoryComponent
{
  val userRepository = new UserRepository
  val userService = new UserService
}
By doing this switch we have now abstracted away the actual component instantiation as well as the wiring into a single “configuration” object.

The neat thing is that we can here switch between different implementations of the services (if we had defined an interface trait and multiple implementations). But even more interestingly, we can create multiple “worlds” or “environments” by simply composing the traits in different combinations.

To show you what I mean, we’ll now create a “testing environment” to be used during unit testing.

Now, instead of instantiating the actual services we instead create mocks to each one of them. We also change the “world” to a trait (why, I will show you in a second).

trait TestEnvironment extends
  UserServiceComponent with
  UserRepositoryComponent with
  org.specs.mock.JMocker
{
  val userRepository = mock(classOf[UserRepository])
  val userService = mock(classOf[UserService])
}
Here we are not merely creating mocks but the mocks we create are wired in as the declared dependencies wherever defined.

Ok, now comes the fun part. Let’s create a unit test in which we are mixing in the TestEnvironment mixin, which is holding all our mocks.

class UserServiceSuite extends TestNGSuite with TestEnvironment {

  @Test { val groups=Array("unit") }
  def authenticateUser = {

    // create a fresh and clean (non-mock) UserService
    // (who's userRepository is still a mock)
    val userService = new UserService

    // record the mock invocation
    expect {
      val user = new User("test", "test")
      one(userRepository).authenticate(user) willReturn user
    }

    ... // test the authentication method
  }

  ...
}
This pretty much sums it up and is just one example on how you can compose your components in the way you want.

Other alternatives

Let’s now take a look at some other ways of doing DI in Scala. This post is already pretty long and therefore I will only walk through the techniques very briefly, but it will hopefully be enough for you to understand how it is done. I have based all these remaining examples on the same little dummy program to make it easier to digest and to compare (taken from some discussion found on the Scala User mailing list). In all these examples you can just copy the code and run it in the Scala interpreter, in case you want to play with it.

Using structural typing

This technique using structural typing was posted by Jamie Webb on the Scala User mailing list some time ago. I like this approach; elegant, immutable, type-safe.

// =======================
// service interfaces
trait OnOffDevice {
  def on: Unit
  def off: Unit
}
trait SensorDevice {
  def isCoffeePresent: Boolean
}

// =======================
// service implementations
class Heater extends OnOffDevice {
  def on = println("heater.on")
  def off = println("heater.off")
}
class PotSensor extends SensorDevice {
  def isCoffeePresent = true
}

// =======================
// service declaring two dependencies that it wants injected,
// is using structural typing to declare its dependencies
class Warmer(env: {
  val potSensor: SensorDevice
  val heater: OnOffDevice
}) {
  def trigger = {
    if (env.potSensor.isCoffeePresent) env.heater.on
    else env.heater.off
  }
}

class Client(env : { val warmer: Warmer }) {
  env.warmer.trigger
}

// =======================
// instantiate the services in a configuration module
object Config {
  lazy val potSensor = new PotSensor
  lazy val heater = new Heater
  lazy val warmer = new Warmer(this) // this is where injection happens
}

new Client(Config)
Using implicit declarations

This approach is simple and straight-forward. But I don’t really like that the actual wiring (importing the implicit declarations) is scattered and tangled with the application code.

// =======================
// service interfaces
trait OnOffDevice {
  def on: Unit
  def off: Unit
}
trait SensorDevice {
  def isCoffeePresent: Boolean
}

// =======================
// service implementations
class Heater extends OnOffDevice {
  def on = println("heater.on")
  def off = println("heater.off")
}
class PotSensor extends SensorDevice {
  def isCoffeePresent = true
}

// =======================
// service declaring two dependencies that it wants injected
class Warmer(
  implicit val sensor: SensorDevice,
  implicit val onOff: OnOffDevice) {

  def trigger = {
    if (sensor.isCoffeePresent) onOff.on
    else onOff.off
  }
}

// =======================
// instantiate the services in a module
object Services {
  implicit val potSensor = new PotSensor
  implicit val heater = new Heater
}

// =======================
// import the services into the current scope and the wiring
// is done automatically using the implicits
import Services._

val warmer = new Warmer
warmer.trigger
Using Google Guice

Scala works nicely with separate DI frameworks and early on we were using google guice. You can use Guice in many different ways, but here we will discuss a slick technique based on a ServiceInjector mixin that my Jan Kriesten showed me.

// =======================
// service interfaces
trait OnOffDevice {
  def on: Unit
  def off: Unit
}
trait SensorDevice {
  def isCoffeePresent: Boolean
}
trait IWarmer {
  def trigger
}
trait Client

// =======================
// service implementations
class Heater extends OnOffDevice {
  def on = println("heater.on")
  def off = println("heater.off")
}
class PotSensor extends SensorDevice {
  def isCoffeePresent = true
}
class @Inject Warmer(
  val potSensor: SensorDevice,
  val heater: OnOffDevice)
  extends IWarmer {

  def trigger = {
    if (potSensor.isCoffeePresent) heater.on
    else heater.off
  }
}

// =======================
// client
class @Inject Client(val warmer: Warmer) extends Client {
  warmer.trigger
}

// =======================
// Guice's configuration class that is defining the
// interface-implementation bindings
class DependencyModule extends Module {
  def configure(binder: Binder) = {
    binder.bind(classOf[OnOffDevice]).to(classOf[Heater])
    binder.bind(classOf[SensorDevice]).to(classOf[PotSensor])
    binder.bind(classOf[IWarmer]).to(classOf[Warmer])
    binder.bind(classOf[Client]).to(classOf[MyClient])
  }
}

// =======================
// Usage: val bean = new Bean with ServiceInjector
trait ServiceInjector {
  ServiceInjector.inject(this)
}

// helper companion object
object ServiceInjector {
  private val injector = Guice.createInjector(
    Array[Module](new DependencyModule))
  def inject(obj: AnyRef) = injector.injectMembers(obj)
}

// =======================
// mix-in the ServiceInjector trait to perform injection
// upon instantiation
val client = new MyClient with ServiceInjector

println(client)
That sums up what I had planned to go through in this article. I hope that you have gained some insight in how one can do DI in Scala, either using language abstractions or a separate DI framework. What works best for you is up to your use-case, requirements and taste.

Update:

Below I have added a Cake Pattern version of the last example for easier comparison between the different DI strategies. Just a note, if you compare the different strategies using this naive example then the Cake Pattern might look a bit overly complicated with its nested (namespace) traits, but it really starts to shine when you have a less then trivial example with many components with more or less complex dependencies to manage.

// =======================
// service interfaces
trait OnOffDeviceComponent {
  val onOff: OnOffDevice
  trait OnOffDevice {
    def on: Unit
    def off: Unit
  }
}
trait SensorDeviceComponent {
  val sensor: SensorDevice
  trait SensorDevice {
    def isCoffeePresent: Boolean
  }
}

// =======================
// service implementations
trait OnOffDeviceComponentImpl extends OnOffDeviceComponent {
  class Heater extends OnOffDevice {
    def on = println("heater.on")
    def off = println("heater.off")
  }
}
trait SensorDeviceComponentImpl extends SensorDeviceComponent {
  class PotSensor extends SensorDevice {
    def isCoffeePresent = true
  }
}
// =======================
// service declaring two dependencies that it wants injected
trait WarmerComponentImpl {
  this: SensorDeviceComponent with OnOffDeviceComponent =>
  class Warmer {
    def trigger = {
      if (sensor.isCoffeePresent) onOff.on
      else onOff.off
    }
  }
}

// =======================
// instantiate the services in a module
object ComponentRegistry extends
  OnOffDeviceComponentImpl with
  SensorDeviceComponentImpl with
  WarmerComponentImpl {

  val onOff = new Heater
  val sensor = new PotSensor
  val warmer = new Warmer
}

// =======================
val warmer = ComponentRegistry.warmer
warmer.trigger