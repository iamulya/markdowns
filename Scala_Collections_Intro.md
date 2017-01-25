###Mutable, Immutable
Scala collections systematically distinguish between mutable and immutable collections. A mutable collection can be updated or extended in place. This means you can change, add, or remove elements of a collection as a side effect. Immutable collections, by contrast, never change. You have still operations that simulate additions, removals, or updates, but those operations will in each case return a new collection and leave the old collection unchanged.

All collection classes are found in the package scala.collection or one of its sub-packages mutable, immutable, and generic. Most collection classes needed by client code exist in three variants, which are located in packages scala.collection, scala.collection.immutable, and scala.collection.mutable, respectively. Each variant has different characteristics with respect to mutability.

A collection in package scala.collection.immutable is guaranteed to be immutable for everyone. Such a collection will never change after it is created. Therefore, you can rely on the fact that accessing the same collection value repeatedly at different points in time will always yield a collection with the same elements.

A collection in package scala.collection.mutable is known to have some operations that change the collection in place. So dealing with mutable collection means you need to understand which code changes which collection when.

A collection in package scala.collection can be either mutable or immutable. For instance, collection.IndexedSeq[T] is a superclass of both collection.immutable.IndexedSeq[T] and collection.mutable.IndexedSeq[T] Generally, the root collections in package scala.collection define the same interface as the immutable collections, and the mutable collections in package scala.collection.mutable typically add some side-effecting modification operations to this immutable interface.

The difference between root collections and immutable collections is that clients of an immutable collection have a guarantee that nobody can mutate the collection, whereas clients of a root collection only promise not to change the collection themselves. Even though the static type of such a collection provides no operations for modifying the collection, it might still be possible that the run-time type is a mutable collection which can be changed by other clients.

By default, Scala always picks immutable collections. For instance, if you just write Set without any prefix or without having imported Set from somewhere, you get an immutable set, and if you write Iterable you get an immutable iterable collection, because these are the default bindings imported from the scala package. To get the mutable default versions, you need to write explicitly collection.mutable.Set, or collection.mutable.Iterable.

A useful convention if you want to use both mutable and immutable versions of collections is to import just the package collection.mutable.

import scala.collection.mutable
Then a word like Set without a prefix still refers to an immutable collection, whereas mutable.Set refers to the mutable counterpart.

The last package in the collection hierarchy is collection.generic. This package contains building blocks for implementing collections. Typically, collection classes defer the implementations of some of their operations to classes in generic. Users of the collection framework on the other hand should need to refer to classes in generic only in exceptional circumstances.

For convenience and backwards compatibility some important types have aliases in the scala package, so you can use them by their simple names without needing an import. An example is the List type, which can be accessed alternatively as
```scala
scala.collection.immutable.List   // that's where it is defined
scala.List                        // via the alias in the scala package
List                              // because scala._
                                  // is always automatically imported
```
Other types aliased are Traversable, Iterable, Seq, IndexedSeq, Iterator, Stream, Vector, StringBuilder, and Range.

The following figure shows all collections in package scala.collection. These are all high-level abstract classes or traits, which generally have mutable as well as immutable implementations.
![alt text](http://docs.scala-lang.org/resources/images/collections.png "Scala Collections")

The following figure shows all collections in package scala.collection.immutable.
![alt text](http://docs.scala-lang.org/resources/images/collections.immutable.png "Scala Immutable Collections")

and here: package scala.collection.mutable
![alt text](http://docs.scala-lang.org/resources/images/collections.mutable.png "Scala Immutable Collections")