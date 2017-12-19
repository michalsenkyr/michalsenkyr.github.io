---
title: Optimizing Spark jobs for maximum performance
date: 2017-12-03 18:09
layout: post
category: spark
tags: scala spark performance
---

* content
{:toc}

Development of Spark jobs seems easy enough on the surface and for the most part it really is. The provided APIs are pretty well designed and feature-rich and if you are familiar with Scala collections or Java streams, you will be done with your implementation in no time. The hard part actually comes when running them on cluster and under full load as not all jobs are created equal in terms of performance. Unfortunately, to implement your jobs in an optimal way, you have to know quite a bit about Spark and its internals.

In this article I will talk about the most common performance problems that you can run into when developing Spark applications and how to avoid or mitigate them.


## 1. Transformations

The most frequent performance problem, when working with the RDD API, is using transformations which are inadequate for the specific use case. I think this usually stems from the users' familiarity with SQL querying languages and its reliance on query optimizations. It is important to realize that the RDD API doesn't apply any such optimizations.

Let's take a look at these two definitions of the same computation:

![RDD groupByKey vs reduceByKey comparison]

The second definition is much faster than the first because it handles data more efficiently in the context of our use case by not collecting all the elements needlessly.

We can observe a similar performance issue when making cartesian joins and later filtering on the resulting data instead of converting to a pair RDD and using an inner join:

![RDD cartesian vs inner join comparison]

The rule of thumb here is to always work with the minimal amount of data at transformation boundaries. The RDD API does its best to optimize background stuff like task scheduling, preferred locations based on data locality, etc. But it does not optimize the computations themselves. It is, in fact, literally impossible for it to do that as each transformation is defined by an opaque function and Spark has no way to see what data we're working with and how.

There is another rule of thumb that can be derived from this: have rich transformations, ie. always do as much as possible in the context of a single transformation. A useful tool for that is the combineByKey transformation:

![RDD combineByKey example]

### DataFrames and Datasets

The Spark community actually recognized these problems and developed two sets of high-level APIs to combat this issue: DataFrame and Dataset. These APIs carry with them additional information about the data and define specific transformations that are recognized throughout the whole framework. When invoking an action, the computation graph is heavily optimized and converted into a corresponding RDD graph, which is executed.

To demonstrate, we can try out two equivalent computations, defined in a very different way, and compare their run times and job graphs:

![DataFrame optimization example]

As we can see, the order of transformations does not matter, which is thanks to a feature called rule-based query optimization. Data sizes are also taken into account to reorder the job in the right way, thanks to cost-based query optimization. Lastly, the DataFrame API also pushes information about the columns that are actually required by the job to limit input reads (this is called predicate pushdown). It is actually very difficult to write an RDD job in such a way as to be on par with what the DataFrame API comes up with.

However, there is one aspect in which DataFrames do not excel and which prompted the creation of another, third, way to represent Spark computations: type safety. As data columns are represented only by name for the purposes of transformation definitions and their valid usage with regards to the actual data types is only checked during run-time, this tends to result in a tedious development process where we need to keep track of all the proper types or we end up with an error. The Dataset API was created as a solution to this.

The Dataset API uses Scala's type inference and implicits-based techniques to pass around Encoders, special classes that describe the data types for Spark's optimizer just as in the case of DataFrames, while retaining compile-time typing in order to do type checking and write transformations naturally. If that sounds complicated, here is an example:

![Dataset example]

Later it was realized that DataFrames can be thought of as just a special case of these Datasets and the API was unified (using a special optimized class called Row as the DataFrame's data type).

However, there is one caveat to keep in mind when it comes to Datasets. As developers became comfortable with the collection-like RDD API, the Dataset API provided its own variant of its most popular methods - filter, map and reduce. These work (as would be expected) with arbitrary functions. As such, Spark cannot understand the details of such functions and its ability to optimize becomes somewhat impaired as it can no longer correctly propagate certain information (e.g. for predicate pushdown). This will be explained further in the section on serialization.

![Dataset map inefficiency example]

## 2. Partitioning

The number two problem that most Spark jobs suffer from, is inadequate partitioning of data. In order for our computations to be efficient, it is important to divide our data into a large enough number of partitions that are as close in size to one another (uniform) as possible, so that Spark can schedule the individual tasks that are operating on them in an agnostic manner and still perform predictably. If the partitions are not uniform, we say that the partitioning is skewed. This can happen for a number of reasons and in different parts of our computation.

![Partitioning skew example]

Our input can already be skewed when reading from the data source. In the RDD API this is often done using the `textFile` and `wholeTextFiles` methods, which have surprisingly different partitioning behaviors. The `textFile` method, which is designed to read individual lines of text from (usually larger) files, loads each input file block as a separate partition by default. It also provides a `minPartitions` parameter that, when greater than the number of blocks, tries to split these partitions further in order to satisfy the specified value. On the other hand, the `wholeTextFiles` method, which is used to read the whole contents of (usually smaller) files, combines the relevant files' blocks into pools by their actual locality inside the cluster and, by default, creates a partition for each of these pools (for more information see Hadoop's [CombineFileInputFormat](http://hadoop.apache.org/docs/current/api/org/apache/hadoop/mapreduce/lib/input/CombineFileInputFormat.html) which is used in its implementation). The `minPartitions` parameter in this case controls the maximum size of these pools (equalling `totalSize/minPartitions`). The default value for all `minPartitions` parameters is 2. This means that it is much easier to get a very low number of partitions with `wholeTextFiles` if using default settings while not managing data locality explicitly on the cluster. Other methods used to read data into RDDs include other formats such as `sequenceFile`, `binaryFiles` and `binaryRecords`, as well as generic methods `hadoopRDD` and `newAPIHadoopRDD` which take custom format implementations (allowing for custom partitioning).

Partitioning characteristics frequently change on shuffle boundaries. Operations that imply a shuffle therefore provide a `numPartitions` parameter that specify the new partition count (by default the partition count stays the same as in the original RDD). Skew can also be introduced via shuffles, especially when joining datasets.

![RDD join skew example]

As the partitioning in these cases depends entirely on the selected key (specifically its Murmur3 hash), care has to be taken to avoid unusually large partitions being created for common keys (e.g. null keys are a common special case). An efficient solution is to separate the relevant records, introduce a salt (random value) to their keys and perform the subsequent action (e.g. reduce) for them in multiple stages to get the correct result.

![RDD join salting example]

Sometimes there are even better solutions, like using map-side joins if one of the datasets is small enough.

![RDD map-side join example]

### DataFrames and Datasets

The high-level APIs share a special approach to partitioning data. All data blocks of the input files are added into common pools, just as in `wholeTextFiles`, but the pools are then divided into partitions according to two settings: `spark.sql.files.maxPartitionBytes`, which specifies a maximum partition size (128MB by default), and `spark.sql.files.openCostInBytes`, which specifies an estimated cost of opening a new file in bytes that could have been read (4MB by default). The framework will figure out the optimal partitioning of input data automatically based on this information.

When it comes to partitioning on shuffles, the high-level APIs are, sadly, quite lacking (at least as of Spark 2.2). The number of partitions can only be specified statically on a job level by specifying the `spark.sql.shuffle.partitions` setting (200 by default).

The high-level APIs can automatically convert join operations into broadcast joins. This is controlled by `spark.sql.autoBroadcastJoinThreshold`, which specifies the maximum size of tables considered for broadcasting (10MB by default) and `spark.sql.broadcastTimeout`, which controls how long executors will wait for broadcasted tables (5 minutes by default).

### Repartitioning

All of the APIs also provide two methods to manipulate the number of partitions. The first one is `repartition` which forces a shuffle in order to redistribute the data among the specified number of partitions (by the aforementioned Murmur hash). As shuffling data is a costly operation, repartitioning should be avoided if possible. There are also more specific variants of this operation: RDDs have `repartitionAndSortWithinPartitions` which can be used with a custom partitioner, whereas DataFrames and Datasets have `repartition` with column parameters to control the partitioning characteristics.

The second method provided by all APIs is `coalesce` which is much more performant than `repartition` because it does not shuffle data but only instructs Spark to read several existing partitions as one. This, however, can only be used to decrease the number of partitions and cannot be used to change partitioning characteristics. There is usually no reason to use it, as Spark is designed to take advantage of larger numbers of small partitions, other than reducing the number of files on output or the number of batches when used together with `foreachPartition` (e.g. to send results to a database).

## 3. Serialization

Another thing that is tricky to take care of correctly is serialization, which comes in two varieties: data serialization and closure serialization. Data serialization refers to the process of encoding the actual data that is being stored in an RDD whereas closure serialization refers to the same process but for the data that is being introduced to the computation externally (like a shared field or variable). It is important to distinguish these two as they work very differently in Spark.

### Data serialization

Spark supports two different serializers for data serialization. The default one is Java serialization which, although it is very easy to use (by simply implementing the `Serializable` interface), is very inefficient. That is why it is advisable to switch to the second supported serializer, [Kryo](https://github.com/EsotericSoftware/kryo), for the majority of production uses. This is done by setting `spark.serializer` to `org.apache.spark.serializer.KryoSerializer`. Kryo is much more efficient and does not require the classes to implement `Serializable` (as they are serialized by Kryo's [FieldSerializer](https://github.com/EsotericSoftware/kryo#fieldserializer) by default). However, in very rare cases, Kryo can fail to serialize some classes, which is the sole reason why it is still not Spark's default. It is also a good idea to register all classes that are expected to be serialized (Kryo will then be able to use indices instead of full class names to identify data types, reducing the size of the serialized data thereby increasing performance even further).

![Kryo vs Java serialization benchmark]

### DataFrames and Datasets

The high-level APIs are much more efficient when it comes to data serialization as they are aware of the actual data types they are working with. Thanks to this, they can generate optimized serialization code tailored specifically to these types and to the way Spark will be using them in the context of the whole computation. For some transformations it might also generate only partial serialization code (e.g. array lookups). This code generation step is called Project Tungsten and is a big part of what makes the high-level APIs so performant.

![Project Tungsten serialization benchmark]

### Closure serialization

In most Spark applications, there is not only the data itself that needs to be serialized. There are also external fields and variables that are used in the individual transformations. Let's consider the following code snippet:

```scala
val factor = config.multiplicationFactor
rdd.map(_ * factor)
```

Here we use a value that is loaded from the application's configuration as part of the computation itself. However, as everything that happens outside the transformation function itself happens on the driver, Spark has to transport the value to the relevant executors. Spark therefore computes what's called a closure to the function in `map` comprising of all external values that it uses, serializes those values and sends them over the network. As closures can be quite complex, a decision was made to only support Java serialization there. Serialization of closures is therefore less efficient than serialization of the data itself, however as closures are only serialized for each executor on each transformation and not for each record, this usually does not cause performance issues. (There is, however, an unpleasant side-effect of needing these values to implement `Serializable`.)

Variables in closures are pretty simple to keep track of. Where there can be quite a bit deal of confusion is using fields. Let's look at the following example:

```scala
class SomeClass(d: Int) extends Serializable {
  val c = 1
  val e = new SomeComplexClass

  def closure(rdd: RDD[Int], b: Int): RDD[Int] = {
    val a = 0
    rdd.map(_ + a + b + c + d)
  }
}
```

Here we can see that `a` is just a variable (just as `factor` before) and is therefore serialized as an `Int`. `b` is a method parameter (which behaves as a variable too), so that also gets serialized as an `Int`. However `c` is a class field and as such cannot be serialized separately. That means that in order to serialize it, Spark needs to serialize the whole instance of `SomeClass` with it (so it has to extend `Serializable`, otherwise we would get a run-time exception). The same is true for `d` as constructor parameters are converted into fields internally. Therefore, in both cases Spark would also have to send the values of `c`, `d` and `e` to the executors. As `e` might be quite costly to serialize, this is definitely not a good solution. We can solve this by avoiding class fields in closures:

```scala
class SomeClass(d: Int) {
  val c = 1
  val e = new SomeComplexClass

  def closure(rdd: RDD[Int], b: Int): RDD[Int] = {
    val a = 0
    val sum = a + b + c + d
    rdd.map(_ + sum)
  }
}
```

Here we prepare the value by storing it in a local variable `sum`. This then gets serialized as a simple `Int` and doesn't drag the whole instance of `SomeClass` with it (so it does not have to extend `Serializable` anymore).

Spark also defines a special construct to improve performance in cases where we need to serialize the same value for multiple transformations. It is called a broadcast variable and is serialized and sent only once, before the computation, to all executors. This is especially useful for large variables like lookup tables.

(TODO: Broadcast example)

Spark provides a useful tool to determine the actual size of objects in memory called [SizeEstimator](https://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/util/SizeEstimator.html) which can help us to decide whether a particular object is a good candidate for a broadcast variable.

(TODO: Add info on accumulators?)

## 4. Memory management

It is important for the application to use its memory space in an efficient manner. As each application's memory requirements are different, Spark divides the memory of an application's driver and executors into multiple parts that are governed by appropriate rules and leaves their size specification to the user via application settings.

## 5. Cluster resources



## Serialization

* Closures
* Kryo
* Broadcast variables
* Accumulators (imprecise)
* Preferred locations (shuffle files, cache)

## RDD operations

* Efficient aggregation (groupByKey vs. reduceByKey, cartesian joins, combineByKey)
* Partitioning (skew) - textFile, wholeTextFiles, DF partitioning
* Tree aggregations

## DataFrames and Datasets

* Catalyst optimizer (rule, cost, source filters)
* Project Tungsten (mem. mgmt, cache-aware, whole-stage codegen)
* Component variants

## Memory usage

* Executor memory (storage vs. execution fraction)
* No swapping, but spilling (spark.local.dir=/tmp)
* Large working set on shuffle -> more tasks (>200ms)
* Estimations (Storage tab, org.apache.spark.util.SizeEstimator.estimate)
* Persistence levels (MEMORY_ONLY, MEMORY_AND_DISK, DISK_ONLY, *_SER, *_2, OFF_HEAP)

## Dynamic allocation

* External shuffle service
* Exponential requests & idle removal

## Speculative execution

