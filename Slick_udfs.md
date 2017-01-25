Scalar Database Functions

If your database system supports a scalar function that is not available as a method in Slick you can define it as a SimpleFunction. There are predefined methods for creating unary, binary and ternary functions with fixed parameter and return types.
```scala
// H2 has a day_of_week() function which extracts the day of week from a timestamp
val dayOfWeek = SimpleFunction.unary[Date, Int]("day_of_week")

// Use the lifted function in a query to group by day of week
val q1 = for {
  (dow, q) <- salesPerDay.map(s => (dayOfWeek(s.day), s.count)).groupBy(_._1)
} yield (dow, q.map(_._2).sum)
```
If you need more flexibility regarding the types (e.g. for varargs, polymorphic functions, or to support Option and non-Option types in a single function), you can use SimpleFunction.apply to get an untyped instance and write your own wrapper function with the proper type-checking:
```scala
def dayOfWeek2(c: Rep[Date]) =
  SimpleFunction[Int]("day_of_week").apply(Seq(c))
```
SimpleBinaryOperator and SimpleLiteral work in a similar way. For even more flexibility (e.g. function-like expressions with unusual syntax), you can use SimpleExpression.
```scala
val current_date = SimpleLiteral[java.sql.Date]("CURRENT_DATE")
salesPerDay.map(_ => current_date)
```

###Other Database Functions And Stored Procedures

For database functions that return complete tables or stored procedures please use Plain SQL Queries. Stored procedures that return multiple result sets are currently not supported.

###Using Custom Scalar Types in Queries

If you need a custom column type you can implement ColumnType. The most common scenario is mapping an application-specific type to an already supported type in the database. This can be done much simpler by using MappedColumnType which takes care of all the boilerplate. It comes with the usual import from the driver.

```scala
// An algebraic data type for booleans
sealed trait Bool
case object True extends Bool
case object False extends Bool

// And a ColumnType that maps it to Int values 1 and 0
implicit val boolColumnType = MappedColumnType.base[Bool, Int](
  { b => if(b == True) 1 else 0 },    // map Bool to Int
  { i => if(i == 1) True else False } // map Int to Bool
)
// You can now use Bool like any built-in column type (in tables, queries, etc.)
```

You can also subclass MappedJdbcType for a bit more flexibility.
If you have a wrapper class (which can optionally be a case class and/or value class) for an underlying value of some supported type, you can make it extend MappedTo to get a macro-generated implicit ColumnType for free. Such wrapper classes are commonly used for type-safe table-specific primary key types:
```scala
// A custom ID type for a table
case class MyID(value: Long) extends MappedTo[Long]

// Use it directly for this table's ID -- No extra boilerplate needed
class MyTable(tag: Tag) extends Table[(MyID, String)](tag, "MY_TABLE") {
  def id = column[MyID]("ID")
  def data = column[String]("DATA")
  def * = (id, data)
}
```

###Using Custom Record Types in Queries

Record types are data structures containing a statically known number of components with individually declared types. Out of the box, Slick supports Scala tuples (up to arity 22) and Slick’s own HList implementation. Record types can be nested and mixed arbitrarily.
In order to use custom record types (case classes, custom HLists, tuple-like types, etc.) in queries you need to tell Slick how to map them between queries and results. You can do that using a Shape extending MappedScalaProductShape.

* Polymorphic Types (e.g. Custom Tuple Types or HLists)
The distinguishing feature of a polymorphic record type is that it abstracts over its element types, so you can use the same record type for both, lifted and plain element types. You can add support for custom polymorphic record types using an appropriate implicit Shape.
Here is an example for a type Pair: