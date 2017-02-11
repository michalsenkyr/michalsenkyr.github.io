---
title: How to create a custom Spark SQL data source (using Parboiled2)
date: 2017-02-11 21:27
layout: post
category: spark
tags: scala spark sparksql datasource parboiled
---

If you work with Big Data, you have probably heard about Apache Spark, the popular engine for large-scale distributed data processing. And if you followed the project's development, you know that its original RDD model was superseded by the much faster DataFrame model.
Unfortunately, to gain in performance the model became much less weildy due to the new requirement of data schema specification. This was improved on by the presently used Dataset model which provided automatic schema inference based on language types, however the core logic remained largely the same. Because of that, extending the model is not such an easy task as one would think (especially if you want to do it properly).

In this article, I will demonstrate how to create a custom data source that uses Parboiled2 to read custom time-ordered logs efficiently.


## Exercise definition

For the purpose of this exercise, let's consider a situation in which we run multiple applications on a cluster. Each application is hosted by multiple nodes and each host node logs its activity into a file on the HDFS filesystem which is rolled at midnight of each day.

![HDFS log directory structure](/images/2017-02-09-spark-sql_datasource/dirtree.svg)

Each activity log is textual (compressed using gzip) and has the following contents:

```
2017-02-08T22:55:20 com.github.michalsenkyr.example.web.RequestHandler DEBUG Received a request
2017-02-08T22:55:23 com.github.michalsenkyr.example.web.RequestHandler DEBUG Sent a response
2017-02-08T22:56:11 com.github.michalsenkyr.example.web.RequestHandler DEBUG Received a request
2017-02-08T23:01:11 com.github.michalsenkyr.example.web.RequestHandler WARN  Request timed out
```

Our goal is to process these log files using Spark SQL. We expect the user's query to always specify the application and time interval for which to retrieve the log records. Additionally, we would like to abstract access to the log files as much as possible. This means that we would like the entire log record database to act as one big table from the user's standpoint and completely hide the file selection process.

## Creating a new Spark-compatible project

In order to build our new project, we will use [Simple Build Tool (SBT)](http://www.scala-sbt.org/) as it integrates natively with the Scala programming language and doesn't require much to set up. [Maven](https://maven.apache.org/) can also be used, but [a plugin is needed for Scala support](http://docs.scala-lang.org/tutorials/scala-with-maven.html).

In addition, we will use the [sbt-assembly](https://github.com/sbt/sbt-assembly) plugin to build uberjar bundles (jars packaged with dependencies) in order to easily send them to Spark.

SBT requires a *build.sbt* file to be present in the root of the project with basic project information (name, version, dependencies, etc.). Ours will look like this:

*build.sbt*

```scala
name := "spark-datasource-test"

version := "1.0-SNAPSHOT"

scalaVersion := "2.11.8"

libraryDependencies ++= Seq(
  "org.parboiled" %% "parboiled" % "2.1.4",
  "org.apache.spark" %% "spark-sql" % "2.1.0" % "provided"
)
```

Note that we're using Scala 2.11.8 instead of the current 2.12.1, because Spark is not yet compatible with the 2.12.x versions.

In order to use the *sbt-assembly* plugin, we also need a *project/assembly.sbt* file with the following contents:

*project/assembly.sbt*

```scala
addSbtPlugin("com.eed3si9n" % "sbt-assembly" % "0.14.3")
```

Now we can run the basic Scala REPL by running:

```bash
sbt console
```

We can build the uberjar by running:

```bash
sbt assembly
```

And we can bundle it with our Spark shell command:

```bash
spark-shell --jars target/scala-2.11/spark-datasource-test-assembly-1.0-SNAPSHOT.jar
```

We can also send it to the cluster using the Spark submit command (assuming we have a main class to run):

```bash
spark-submit --class com.github.michalsenkyr.example.MainClass target/scala-2.11/spark-datasource-test-assembly-1.0-SNAPSHOT.jar
```

## Parsing using Parboiled2

First, let's describe the log record format and create a parser that will be able to read it.

![Log record format](/images/2017-02-09-spark-sql_datasource/logrecord.svg)

As we can see, each line contains a separate record, which is divided into four columns:

1. Formatted date and time (ISO-8601)
2. Source component
3. Log level
4. Message (can contain whitespace)

To parse this log, we will use the powerful [Parboiled2 library](https://github.com/sirthias/parboiled2) which uses Scala macros to construct very efficient parsers from an easy to understand [DSL](https://github.com/sirthias/parboiled2#the-rule-dsl). It uses [parsing expression grammars (PEG)](https://en.wikipedia.org/wiki/Parsing_expression_grammar) instead of the popular regular expressions as they are strictly more powerful, representable in DSL without interpreting a string representation first and can be decomposed into individual parsing components which makes them much easier to read.

Parboiled2 provides many basic building components. Here is a list of the most important ones (the ones we're interested in are bolded):

* **`ANY`** - matches any character
* `anyOf(s)` - matches any character contained in string `s`
* **`noneOf(s)`** - matches any character except the ones contained in string `s`
* `ignoreCase(s)` - matches the given string case-insensitively (Note: `s` must be lowercase)
* `a?` - matches zero or one occurence of `a`
* `a*` - matches zero or more occurences of `a`
* **`a+`** - matches one or more occurences of `a`
* **`a ~ b`** - matches the concatenation of `a` and `b`
* `a | b` - matches either `a` or `b`
* `&a` - positive match (without consuming)
* `!a` - negative match (without consuming)
* `capture(a)` - matches `a` and captures its value for later use
* `a ~> (f)` - transforms the captured values from `a` using function `f`
* **`a ~> CaseClass`** - constructs a case class `CaseClass` using the captured values from `a`
* `MATCH` - empty match
* `MISMATCH` - matches nothing (always fails)
* **`EOI`** - matches the end of input

Every Parboiled2 parser is divided into rules which combine these building blocks as well as other rules. These rules are specified using the `rule` notation in the following way:

```scala
def ruleName = rule {
  ...
}
```

We can divide our log record parser into the following rules:

![Rule graph](/images/2017-02-09-spark-sql_datasource/rules.svg)

Descriptions of the rules' operations:

* `WhiteSpaceChar` - a whitespace character
* `NonWhiteSpaceChar` - a non-whitespace character
* `WhiteSpace` - a non-empty string of whitespace characters
* `Field` - a string of non-whitespace characters (capture is added to put the value on stack)
* `MessageField` - match (and capture) the rest of the line
* `DateTimeField` - converts a `Field` into a `java.sql.Timestamp` instance (one of the classes natively supported by Spark SQL)
* `Record` - match the whole record and return a case class containing the captured values
* `Line` - as we will parse the input by individual lines, we expect input to end (`EOI`) after a record

The code to define these rules is as follows:

```scala
case class LogRecord(dateTime: Timestamp, component: String, level: String, message: String)

class LogParser(val input: ParserInput) extends Parser {
  def WhiteSpaceChar = rule {
    anyOf(" \t")
  }

  def NonWhiteSpaceChar = rule {
    noneOf(" \t")
  }

  def WhiteSpace = rule {
    WhiteSpaceChar+
  }

  def Field = rule {
    capture(NonWhiteSpaceChar+)
  }

  def MessageField = rule {
    capture(ANY+)
  }

  def DateTimeField = rule {
    Field ~> { str =>
      Timestamp.valueOf(LocalDateTime.parse(str, DateTimeFormatter.ISO_LOCAL_DATE_TIME))
    }
  }

  def Record = rule {
    DateTimeField ~ WhiteSpace ~
      Field ~ WhiteSpace ~
      Field ~ WhiteSpace ~
      MessageField ~> LogRecord
  }

  def Line = rule {
    Record ~ EOI
  }
}
```

Special considerations:

* `NonWhiteSpaceChar` rule - can be also written as `!WhiteSpaceChar ~ ANY` (negative match rule must be combined with `ANY` as it does not consume a character)
* `DateTimeField` rule - `LocalDateTime` intermediate parser is used because `java.sql.Timestamp.valueOf` doesn't support ISO-8601
* `Line` rule - `EOI` is not strictly necessary as we match the rest of the line using the `MessageField` rule, but it is good practice to mark the top-level rule

Let's test it out using Scala REPL:

```
scala> println(new LogParser("2017-02-09T00:09:27  com.github.michalsenkyr.example.parser.ParserTester  INFO  Started parsing").Line.run())
Success((2017-02-09 00:09:27.0,com.github.michalsenkyr.example.parser.ParserTester,INFO,Started parsing))
```

## Implementing a Spark data source relation

In order to create a proper Spark data source, we need to implement the [`BaseRelation` abstract class](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.sources.BaseRelation). Our implementation must also mix in one of the `Scan` traits. These determine how much information on the user's query we want to be provided in order to retrieve the required data more efficiently.

Possible `Scan` traits include:

* [`TableScan`](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.sources.TableScan) - the most basic scan - provides no information on the executed query
* [`PrunedScan`](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.sources.PrunedScan) - provides information on the columns required by the query
* [`PrunedFilteredScan`](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.sources.PrunedFilteredScan) - provides information on the columns required by the query as well as the filters specified (note: these are the filters as understood by Spark's query planner and can differ from those provided by the user)
* [`CatalystScan`](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.sources.CatalystScan) - provides raw expressions from the query planner (note: this is experimental and isn't intended for common use)

We will implement the most specific `PrunedFilteredScan` as we need the information on the query's filters in order to choose the correct log file. This trait requires the `buildScan` method to be implemented, which provides the SQL context and the query's filters as parameters. Each filter can be easily pattern-matched and is one of the following:

* `IsNull`, `IsNotNull` - null tests
* `EqualTo`, `EqualNullSafe` - equality tests
* `GreaterThan`, `LessThan`, `GreaterThanOrEqual`, `LessThanOrEqual` - comparisons
* `StringStartsWith`, `StringContains`, `StringEndsWith` - string tests
* `In` - inclusion test
* `Not`, `And`, `Or` - boolean composition

As the query planner already minimizes the query filters and puts them into a convenient array (containing all the required conditions), we only need an equality test to build the application portion and the comparisons to build the date portion of our path pattern. That is assuming, of course, that we do not want to handle more complex queries (like selecting more applications or more time intervals at once), which, for the sake of simplicity, we do not.

To read the resulting files into an RDD, we can use [SparkContext](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.SparkContext)'s textFile method (which reads files line by line and properly manages partitioning).

```scala
override def buildScan(requiredColumns: Array[String], filters: Array[Filter]): RDD[Row] = {
  val (application, from, to) = filters
    .foldLeft((Option.empty[String], Option.empty[Timestamp], Option.empty[Timestamp])) {
      case (x, EqualTo("application", value: String)) => x.copy(_1 = Some(value))
      case (x, GreaterThan("dateTime", value: Timestamp)) => x.copy(_2 = Some(value))
      case (x, GreaterThanOrEqual("dateTime", value: Timestamp)) => x.copy(_2 = Some(value))
      case (x, LessThan("dateTime", value: Timestamp)) => x.copy(_3 = Some(value))
      case (x, LessThanOrEqual("dateTime", value: Timestamp)) => x.copy(_3 = Some(value))
      case (x, _) => x
    }
  if (application.isEmpty) throw LogDataSourceException("Application not specified")
  if (from.isEmpty) throw LogDataSourceException("Timestamp lower bound not specified")
  val (fromLimit, toLimit) = (
    from.get.toLocalDateTime.toLocalDate,
    to.map(_.toLocalDateTime.toLocalDate).getOrElse(LocalDate.now)
  )
  val pathPattern =
    Iterator.iterate(fromLimit)(_.plusDays(1))
      .takeWhile(!_.isAfter(toLimit))
      .map(_.format(DateTimeFormatter.ISO_LOCAL_DATE))
      .mkString(s"$path/${application.get}/*/{", ",", "}.gz")

  sqlContext.sparkContext.textFile(pathPattern).map { line =>
    val record = new LogParser(line).Line.run().get
    Row(application.get, record.dateTime, record.component, record.level, record.message)
  }
}
```

Here we first go through all the filters and extract required information on the application and the timestamp bounds (upper bound is optional). Then we derive the date limits (of all the files that include the log records from the query) for the path pattern and construct it. Lastly, we obtain the Spark's context and use its `textFile` method to facilitate reading the record lines into an RDD and map them through our parser.

Next, we need to properly define the schema of the `Row`s that our relation returns and provide them to Spark by implementing the `schema` method. Spark's DataFrame API supports the [following types](http://spark.apache.org/docs/latest/sql-programming-guide.html#data-types):

* `NullType` - represents a `null` value
* `BooleanType`
* `NumericType`
  * `ByteType`, `ShortType`, `IntegerType`, `LongType`
  * `FloatType`, `DoubleType`
  * `DecimalType` - represents a `java.math.BigDecimal`
* `StringType`
* `DateType`, `TimestampType` - represent types in the  `java.sql` package
* `CalendarIntervalType` - Spark's custom date-time interval value
* `BinaryType` - represents a byte array
* `ArrayType` - represents a general array of elements
* `MapType`
* `StructType` - represents a structure with named fields (contains a `Row`)

Our schema definition will therefore need to look like this:

```scala
override def schema: StructType = StructType(Seq(
  StructField("application", StringType, false),
  StructField("dateTime", TimestampType, false),
  StructField("component", StringType, false),
  StructField("level", StringType, false),
  StructField("message", StringType, false)))
```

Note that we added the aforementioned *application* field to match the produced `Row`.

Putting it all together we get:

```scala
class LogRelation(val sqlContext: SQLContext, val path: String) extends BaseRelation with PrunedFilteredScan {
  override def schema: StructType = StructType(Seq(
    StructField("application", StringType, false),
    StructField("dateTime", TimestampType, false),
    StructField("component", StringType, false),
    StructField("level", StringType, false),
    StructField("message", StringType, false)))

  override def buildScan(requiredColumns: Array[String], filters: Array[Filter]): RDD[Row] = {
    val (application, from, to) = filters
      .foldLeft((Option.empty[String], Option.empty[Timestamp], Option.empty[Timestamp])) {
        case (x, EqualTo("application", value: String)) => x.copy(_1 = Some(value))
        case (x, GreaterThan("dateTime", value: Timestamp)) => x.copy(_2 = Some(value))
        case (x, GreaterThanOrEqual("dateTime", value: Timestamp)) => x.copy(_2 = Some(value))
        case (x, LessThan("dateTime", value: Timestamp)) => x.copy(_3 = Some(value))
        case (x, LessThanOrEqual("dateTime", value: Timestamp)) => x.copy(_3 = Some(value))
        case (x, _) => x
      }
    if (application.isEmpty) throw LogDataSourceException("Application not specified")
    if (from.isEmpty) throw LogDataSourceException("Timestamp lower bound not specified")
    val (fromLimit, toLimit) = (
      from.get.toLocalDateTime.toLocalDate,
      to.map(_.toLocalDateTime.toLocalDate).getOrElse(LocalDate.now)
    )
    val pathPattern =
      Iterator.iterate(fromLimit)(_.plusDays(1))
        .takeWhile(!_.isAfter(toLimit))
        .map(_.format(DateTimeFormatter.ISO_LOCAL_DATE))
        .mkString(s"$path/${application.get}/*/{", ",", "}.gz")

    sqlContext.sparkContext.textFile(pathPattern).map { line =>
      val record = new LogParser(line).Line.run().get
      Row(application.get, record.dateTime, record.component, record.level, record.message)
    }
  }
}
```

To try out our new relation, we can construct it explicitly in the Spark shell:

```
scala> new LogRelation(spark.sqlContext, "hdfs:///logs").buildScan(Array("dateTime", "application", "level", "message"), Array(EqualTo("application", "web"), GreaterThan("dateTime", Timestamp.valueOf("2017-02-08 22:55:00")), LessThan("dateTime", Timestamp.valueOf("2017-02-08 23:05:00")))).foreach(println)
[2017-02-08 22:55:20.0,com.github.michalsenkyr.example.web.RequestHandler,DEBUG,Received a request]
[2017-02-08 22:55:23.0,com.github.michalsenkyr.example.web.RequestHandler,DEBUG,Sent a response]
[2017-02-08 22:56:11.0,com.github.michalsenkyr.example.web.RequestHandler,DEBUG,Received a request]
[2017-02-08 23:01:11.0,com.guthub.michalsenkyr.example.web.RequestHandler,WARN,Request timed out]
```

Note: We can support insertions in a relation by mixing in the [`InsertableRelation` trait](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.sources.InsertableRelation) and implementing its `insert` method.

## Implementing and registering a new data source

In order to use our new relation, we need to tell Spark SQL how to create it. As different relations use different parameters, Spark SQL accepts these in the form of a `Map[String, String]` which is specified by the user using different methods on the [`DataFrameReader` object](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.DataFrameReader) obtained using `spark.read`. The special *format* parameter is used by the `DataFrameReader` to select the data source to be created if it has been registered. That's when [`DataSourceRegister`](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.sources.DataSourceRegister) comes into play.

Extending `DataSourceRegister` requires us to implement the `shortName` method which is used to identify the data source (used by the *format* parameter). We also need to mix in one or more of the `Provider` traits and implement their `createRelation` methods:

* [`RelationProvider`](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.sources.RelationProvider) - basic provider which receives the `SQLContext` and the user-defined parameters
* [`SchemaRelationProvider`](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.sources.SchemaRelationProvider) - additionally accepts a custom `schema` and is used for generic data sources
* [`CreatableRelationProvider`](https://spark.apache.org/docs/latest/api/scala/index.html#org.apache.spark.sql.sources.CreatableRelationProvider) - additionally accepts a `SaveMode` and a `DataFrame` containing the data to save

As our relation has a defined schema and we only need to load from it, we mix in the `RelationProvider` trait and write a very simple implementation:

```scala
class LogDataSource extends DataSourceRegister with RelationProvider {
  override def shortName(): String = "log"

  override def createRelation(sqlContext: SQLContext,
                              parameters: Map[String, String]): BaseRelation =
    new LogRelation(sqlContext, parameters("path"))
}
```

This data source then has to be registered. Spark SQL handles this using a resource file which is read from every jar loaded by the application's classloader. We will need to create it and specify our `DataSource` class name inside it:

*META-INF/services/org.apache.spark.sql.sources.DataSourceRegister*

```
com.github.michalsenkyr.example.datasource.LogDataSource
```

Now we can try out loading our data source using the Spark shell:

```
scala> spark.read.format("log").load("hdfs:///logs").where('dateTime >= Timestamp.valueOf("2017-02-08 22:55:00") && 'dateTime < Timestamp.valueOf("2017-02-08 23:05:00") && 'application === "web").show()
+--------------------+--------------------+-----+------------------+
|            dateTime|         application|level|           message|
+--------------------+--------------------+-----+------------------+
|2017-02-08 22:55:...|com.github.michal...|DEBUG|Received a request|
|2017-02-08 22:55:...|com.github.michal...|DEBUG|   Sent a response|
|2017-02-08 22:56:...|com.github.michal...|DEBUG|Received a request|
|2017-02-08 23:01:...|com.guthub.michal...| WARN| Request timed out|
+--------------------+--------------------+-----+------------------+
```

We can also define an extension method on the `DataFrameReader` class to make this even easier:

```scala
implicit class LogDataFrameReader(val reader: DataFrameReader) extends AnyVal {
  def log(path: String) = reader.format("log").load(path)
}
```

Now as long as `LogDataFrameReader` is within scope, we can use the `log` method to load our data:

```
scala> import com.github.michalsenkyr.example.datasource.LogDataFrameReader
import com.github.michalsenkyr.example.datasource.LogDataFrameReader

scala> spark.read.log("hdfs:///logs").where('dateTime >= Timestamp.valueOf("2017-02-08 22:55:00") && 'dateTime < Timestamp.valueOf("2017-02-08 23:05:00") && 'application === "web").show()
+--------------------+--------------------+-----+------------------+
|            dateTime|         application|level|           message|
+--------------------+--------------------+-----+------------------+
|2017-02-08 22:55:...|com.github.michal...|DEBUG|Received a request|
|2017-02-08 22:55:...|com.github.michal...|DEBUG|   Sent a response|
|2017-02-08 22:56:...|com.github.michal...|DEBUG|Received a request|
|2017-02-08 23:01:...|com.guthub.michal...| WARN| Request timed out|
+--------------------+--------------------+-----+------------------+
```

## Practical usage

Up to this point we were working within the scope of a single project. However, in practice, we should separate our data source functionality into its own library and distribute it with our application, which we submit to the cluster using the *spark-submit* script.

One way of achieving this is to simply include the library in our application's uberjar by specifying it as a compile dependency. This works with small or rarely reused libraries (we actually already did that with the Parboiled2 library).

Another way is to package it separately, making it a provided dependency (like we did with *spark-sql*) and sending it alongside our application:

```bash
spark-shell --jars example-library.jar,example-application.jar
```

or

```bash
spark-submit --jars example-jibrary.jar --class com.github.michalsenkyr.example.MainClass example-application.jar
```

## Conclusion

Spark's Dataframe and DataSet models were a great innovation in terms of performance but brought with them additional layers of (fully justified) complexity. Spark's developers did their best to abstract from the internals and provide developers with the simplest way possible to extend the framework's capabilities regarding the support of various data sources using the configurable relation and data source register API. Thanks to this, lots of developers were able to contribute [their own implementations](https://spark-packages.org/?q=tags%3A%22Data%20Sources%22).

Parboiled2 provides a powerful and easy-to-use parser DSL leveraging Scala's macro system. It takes some getting used to because of the stack-based mechanics and a syntax that differs quite a bit from the often used regular expressions but it doesn't add too much in terms of verbosity and enables the framework to create extremely efficient parsing solutions suitable for highly-performant Spark applications.
