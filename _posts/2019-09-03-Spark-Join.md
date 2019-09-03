---

layout:     post
title:      "深入理解Spark Join"
subtitle:   ""
date:       2019-09-03
author:     "zhoup"
header-img: "img/post-bg-unix-linux.jpg"
tags:
    - 大数据
    - Spark
---

> This document is not completed and will be updated anytime.

开发Spark程序的时候，join是最常见的操作之一。熟悉Spark的都知道，Join操作也是影响spark性能的因素之一，因此，如何使用和如何用好Join是spark调优的一大关键因素。

Spark提供了三种类型的join：

- Sort-Merge Join
- BroadCast Join
- Shuffle Hash Join

## Sort-Merge Join

Sort-Merge Join 包含2步操作。第一步是将数据排序，第二个操作是通过迭代元素来合并分区中的排序后的数据，并根据join的key连接具有相同值的行。

从Spark 2.3开始，Spark就将Sort-Merge Join作为默认的join算法。但是，可以通过设置参数：**spark.sql.join.preferSortMergeJoin=false**来关闭它。如下所示：

```scala
scala>       spark.sql(
     |         s"""
     |            |select  t1.device, t1.refine_final_flag, t2.feature_field, t2.pkg
     |            |from tmp_t1 t1
     |            |inner join tmp_t2 t2
     |            |on t1.pkg = t2.pkg
     |          """.stripMargin).explain()
== Physical Plan ==
*Project [device#1, refine_final_flag#3, feature_field#12, pkg#15]
+- *SortMergeJoin [pkg#2], [pkg#15], Inner
   :- *Sort [pkg#2 ASC NULLS FIRST], false, 0
   :  +- Exchange hashpartitioning(pkg#2, 200)
   :     +- *Filter isnotnull(pkg#2)
   :        +- HiveTableScan [device#1, refine_final_flag#3, pkg#2], MetastoreRelation dm_mobdi_master, master_reserved_new, [isnotnull(day#0), (day#0 >= 20190823), (day#0 <= 20190825)]
   +- *Sort [pkg#15 ASC NULLS FIRST], false, 0
      +- Exchange hashpartitioning(pkg#15, 200)
         +- Union
            :- *HashAggregate(keys=[pkg#15, cate_l1_id#20], functions=[])
            :  +- Exchange hashpartitioning(pkg#15, cate_l1_id#20, 200)
            :     +- *HashAggregate(keys=[pkg#15, cate_l1_id#20], functions=[])
            :        +- *Filter isnotnull(pkg#15)
            :           +- HiveTableScan [pkg#15, cate_l1_id#20], MetastoreRelation dm_sdk_mapping, app_category_mapping_par, [isnotnull(version#14), (version#14 = 1000)]
            +- *HashAggregate(keys=[pkg#23, cate_l2_id#29], functions=[])
               +- Exchange hashpartitioning(pkg#23, cate_l2_id#29, 200)
                  +- *HashAggregate(keys=[pkg#23, cate_l2_id#29], functions=[])
                     +- *Filter isnotnull(pkg#23)
                        +- HiveTableScan [pkg#23, cate_l2_id#29], MetastoreRelation dm_sdk_mapping, app_category_mapping_par, [isnotnull(version#22), (version#22 = 1000)]


```

Sort-Merge Join适用于join的时候左右两表都比较大的情况，这种情况下，通过join前的排序，使相关联的两个分区的数据都为有序状态，在进行键的相等性比较的时候速度就会很快。

Sort-Merge Join可表示为如下状态：

![sort-merge-shuffle.png](https://github.com/zhou191101/draw/blob/master/spark/sort-merge-shuffle.png?raw=true)

## BroadCast Join

Spark的shuffle操作是极其影响性能的，因此，如果能避免shuffle，对spark性能的提升会有极其显著的效果。BroadCast Join就是由此而诞生的。它的原理是：将join右边的表广播到每个executor节点去，因此，join的时候就不会产生shuffle，但是，如果join右边的表过大，那么势必会造成driver和executor的内存负担，甚至产生oom。

Broadcast Join是由参数`spark.sql.autoBroadcastJoinThreshold`来控制，默认为10485760 (10 MB)。如果超过该阈值，那么Spark就不会使用broadcast join，如果该参数设置为-1，则代表不使用broadcast join。注意：broadcast的表需要是Hive原生的表或者通过cache缓存在内存中的表。

Spark将broadcast的表的最大的大小从2GB提升到8GB，因此，广播超过8GB的表是不可能的。

操作流程如下图所示：

![spark-broadcast-join.png](https://github.com/zhou191101/draw/blob/master/spark/spark-broadcast-join.png?raw=true)

例如上面的例子：

将tmp_t2缓存在内存中，发现表大小为：2.8M

![cache-table.png](https://github.com/zhou191101/draw/blob/master/spark/cache-table.png?raw=true)

少于默认的10M，但是查看Spark的执行计划还是使用的Sort-Merge Join。

通过以下操作：

```scala
scala> spark.sql("cache table tmp_t2") 

scala>       spark.sql(
     |         s"""
     |            |select  t1.device, t1.refine_final_flag, t2.feature_field, t2.pkg
     |            |from tmp_t1 t1
     |            |inner join tmp_t2 t2
     |            |on t1.pkg = t2.pkg
     |          """.stripMargin).explain()
== Physical Plan ==
*Project [device#1, refine_final_flag#3, feature_field#12, pkg#15]
+- *BroadcastHashJoin [pkg#2], [pkg#15], Inner, BuildRight
   :- *Filter isnotnull(pkg#2)
   :  +- HiveTableScan [device#1, refine_final_flag#3, pkg#2], MetastoreRelation dm_mobdi_master, master_reserved_new, [isnotnull(day#0), (day#0 >= 20190823), (day#0 <= 20190825)]
   +- BroadcastExchange HashedRelationBroadcastMode(List(input[0, string, false]))
      +- *Filter isnotnull(pkg#15)
         +- InMemoryTableScan [pkg#15, feature_field#12], [isnotnull(pkg#15)]
               +- InMemoryRelation [pkg#15, feature_field#12], true, 10000, StorageLevel(disk, memory, deserialized, 1 replicas), `tmp_t2`
                     +- Union
                        :- *HashAggregate(keys=[pkg#15, cate_l1_id#20], functions=[])
                        :  +- Exchange hashpartitioning(pkg#15, cate_l1_id#20, 200)
                        :     +- *HashAggregate(keys=[pkg#15, cate_l1_id#20], functions=[])
                        :        +- HiveTableScan [pkg#15, cate_l1_id#20], MetastoreRelation dm_sdk_mapping, app_category_mapping_par, [isnotnull(version#14), (version#14 = 1000)]
                        +- *HashAggregate(keys=[pkg#23, cate_l2_id#29], functions=[])
                           +- Exchange hashpartitioning(pkg#23, cate_l2_id#29, 200)
                              +- *HashAggregate(keys=[pkg#23, cate_l2_id#29], functions=[])
                                 +- HiveTableScan [pkg#23, cate_l2_id#29], MetastoreRelation dm_sdk_mapping, app_category_mapping_par, [isnotnull(version#22), (version#22 = 1000)]

```

将表tmp_t2 cache到内存中后，再查看spark的执行计划，发现变为Broadcast Join

通过源码可知broadcast join触发的条件：

```scala
    /**
     * Matches a plan whose output should be small enough to be used in broadcast join.
     */
    private def canBroadcast(plan: LogicalPlan): Boolean = {
      plan.stats.sizeInBytes >= 0 && plan.stats.sizeInBytes <= conf.autoBroadcastJoinThreshold
    }

```



braodcast还可以通过以下方式来产生：

```sql
-- We accept BROADCAST, BROADCASTJOIN and MAPJOIN for broadcast hint
SELECT /*+ BROADCAST(r) */ * FROM records r JOIN src s ON r.key = s.key
```

> 注：full join 不支持BroadCast Join；在left ，left semi, left anti and the internal join type ExistenceJoin中，只能广播右表；在right join中，只能广播左表；inner join两个表都可以广播。

## Shuffle Hash Join

当join右侧的表很小的时候，我们通常选择将小表broadcast出去来避免shuffle。但是因为broadcast首先会将表发送到Driver端，然后再分布到别等执行节点去，势必会造成内存的开销。因此当表的大小到了一定的规模后，使用broadcast就会对性能造成影响。

当每个分区的平均大小足够小而能用hashmap装下，那么就使用shuffle hash join，它是基于map reduce的原理来工作。当sort merge join关闭或者通过以下函数：

```scala
    /**
     * Matches a plan whose single partition should be small enough to build a hash table.
     *
     * Note: this assume that the number of partition is fixed, requires additional work if it's
     * dynamic.
     */
    private def canBuildLocalHashMap(plan: LogicalPlan): Boolean = {
      plan.stats.sizeInBytes < conf.autoBroadcastJoinThreshold * conf.numShufflePartitions
    }
```

来判断Shuffle Hash Join是否比broadcast join更好。但是，可能会出现一些场景，Shuffle Hash Join并不优于其他场景来使用join。创建hash表的代价很高，只有在单分区足够创建hash表的开销的时候才能实现。

可以通过Spark将数据通过分区的形式划分成能份较小的数据集，再对两个表中的相对应分区的数据进行join，采用分治的策略，减少了driver和executor的压力。

如下图所示：

![spark-shuffle-hash-join.png](https://github.com/zhou191101/draw/blob/master/spark/spark-shuffle-hash-join.png?raw=true)

Shuffle Hash Join分为两步：

1. 对两张表分别按照join keys进行重分区，即shuffle，目的是为了让有相同join keys值的记录分到对应的分区中
2. 对对应分区中的数据进行join，此处先将小表分区构造为一张hash表，然后根据大表分区中记录的join keys值拿出来进行匹配



Shuffle Hash Join的条件有以下几个：

1. 分区的平均大小不超过spark.sql.autoBroadcastJoinThreshold所配置的值，默认是10M
2. 基表不能被广播，比如left outer join时，只能广播右表
3. 一侧的表要明显小于另外一侧，小的一侧将被广播（明显小于的定义为3倍小，此处为经验值）

我们可以看到，在一定大小的表中，SparkSQL从时空结合的角度来看，将两个表进行重新分区，并且对小表中的分区进行hash化，从而完成join。在保持一定复杂度的基础上，尽量减少driver和executor的内存压力，提升了计算时的稳定性。

## 源码分析

Spark SQL物理执行计划的join通过`JoinSelection`来控制，如下：

```scala
  object JoinSelection extends Strategy with PredicateHelper {

    /**
     * Matches a plan whose output should be small enough to be used in broadcast join.
     */
    private def canBroadcast(plan: LogicalPlan): Boolean = {
      plan.stats.sizeInBytes >= 0 && plan.stats.sizeInBytes <= conf.autoBroadcastJoinThreshold
    }

    /**
     * Matches a plan whose single partition should be small enough to build a hash table.
     *
     * Note: this assume that the number of partition is fixed, requires additional work if it's
     * dynamic.
     */
    private def canBuildLocalHashMap(plan: LogicalPlan): Boolean = {
      plan.stats.sizeInBytes < conf.autoBroadcastJoinThreshold * conf.numShufflePartitions
    }

    /**
     * Returns whether plan a is much smaller (3X) than plan b.
     *
     * The cost to build hash map is higher than sorting, we should only build hash map on a table
     * that is much smaller than other one. Since we does not have the statistic for number of rows,
     * use the size of bytes here as estimation.
     */
    private def muchSmaller(a: LogicalPlan, b: LogicalPlan): Boolean = {
      a.stats.sizeInBytes * 3 <= b.stats.sizeInBytes
    }

    private def canBuildRight(joinType: JoinType): Boolean = joinType match {
      case _: InnerLike | LeftOuter | LeftSemi | LeftAnti | _: ExistenceJoin => true
      case _ => false
    }

    private def canBuildLeft(joinType: JoinType): Boolean = joinType match {
      case _: InnerLike | RightOuter => true
      case _ => false
    }

    private def broadcastSide(
        canBuildLeft: Boolean,
        canBuildRight: Boolean,
        left: LogicalPlan,
        right: LogicalPlan): BuildSide = {

      def smallerSide =
        if (right.stats.sizeInBytes <= left.stats.sizeInBytes) BuildRight else BuildLeft

      if (canBuildRight && canBuildLeft) {
        // Broadcast smaller side base on its estimated physical size
        // if both sides have broadcast hint
        smallerSide
      } else if (canBuildRight) {
        BuildRight
      } else if (canBuildLeft) {
        BuildLeft
      } else {
        // for the last default broadcast nested loop join
        smallerSide
      }
    }

    private def canBroadcastByHints(joinType: JoinType, left: LogicalPlan, right: LogicalPlan)
      : Boolean = {
      val buildLeft = canBuildLeft(joinType) && left.stats.hints.broadcast
      val buildRight = canBuildRight(joinType) && right.stats.hints.broadcast
      buildLeft || buildRight
    }

    private def broadcastSideByHints(joinType: JoinType, left: LogicalPlan, right: LogicalPlan)
      : BuildSide = {
      val buildLeft = canBuildLeft(joinType) && left.stats.hints.broadcast
      val buildRight = canBuildRight(joinType) && right.stats.hints.broadcast
      broadcastSide(buildLeft, buildRight, left, right)
    }

    private def canBroadcastBySizes(joinType: JoinType, left: LogicalPlan, right: LogicalPlan)
      : Boolean = {
      val buildLeft = canBuildLeft(joinType) && canBroadcast(left)
      val buildRight = canBuildRight(joinType) && canBroadcast(right)
      buildLeft || buildRight
    }

    private def broadcastSideBySizes(joinType: JoinType, left: LogicalPlan, right: LogicalPlan)
      : BuildSide = {
      val buildLeft = canBuildLeft(joinType) && canBroadcast(left)
      val buildRight = canBuildRight(joinType) && canBroadcast(right)
      broadcastSide(buildLeft, buildRight, left, right)
    }

    def apply(plan: LogicalPlan): Seq[SparkPlan] = plan match {

      // --- BroadcastHashJoin --------------------------------------------------------------------

      // broadcast hints were specified
      case ExtractEquiJoinKeys(joinType, leftKeys, rightKeys, condition, left, right)
        if canBroadcastByHints(joinType, left, right) =>
        val buildSide = broadcastSideByHints(joinType, left, right)
        Seq(joins.BroadcastHashJoinExec(
          leftKeys, rightKeys, joinType, buildSide, condition, planLater(left), planLater(right)))

      // broadcast hints were not specified, so need to infer it from size and configuration.
      case ExtractEquiJoinKeys(joinType, leftKeys, rightKeys, condition, left, right)
        if canBroadcastBySizes(joinType, left, right) =>
        val buildSide = broadcastSideBySizes(joinType, left, right)
        Seq(joins.BroadcastHashJoinExec(
          leftKeys, rightKeys, joinType, buildSide, condition, planLater(left), planLater(right)))

      // --- ShuffledHashJoin ---------------------------------------------------------------------

      case ExtractEquiJoinKeys(joinType, leftKeys, rightKeys, condition, left, right)
         if !conf.preferSortMergeJoin && canBuildRight(joinType) && canBuildLocalHashMap(right)
           && muchSmaller(right, left) ||
           !RowOrdering.isOrderable(leftKeys) =>
        Seq(joins.ShuffledHashJoinExec(
          leftKeys, rightKeys, joinType, BuildRight, condition, planLater(left), planLater(right)))

      case ExtractEquiJoinKeys(joinType, leftKeys, rightKeys, condition, left, right)
         if !conf.preferSortMergeJoin && canBuildLeft(joinType) && canBuildLocalHashMap(left)
           && muchSmaller(left, right) ||
           !RowOrdering.isOrderable(leftKeys) =>
        Seq(joins.ShuffledHashJoinExec(
          leftKeys, rightKeys, joinType, BuildLeft, condition, planLater(left), planLater(right)))

      // --- SortMergeJoin ------------------------------------------------------------

      case ExtractEquiJoinKeys(joinType, leftKeys, rightKeys, condition, left, right)
        if RowOrdering.isOrderable(leftKeys) =>
        joins.SortMergeJoinExec(
          leftKeys, rightKeys, joinType, condition, planLater(left), planLater(right)) :: Nil

      // --- Without joining keys ------------------------------------------------------------

      // Pick BroadcastNestedLoopJoin if one side could be broadcast
      case j @ logical.Join(left, right, joinType, condition)
          if canBroadcastByHints(joinType, left, right) =>
        val buildSide = broadcastSideByHints(joinType, left, right)
        joins.BroadcastNestedLoopJoinExec(
          planLater(left), planLater(right), buildSide, joinType, condition) :: Nil

      case j @ logical.Join(left, right, joinType, condition)
          if canBroadcastBySizes(joinType, left, right) =>
        val buildSide = broadcastSideBySizes(joinType, left, right)
        joins.BroadcastNestedLoopJoinExec(
          planLater(left), planLater(right), buildSide, joinType, condition) :: Nil

      // Pick CartesianProduct for InnerJoin
      case logical.Join(left, right, _: InnerLike, condition) =>
        joins.CartesianProductExec(planLater(left), planLater(right), condition) :: Nil

      case logical.Join(left, right, joinType, condition) =>
        val buildSide = broadcastSide(
          left.stats.hints.broadcast, right.stats.hints.broadcast, left, right)
        // This join could be very slow or OOM
        joins.BroadcastNestedLoopJoinExec(
          planLater(left), planLater(right), buildSide, joinType, condition) :: Nil

      // --- Cases where this strategy does not apply ---------------------------------------------

      case _ => Nil
    }
  }

```

首先看注释，该策略会首先使用`ExtractEquiJoinKeys`来确定join至少有一个谓词是可以去估算的，如果有的话，就要根据这些谓词来去计算选择哪种join，这里分三种Join，广播Join，Shuffle Hash Join，还有最常见的Sort Merge Join。如果没有谓词可以估算的话，那么也是有两种方式：`BroadcastNestedLoopJoin`和`CartesianProduct`。

### Broadcast Join

涉及到两个方法：`canBroadcast()`和`canBuildxxx`(`canBuildLeft()` 和`canBuildRight()`)

```scala
/**
 * Matches a plan whose output should be small enough to be used in broadcast join.
 */
private def canBroadcast(plan: LogicalPlan): Boolean = {
  plan.stats.sizeInBytes >= 0 && plan.stats.sizeInBytes <= conf.autoBroadcastJoinThreshold
}
```

可以看到会去conf里面查找`autoBroadcastJoinThreshold`即AUTO_BROADCASTJOIN_THRESHOLD，这个参数为-1，则不可用。

```scala
    private def canBuildRight(joinType: JoinType): Boolean = joinType match {
      case _: InnerLike | LeftOuter | LeftSemi | LeftAnti | _: ExistenceJoin => true
      case _ => false
    }

    private def canBuildLeft(joinType: JoinType): Boolean = joinType match {
      case _: InnerLike | RightOuter => true
      case _ => false
    }
```

`canBuildRight()`和`canBuildLeft()`表示有一边为主的时候为true。

有两种方式会产生broadcast join：

```scala
      // broadcast hints were specified
      case ExtractEquiJoinKeys(joinType, leftKeys, rightKeys, condition, left, right)
        if canBroadcastByHints(joinType, left, right) =>
        val buildSide = broadcastSideByHints(joinType, left, right)
        Seq(joins.BroadcastHashJoinExec(
          leftKeys, rightKeys, joinType, buildSide, condition, planLater(left), planLater(right)))


```

第一种是显示broadcast的，比如在代码中显示的broadcast需要join的某个表。

```scala

      // broadcast hints were not specified, so need to infer it from size and configuration.
      case ExtractEquiJoinKeys(joinType, leftKeys, rightKeys, condition, left, right)
        if canBroadcastBySizes(joinType, left, right) =>
        val buildSide = broadcastSideBySizes(joinType, left, right)
        Seq(joins.BroadcastHashJoinExec(
          leftKeys, rightKeys, joinType, buildSide, condition, planLater(left), planLater(right)))
```

第二种是未broadcast的，需要根据参数`AUTO_BROADCASTJOIN_THRESHOLD`以及表的大小来判断是否需要broadcast join。

最后都调用`joins.BroadcastHashJoinExec(leftKeys, rightKeys, joinType, buildSide, condition, planLater(left), planLater(right))`执行，跟进源码。

```scala
case class BroadcastHashJoinExec(
    leftKeys: Seq[Expression],
    rightKeys: Seq[Expression],
    joinType: JoinType,
    buildSide: BuildSide,
    condition: Option[Expression],
    left: SparkPlan,
    right: SparkPlan)
  extends BinaryExecNode with HashJoin with CodegenSupport {

  override lazy val metrics = Map(
    "numOutputRows" -> SQLMetrics.createMetric(sparkContext, "number of output rows"))

  override def requiredChildDistribution: Seq[Distribution] = {
    val mode = HashedRelationBroadcastMode(buildKeys)
    // 要被广播的表
    buildSide match {
      case BuildLeft =>
        BroadcastDistribution(mode) :: UnspecifiedDistribution :: Nil
      case BuildRight =>
        UnspecifiedDistribution :: BroadcastDistribution(mode) :: Nil
    }
  }

  protected override def doExecute(): RDD[InternalRow] = {
    val numOutputRows = longMetric("numOutputRows")

    val broadcastRelation = buildPlan.executeBroadcast[HashedRelation]()
    streamedPlan.execute().mapPartitions { streamedIter =>
      val hashed = broadcastRelation.value.asReadOnlyCopy()
      TaskContext.get().taskMetrics().incPeakExecutionMemory(hashed.estimatedSize)
      join(streamedIter, hashed, numOutputRows)
    }
  }
```

这边，将`buildPlan`广播出去以后，将`streamedPlan`调用`execute()`过后返回的RDD[InternalRow]，调用`mapPartitions`，根据每个分区和广播的小表进行join操作。

### Shuffle Hash Join

涉及到`canBuildLocalHashMap()`方法、`muchSmaller()`方法和一个配置项**PREFER_SORTMERGEJOIN**，这个配置项的解释是：

```
When true, prefer sort merge join over shuffle hash join.
```

意思是，如果未true，则首选sort merge join。

触发此Join的条件是需要判断是否满足每个分区的平均大小能小到足够使用hash表放下:

```scala
    /**
     * Matches a plan whose single partition should be small enough to build a hash table.
     *
     * Note: this assume that the number of partition is fixed, requires additional work if it's
     * dynamic.
     */
    private def canBuildLocalHashMap(plan: LogicalPlan): Boolean = {
      plan.stats.sizeInBytes < conf.autoBroadcastJoinThreshold * conf.numShufflePartitions
    }
```

`muchSmaller()`:

```scala
    /**
     * Returns whether plan a is much smaller (3X) than plan b.
     *
     * The cost to build hash map is higher than sorting, we should only build hash map on a table
     * that is much smaller than other one. Since we does not have the statistic for number of rows,
     * use the size of bytes here as estimation.
     */
    private def muchSmaller(a: LogicalPlan, b: LogicalPlan): Boolean = {
      a.stats.sizeInBytes * 3 <= b.stats.sizeInBytes
    }
```

实际上就是b是a的三倍大小

Shuffle Hash Join触发的源码如下：

```
      // --- ShuffledHashJoin ---------------------------------------------------------------------

      case ExtractEquiJoinKeys(joinType, leftKeys, rightKeys, condition, left, right)
         if !conf.preferSortMergeJoin && canBuildRight(joinType) && canBuildLocalHashMap(right)
           && muchSmaller(right, left) ||
           !RowOrdering.isOrderable(leftKeys) =>
        Seq(joins.ShuffledHashJoinExec(
          leftKeys, rightKeys, joinType, BuildRight, condition, planLater(left), planLater(right)))

      case ExtractEquiJoinKeys(joinType, leftKeys, rightKeys, condition, left, right)
         if !conf.preferSortMergeJoin && canBuildLeft(joinType) && canBuildLocalHashMap(left)
           && muchSmaller(left, right) ||
           !RowOrdering.isOrderable(leftKeys) =>
        Seq(joins.ShuffledHashJoinExec(
          leftKeys, rightKeys, joinType, BuildLeft, condition, planLater(left), planLater(right)))

```

最后调用`joins.ShuffledHashJoinExec(leftKeys, rightKeys, joinType, BuildLeft, condition, planLater(left), planLater(right))`方法，跟进源码：

```scala

/**
 * Performs a hash join of two child relations by first shuffling the data using the join keys.
 */
case class ShuffledHashJoinExec(
    leftKeys: Seq[Expression],
    rightKeys: Seq[Expression],
    joinType: JoinType,
    buildSide: BuildSide,
    condition: Option[Expression],
    left: SparkPlan,
    right: SparkPlan)
  extends BinaryExecNode with HashJoin {

  override lazy val metrics = Map(
    "numOutputRows" -> SQLMetrics.createMetric(sparkContext, "number of output rows"),
    "buildDataSize" -> SQLMetrics.createSizeMetric(sparkContext, "data size of build side"),
    "buildTime" -> SQLMetrics.createTimingMetric(sparkContext, "time to build hash map"))

  override def requiredChildDistribution: Seq[Distribution] =
    HashClusteredDistribution(leftKeys) :: HashClusteredDistribution(rightKeys) :: Nil

  private def buildHashedRelation(iter: Iterator[InternalRow]): HashedRelation = {
    val buildDataSize = longMetric("buildDataSize")
    val buildTime = longMetric("buildTime")
    val start = System.nanoTime()
    val context = TaskContext.get()
    val relation = HashedRelation(iter, buildKeys, taskMemoryManager = context.taskMemoryManager())
    buildTime += (System.nanoTime() - start) / 1000000
    buildDataSize += relation.estimatedSize
    // This relation is usually used until the end of task.
    context.addTaskCompletionListener[Unit](_ => relation.close())
    relation
  }

  protected override def doExecute(): RDD[InternalRow] = {
    val numOutputRows = longMetric("numOutputRows")
    streamedPlan.execute().zipPartitions(buildPlan.execute()) { (streamIter, buildIter) =>
      val hashed = buildHashedRelation(buildIter)
      join(streamIter, hashed, numOutputRows)
    }
  }
}

```

`ShuffledHashJoin`和`BroadcastJoin`在构造Hash Table上有不同，后者是依靠广播生成的`HashedRelation`，前者是调用`zipPartitions`方法，该方法的作用是将两个有相同分区数的RDD合并，映射参数是两个RDD的迭代器，可以看到在这里是`(streamIter, buildIter)`，然后对`buildIter`构造HashRelation。这也就说明：**BroadcastJoin的HashRelation是小表的全部数据，而ShuffledHashJoin的HashRelation只是小表跟大表在同一分区内的一部分数据**。

###SortMergeJoin 

```scala
      // --- SortMergeJoin ------------------------------------------------------------

      case ExtractEquiJoinKeys(joinType, leftKeys, rightKeys, condition, left, right)
        if RowOrdering.isOrderable(leftKeys) =>
        joins.SortMergeJoinExec(
          leftKeys, rightKeys, joinType, condition, planLater(left), planLater(right)) :: Nil

```

最后调用`oins.SortMergeJoinExec(leftKeys, rightKeys, joinType, condition, planLater(left), planLater(right))`方法，跟进源码：

```scala
  protected override def doExecute(): RDD[InternalRow] = {
    val numOutputRows = longMetric("numOutputRows")
    val spillThreshold = getSpillThreshold
    val inMemoryThreshold = getInMemoryThreshold
    left.execute().zipPartitions(right.execute()) { (leftIter, rightIter) =>
      val boundCondition: (InternalRow) => Boolean = {
        condition.map { cond =>
          newPredicate(cond, left.output ++ right.output).eval _
        }.getOrElse {
          (r: InternalRow) => true
        }
      }

```

同样是调用`zipPartitions()`方法。	

## 总结

在很多时候， Sort-Merge Join是一种很好的选择，它能将数据溢写到磁盘而不必向Shuffle Hash join一样一直将数据持久化在内存中。如果有可能，最好使用BroadCast Join，因为它避免了shuffle。如果你能很明确的确定Shuffle Hash join优于Sort-Merge Join，那么将Sort-Merge Join关闭掉就行了。

## 参考

* https://medium.com/datakaresolutions/optimize-spark-sql-joins-c81b4e3ed7da
* https://www.linkedin.com/pulse/spark-sql-3-common-joins-explained-ram-ghadiyaram/
* http://spark.apache.org/docs/latest/sql-performance-tuning.html
* https://www.imooc.com/article/261893

