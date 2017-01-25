You can tell Slick how to connect to the JDBC database of your choice by creating a Database object, which encapsulates the information. There are several factory methods on slick.jdbc.JdbcBackend.Database that you can use depending on what connection data you have available.

* Using Typesafe Config

The prefered way to configure database connections is through Typesafe Config in your application.conf, which is also used by Play and Akka for their configuration.
```scala
mydb = {
  dataSourceClass = "org.postgresql.ds.PGSimpleDataSource"
  properties = {
    databaseName = "mydb"
    user = "myuser"
    password = "secret"
  }
  numThreads = 10
}
```

Such a configuration can be loaded with Database.forConfig (see the API documentation of this method for details on the configuration parameters).
```scala
val db = Database.forConfig("mydb")
```

* Using a JDBC URL

You can pass a JDBC URL to forURL. (see your database’s JDBC driver’s documentation for the correct URL syntax).

```scala
val db = Database.forURL("jdbc:h2:mem:test1;DB_CLOSE_DELAY=-1", driver="org.h2.Driver")
```

Here we are connecting to a new, empty, in-memory H2 database called test1 and keep it resident until the JVM ends (DB_CLOSE_DELAY=-1, which is H2 specific).

* Using a Database URL

A Database URL, a platform independent URL in the form vendor://user:password@host:port/db, is often provided by platforms such as Heroku. You can use a Database URL in Typesafe Config as shown here:
```scala
databaseUrl {
  dataSourceClass = "slick.jdbc.DatabaseUrlDataSource"
  properties = {
    driver = "slick.driver.PostgresDriver$"
    url = "postgres://user:pass@host/dbname"
  }
}
```

By default, the data source will use the value of the DATABASE_URL environment variable. Thus you may omit the url property if the DATABASE_URL environment variable is set. You may also define a custom environment variable with standard Typesafe Config syntax, such as ${?MYSQL_DATABASE_URL}.
Or you may pass a DatabaseUrlDataSource object to forDataSource.
```scala
val db = Database.forDataSource(dataSource: slick.jdbc.DatabaseUrlDataSource)
```

* Using a DataSource

You can pass a DataSource object to forDataSource. If you got it from the connection pool of your application framework, this plugs the pool into Slick.
```scala
val db = Database.forDataSource(dataSource: javax.sql.DataSource)
```

* Using a JNDI Name

If you are using JNDI you can pass a JNDI name to forName under which a DataSource object can be looked up.
```scala
val db = Database.forName(jndiName: String)
```

* Database thread pool

Every Database contains an AsyncExecutor that manages the thread pool for asynchronous execution of Database I/O Actions. Its size is the main parameter to tune for the best performance of the Database object. It should be set to the value that you would use for the size of the connection pool in a traditional, blocking application (see About Pool Sizing in the HikariCP documentation for further information). When using Database.forConfig, the thread pool is configured directly in the external configuration file together with the connection parameters. If you use any other factory method to get a Database, you can either use a default configuration or specify a custom AsyncExecutor:

```scala
val db = Database.forURL("jdbc:h2:mem:test1;DB_CLOSE_DELAY=-1", driver="org.h2.Driver",
  executor = AsyncExecutor("test1", numThreads=10, queueSize=1000))
```
* Connection pools

When using a connection pool (which is always recommended in production environments) the minimum size of the connection pool should also be set to at least the same size. The maximum size of the connection pool can be set much higher than in a blocking application. Any connections beyond the size of the thread pool will only be used when other connections are required to keep a database session open (e.g. while waiting for the result from an asynchronous computation in the middle of a transaction) but are not actively doing any work on the database.
Note that reasonable defaults for the connection pool sizes are calculated from the thread pool size when using Database.forConfig.
Slick uses prepared statements wherever possible but it does not cache them on its own. You should therefore enable prepared statement caching in the connection pool’s configuration.

* DatabaseConfig

On top of the configuration syntax for Database, there is another layer in the form of DatabaseConfig which allows you to configure a Slick driver plus a matching Database together. This makes it easy to abstract over different kinds of database systems by simply changing a configuration file.
Here is a typical DatabaseConfig with a Slick driver object in driver and the database configuration in db:

```scala
tsql {
  driver = "slick.driver.H2Driver$"
  db {
    connectionPool = disabled
    driver = "org.h2.Driver"
    url = "jdbc:h2:mem:tsql1;INIT=runscript from 'src/main/resources/create-schema.sql'"
  }
}
```