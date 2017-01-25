###Slick Getting Started
The easiest way to get started is with a working application in Typesafe Activator. The following templates are created by the Slick team, with updated versions being made for new Slick releases:
To learn the basics of Slick, start with the Hello Slick template. It contains an extended version of the tutorial and code from this page.
The Slick Plain SQL Queries template shows you how to do SQL queries with Slick.
The Slick Multi-DB Patterns template shows you how to write Slick applications that can use different database systems and how to use custom database functions in Slick queries.
The Slick TestKit Example template shows you how to use Slick TestKit to test your own Slick drivers.
There are more Slick templates created by the community, as well as versions of our own templates for other Slick releases. You can find all Slick templates on the Typesafe web site.
Adding Slick to Your Project

To include Slick in an existing project use the library published on Maven Central. For sbt projects add the following to your build definition - build.sbt or project/Build.scala:
libraryDependencies ++= Seq(
  "com.typesafe.slick" %% "slick" % "3.1.1",
  "org.slf4j" % "slf4j-nop" % "1.6.4"
)
For Maven projects add the following to your <dependencies> (make sure to use the correct Scala version prefix, _2.10 or _2.11, to match your project’s Scala version):
<dependency>
  <groupId>com.typesafe.slick</groupId>
  <artifactId>slick_2.10</artifactId>
  <version>3.1.1</version>
</dependency>
<dependency>
  <groupId>org.slf4j</groupId>
  <artifactId>slf4j-nop</artifactId>
  <version>1.6.4</version>
</dependency>
Slick uses SLF4J for its own debug logging so you also need to add an SLF4J implementation. Here we are using slf4j-nop to disable logging. You have to replace this with a real logging framework like Logback if you want to see log output.
The Reactive Streams API is pulled in automatically as a transitive dependency.
If you want to use Slick’s connection pool support for HikariCP, you need to add the slick-hikaricp module as a dependency in the same way as shown for slick above. It will automatically provide a compatible version of HikariCP as a transitive dependency.

####Quick Introduction

To use Slick you first need to import the API for the database you will be using, like:

```scala
// Use H2Driver to connect to an H2 database
import slick.driver.H2Driver.api._

import scala.concurrent.ExecutionContext.Implicits.global
```

Since we are using H2 as our database system, we need to import features from Slick’s H2Driver. A driver’s api object contains all commonly needed imports from the driver and other parts of Slick such as database handling.
Slick’s API is fully asynchronous and runs database call in a separate thread pool. For running user code in composition of DBIOAction and Future values, we import the global ExecutionContext. When using Slick as part of a larger application (e.g. with Play or Akka) the framework may provide a better alternative to this default ExecutionContext.

####Database Configuration
In the body of the application we create a Database object which specifies how to connect to a database. In most cases you will want to configure database connections with Typesafe Config in your application.conf, which is also used by Play and Akka for their configuration:

```scala
h2mem1 = {
  url = "jdbc:h2:mem:test1"
  driver = org.h2.Driver
  connectionPool = disabled
  keepAliveConnection = true
}
```

For the purpose of this example we disable the connection pool (there is no point in using one for an embedded in-memory database) and request a keep-alive connection (which ensures that the database does not get dropped while we are using it). The database can be easily instantiated from the configuration like this:
```scala
val db = Database.forConfig("h2mem1")
try {
  // ...
} finally db.close
```

> A Database object usually manages a thread pool and a connection pool. You should always shut it down properly when it is no longer needed (unless the JVM process terminates anyway).

####Schema
Before we can write Slick queries, we need to describe a database schema with Table row classes and TableQuery values for our tables. You can either use the code generator to automatically create them for your database schema or you can write them by hand:

```scala
// Definition of the SUPPLIERS table
class Suppliers(tag: Tag) extends Table[(Int, String, String, String, String, String)](tag, "SUPPLIERS") {
  def id = column[Int]("SUP_ID", O.PrimaryKey) // This is the primary key column
  def name = column[String]("SUP_NAME")
  def street = column[String]("STREET")
  def city = column[String]("CITY")
  def state = column[String]("STATE")
  def zip = column[String]("ZIP")
  // Every table needs a * projection with the same type as the table's type parameter
  def * = (id, name, street, city, state, zip)
}
val suppliers = TableQuery[Suppliers]

// Definition of the COFFEES table
class Coffees(tag: Tag) extends Table[(String, Int, Double, Int, Int)](tag, "COFFEES") {
  def name = column[String]("COF_NAME", O.PrimaryKey)
  def supID = column[Int]("SUP_ID")
  def price = column[Double]("PRICE")
  def sales = column[Int]("SALES")
  def total = column[Int]("TOTAL")
  def * = (name, supID, price, sales, total)
  // A reified foreign key relation that can be navigated to create a join
  def supplier = foreignKey("SUP_FK", supID, suppliers)(_.id)
}
val coffees = TableQuery[Coffees]
```

All columns get a name (usually in camel case for Scala and upper case with underscores for SQL) and a Scala type (from which the SQL type can be derived automatically). The table object also needs a Scala name, SQL name and type. The type argument of the table must match the type of the special * projection. In simple cases this is a tuple of all columns but more complex mappings are possible.
The foreignKey definition in the coffees table ensures that the supID field can only contain values for which a corresponding id exists in the suppliers table, thus creating an n to one relationship: A Coffees row points to exactly one Suppliers row but any number of coffees can point to the same supplier. This constraint is enforced at the database level.

* Populating the Database
The connection to the embedded H2 database engine provides us with an empty database. Before we can execute queries, we need to create the database schema (consisting of the coffees and suppliers tables) and insert some test data:

```scala
val setup = DBIO.seq(
  // Create the tables, including primary and foreign keys
  (suppliers.schema ++ coffees.schema).create,

  // Insert some suppliers
  suppliers += (101, "Acme, Inc.",      "99 Market Street", "Groundsville", "CA", "95199"),
  suppliers += ( 49, "Superior Coffee", "1 Party Place",    "Mendocino",    "CA", "95460"),
  suppliers += (150, "The High Ground", "100 Coffee Lane",  "Meadows",      "CA", "93966"),
  // Equivalent SQL code:
  // insert into SUPPLIERS(SUP_ID, SUP_NAME, STREET, CITY, STATE, ZIP) values (?,?,?,?,?,?)

  // Insert some coffees (using JDBC's batch insert feature, if supported by the DB)
  coffees ++= Seq(
    ("Colombian",         101, 7.99, 0, 0),
    ("French_Roast",       49, 8.99, 0, 0),
    ("Espresso",          150, 9.99, 0, 0),
    ("Colombian_Decaf",   101, 8.99, 0, 0),
    ("French_Roast_Decaf", 49, 9.99, 0, 0)
  )
  // Equivalent SQL code:
  // insert into COFFEES(COF_NAME, SUP_ID, PRICE, SALES, TOTAL) values (?,?,?,?,?)
)

val setupFuture = db.run(setup)
```

The TableQuery‘s ddl method creates DDL(data definition language) objects with the database-specific code for creating and dropping tables and other database entities. Multiple DDL values can be combined with ++ to allow all entities to be created and dropped in the correct order, even when they have circular dependencies on each other.
Inserting the tuples of data is done with the += and ++= methods, similar to how you add data to mutable Scala collections.
The create, += and ++= methods return an Action which can be executed on a database at a later time to produce a result. There are several different combinators for combining multiple Actions into sequences, yielding another Action. Here we use the simplest one, Action.seq, which can concatenate any number of Actions, discarding the return values (i.e. the resulting Action produces a result of type Unit). We then execute the setup Action asynchronously with db.run, yielding a Future[Unit].

> Database connections and transactions are managed automatically by Slick. By default connections are acquired and released on demand and used in auto-commit mode. In this mode we have to populate the suppliers table first because the coffees data can only refer to valid supplier IDs. We could also use an explicit transaction bracket encompassing all these statements (db.run(setup.transactionally)). Then the order would not matter because the constraints are only enforced at the end when the transaction is committed.

###Querying
The simplest kind of query iterates over all the data in a table:
```scala
// Read all coffees and print them to the console
println("Coffees:")
db.run(coffees.result).map(_.foreach {
  case (name, supID, price, sales, total) =>
    println("  " + name + "\t" + supID + "\t" + price + "\t" + sales + "\t" + total)
})
// Equivalent SQL code:
// select COF_NAME, SUP_ID, PRICE, SALES, TOTAL from COFFEES
```

This corresponds to a SELECT * FROM COFFEES in SQL (except that the * is the table’s * projection we defined earlier and not whatever the database sees as *). The type of the values we get in the loop is, unsurprisingly, the type parameter of Coffees.
Let’s add a projection to this basic query. This is written in Scala with the map method or a for comprehension:

```scala
// Why not let the database do the string conversion and concatenation?
val q1 = for(c <- coffees)
  yield LiteralColumn("  ") ++ c.name ++ "\t" ++ c.supID.asColumnOf[String] ++
    "\t" ++ c.price.asColumnOf[String] ++ "\t" ++ c.sales.asColumnOf[String] ++
    "\t" ++ c.total.asColumnOf[String]
// The first string constant needs to be lifted manually to a LiteralColumn
// so that the proper ++ operator is found

// Equivalent SQL code:
// select '  ' || COF_NAME || '\t' || SUP_ID || '\t' || PRICE || '\t' SALES || '\t' TOTAL from COFFEES

db.stream(q1.result).foreach(println)
```

The output will be the same: For each row of the table, all columns get converted to strings and concatenated into one tab-separated string. The difference is that all of this now happens inside the database engine, and only the resulting concatenated string is shipped to the client. Note that we avoid Scala’s + operator (which is already heavily overloaded) in favor of ++ (commonly used for sequence concatenation). Also, there is no automatic conversion of other argument types to strings. This has to be done explicitly with the type conversion method asColumnOf.
This time we also use Reactive Streams to get a streaming result from the database and print the elements as they come in instead of materializing the whole result set upfront.
Joining and filtering tables is done the same way as when working with Scala collections:

```scala
// Perform a join to retrieve coffee names and supplier names for
// all coffees costing less than $9.00
val q2 = for {
  c <- coffees if c.price < 9.0
  s <- suppliers if s.id === c.supID
} yield (c.name, s.name)
// Equivalent SQL code:
// select c.COF_NAME, s.SUP_NAME from COFFEES c, SUPPLIERS s where c.PRICE < 9.0 and s.SUP_ID = c.SUP_ID
```

>Note the use of === instead of == for comparing two values for equality and =!= instead of != for inequality. This is necessary because these operators are already defined (with unsuitable types and semantics) on the base type Any, so they cannot be replaced by extension methods. The other comparison operators are the same as in standard Scala code: <, <=, >=, >.

The generator expression suppliers if s.id === c.supID follows the relationship established by the foreign key Coffees.supplier. Instead of repeating the join condition here we can use the foreign key directly:

```scala
val q3 = for {
  c <- coffees if c.price < 9.0
  s <- c.supplier
} yield (c.name, s.name)
// Equivalent SQL code:
// select c.COF_NAME, s.SUP_NAME from COFFEES c, SUPPLIERS s where c.PRICE < 9.0 and s.SUP_ID = c.SUP_ID
```