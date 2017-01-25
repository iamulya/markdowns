The Slick code generator is a convenient tool for working with an existing or evolving database schema. It can be run stand-alone or integrated into you sbt build for creating all code Slick needs to work.

###Overview
By default, the code generator generates Table classes, corresponding TableQuery values, which can be used in a collection-like manner, as well as case classes for holding complete rows of values. For tables with more than 22 columns the generator automatically switches to Slick’s experimental HList implementation for overcoming Scala’s tuple size limit. (In Scala <= 2.10.3 use HCons instead of :: as a type contructor due to performance issues during compilation, which are fixed in 2.10.4 and later.)

###Standalone use
To include Slick’s code generator use the published library. For sbt projects add following to your build definition - build.sbt or project/Build.scala:
libraryDependencies += "com.typesafe.slick" %% "slick-codegen" % "3.1.1"
For Maven projects add the following to your <dependencies>:
```xml
<dependency>
  <groupId>com.typesafe.slick</groupId>
  <artifactId>slick-codegen_2.10</artifactId>
  <version>3.1.1</version>
</dependency>
```

Slick’s code generator comes with a default runner that can be used from the command line or from Java/Scala. You can simply execute
```scala
slick.codegen.SourceCodeGenerator.main(
  Array(slickDriver, jdbcDriver, url, outputFolder, pkg)
)
```
or
```scala
slick.codegen.SourceCodeGenerator.main(
  Array(slickDriver, jdbcDriver, url, outputFolder, pkg, user, password)
)
```
and provide the following values:
* slickDriver Fully qualified name of Slick driver class, e.g. “slick.driver.H2Driver”
* jdbcDriver Fully qualified name of jdbc driver class, e.g. “org.h2.Driver”
* url jdbc url, e.g. “jdbc:postgresql://localhost/test”
* outputFolder Place where the package folder structure should be put
* pkg Scala package the generated code should be places in
* user database connection user name
* password database connection password

###Integrated into sbt
The code generator can be run before every compilation or manually in sbt. An example project showing both can be found [here](https://github.com/slick/slick-codegen-example).

###Generated Code

By default, the code generator places a file Tables.scala in the given folder in a subfolder corresponding to the package. The file contains an object Tables from which the code can be imported for use right away. Make sure you use the same Slick driver. The file also contains a trait Tables which can be used in the cake pattern.

> When using the generated code, be careful not to mix different database drivers accidentally. The default object Tables uses the driver used during code generation. Using it together with a different driver for queries will lead to runtime errors. The generated trait Tables can be used with a different driver, but be aware, that this is currently untested and not officially supported. It may or may not work in your case. We will officially support this at some point in the future.

###Customization

The generator can be flexibly customized by overriding methods to programmatically generate any code based on the data model. This can be used for minor customizations as well as heavy, model driven code generation, e.g. for framework bindings in Play, other data-related, repetitive sections of applications, etc.
This example shows a customized code-generator and how to setup up a multi-project sbt build, which compiles and runs it before compiling the main sources.
The implementation of the code generator is structured into a small hierarchy of sub-generators responsible for different fragments of the complete output. The implementation of each sub-generator can be swapped out for a customized one by overriding the corresponding factory method. SourceCodeGenerator contains a factory method Table, which it uses to generate a sub-generator for each table. The sub-generator Table in turn contains sub-generators for Table classes, entity case classes, columns, key, indices, etc. Custom sub-generators can easily be added as well.
Within the sub-generators the relevant part of the Slick data model can be accessed to drive the code generation.
Please see the api documentation for info on all of the methods that can be overridden for customization.
Here is an example for customizing the generator:
```scala
import slick.codegen.SourceCodeGenerator
// fetch data model
val modelAction = H2Driver.createModel(Some(H2Driver.defaultTables)) // you can filter specific tables here
val modelFuture = db.run(modelAction)
// customize code generator
val codegenFuture = modelFuture.map(model => new SourceCodeGenerator(model) {
  // override mapped table and class name
  override def entityName =
    dbTableName => dbTableName.dropRight(1).toLowerCase.toCamelCase
  override def tableName =
    dbTableName => dbTableName.toLowerCase.toCamelCase

  // add some custom import
  override def code = "import foo.{MyCustomType,MyCustomTypeMapper}" + "\n" + super.code

  // override table generator
  override def Table = new Table(_){
    // disable entity class generation and mapping
    override def EntityType = new EntityType{
      override def classEnabled = false
    }

     // override contained column generator
    override def Column = new Column(_){
      // use the data model member of this column to change the Scala type,
      // e.g. to a custom enum or anything else
      override def rawType =
        if(model.name == "SOME_SPECIAL_COLUMN_NAME") "MyCustomType" else super.rawType
    }
  }
})
codegenFuture.onSuccess { case codegen =>
  codegen.writeToFile(
    "slick.driver.H2Driver","some/folder/","some.packag","Tables","Tables.scala"
  )
}
```