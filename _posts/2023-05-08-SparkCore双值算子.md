---
layout: post
title: SparkCore双值算子
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

#### RDD_DoubleValue

##### cartesian

```scala
// 笛卡尔集
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("DoubleValue")
val sc = new SparkContext(sparkConf)
val rdd1 = sc.makeRDD(
    List(1, 2, 3, 4)
)
val rdd2 = sc.makeRDD(
    List(3, 4, 5)
)
val rdd3 = rdd1.cartesian(rdd2)
rdd3.collect().foreach(println)

sc.stop()
// 返回结果(1,3)(1,4)(1,5)
//        (2,3)(2,4)(2,5)
//        (3,3)(3,4)(3,5)
//        (4,3)(4,4)(4,5)
```

##### intersection

```scala
// 交集
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("DoubleValue")
val sc = new SparkContext(sparkConf)
val rdd1 = sc.makeRDD(
    List(1, 2, 3, 4)
)
val rdd2 = sc.makeRDD(
    List(3, 4, 5)
)
val rdd3 = rdd1.intersection(rdd2)
rdd3.collect().foreach(println)

sc.stop()
// 返回结果 3 4 
```

##### subtract

```scala
// 差集,返回第一个rdd中有而第二个没的
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("DoubleValue")
val sc = new SparkContext(sparkConf)
val rdd1 = sc.makeRDD(
    List(1, 2, 3, 4)
)
val rdd2 = sc.makeRDD(
    List(3, 4, 5)
)
val rdd3 = rdd1.subtract(rdd2)
rdd3.collect().foreach(println)

sc.stop()
// 返回结果 1 2
```

##### union

```scala
// 并集两个RDD,不会去重
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("DoubleValue")
val sc = new SparkContext(sparkConf)
val rdd1 = sc.makeRDD(
    List(1, 2, 3, 4)
)
val rdd2 = sc.makeRDD(
    List(3, 4, 5)
)
val rdd3 = rdd1.union(rdd2)
rdd3.collect().foreach(println)

sc.stop()
// 返回结果 1 2 3 4 3 4 5
```

##### zip

```scala
// 两个RDD拉链
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("DoubleValue")
val sc = new SparkContext(sparkConf)
val rdd1 = sc.makeRDD(
    List(1, 2, 3, 4)
)
val rdd2 = sc.makeRDD(
    List(3, 4, 5, 6)
)
val rdd3 = rdd1.zip(rdd2)
rdd3.collect().foreach(println)

sc.stop()
// 返回结果(1,3) (2,4) (3,5) (4,6)
```
