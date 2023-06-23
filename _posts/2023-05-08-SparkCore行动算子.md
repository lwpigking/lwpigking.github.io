---
layout: post
title: SparkCore行动算子
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

#### RDD_Action(行动算子)

行动算子：触发任务的调度和作业的执行

##### aggregate

```scala
//第一个是分区内操作
//第二个是分区间操作
//aggregate 方法是一个聚合函数，接受多个输入，并按照一定的规则运算以后输出一个结果值。
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("action")
val sc = new SparkContext(sparkConf)
val rdd = sc.makeRDD (
    List(1, 2, 3, 4),2
)
val result = rdd.aggregate(0)(_ + _,_ + _)
// 各自分区内：0+1+2=3 0+3+4=7 
// 分区间：0+3+7=10
val result2 = rdd.aggregate(10)(_ + _,_ + _)
// 各自分区间内：1+2+10=13 3+4+10=17
// 分区间：10+13+17=40
// ******初始值也会参与分区间的计算*******
// 如果分区数为8:13+17+10*(8-1)=100
println(result)
println(result2)

// 输出结果为10和40
```

##### collect

```scala
// collect在sparksql中特别重要
// 以一张静态网页举例，该网页全是table，里面都是td和tr
// 现在要爬取该网页tr里面的文字内容
// 用python的pandas.read_html()即可提取该网页并且变成DataFrame进行存储
// SparkSQL在show()后的结果就和该网页一样，有大量的tr格式
// 加上collect()后就和read_html()一样，可以取消格式并且将数据变成集合进行存储
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("action")
val sc = new SparkContext(sparkConf)
val rdd = sc.makeRDD (
    List(1, 2, 3, 4)
)
println(rdd.collect()(0))
// 输出结果为1
```

##### count

```scala
// count是计算rdd中的元素个数
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("action")
val sc = new SparkContext(sparkConf)
val rdd = sc.makeRDD (
    List(1, 2, 3, 4)
)
val countResult = rdd.count()
println(countResult)
// 输出结果为4
```

##### countByKey

```scala
// countByKey是统计每种key的个数
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("action")
val sc = new SparkContext(sparkConf)
val rdd = sc.makeRDD (
    List(
        (1, "a"),(1, "a"),(1, "a"),(1, "a"),(2,"b"),(3,"c"),(3,"c")
    )
)
val result = rdd.countByKey()
println(result.mkString(","))
// 输出结果 1 -> 4,2 -> 1,3 -> 2
```

##### first

```scala
// first返回rdd中的第一个元素
// 等价于rdd.collect()(0)
// rdd.first() == rdd.collect()(0)
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("action")
val sc = new SparkContext(sparkConf)
val rdd = sc.makeRDD (
    List(1, 2, 3, 4)
)
println(rdd.first())
// 返回结果为1
```

##### fold

```scala
// aggregate的简化版
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("action")
val sc = new SparkContext(sparkConf)
val rdd = sc.makeRDD (
    List(1, 2, 3, 4),2
)
val result = rdd.fold(10)(_ + _)
println(result)
// 返回结果为40
```

##### reduce

```scala
// reduce聚合rdd中所有的元素
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("action")
val sc = new SparkContext(sparkConf)
val rdd = sc.makeRDD (
    List(1, 2, 3, 4),2
)
val rdd2 = rdd.reduce(_ + _)
println(rdd2)
// 返回结果为10
```

##### take

```scala
// 返回一个有RDD的前n个元素组成的素组
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("action")
val sc = new SparkContext(sparkConf)
val rdd = sc.makeRDD (
    List(1, 2, 3, 4)
)
val takeResult = rdd.take(2)
println(takeResult.mkString(","))
// 返回结果为1,2
```

##### takeOrderd

```scala
// 返回RDD排序后的前n个元素组成的素组
// 先将RDD进行排序，然后取前n个元素
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("action")
val sc = new SparkContext(sparkConf)
val rdd = sc.makeRDD (
    List(1, 2, 4, 3,3,5,7,0)
)
val result = rdd.takeOrdered(5)
println(result.mkString(","))
// 返回结果为0,1,2,3,3
```
