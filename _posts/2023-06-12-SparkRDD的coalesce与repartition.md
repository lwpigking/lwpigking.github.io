---
layout: post
title: SparkCore的coalesce与repartition
categories: [Spark]
description: Spark
keywords: Spark
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

本文作者：张老师！！！！！！

RDD的分区数量可以通过 coalesce 与 repartition 这两个算子修改RDD分区数量
1 repartition 会有 shuffle过程， RDD的分区可以增大也可以减少

```scala
// 创建一个RDD 分区数量为3
val rdd1 = sc.makeRDD(Array(1,2,3,4,5,6),3)

// 修改rdd1的分区数量为4
rdd1.repartition(4)

// 输出一下RDD的分区个数
res0.partitions.size

// 修改rdd1的分区数量为2
rdd1.repartition(2)

// 输出一下RDD的分区个数
res2.partitions.size
```

2 coalesce 没有shuffle过程，RDD的分区只可以减少不可以增加

```scala
// 创建一个RDD 分区数量为3
val rdd1 = sc.makeRDD(Array(1,2,3,4,5,6),3)

// 修改rdd1的分区数量为4
rdd1.coalesce(6)

// 输出一下RDD的分区个数
res4.partitions.size

// 修改rdd1的分区数量为2
rdd1.coalesce(1)

// 输出一下RDD的分区个数
res6.partitions.size
```

1）在IDEA查看 coalesce 与 repartition 源码 可以发现 其实repartition在更改分区的时候也是调用的coalesce算子，只不过通过 repartition 对RDD进行分区的时候 调用coalesce 算子时shuffle=true。
2）而在使用coalesce对RDD分区做修改的时候shuffle=false。
3）coalesce的使用场景在对数据做过滤的时候，过滤掉了大量的数据，分区过多会浪费性能，所以采用coalesce减少RDD的分区数量
