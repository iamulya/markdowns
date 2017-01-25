###What is Slick?

Slick (“Scala Language-Integrated Connection Kit”) is Typesafe‘s Functional Relational Mapping (FRM) library for Scala that makes it easy to work with relational databases. It allows you to work with stored data almost as if you were using Scala collections while at the same time giving you full control over when a database access happens and which data is transferred. You can also use SQL directly. Execution of database actions is done asynchronously, making Slick a perfect fit for your reactive applications based on Play and Akka.

```scala
val limit = 10.0

// Your query could look like this:
( for( c <- coffees; if c.price < limit ) yield c.name ).result

// Equivalent SQL: select COF_NAME from COFFEES where PRICE < 10.0
```

When using Scala instead of raw SQL for your queries you benefit from compile-time safety and compositionality. Slick can generate queries for different back-end databases including your own, using its extensible query compiler.

###Functional Relational Mapping

Functional programmers have long suffered Object-Relational and Object-Math impedance mismatches when connecting to relational databases. Slick’s new Functional Relational Mapping (FRM) paradigm allows mapping to be completed within Scala, with loose-coupling, minimal configuration requirements, and a number of other major advantages that abstract the complexities away from connecting with relational databases.
We don’t try to fight the relational model, we embrace it through a functional paradigm. Instead of trying to bridge the gap between the object model and the database model, we’ve brought the database model into Scala so developers don’t need to write SQL code.

```scala
class Coffees(tag: Tag) extends Table[(String, Double)](tag, "COFFEES") {
  def name = column[String]("COF_NAME", O.PrimaryKey)
  def price = column[Double]("PRICE")
  def * = (name, price)
}
val coffees = TableQuery[Coffees]
```

Slick integrates databases directly into Scala, allowing stored and remote data to be queried and processed in the same way as in-memory data, using ordinary Scala classes and collections.

```scala
// Query that only returns the "name" column
// Equivalent SQL: select NAME from COFFEES
coffees.map(_.name)

// Query that limits results by price < 10.0
// Equivalent SQL: select * from COFFEES where PRICE < 10.0
coffees.filter(_.price < 10.0)
```

This enables full control over when a database is accessed and which data is transferred. The language integrated query model in Slick’s FRM is inspired by the LINQ project at Microsoft and leverages concepts tracing all the way back to the early work of Mnesia at Ericsson.
Some of the key benefits of Slick’s FRM approach for functional programming include:
* Efficiency with Pre-Optimization
FRM is more efficient way to connect; unlike ORM it has the ability to pre-optimize its communication with the database - and with FRM you get this out of the box. The road to making an app faster is much shorter with FRM than ORM.
* No More Tedious Troubleshooting with Type Safety
FRM brings type safety to building database queries. Developers are more productive because the compiler finds errors automatically versus the typical tedious troubleshooting required of finding errors in untyped strings.

```scala
// The result of "select PRICE from COFFEES" is a Seq of Double
// because of the type safe column definitions
val coffeeNames: Future[Seq[Double]] = db.run(
  coffees.map(_.price).result
)

// Query builders are type safe:
coffees.filter(_.price < 10.0)
// Using a string in the filter would result in a compilation error
```

Misspelled the column name price? The compiler will tell you:

```scala
GettingStartedOverview.scala:89: value prices is not a member of com.typesafe.slick.docs.GettingStartedOverview.Coffees
        coffees.map(_.prices).result
                      ^
```

The same goes for type errors:

```scala
GettingStartedOverview.scala:89: type mismatch;
 found   : slick.driver.H2Driver.StreamingDriverAction[Seq[String],String,slick.dbio.Effect.Read]
    (which expands to)  slick.profile.FixedSqlStreamingAction[Seq[String],String,slick.dbio.Effect.Read]
 required: slick.dbio.DBIOAction[Seq[Double],slick.dbio.NoStream,Nothing]
        coffees.map(_.name).result
                            ^
```

* A More Productive, Composable Model for Building Queries
FRM supports a composable model for building queries. It’s a very natural model to compose pieces together to build a query, and then reuse pieces across your code base.

```scala
// Create a query for coffee names with a price less than 10, sorted by name
coffees.filter(_.price < 10.0).sortBy(_.name).map(_.name)
// The generated SQL is equivalent to:
// select name from COFFEES where PRICE < 10.0 order by NAME
```

###Reactive Applications

Slick is easy to use in asynchronous, non-blocking application designs, and supports building applications according to the Reactive Manifesto. Unlike simple wrappers around traditional, blocking database APIs, Slick gives you:
* Clean separation of I/O and CPU-intensive code: Isolating I/O allows you to keep your main thread pool busy with CPU-intensive parts of the application while waiting for I/O in the background.
* Resilience under load: When a database cannot keep up with the load of your application, Slick will not create more and more threads (thus making the situation worse) or lock out all kinds of I/O. Back-pressure is controlled efficiently through a queue (of configurable size) for database I/O actions, allowing a certain number of requests to build up with very little resource usage and failing immediately once this limit has been reached.
* Reactive Streams for asynchronous streaming.
* Efficient utilization of database resources: Slick can be tuned easily and precisely for the parallelism (number of concurrent active jobs) and resource usage (number of currently suspended database sessions) of your database server.

###Plain SQL Support

The Scala-based query API for Slick allows you to write database queries like queries for Scala collections. Please see Getting Started for an introduction. Most of this user manual focuses on this API.
If you want to write your own SQL statements and still execute them asynchronously like a normal Slick queries, you can use the Plain SQL API:

```scala
val limit = 10.0

sql"select COF_NAME from COFFEES where PRICE < $limit".as[String]

// Automatically using a bind variable to be safe from SQL injection:
// select COF_NAME from COFFEES where PRICE < ?
```