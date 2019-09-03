---
layout:     post
title:      "深入理解Spark Join"
subtitle:   ""
date:       2019-09-03
author:     "zhoup"
header-img: "img/post-bg-unix-linux.jpg"
tags:
    - 大数据
    - Java
    - Netty
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

braodcast还可以通过以下方式来产生：

```sql
-- We accept BROADCAST, BROADCASTJOIN and MAPJOIN for broadcast hint
SELECT /*+ BROADCAST(r) */ * FROM records r JOIN src s ON r.key = s.key
```

## Shuffle Hash Join

当join右侧的表很小的时候，我们通常选择将小表broadcast出去来避免shuffle。但是因为broadcast首先会将表发送到Driver端，然后再分布到别等执行节点去，势必会造成内存的开销。因此当表的大小到了一定的规模后，使用broadcast就会对性能造成影响。

shuffle hash join基于map reduce的原理来工作。当sort merge join关闭或者通过以下函数：

```scala
def canBuildLocalHashMap(plan:LogicalPlan):Boolean= { plan.statistics.sizeInBytes < conf.autoBroadcastJoinThreshold * conf.numShufflePartitions
}
```

来判断Shuffle Hash Join是否比broadcast join更好。但是，kennel会出现一些场景，Shuffle Hash Join并不优于其他场景来使用join。创建hash表的代价很高，只有在单分区足够创建hash表的开销的时候才能实现。

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

## 总结

在很多时候， Sort-Merge Join是一种很好的选择，它能将数据溢写到磁盘而不必向Shuffle Hash join一样一直将数据持久化在内存中。如果有可能，最好使用BroadCast Join，因为它避免了shuffle。如果你能很明确的确定Shuffle Hash join优于Sort-Merge Join，那么将Sort-Merge Join关闭掉就行了。

## 参考

* https://medium.com/datakaresolutions/optimize-spark-sql-joins-c81b4e3ed7da
* https://www.linkedin.com/pulse/spark-sql-3-common-joins-explained-ram-ghadiyaram/
* http://spark.apache.org/docs/latest/sql-performance-tuning.html