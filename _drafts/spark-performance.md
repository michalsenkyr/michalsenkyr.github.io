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

The Spark community actually recognized these problems and developed two sets of high-level APIs to combat this issue: DataFrame and Dataset.

## 2. Partitioning

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

