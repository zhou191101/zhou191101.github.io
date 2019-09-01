---
layout:     post
title:      "Spark RDD"
subtitle:   "理解Spark RDD"
date:       2019-09-01
author:     "zhoup"
header-img: "img/post-bg-unix-linux.jpg"
tags:
    - 大数据
    - Spark
---

> This document is not completed and will be updated anytime.

# What

## 什么是Spark RDD？

RDD(Resilient Distributed Dataset),弹性分布式数据集。

RDD是Spark最基础的抽象，它本质上是一个不可变的记录分区的集合，RDD只能通过在：1、稳定的存储器或 2、其他的RDD的数据上的确定性操作来创建。RDD提供了丰富的操作类型，比如map、filter、join等。

用户可以控制RDD的其他方面：持久化和分区。用户可以选择重用哪个RDD，并为其定制存储策略（比如磁盘或者内存）。也可以让RDD中的数据根据记录的key分布到集群到多个机器。

RDD的五个属性：

- A list of partitions
- A function for computing each split
- A list of dependencies on other RDDs
- Optionally, a Partitioner for key-value RDDs (e.g. to say that the RDD is hash-partitioned)
- Optionally, a list of preferred locations to compute each split on (e.g. block locations for an HDFS file)

## A list of partitions（一组分区）

RDD逻辑上是分区的，每个分区的数据是抽象存在的。
分区是数据集的最小分片。

## A function for computing each split（一个函数）

RDD计算的时候会通过一个compute函数得到每个分区的数据。如果RDD是通过已有的文件系统构建，则compute函数是读取指定文件系统中的数据，如果RDD是通过其他RDD转换而来，则compute函数是执行转换逻辑将其他RDD的数据进行转换。

```
  /**
   * :: DeveloperApi ::
   * Implemented by subclasses to compute a given partition.
   */
  @DeveloperApi
  def compute(split: Partition, context: TaskContext): Iterator[T]
```

RDD是Spark的一个抽象类，实现自定义的RDD必须实现compute方法。

## A list of dependencies on other RDDs（一组依赖关系）

RDD与RDD是相互依赖的，RDD通过算子操作进行转换，转换得到新的RDD，转换得到的新RDD包含了从其他RDD衍生的所必需的信息。RDD之间维护着这种血缘关系，也称为依赖。

## Optionally, a Partitioner for key-value RDDs（一个分区器）

对于健值对的RDD，key就是RDD的分区器

## Optionally, a list of preferred locations to compute each split（一个列表，存储存取每个partition的preferred位置） 

一个列表，存储存取每个partition的preferred位置。对于一个HDFS文件来说，存储每个partition所在的块的位置。

# RDD的优点

- 可以使用lineage恢复数据，RDD不需要检查点的开销。此外，当出现失败时，RDDs的分区中只有丢失的那部分需要重新计算，而且该计算可在多个节点上并发完成，不必回滚整个程序
- 不可变性让系统像 MapReduce 那样用后备任务代替运行缓慢的任务来减少缓慢节点 (stragglers) 的影响
- 在RDDs上的批量操作过程中，任务的执行可以根据数据的所处的位置来进行优化，从而提高性能；只要所进行的操作是只基于扫描的，当内存不足时，RDD的性能下降也是平稳的。不能载入内存的分区可以存储在磁盘上，其性能也会与当前其他数据并行系统相当。

# RDD Operator

RDD支持两种类型的操作：transformation和action。

transformation:Spark的所有transformation操作都是lazy的。只有当action操作触发时，才会计算transformation的内容。
action：触发Spark计算任务的操作。

## Transformation

- map(func)
- filter(func)
- flatMap(func)
- mapPartitions(func)
- mapPartitionsWithIndex(func)
- sample(withReplacement, fraction, seed)
- union(otherDataset)
- intersection(otherDataset)
- distinct([numPartitions]))
- groupByKey([numPartitions])
- reduceByKey(func, [numPartitions])
- aggregateByKey(zeroValue)(seqOp, combOp, [numPartitions])
- sortByKey([ascending], [numPartitions])
- join(otherDataset, [numPartitions])
- cogroup(otherDataset, [numPartitions])
- cartesian(otherDataset)
- pipe(command, [envVars])
- coalesce(numPartitions)
- repartition(numPartitions)
- repartitionAndSortWithinPartitions(partitioner)

## Actions

- reduce(func)
- collect()
- count()
- first()
- take(n)
- takeSample(withReplacement, num, [seed])
- takeOrdered(n, [ordering])
- saveAsTextFile(path)
- saveAsSequenceFile(path) 
- saveAsObjectFile(path) 
- countByKey()
- foreach(func)

## 理解闭包

什么叫闭包：跨作用域访问函数变量。又指一个拥有许多变量和绑定了这些变量的函数表达式（通常是一个函数），因而这些变量也是该表达式的一部分。

在Spark中，如下示例：

```
scala> val rdd = sc.parallelize(List(1,2,3))
scala> var counter =0
scala> rdd.foreach(x=>counter+x)
scala> print(counter)
0
```

最终结果显示为0，与我们预期的不一样。

对于以上变量counter，由于在main函数和在rdd对象的foreach函数是属于不同的闭包，所以，传进foreach的counter是一个副本，初始值为0。在foreach函数中是副本的叠加，不管副本如何变化，都不会影响到main函数中的counter，因此，最终结果为0。
为了确保这种场景的成功，在Spark中应该使用累加器（Accumulator）。