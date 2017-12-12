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

Our input can already be skewed when reading from the data source. In the RDD API this is often done using the `textFile` and `wholeTextFiles` methods, which have surprisingly different partitioning behaviors. The `textFile` method, which is designed to read individual lines of text from (usually larger) files, loads each input file block as a separate partition by default. It also provides a `minPartitions` parameter that, when greater than the number of blocks, tries to split these partitions further in order to satisfy the specified value. On the other hand, the `wholeTextFiles` method, which is used to read the whole contents of (usually smaller) files, combines the relevant files' blocks into pools by their actual locality inside the cluster and, by default, creates a partition for each of these pools. The `minPartitions` parameter in this case controls the maximum size of these pools (equalling `totalSize/minPartitions`). The default value for all `minPartitions` parameters is 2. This means that it is much easier to get a very low number of partitions with `wholeTextFiles` if using default settings while not managing data locality explicitly on the cluster. Other methods used to read data into RDDs include `sequenceFile` (TODO: Add partitioning info) and `hadoopRDD`/`newAPIHadoopRDD` where we can specify custom partitioning (TODO: Verify and add further info).

Partitioning characteristics frequently change on shuffle boundaries. Operations that imply a shuffle therefore provide a `numPartitions` parameter that specify the new partition count (by default the partition count stays the same as in the original RDD). Skew can also be introduced via shuffles, especially when joining datasets.

![RDD join skew example]

As the partitioning in these cases depends entirely on the selected key (specifically its Murmur3 hash), care has to be taken to avoid unusually large partitions being created for common keys (e.g. null keys are a common special case). An efficient solution is to separate the relevant records, introduce a salt (random value) to their keys and perform the subsequent action (e.g. reduce) for them in multiple stages to get the correct result.

![RDD join salting example]

Sometimes there are even better solutions, like using map-side joins if one of the datasets is small enough.

![RDD map-side join example]

### DataFrames and Datasets

The high-level APIs share a special approach to partitioning data. All data blocks of the input files are added into a common pool and this pool is divided into partitions according to 

## 3. Serialization

## 4. Memory management

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

