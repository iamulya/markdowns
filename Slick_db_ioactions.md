Anything that you can execute on a database, whether it is a getting the result of a query (myQuery.result), creating a table (myTable.schema.create), inserting data (myTable += item) or something else, is an instance of DBIOAction, parameterized by the result type it will produce when you execute it.
Database I/O Actions can be combined with several different combinators (see the DBIOAction class and DBIO object for details), but they will always be executed strictly sequentially and (at least conceptually) in a single database session.
In most cases you will want to use the type aliases DBIO and StreamingDBIO for non-streaming and streaming Database I/O Actions. They omit the optional effect types supported by DBIOAction.

* Executing Database I/O Actions

DBIOActions can be executed either with the goal of producing a fully materialized result or streaming data back from the database.
1. Materialized
You can use run to execute a DBIOAction on a Database and produce a materialized result. This can be, for example, a scalar query result (myTable.length.result), a collection-valued query result (myTable.to[Set].result), or any other action. Every DBIOAction supports this mode of execution.
Execution of the action starts when run is called, and the materialized result is returned as a Future which is completed asynchronously as soon as the result is available:
```scala
val q = for (c <- coffees) yield c.name
val a = q.result
val f: Future[Seq[String]] = db.run(a)


f.onSuccess { case s => println(s"Result: $s") }
```

2. Streaming
Collection-valued queries also support streaming results. In this case, the actual collection type is ignored and elements are streamed directly from the result set through a Reactive Streams Publisher, which can be processed and consumed by Akka Streams.
Execution of the DBIOAction does not start until a Subscriber is attached to the stream. Only a single Subscriber is supported, and any further attempts to subscribe again will fail. Stream elements are signaled as soon as they become available in the streaming part of the DBIOAction. The end of the stream is signaled only after the entire action has completed. For example, when streaming inside a transaction and all elements have been delivered successfully, the stream can still fail afterwards if the transaction cannot be committed.

```scala
val q = for (c <- coffees) yield c.name
val a = q.result
val p: DatabasePublisher[String] = db.stream(a)

// .foreach is a convenience method on DatabasePublisher.
// Use Akka Streams for more elaborate stream processing.
p.foreach { s => println(s"Element: $s") }
```
When streaming a JDBC result set, the next result page will be buffered in the background if the Subscriber is not ready to receive more data, but all elements are signaled synchronously and the result set is not advanced before synchronous processing is finished. This allows synchronous callbacks to low-level JDBC values like Blob which depend on the state of the result set. The convenience method mapResult is provided for this purpose:

```scala
val q = for (c <- coffees) yield c.image
val a = q.result
val p1: DatabasePublisher[Blob] = db.stream(a)
val p2: DatabasePublisher[Array[Byte]] = p1.mapResult { b =>
  b.getBytes(0, b.length().toInt)
}
```

###Composing Database I/O Actions

DBIOActions describe sequences of individual actions to execute in strictly sequential order on one database session (at least conceptually), therefore the most commonly used combinators deal with sequencing. Since a DBIOAction eventually results in a Success or Failure, its combinators, just like the ones on Future, have to distinguish between successful and failed executions. Unless specifically noted, all combinators only apply to successful actions. Any failure will abort the sequence of execution and result in a failed Future or Reactive Stream.

####Sequential Execution
The simplest combinator is DBIO.seq which takes a varargs list of actions to run in sequence, discarding their return value. If you need the return value, you can use andThen to combine two actions and keep the result of the second one. If you need both return values of two actions, there is the zip combinator. For getting all result values from a sequence of actions (of compatible types), use DBIO.sequence. All these combinators work with pre-existing DBIOActions which are composed eagerly.
If an action depends on a previous action in the sequence, you have to compute it on the fly with flatMap or map. These two methods plus filter enable the use of for comprehensions for action sequencing. Since they take function arguments, they also require an implicit ExecutionContext on which to run the function. This way Slick ensures that no non-database code is run on the database thread pool.

> You should prefer the less flexible methods without an ExecutionContext where possible. The resulting actions can be executed more efficiently.

Similar to DBIO.sequence for upfront composition, there is DBIO.fold for working with sequences of actions and composing them based on the previous result.

* Error Handling

You can use andFinally to perform a cleanup action, no matter whether the previous action succeeded or failed. This is similar to using try ... finally ... in imperative Scala code. A more flexible version of andFinally is cleanUp. It lets you transform the failure and decide how to fail the resulting action if both the original one and the cleanup failed.

> For even more flexible error handling use asTry and failed. Unlike with andFinally and cleanUp the resulting actions cannot be used for streaming.

* Primitives
You can convert a Future into an action with DBIO.from. This allows the result of the Future to be used in an action sequence. A pre-existing value or failure can be converted with DBIO.successful and DBIO.failed, respectively.

* Debugging
The named combinator names an action. This name can be seen in debug logs if you enable the slick.backend.DatabaseComponent.action logger.

* Transactions and Pinned Sessions
When executing a DBIOAction which is composed of several smaller actions, Slick acquires sessions from the connection pool and releases them again as needed so that a session is not kept in use unnecessarily while waiting for the result from a non-database computation (e.g. the function passed to flatMap that determines the next Action to run). All DBIOAction combinators which combine two database actions without any non-database computations in between (e.g. andThen or zip) can fuse these actions for more efficient execution, with the side-effect that the fused action runs inside a single session. You can use withPinnedSession to force the use of a single session, keeping the existing session open even when waiting for non-database computations.
There is a similar combinator called transactionally to force the use of a transaction. This guarantees that the entire DBIOAction that is executed will either succeed or fail atomically.

> Failure is not guaranteed to be atomic at the level of an individual DBIOAction that is wrapped with transactionally, so you should not apply error recovery combinators at that point. An actual database transaction is inly created and committed / rolled back for the outermost transactionally action.

```scala
val a = (for {
  ns <- coffees.filter(_.name.startsWith("ESPRESSO")).map(_.name).result
  _ <- DBIO.seq(ns.map(n => coffees.filter(_.name === n).delete): _*)
} yield ()).transactionally

val f: Future[Unit] = db.run(a)
```

* JDBC Interoperability

In order to drop down to the JDBC level for functionality that is not available in Slick, you can use a SimpleDBIO action which is run on a database thread and gets access to the JDBC Connection:

```scala
val getAutoCommit = SimpleDBIO[Boolean](_.connection.getAutoCommit)
```