---
layout: post
title: SparkCore创建RDD
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

#### 创建RDD

```scala
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("RDD")
val sparkContext = new SparkContext(sparkConf)

// 从内存中创建RDD，将内存中集合的数据处理的数据源
val rdd1 = sparkContext.parallelize(
 List(1,2,3,4)
)

val rdd2 = sparkContext.makeRDD(
 List(1,2,3,4)
)

rdd1.collect().foreach(println)
rdd2.collect().foreach(println)

sparkContext.stop()
```

```scala
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("RDD")
val sparkContext = new SparkContext(sparkConf)

// 从文件中创建RDD
val rdd = sparkContext.textFile("data/1.txt")

rdd.collect().foreach(println)

sparkContext.stop()
```

#### 并行度与分区

```scala
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("RDD")
val sparkContext = new SparkContext(sparkConf)

// RDD的并行度与分区
// makeRDD第二个参数表示分区数量
val rdd = sparkContext.makeRDD(
	List(1,2,3,4),2
)

// 将处理的数据保存成分区文件
rdd.saveAsTextFile("output")

sparkContext.stop()
```
