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

The most frequent performance problem, when working with the RDD API, is using transformations which are inadequate for the specific use case. I think this usually stems from the users' familiarity with SQL querying languages and its reliance on query optimizations. It is important to realize that the RDD API doesn't do any such optimizations.

Let's take a look at these two definitions of the same computation:

![RDD groupByKey vs reduceByKey comparison]

The second definition is much faster than the first because it handles data more efficiently in the context of our use case.

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

