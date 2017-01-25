### The Problem: The Object/Relational Mismatch

As developers, we know that the object "style" differs significantly from the relational "style", despite many surface-level similarities; rare indeed is the project simple enough to allow for an isomorphic mapping between tables and classes. Attempt to store objects in their native format have largely failed; attempts to write programs in relational languages have also largely failed. As a result, developers face—and, thanks to the popularity of both, will continue to face—a task of "hybridizing" objects and relational data. This is known as the object/relational impedance mismatch.
Many developers, when asked to define the basic tenets of this mismatch, can offer little by way of concrete concerns; to many, it is a "Stewart-esque" problem, hard to define yet clearly recognizable. (U.S. Supreme Court Justice Potter Stewart, when asked to provide a definition of illicit materials, said, "I shall not today attempt further to define the kinds of material but I know it when I see it.") Because solutions can only come from a clear definition of the problem, let's call out some of the basic problems with mixing-and-matching objects and relational data stores in the same program:
* Conflicting type systems: Although they agree on a majority of the types that are available for relational tuple items, vendor databases tend to have unique data types that are not isomorphic in our favorite object-centric programming languages, or have slightly differing semantics. Dates/times, for example, frequently represent a point of translation pain. Perhaps more importantly, the access mechanism for retrieving data out of the database (SQL) yields data that is fundamentally untyped. This will have powerful ramifications for the query mechanism (described below), which will permit query results to be formulated from any part of the data store without restriction beyond that required by the programmer/DBA-defined database integrity constraints.
* Conflicting design goals: It may seem trite to call it out explicitly, but the database system focuses specifically on the storage and retrieval of data, whereas an object system focuses specifically on the union of state and behavior for easier programmer manipulation. (In fact, early object systems were build to focus on user manipulation—the first object systems were essentially user objects intended for direct user access, rather than intermediate objects strung together to form complicated user interface code layers. (Naked Objects by Richard Pawson)) For this reason, it's not uncommon for object system code and database code to require a degree of decoupling from one another, in order to accommodate this differentiation of purpose.
* Conflicting architectural style: Most database products are built to assume a fundamentally client/server style of interaction, assuming the database is located elsewhere on the network, and programs accessing the database will be doing so via some sort of remote access protocol. Object systems assume the precise opposite, and in fact, perform significantly worse when distributed.
* Differing structural relationships: Relational data stores track entities in terms of relations between tuples and tuplesets; object-oriented systems instead prefer to track entities in terms of classes, compilation of state and behavior that relates to one another through IS-A and/or HAS-A style unidirectional connections. Where databases use foreign-key relationships to indicate relations, objects use references or pointers—a seemingly trivial difference, except that while each is a fundamentally one-way relationship, one is from the relator to the "relatee," and the other is the reverse. Object types frequently are designed in a manner most convenient for developers to navigate and use, such that relationships between objects can be varied and wildly chaotic; database relationships are much more strictly structured and tightly defined according to the rules of Normal Form.
* Differing identity constructs: Object systems use an implicit sense of identity to distinguish between objects of similar state (the ubiquitous this pointer or reference), yet databases require that sense of identity to be explicit via primary key column or columns. In fact, in modern object-oriented languages an object system cannot be built without a sense of object identity, whereas relational tables can have no primary key whatsoever, if desired.
* Transactional boundaries: Object systems do not have any sense of "transactional demarcation" when working with the objects, whereas database instances must in order to deal with the multi-user requirements of a modern client/server-based system.
* Query/access capabilities: Retrieving data stored in a relational database makes use of SQL, a declarative language predicated on the mathematical theory of relational algebra and predicate calculus, as described by Codd, and later, Date, in his seminal work on the matter. (An Introduction to Database Systems, 8th Ed., by Chris Date) Data retrieved is generally independent of the data required to navigate and relate that data, meaning that entirely unrelated data elements can be returned as part of a single query, so long as their relationships can be accurately described by SQL. In object systems, the entire object is required in order to navigate from one object to the next, meaning that the entire graph of objects is necessary in order to find two disparate parts of data—for a system intended to remain entirely in working memory, this is of no concern, but for a system whose principal access is intended to be distributed, as relational database are, this can be a crippling problem.

While there are undoubtedly more elements that could be brought up, these are ones that frequently come to mind when discussing the O/R mismatch. In each of these cases, what's mourned is the loss of transparency when dealing with the data store. Or, perhaps to be more accurate, the lack of opacity, to simply let the system handle the data storage details and leave the programmers to focus more on the business domain logic at hand rather than the details of persistence.
What follows is a discussion of the various means programmers have used over the past two decades to access relational systems, and how well these means have met the problems encompassed within the O/R impedance mismatch. When finished, we'll see how Microsoft's newest entry into this problem space, Project LINQ, addresses the mismatch by taking a broader approach.

####Object-Oriented Database Management Systems
One approach to solving the object/relational mismatch is to do away with the relational half of the mismatch entirely, preferring instead to store, retrieve and deal with objects in their native form, rather than "translate" them into a relational form. This would have the advantage of unifying the storage schema with the working program's type definitions, eliminate the mismatch in query APIs, and so on.
This approach was pursued with great vigor in the mid-to-late 1990s, and now, in 2005, it is fairly safe to say that the OODBMS is dead. While it is beyond the scope of this paper to go into a full discussion of why the object database concept did not achieve the sort of success many had predicted (and hoped) it would reach, it is fair to say that the OODBMS suffered from two main detractions, one technical and one political.
For starters, companies had already made sizable investment in relational technologies, and were reluctant to weather another transition again without immediate and sizable return on the investment, which the OODBMS community could never adequately describe or justify. ("Making it easier on your developers" is generally not a sufficient reason for a company to go through a technological upheaval.)
From a technical perspective, however, the OODBMS suffered one major flaw, in that practitioners of OODBMS design failed to recognize that tying the data store schema too tightly to the developers working model is a flaw, not an advantage. Because the data store is often a data repository for more than just one program, when programs' intents begin to differ, so must their working model, leading to direct conflict between development teams. (Many large-scale corporations have gone through the same problem in a different guise: there have been a number of "unified relational model" projects in which the entire company attempts to standardize on a single model; such efforts all almost universally failed, due to the basic fact that a Customer to the billing department is very different from what a Customer looks like to the Accounting department.)
The OODBMS represented the last great effort to bring objects into the data store.

####Call-Level Interfaces (ODBC, JDBC, ADO.NET)
First popularized with ODBC (Open DataBase Connectivity) back in the mid-90s and replicated in Java (JDBC) and .NET (ADO.NET), the call-level interface provides a basic "object-ish" API accessing relational data by taking a SQL query, passed as a string literal into the library, and returning either a scalar result or a collection of tuples that are consumed as a list of individual tuples (what developers will casually refer to as "rows")—each with named elements ("columns")—containing the name of, and the data for, that element.
In of itself, this style is intrinsically easy to understand and utilize, as it minimizes the degree of encapsulation away from the fundamental architecture of the RDBMS: Queries or statements are formulated as strings (an easy data type to manipulate—even in C++, at the time of this writing); the format of the query or statement is in straight SQL; and the results are often easily typecast to other programming language types through the user of utility methods provided by the library. Depending on the CLI library in question, if this is a potentially unsafe conversion, say from a VARCHAR database column to an integer programming language type, the attempted conversion will either silently fail or else raise some kind of programmatic exception or error to be handled at the programmer's discretion. (CLI refers as an acronym for "Call-Level Interface," and is not to be confused with the (unfortunately overloaded) acronym "Common Language Infrastructure," the ECMA and ISO specification defining the behavior of execution environments such as the CLR.)
Because the data retrieved is essentially typeless beyond this basic tupleset/tuple notion, the returned data also contains a description of the tupleset, also known as the metadata for the set. Developers will sometimes consume this metadata in order to validate the data they believe they are receiving, or to provide some kind of user-visible hint as to the data being returned, but more often than not this metadata is simply discarded.
Unfortunately, CLI code has a tendency to be verbose, forcing the programmer to go through a wide variety of steps before reaching the point of real business value—fetching or updating the data. For example, consider this JDBC code, used to retrieve all the Customers from the Northwind database (a sample database whose definition ships with SQL Server and Visual Studio) who live in Seattle:

```java
Connection conn = // obtained from someplace
try {
  PreparedStatement stmt =
    conn.prepareStatement("SELECT * FROM Customers WHERE city=?");
  stmt.setString(1, "Seattle");
  ResultSet rslt = stmt.executeQuery();
  while (rslt.getNext()) {
    // Get the contents of the rows
    String companyName = rslt.getString(1);
    String contactName = rslt.getString(2);
    // and so on
  }
}
catch (SQLException sqlEx) {
  // Handle it appropriately, whatever that means
}
finally {
  conn.close();
}
```
Note that this code sample has chosen to be explicit about its error handling, rather than pass the JDBC-mandated SQLException to higher layers of code to be handled, as proper n-tier design would suggest that persistence concerns should be handled entirely within the persistence layer. ADO.NET, the relational data access CLI for .NET, looks similar, almost to the letter, except for the mandatory JDBC exception handling.
In addition to its verbosity, once common criticism of the CLI is that it allows a dangerous tendency to "scatter" SQL code across the codebase, making it difficult to evolve the database schema in a manner independent of the object codebase. For smaller projects, this is arguably not a concern, as the programmer and database designer will often be the same person. In larger teams, however, this is rarely the case, and certainly not all programmers will be database designers or vice versa. Solving this problem requires developer discipline, either by placing "comment tags" in front of all such code access, making it easier for programmers to find and update SQL statements in response to database schema changes, or by reading SQL statements in from an external storage mechanism, such as text-based or XML-based file.
One advantage this approach does have, however, is that there are no layers of indirection or encapsulation to have to work around; in fact, there's really nothing there between the programmer and the data store. Programmers are executing queries "in the raw" against the data store, and receiving the data in the fundamentally untyped nature of SQL tuplesets. This is both strength and weakness: while it enables developers to take advantage of database vendor value-added features in the query language dialect for that database, it also requires developers to be familiar with said features and its nuances and drawbacks.

####Code-Generation
In order to mitigate some of the criticisms of the CLI, developers have fallen back to (and, some would argue, perfected) the ancient art of code generation by executing desired SQL statements through the CLI against the database, examining the returned metadata, and creating strongly typed class wrappers around that information. In some cases, the code is written to allow for a more encapsulated style, such that a given instance of the generated class can be used to perform all data-oriented operations (also known as CRUD, short for Create/Retrieve/Update/Delete), making it an Active Record.
Tools which do this, which often as not are hand-rolled within an organization, still face some of the basic problems of the CLI on which they rest: because, for example, the code generated is based on relational metadata, the generated classes do little to provide the kind of object-oriented Domain Model most developers steeped in object thinking prefer to design and use. As a result, some kind of "mapping" must still occur between the Domain Model and the generated data access code that can result in discomfort in places.

####Embedded SQL (SQLJ)
SQLJ (or SQL/J, as it's sometimes written) is the latest evolution in the embedded SQL approach, in which SQL statements are written directly inside the programming language (hence the 'embedded' moniker). The code is passed through a translator, which extracts the SQL statements into an alternate form (specific to the environment), and passes the remainder of the code on to the compiler for compilation. One of the most popular embedded SQL environments was the Oracle Pro*C compiler; unfortunately, Pro*C was specific to Oracle, and as a result was passed over by ODBC, which offered a vendor-neutral database access solution.
SQLJ programs are a direct variant of the Java programming language, in which SQL statements are passed through a SQLJ translator, and extracted into .java files containing normal Java code, and usually a .ser file containing the SQL statements and other configuration data. (The details vary from translator to translator.) The code is then compiled using the normal Java compiler, and at runtime uses the .ser file in conjunction with the SQLJ runtime libraries to access and update the data store.
For example, using SQLJ to access a relational database containing the Microsoft Northwind database would look like the following:

```java
#sql public iterator CustomerIter (int, String, String);

// Declare and initialize host variables
int id=0; String name=null; String city=null;

// Declare an iterator instance
CustomerIter custs;
#sql custs = { SELECT CustomerId, CompnyName, City FROM Customers };
while (true) {
  #sql { FETCH :custs INTO :id, :name, :city };

  if (custs.endFetch())
    break;

  // Use id, name, city
}
custs.close();
```

Doing this minimizes the verbosity of the code, but despite appearances, embedded SQL statements don't fit within the type system of the language, and as a result have to "graft on" to the underlying type system. This means that there will still be database-to-language mapping issues that will remain unresolvable—the INTEGER column that allows NULL as a valid value, for example, is impossible to represent in a Java int variable.
What the SQLJ translator will do, however, is check the validity of the SQL at compile-time, thus catching the misspelled "CompnyName" (which should be CompanyName) in the code above. This is in keeping with the principles of a statically typed and bound language. Because Java is statically typed, however, the iterator type itself (CustomerIter) must be declared beforehand, and is done so at the top of the example; should the SQL query using the CustomerIter result type change, the CustomerIter explicit declaration must change with it.
Despite the advantages of SQLJ, and its position as an accepted ANSI standard, SQLJ never grew to acceptance within the Java community, and as of this writing even the reference implementation and Web site to support SQLJ has been discontinued. The reason? Java developers had found a new love: automated object/relational mapping.

####Automated Mapping (EJB 3.0/JDO 2.0/(N)Hibernate)
An automating object/relational mapping tool distinguishes itself from its brethren by its attempt to seamlessly and silently create a translation layer to transform objects into relational data and back again. In the ideal automated mapping, the programmer working with the object model sees no SQL, no hint of the relational system "peeking through" to complicate the object programmer's life. In essence, the desire is to create a transparent persistence layer.
Numerous papers have been written about the approaches object/relational developers can take to provide said mapping, and many vendors have pursued this task with vigor and energy unequalled by any other task in Computer Science. Java, no different than any other strongly typed object-oriented environment, has done so through several efforts, most notably first the Entity Bean portion of the EJB Specification (which has seen several significant revisions already prior to the forthcoming 3.0 release), then later the Java Data Objects (JDO) Specification. Concurrently, developers frustrated with the limitations of both the Entity Bean and JDO specs pursued their own approaches, the most successful of the lot being the Hibernate library. Now, the EJB 3 specification seeks to reunify the JDO and EJB persistence approaches, with major input from Hibernate's principal authors, in an effort to rein in the persistence development efforts and bring order back to the Java persistence chaos.
Welcome to the quagmire.
Unfortunately, because Hibernate, JDO 2 and EJB 3 all sit "on top" of the language as written, they suffer from many of the same problems as all automated mapping tools do: attempts to hide data retrieval strategies behind object interfaces, which goes hand in hand with attempts to model relationships between tables in the data store with relationships between objects, and an inability to address strongly typed "partial object" queries being two of the major problems.
For example, consider this Entity type for Customers (again, against Northwind) that uses the proposed EJB 3 annotations defined in the Public Draft:

```java
@Entity
public class Customer {
  @Id(generate=GeneratorType.NONE) String customerID;
  @Basic String companyName;
  @Basic String contactName;
  @Basic String contactTitle;
  @Basic String address;
  @Basic String city;
  @Basic String region;
  @Basic String postalCode;
  @Basic String country;
  @Basic String phone;
  @Basic String fax;
  @OneToMany(targetEntity=com.tedneward.Orders.class,
             mappedBy="customer")
  Collection orders;

  public Customer() { }

  public String getId() { return customerID; }
  // usual collection of get/set pairs go here
}
```

Notice how, like many such type-annotated approaches, the details of the association of Customers to Orders is defined in the code, using an underlying Java Collection object to hold the orders for a particular customer. This represents a concrete example of the conflicting architectural style problem described above; when retrieving a Customer instance (that is, creating an instance of type Customer and fetching its corresponding data from the database for population into the fields of the newly created Customer object), should the database fetch all of the order information for that Customer as well?
This answer is one that's plagued many O/R mapping systems—this kind of location transparency, where the developer need not pay attention to the details of how and when the data is retrieved—is difficult to get right by default. If the system chooses to eager-load the data, and retrieve the orders as part of the customer, then fetching each Customer for displaying Customer company name and country on a drilldown page is horribly wasteful, consuming potentially millions of bytes of network bandwidth to retrieve orders not needed for this page. If the system chooses to lazy-load the data, and retrieve the orders only upon demand, then the system will fall victim to the N+1 query problem, as it became known in the Java/EJB lexicon: each request to the each Customer for its orders will result in a distinct and unique query back to the database for the order associated with that Customer, thus requiring "N" queries (for each Order) plus the query for the Customer itself. Whichever way an O/R mapping selects, there is a usage model that will generate the absolute worst performance possible.
The EJB3 persistence scheme attempts to compensate for this problem, as many modern O/R mappers do, by allowing developers to define, in the Customer type, how the Orders should be fetched:

```java
public class Customer {
  // . . .

  @OneToMany(targetEntity=com.tedneward.Orders.class,
             fetch=FetchType.LAZY,
             mappedBy="customer")
  Collection orders;
}
```

In this case it means that we, as the developer, are telling the system that we deliberately want Orders to be fetched lazily, only when the Customer demands the order collection in question. (Note that there are still some ambiguities here, as it's not clear from the EJB 3 specification whether all Orders will be fetched when the Collection is first referenced, or whether Orders will be individually fetched-on-demand; either solution would be reasonable in certain situations.)
Unfortunately, even this is not enough to handle all possible scenarios effectively, as there will be times, even for the same Customer type, when both eager-loading and lazy-loading would be appropriate, the classic scenario being that drilldown page: we want lazy-loading when fetching the initial list of all Customers, but then want to eager-load the Customer and its Orders when retrieving the individual Customer to drill into.
Because of this problem, EJB 3 and JDO offer a query language to write customized, complex queries that are able to retrieve exactly the data desired when desired. In EJB, this language was known as EJBQL, and for JDO, JDOQL. (Hibernate defined its own query language over time as well, known—naturally—as HQL.) Thus, a developer could query precisely for what was required at the time it is required—as with SQL—and not pay the additional price of the automated mapping. Thus, an EJB3 developer could write:

```java
public class Customer {
  public List findOrdersForCustomer(String customer) {
    return em.createQuery(
      "SELECT o FROM Orders o, Customer c " +
      "WHERE c.CustomerName LIKE :customer AND " +
      "o.CustomerID == c.CustomerID")
    .setParameter("customer", customer)
    .setMaxResults(10)
    .getResultList();
  }
}
```
(Notice how EJB 3 allows for call-chaining, where each object method returns itself, allowing for what appears to be multiple method calls in one virtual line. This is designed to minimize the verbosity of the Query API, arguably at the expense of readability.) However, we now have a Call-Level Interface API, with all of the basic problems plaguing that approach: untyped results being one of the core problems.
One such automated mapping that is taking the development world by storm is that of Rails, a Web-and-database framework written on top of Ruby, a dynamic object-oriented scripting language developed in the open source community. Basing itself on common-sense defaults and scaffolding, Rails enables a relatively straightforward mapping against the database by keying off of constructs in the types defined in code itself; for example, the following code defines a class that defines itself around the "Customer" table of the Northwind database:

```ruby
require "rubygems"
require_gem "activerecord"

class Customer < ActiveRecord::Base
end

customer = Customer.find(123)
customer.ContactName = "Dave Thomas"
customer.save
```

Experienced developers will quickly notice the lack of anything resembling properties or methods in the Customer class (defined to inherit from the Rails ActiveRecord::Base class). Rails is able to provide all the columns of the table as fields on the object "customer" because of the highly dynamic nature of Ruby—when a method or property is accessed on an object where no such property exists, Ruby will call a base class method designed to trap all such "unknown" accesses. ActiveRecord::Base overrides that method to then examine the name of the method invoked, compare it against the metadata for the table retrieved from the database, and return that data from the row retrieved earlier.
While interesting, this highly dynamic approach faces two fundamental problems: it is highly sensitive to changes to the database schema definitions, and it exposes the schema to developers to deal with in a direct fashion. While Rails presents some ways around these two problems, fundamentally they are similar in nature to the other automated libraries already discussed.
What is perhaps most interesting to this discussion, however, is the ease by which Rails allows the introduction of new types into the system. Should a developer require some kind of "partial object result" of Customers for a particular query, such as is necessary for drilldown screens navigating through all Customers before displaying all data for a particular Customer? It is trivial (two lines of code) to create a Rails type that encompasses that particular data (which would likely have to be formulated as a database view, since Rails offers no such wrapping around arbitrary SQL statements).

####