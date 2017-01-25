 The API for building queries is a _lifted embedding_, which means that you are not working with standard Scala types but with types that are lifted into a Rep type constructor. This becomes clearer when you compare the types of a simple Scala collections example
 ```scala
case class Coffee(name: String, price: Double)
val coffees: List[Coffee] = //...

val l = coffees.filter(_.price > 8.0).map(_.name)
//                       ^       ^          ^
//                       Double  Double     String
```
... with the types of similar code in Slick:
```scala
class Coffees(tag: Tag) extends Table[(String, Double)](tag, "COFFEES") {
  def name = column[String]("COF_NAME")
  def price = column[Double]("PRICE")
  def * = (name, price)
}
val coffees = TableQuery[Coffees]

val q = coffees.filter(_.price > 8.0).map(_.name)
//                       ^       ^          ^
//               Rep[Double]  Rep[Double]  Rep[String]
```
All plain types are lifted into Rep. The same is true for the table row type Coffees which is a subtype of Rep[(String, Double)]. Even the literal 8.0 is automatically lifted to a Rep[Double] by an implicit conversion because that is what the > operator on Rep[Double] expects for the right-hand side. This lifting is necessary because the lifted types allow us to generate a syntax tree that captures the query computations. Getting plain Scala functions and values would not give us enough information for translating those computations to SQL.

###Expressions

Scalar (non-record, non-collection) values are represented by type Rep[T] for which an implicit TypedType[T] exists.
The operators and other methods which are commonly used in queries are added through implicit conversions defined in ExtensionMethodConversions. The actual methods can be found in the classes AnyExtensionMethods, ColumnExtensionMethods, NumericColumnExtensionMethods, BooleanColumnExtensionMethods and StringColumnExtensionMethods (cf. ExtensionMethods).

> Most operators mimic the plain Scala equivalents, but you have to use === instead of == for comparing two values for equality and =!= instead of != for inequality. This is necessary because these operators are already defined (with unsuitable types and semantics) on the base type Any, so they cannot be replaced by extension methods.

Collection values are represented by the Query class (a Rep[Seq[T]]) which contains many standard collection methods like flatMap, filter, take and groupBy. Due to the two different component types of a Query (lifted and plain, e.g. Query[(Rep[Int], Rep[String]), (Int, String), Seq]), the signatures for these methods are very complex but the semantics are essentially the same as for Scala collections.
Additional methods for queries of scalar values are added via an implicit conversion to SingleColumnQueryExtensionMethods.

###Sorting and Filtering

There are various methods with sorting/filtering semantics (i.e. they take a Query and return a new Query of the same type), for example:
```scala
val q1 = coffees.filter(_.supID === 101)
// compiles to SQL (simplified):
//   select "COF_NAME", "SUP_ID", "PRICE", "SALES", "TOTAL"
//     from "COFFEES"
//     where "SUP_ID" = 101

val q2 = coffees.drop(10).take(5)
// compiles to SQL (simplified):
//   select "COF_NAME", "SUP_ID", "PRICE", "SALES", "TOTAL"
//     from "COFFEES"
//     limit 5 offset 10

val q3 = coffees.sortBy(_.name.desc.nullsFirst)
// compiles to SQL (simplified):
//   select "COF_NAME", "SUP_ID", "PRICE", "SALES", "TOTAL"
//     from "COFFEES"
//     order by "COF_NAME" desc nulls first

// building criteria using a "dynamic filter" e.g. from a webform.
val criteriaColombian = Option("Colombian")
val criteriaEspresso = Option("Espresso")
val criteriaRoast:Option[String] = None

val q4 = coffees.filter { coffee =>
  List(
      criteriaColombian.map(coffee.name === _),
      criteriaEspresso.map(coffee.name === _),
      criteriaRoast.map(coffee.name === _) // not a condition as `criteriaRoast` evaluates to `None`
  ).collect({case Some(criteria)  => criteria}).reduceLeftOption(_ || _).getOrElse(true: Rep[Boolean])
}
// compiles to SQL (simplified):
//   select "COF_NAME", "SUP_ID", "PRICE", "SALES", "TOTAL"
//     from "COFFEES"
//     where ("COF_NAME" = 'Colombian' or "COF_NAME" = 'Espresso')
```

###Joining and Zipping

Joins are used to combine two different tables or queries into a single query. There are two different ways of writing joins: Applicative and monadic.

* Applicative joins
Applicative joins are performed by calling a method that joins two queries into a single query of a tuple of the individual results. They have the same restrictions as joins in SQL, i.e. the right-hand side may not depend on the left-hand side. This is enforced naturally through Scala’s scoping rules.

```scala
val crossJoin = for {
  (c, s) <- coffees join suppliers
} yield (c.name, s.name)
// compiles to SQL (simplified):
//   select x2."COF_NAME", x3."SUP_NAME" from "COFFEES" x2
//     inner join "SUPPLIERS" x3

val innerJoin = for {
  (c, s) <- coffees join suppliers on (_.supID === _.id)
} yield (c.name, s.name)
// compiles to SQL (simplified):
//   select x2."COF_NAME", x3."SUP_NAME" from "COFFEES" x2
//     inner join "SUPPLIERS" x3
//     on x2."SUP_ID" = x3."SUP_ID"

val leftOuterJoin = for {
  (c, s) <- coffees joinLeft suppliers on (_.supID === _.id)
} yield (c.name, s.map(_.name))
// compiles to SQL (simplified):
//   select x2."COF_NAME", x3."SUP_NAME" from "COFFEES" x2
//     left outer join "SUPPLIERS" x3
//     on x2."SUP_ID" = x3."SUP_ID"

val rightOuterJoin = for {
  (c, s) <- coffees joinRight suppliers on (_.supID === _.id)
} yield (c.map(_.name), s.name)
// compiles to SQL (simplified):
//   select x2."COF_NAME", x3."SUP_NAME" from "COFFEES" x2
//     right outer join "SUPPLIERS" x3
//     on x2."SUP_ID" = x3."SUP_ID"

val fullOuterJoin = for {
  (c, s) <- coffees joinFull suppliers on (_.supID === _.id)
} yield (c.map(_.name), s.map(_.name))
// compiles to SQL (simplified):
//   select x2."COF_NAME", x3."SUP_NAME" from "COFFEES" x2
//     full outer join "SUPPLIERS" x3
//     on x2."SUP_ID" = x3."SUP_ID"
```

Note the use of map in the yield clauses of the outer joins. Since these joins can introduce additional NULL values (on the right-hand side for a left outer join, on the left-hand sides for a right outer join, and on both sides for a full outer join), the respective sides of the join are wrapped in an Option (with None representing a row that was not matched).

* Monadic joins
Monadic joins are created with flatMap. They are theoretically more powerful than applicative joins because the right-hand side may depend on the left-hand side. However, this is not possible in standard SQL, so Slick has to compile them down to applicative joins, which is possible in many useful cases but not in all of them (and there are cases where it is possible in theory but Slick cannot perform the required transformation yet). If a monadic join cannot be properly translated, it will fail at runtime.
A cross-join is created with a flatMap operation on a Query (i.e. by introducing more than one generator in a for-comprehension):
```scala
val monadicCrossJoin = for {
  c <- coffees
  s <- suppliers
} yield (c.name, s.name)
// compiles to SQL:
//   select x2."COF_NAME", x3."SUP_NAME"
//     from "COFFEES" x2, "SUPPLIERS" x3
```
If you add a filter expression, it becomes an inner join:
```scala
val monadicInnerJoin = for {
  c <- coffees
  s <- suppliers if c.supID === s.id
} yield (c.name, s.name)
// compiles to SQL:
//   select x2."COF_NAME", x3."SUP_NAME"
//     from "COFFEES" x2, "SUPPLIERS" x3
//     where x2."SUP_ID" = x3."SUP_ID"
```
The semantics of these monadic joins are the same as when you are using flatMap on Scala collections.

> Slick currently generates implicit joins in SQL (select ... from a, b where ...) for monadic joins, and explicit joins (select ... from a join b on ...) for applicative joins. This is subject to change in future versions.

* Zip joins
In addition to the usual applicative join operators supported by relational databases (which are based off a cross join or outer join), Slick also has zip joins which create a pairwise join of two queries. The semantics are again the same as for Scala collections, using the zip and zipWith methods:
```scala
val zipJoinQuery = for {
  (c, s) <- coffees zip suppliers
} yield (c.name, s.name)

val zipWithJoin = for {
  res <- coffees.zipWith(suppliers, (c: Coffees, s: Suppliers) => (c.name, s.name))
} yield res
```
A particular kind of zip join is provided by zipWithIndex. It zips a query result with an infinite sequence starting at 0. Such a sequence cannot be represented by an SQL database and Slick does not currently support it, either. The resulting zipped query, however, can be represented in SQL with the use of a row number function, so zipWithIndex is supported as a primitive operator:
```scala
val zipWithIndexJoin = for {
  (c, idx) <- coffees.zipWithIndex
} yield (c.name, idx)
```
* Unions

Two queries can be concatenated with the ++ (or unionAll) and union operators if they have compatible types:
```scala
val q1 = coffees.filter(_.price < 8.0)
val q2 = coffees.filter(_.price > 9.0)

val unionQuery = q1 union q2
// compiles to SQL (simplified):
//   select x8."COF_NAME", x8."SUP_ID", x8."PRICE", x8."SALES", x8."TOTAL"
//     from "COFFEES" x8
//     where x8."PRICE" < 8.0
//   union select x9."COF_NAME", x9."SUP_ID", x9."PRICE", x9."SALES", x9."TOTAL"
//     from "COFFEES" x9
//     where x9."PRICE" > 9.0

val unionAllQuery = q1 ++ q2
// compiles to SQL (simplified):
//   select x8."COF_NAME", x8."SUP_ID", x8."PRICE", x8."SALES", x8."TOTAL"
//     from "COFFEES" x8
//     where x8."PRICE" < 8.0
//   union all select x9."COF_NAME", x9."SUP_ID", x9."PRICE", x9."SALES", x9."TOTAL"
//     from "COFFEES" x9
//     where x9."PRICE" > 9.0
```
Unlike union which filters out duplicate values, ++ simply concatenates the results of the individual queries, which is usually more efficient.

* Aggregation

The simplest form of aggregation consists of computing a primitive value from a Query that returns a single column, usually with a numeric type, e.g.:
```scala
val q = coffees.map(_.price)

val q1 = q.min
// compiles to SQL (simplified):
//   select min(x4."PRICE") from "COFFEES" x4

val q2 = q.max
// compiles to SQL (simplified):
//   select max(x4."PRICE") from "COFFEES" x4

val q3 = q.sum
// compiles to SQL (simplified):
//   select sum(x4."PRICE") from "COFFEES" x4

val q4 = q.avg
// compiles to SQL (simplified):
//   select avg(x4."PRICE") from "COFFEES" x4
```
Note that these aggregate queries return a scalar result, not a collection. Some aggregation functions are defined for arbitrary queries (of more than one column):
```scala
val q1 = coffees.length
// compiles to SQL (simplified):
//   select count(1) from "COFFEES"

val q2 = coffees.exists
// compiles to SQL (simplified):
//   select exists(select * from "COFFEES")
```

Grouping is done with the groupBy method. It has the same semantics as for Scala collections:

```scala
val q = (for {
  c <- coffees
  s <- c.supplier
} yield (c, s)).groupBy(_._1.supID)

val q2 = q.map { case (supID, css) =>
  (supID, css.length, css.map(_._1.price).avg)
}
// compiles to SQL:
//   select x2."SUP_ID", count(1), avg(x2."PRICE")
//     from "COFFEES" x2, "SUPPLIERS" x3
//     where x3."SUP_ID" = x2."SUP_ID"
//     group by x2."SUP_ID"
```
The intermediate query q contains nested values of type Query. These would turn into nested collections when executing the query, which is not supported at the moment. Therefore it is necessary to flatten the nested queries immediately by aggregating their values (or individual columns) as done in q2.

* Querying

A Query can be converted into an Action by calling its result method. The Action can then be executed directly in a streaming or fully materialized way, or composed further with other Actions:
```scala
val q = coffees.map(_.price)
val action = q.result
val result: Future[Seq[Double]] = db.run(action)
val sql = action.statements.head
```

If you only want a single result value, you can call head or headOption on the result Action.

* Deleting

Deleting works very similarly to querying. You write a query which selects the rows to delete and then get an Action by calling the delete method on it:

```scala
val q = coffees.filter(_.supID === 15)
val action = q.delete
val affectedRowsCount: Future[Int] = db.run(action)
val sql = action.statements.head
```

A query for deleting must only select from a single table. Any projection is ignored (it always deletes full rows).

* Inserting

Inserts are done based on a projection of columns from a single table. When you use the table directly, the insert is performed against its * projection. Omitting some of a table’s columns when inserting causes the database to use the default values specified in the table definition, or a type-specific default in case no explicit default was given. All methods for building insert Actions are defined in CountingInsertActionComposer and ReturningInsertActionComposer.
```scala
val insertActions = DBIO.seq(
  coffees += ("Colombian", 101, 7.99, 0, 0),

  coffees ++= Seq(
    ("French_Roast", 49, 8.99, 0, 0),
    ("Espresso",    150, 9.99, 0, 0)
  ),

  // "sales" and "total" will use the default value 0:
  coffees.map(c => (c.name, c.supID, c.price)) += ("Colombian_Decaf", 101, 8.99)
)

// Get the statement without having to specify a value to insert:
val sql = coffees.insertStatement

// compiles to SQL:
//   INSERT INTO "COFFEES" ("COF_NAME","SUP_ID","PRICE","SALES","TOTAL") VALUES (?,?,?,?,?)
```

When you include an AutoInc column in an insert operation, it is silently ignored, so that the database can generate the proper value. In this case you usually want to get back the auto-generated primary key column. By default, += gives you a count of the number of affected rows (which will usually be 1) and ++= gives you an accumulated count in an Option (which can be None if the database system does not provide counts for all rows). This can be changed with the returning method where you specify the columns to be returned (as a single value or tuple from += and a Seq of such values from ++=):

```scala
val userId =
  (users returning users.map(_.id)) += User(None, "Stefan", "Zeiger")
```
> Many database systems only allow a single column to be returned which must be the table’s auto-incrementing primary key. If you ask for other columns a SlickException is thrown at runtime (unless the database actually supports it).

You can follow the returning method with the into method to map the inserted values and the generated keys (specified in returning) to a desired value. Here is an example of using this feature to return an object with an updated id:
```scala
val userWithId =
  (users returning users.map(_.id)
         into ((user,id) => user.copy(id=Some(id)))
  ) += User(None, "Stefan", "Zeiger")
```
Instead of inserting data from the client side you can also insert data created by a Query or a scalar expression that is executed in the database server:
```scala
class Users2(tag: Tag) extends Table[(Int, String)](tag, "users2") {
  def id = column[Int]("id", O.PrimaryKey)
  def name = column[String]("name")
  def * = (id, name)
}
val users2 = TableQuery[Users2]

val actions = DBIO.seq(
  users2.schema.create,
  users2 forceInsertQuery (users.map { u => (u.id, u.first ++ " " ++ u.last) }),
  users2 forceInsertExpr (users.length + 1, "admin")
)
```
In these cases, AutoInc columns are not ignored.

* Updating

Updates are performed by writing a query that selects the data to update and then replacing it with new data. The query must only return raw columns (no computed values) selected from a single table. The relevant methods for updating are defined in UpdateExtensionMethods.
```scala
val q = for { c <- coffees if c.name === "Espresso" } yield c.price
val updateAction = q.update(10.49)

// Get the statement without having to specify an updated value:
val sql = q.updateStatement

// compiles to SQL:
//   update "COFFEES" set "PRICE" = ? where "COFFEES"."COF_NAME" = 'Espresso'
```
There is currently no way to use scalar expressions or transformations of the existing data in the database for updates.

* Upserting

Upserting is performed by supplying a row to be either inserted or updated. The object must contain the table’s primary key, since the update portion needs to be able to find a uniquelly matching row.
```scala
val updated = users.insertOrUpdate(User(Some(1), "Admin", "Zeiger"))
// returns: number of rows updated

val updatedAdmin = (users returning users).insertOrUpdate(User(Some(1), "Slick Admin", "Zeiger"))
// returns: None if updated, Some((Int, String)) if row inserted
```

* Compiled Queries

Database queries typically depend on some parameters, e.g. an ID for which you want to retrieve a matching database row. You can write a regular Scala function to create a parameterized Query object each time you need to execute that query but this will incur the cost of recompiling the query in Slick (and possibly also on the database if you don’t use bind variables for all parameters). It is more efficient to pre-compile such parameterized query functions:

```scala
def userNameByIDRange(min: Rep[Int], max: Rep[Int]) =
  for {
    u <- users if u.id >= min && u.id < max
  } yield u.first

val userNameByIDRangeCompiled = Compiled(userNameByIDRange _)

// The query will be compiled only once:
val namesAction1 = userNameByIDRangeCompiled(2, 5).result
val namesAction2 = userNameByIDRangeCompiled(1, 3).result
// Also works for .insert, .update and .delete
```
This works for all functions that take parameters consisting only of individual columns or or records of columns and return a Query object or a scalar query. See the API documentation for Compiled and its subclasses for details on composing compiled queries.
Be aware that take and drop take ConstColumn[Long] parameters. Unlike Rep[Long]], which could be substituted by another value computed by a query, a ConstColumn can only be literal value or a parameter of a compiled query. This is necessary because the actual value has to be known by the time the query is prepared for execution by Slick.
```scala
val userPaged = Compiled((d: ConstColumn[Long], t: ConstColumn[Long]) => users.drop(d).take(t))

val usersAction1 = userPaged(2, 1).result
val usersAction2 = userPaged(1, 3).result
```
You can use a compiled query for querying, inserting, updating and deleting data. For backwards-compatibility with Slick 1.0 you can still create a compiled query by calling flatMap on a Parameters object. In many cases this enables you to write a single for comprehension for a compiled query:

```scala
val userNameByID = for {
  id <- Parameters[Int]
  u <- users if u.id === id
} yield u.first

val nameAction = userNameByID(2).result.head

val userNameByIDRange = for {
  (min, max) <- Parameters[(Int, Int)]
  u <- users if u.id >= min && u.id < max
} yield u.first

val namesAction = userNameByIDRange(2, 5).result
```