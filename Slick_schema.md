If you already have a schema in the database, you can also use the code generator to take this work off your hands.

###Table Rows

In order to use the Scala API for type-safe queries, you need to define Table row classes for your database schema. These describe the structure of the tables:
```scala
class Coffees(tag: Tag) extends Table[(String, Int, Double, Int, Int)](tag, "COFFEES") {
  def name = column[String]("COF_NAME", O.PrimaryKey)
  def supID = column[Int]("SUP_ID")
  def price = column[Double]("PRICE")
  def sales = column[Int]("SALES", O.Default(0))
  def total = column[Int]("TOTAL", O.Default(0))
  def * = (name, supID, price, sales, total)
}
```
All columns are defined through the column method. Each column has a Scala type and a column name for the database (usually in upper-case). The following primitive types are supported out of the box for JDBC-based databases in JdbcProfile (with certain limitations imposed by the individual database drivers):
```scala
Numeric types: Byte, Short, Int, Long, BigDecimal, Float, Double
LOB types: java.sql.Blob, java.sql.Clob, Array[Byte]
Date types: java.sql.Date, java.sql.Time, java.sql.Timestamp
Boolean
String
Unit
java.util.UUID
```
Nullable columns are represented by Option[T] where T is one of the supported primitive types.
> Currently all operations on Option values use the database’s null propagation semantics which may differ from Scala’s Option semantics. In particular, None === None evaluates to None. This behaviour may change in a future major release of Slick.

After the column name, you can add optional column options to a column definition. The applicable options are available through the table’s O object. The following ones are defined for JdbcProfile:
* PrimaryKey:
Mark the column as a (non-compound) primary key when creating the DDL statements.
* Default[T](defaultValue: T):
Specify a default value for inserting data into the table without this column. This information is only used for creating DDL statements so that the database can fill in the missing information.
* DBType(dbType: String) :
Use a non-standard database-specific type for the DDL statements (e.g. DBType("VARCHAR(20)") for a String column).
* AutoInc:
Mark the column as an auto-incrementing key when creating the DDL statements. Unlike the other column options, this one also has a meaning outside of DDL creation: Many databases do not allow non-AutoInc columns to be returned when inserting data (often silently ignoring other columns), so Slick will check if the return column is properly marked as AutoInc where needed.
* NotNull, Nullable:
Explicitly mark the column as nullable or non-nullable when creating the DDL statements for the table. Nullability is otherwise determined from the type (Option or non-Option). There is usually no reason to specify these options.

Every table requires a * method containing a default projection. This describes what you get back when you return rows (in the form of a table row object) from a query. Slick’s * projection does not have to match the one in the database. You can add new columns (e.g. with computed values) or omit some columns as you like. The non-lifted type corresponding to the * projection is given as a type parameter to Table. For simple, non-mapped tables, this will be a single column type or a tuple of column types.
If your database layout requires schema names, you can specify the schema name for a table in front of the table name, wrapped in Some():
```scala
class Coffees(tag: Tag)
  extends Table[(String, Int, Double, Int, Int)](tag, Some("MYSCHEMA"), "COFFEES") {
  //...
}
```

###Table Query

Alongside the Table row class you also need a TableQuery value which represents the actual database table:

```scala
val coffees = TableQuery[Coffees]
```

The simple TableQuery[T] syntax is a macro which expands to a proper TableQuery instance that calls the table’s constructor (new TableQuery(new T(_))).
You can also extend TableQuery to use it as a convenient namespace for additional functionality associated with the table:
object coffees extends TableQuery(new Coffees(_)) {
  val findByName = this.findBy(_.name)
}

###Mapped Tables

It is possible to define a mapped table that uses a custom type for its * projection by adding a bi-directional mapping with the <> operator:
case class User(id: Option[Int], first: String, last: String)

class Users(tag: Tag) extends Table[User](tag, "users") {
  def id = column[Int]("id", O.PrimaryKey, O.AutoInc)
  def first = column[String]("first")
  def last = column[String]("last")
  def * = (id.?, first, last) <> (User.tupled, User.unapply)
}
val users = TableQuery[Users]
It is optimized for case classes (with a simple apply method and an unapply method that wraps its result in an Option) but it can also be used with arbitrary mapping functions. In these cases it can be useful to call .shaped on a tuple on the left-hand side in order to get its type inferred properly. Otherwise you may have to add full type annotations to the mapping functions.
For case classes with hand-written companion objects, .tupled only works if you manually extend the correct Scala function type. Alternatively you can use (User.apply _).tupled. See SI-3664 and SI-4808.

###Constraints

A foreign key constraint can be defined with a Table’s foreignKey method. It first takes a name for the constraint, the referencing column(s) and the referenced table. The second argument list takes a function from the referenced table to its referenced column(s) as well as ForeignKeyAction for onUpdate and onDelete, which are optional and default to NoAction. When creating the DDL statements for the table, the foreign key definition is added to it.

```scala
class Suppliers(tag: Tag) extends Table[(Int, String, String, String, String, String)](tag, "SUPPLIERS") {
  def id = column[Int]("SUP_ID", O.PrimaryKey)
  //...
}
val suppliers = TableQuery[Suppliers]

class Coffees(tag: Tag) extends Table[(String, Int, Double, Int, Int)](tag, "COFFEES") {
  def supID = column[Int]("SUP_ID")
  //...
  def supplier = foreignKey("SUP_FK", supID, suppliers)(_.id, onUpdate=ForeignKeyAction.Restrict, onDelete=ForeignKeyAction.Cascade)
  // compiles to SQL:
  //   alter table "COFFEES" add constraint "SUP_FK" foreign key("SUP_ID")
  //     references "SUPPLIERS"("SUP_ID")
  //     on update RESTRICT on delete CASCADE
}
val coffees = TableQuery[Coffees]
```

Independent of the actual constraint defined in the database, such a foreign key can be used to navigate to the referenced data with a join. For this purpose, it behaves the same as a manually defined utility method for finding the joined data:
```scala
def supplier = foreignKey("SUP_FK", supID, suppliers)(_.id, onUpdate=ForeignKeyAction.Restrict, onDelete=ForeignKeyAction.Cascade)
def supplier2 = suppliers.filter(_.id === supID)
```
A primary key constraint can be defined in a similar fashion by adding a method that calls primaryKey. This is useful for defining compound primary keys (which cannot be done with the O.PrimaryKey column option):
```scala
class A(tag: Tag) extends Table[(Int, Int)](tag, "a") {
  def k1 = column[Int]("k1")
  def k2 = column[Int]("k2")
  def * = (k1, k2)
  def pk = primaryKey("pk_a", (k1, k2))
  // compiles to SQL:
  //   alter table "a" add constraint "pk_a" primary key("k1","k2")
}
```
Other indexes are defined in a similar way with the index method. They are non-unique by default unless you set the unique parameter:
```scala
class A(tag: Tag) extends Table[(Int, Int)](tag, "a") {
  def k1 = column[Int]("k1")
  def k2 = column[Int]("k2")
  def * = (k1, k2)
  def idx = index("idx_a", (k1, k2), unique = true)
  // compiles to SQL:
  //   create unique index "idx_a" on "a" ("k1","k2")
}
```
All constraints are discovered reflectively by searching for methods with the appropriate return types which are defined in the table. This behavior can be customized by overriding the tableConstraints method.

###Data Definition Language

DDL statements for a table can be created with its TableQuery‘s schema method. Multiple DDL objects can be concatenated with ++ to get a compound DDL object which can create and drop all entities in the correct order, even in the presence of cyclic dependencies between tables. The create and drop methods produce the Actions for executing the DDL statements:
```scala
val schema = coffees.schema ++ suppliers.schema
db.run(DBIO.seq(
  schema.create,
  //...
  schema.drop
))
```
You can use the the statements method to get the SQL code, like for most other SQL-based Actions. Schema Actions are currently the only Actions that can produce more than one statement.
```scala
schema.create.statements.foreach(println)
schema.drop.statements.foreach(println)
```