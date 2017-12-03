---
title: Optimizing Spark jobs for maximum performance
date: 2017-12-03 18:09
layout: post
category: spark
tags: scala spark performance
---

* content
{:toc}

## 1. Transformations (RDD)

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

