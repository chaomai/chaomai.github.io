title: "Notes of CS100.1x Introduction to Big Data with Apache Spark (1)"
date: 2015-06-26 17:15:43
description: Notes of Lecture 3 Big Data, Hardware Trends, and Apache Spark.
categories:
    - spark
tags:
    - spark
    - pyspark
    - edx
---

## Lecture 3: Big Data, Hardware Trends, and Apache Spark and Lecture 4: Spark Essentials

### The Big Data Problem

* Growing data sources
* Storage getting cheapper
* But stalling CPU and storage bottlenecks

### Hardware for Big Data

Problems with cheap hardware

* Failures
* Network
* Uneven performance

### What's Hard About Cluster Computing

* Divide work across machines
  * Must consider network, data locality
  * Moving data may be veay expensive
* Deal with failures

## Spark Essentials

### PySpark

A Spark program consists of two programs, a driver program
and a workers program.

* Drivers program: runs on the driver machine.
* Worker programs: run on cluster nodes
or in local threads.

RDDs are distributed across the workers.

### RDD

An RDD is immutable, so once it is created, it cannot be changed.

types of operations:

* transformations

  * lazily evaluated.
  * A transformed RDD is executed only when an action runs on it.
  * can also persist, or cache RDDs in memory or on disk.
?
* actions

  * cause Spark to execute the recipe to transform the source data.

### Spark Programming Model

```python
lines = sc.textFile("...", 4)
comments = lines.filter(isComment)
print lines.count(), comments.count()
```

`comments.count()` is going to cause Spark to re-compute lines. reread all of the data from that text file again, sum within the partition the number of lines, so the number of elements, and then combine those sums in the driver.

### Caching RDDS

```python
lines = sc.textFile("...", 4)
lines.cache()
comments = lines.filter(isComment)
print lines.count(), comments.count()
```

create the comments RDD directly, instead of reading from disk.

### Spark Program Lifecycle

1. create RDDs from some external data source or parallelize a collection in your driver program.
2. lazily transform these RDDs into new RDDs.
3. cache some of those RDDs for future reuse.
4. perform actions to execute parallel computation and to produce results.

### Spark Broadcast Variables

```python
broadcast_var = sc.broadcast([1, 2, 3])
...
broadcast_var.value
```

Keep a read-only variable cached at a worker and will be reused every time we need to access it instead of constructing another closure.

### Spark Accumulators

could only be added to by an associative operation. They're used to very efficiently implement parallel counters and sums, and only the driver can read an accumulator's values, not the tasks.

can be used in actions or transformations:

* actions: each tasks update to the accumulator is guaranteed by spark to **only be applied once**.
* transformations: no guarantee.

support the types:

* integers
* double
* long
* float
* custom types

## About pySpark

### Spark Context

When running Spark, you start a new Spark application by creating a SparkContext. When the SparkContext is created, it asks the master for some cores to use to do work. The master sets these cores aside just for you; they **won't be used for other applications**.

Driver programs access Spark through a SparkContext object, which represents **a connection to a computing cluster**. A Spark context object (sc) is the main entry point for Spark functionality. A Spark context can be used to create Resilient Distributed Datasets (RDDs) on a cluster.

![](http://spark-mooc.github.io/web-assets/images/executors.png)

### Resilient Distributed Datasets (RDDs)

![](http://spark-mooc.github.io/web-assets/images/partitions.png)

### `map()`

When you run `map()` on a dataset, a **single stage of tasks** is launched. A stage is *a group of tasks that all perform the same computation, but on different input data*. **One task is launched for each partition**. A task is *a unit of execution that runs on a single machine*. When we run `map(f)` within a partition, a new task applies f to all of the entries in a particular partition, and outputs a new partition.

![](http://spark-mooc.github.io/web-assets/images/map.png)

When applying the `map()` transformation, each item in the parent RDD will map to one element in the new RDD.

### `collect()`

the data returned to the driver **must fit into the driver's available memory**. If not, the driver will crash.

### `first()` and `take()`

`first()` and `take()` actions, the elements that are returned depend on how the RDD is partitioned.

### `takeOrdered()`

The key advantage of using `takeOrdered()` instead of `first()` or `take()` is that `takeOrdered()` returns a **deterministic result**, while the other two actions may return different results, *depending on the number of partitions or execution environment*.

`takeOrdered()` returns the list sorted in **ascending order**. The `top()` action is similar to `takeOrdered()` except that it returns the list in **descending order**.

### `reduce()`

reduces the elements of a RDD to a single value by applying a function that takes two parameters and returns a single value.

The function should be **commutative and associative（可交换和可结合）**, as `reduce()` is applied at the partition level and then again to aggregate results from partitions.

### `takeSample()` and `countByValue()`

The `takeSample()` action returns an array with a random sample of elements from the dataset. It takes in a `withReplacement` argument, which specifies whether it is okay to randomly pick the same item multiple times from the parent RDD. It also takes an optional `seed` parameter that allows you to specify a seed value for the random number generator, so that reproducible results can be obtained.

The `countByValue()` action returns **the count of each unique value** in the RDD as a dictionary that maps values to counts.

### `flatMap()`

```python
simpleRDD = sc.parallelize([[[9,9,9],2,3], [[9,9,9],3,4], [[9,9,9],5,6]])
print simpleRDD.map(lambda x:x).collect()
print simpleRDD.flatMap(lambda x:x).collect()

# output
[[[9, 9, 9], 2, 3], [[9, 9, 9], 3, 4], [[9, 9, 9], 5, 6]]
[[9, 9, 9], 2, 3, [9, 9, 9], 3, 4, [9, 9, 9], 5, 6]
```

### `groupByKey()` and `reduceByKey()`

Both of these transformations operate on *pair RDDs*. A pair RDD is an RDD where *each element is a pair tuple (key, value)*.

![](http://spark-mooc.github.io/web-assets/images/reduce_by.png)

![](http://spark-mooc.github.io/web-assets/images/group_by.png)

`reduceByKey()` operates by applying the function first within each partition on a per-key basis and then across the partitions.

* the `reduceByKey()` transformation works much better for large distributed datasets. This is because Spark knows it can *combine output with a common key on each partition before shuffling* (redistributing) the data across nodes. Only use `groupByKey()` if the operation would not benefit from reducing the data before the shuffle occurs.
* On the other hand, when using the `groupByKey()` transformation - all the key-value pairs are shuffled around, causing a lot of unnecessary data to being transferred over the network.

### `cache()` and `unpersist()`

if you cache too many RDDs and Spark runs out of memory, it will delete the least recently used (LRU) RDD first. The RDD will be automatically recreated when accessed.

tell Spark to stop caching it in memory by using the RDD's `unpersist()` method.
