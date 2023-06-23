---
layout: post
title: SparkCore键值算子
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

#### RDD_KeyValue

##### partitionBy

```scala
// partitionBy根据指定的分区规则对数据进行重分区
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("partitionBy")
val sc = new SparkContext(sparkConf)
val rdd = sc.makeRDD(List(1,2,3,4))

// 隐式转换(二次编译)
val mapRDD = rdd.map((_,1))

mapRDD.partitionBy(new HashPartitioner(2))

sc.stop()
```

##### aggregateByKey

```scala
// 对数据的key按照不同的规则进行分区内计算和分区间计算
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("aggregateByKey")
val sc = new SparkContext(sparkConf)
val dataRDD1 = sc.makeRDD(List(("a",1),("a",2),("a",3),("a",4)),2)

// aggregateByKey存在函数柯里化，有两个参数列表

// 第一个参数：需要传递一个参数，表示为初始值
// 			 主要用于当碰见第一个key的时候，和value进行分区内计算

// 第二个参数：需要传递2个函数
//            第一个表示分区内计算规则
//            第二个表示分区间计算规则

val dataRDD2 = dataRDD1.aggregateByKey(0)(
    (x, y) => math.max(x, y), // 分区内求最大值
    (x, y) => x + y // 分区间求和
)
dataRDD2.collect().foreach(println)
sc.stop()
// 返回结果 (a,6)
```

```scala
// aggregateByKey最终的返回数据结果应该和初始值的类型保持一致
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("aggregateByKey")
val sc = new SparkContext(sparkConf)
val rdd = sc.makeRDD(
    List(
        ("a",1),("a",2),("b",3),
        ("b",4),("b",5),("a",6)
    ),2
)

// 获取相同key的数据的平均值 => (a, 3),(b, 4)
val newRDD = rdd.aggregateByKey( (0,0) )(
	( t, v ) => {
        (t._1 + v, t._2 + 1)
    },
    (t1, t2) => {
        (t1._1 + t2._1, t1._2 + t2._2)
    }
)

val resultRDD = newRDD.mapValues{
    case(num, cnt) => {
        num / cnt
    }
}

resultRDD.collect().foreach(println)

sc.stop()
```

##### combineByKey

```scala
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("combineByKey")
val sc = new SparkContext(sparkConf)
val rdd = sc.makeRDD(
    List(
        ("a",1),("a",2),("b",3),
        ("b",4),("b",5),("a",6)
    ),2
)

// combineByKey方法需要三个参数
// 第一个参数表示：将相同key的第一个数据进行结构的转换，实现操作
// 第二个参数表示：分区内的计算规则
// 第三个参数表示：分区间的计算规则

val newRDD = rdd.combineByKey(
	v => (v, 1),
    ( t:(Int, Int), v ) => {
        (t._1 + v, t._2 + 1)
    },
    (t1:(Int, Int), t2:(Int, Int)) => {
        (t1._1 + t2._1, t1._2 + t2._2)
    }
)

val resultRDD = newRDD.mapValues{
    case(num, cnt) => {
        num / cnt
    }
}

sc.stop()
```

##### cogroup

```scala
// 对两个RDD中的KV元素，每个RDD中相同key中的元素分别聚合成一个集合
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("cogroup")
val sc = new SparkContext(sparkConf)
val rdd1 = sc.makeRDD(
    List(
        ("a", 1), ("b", 2), ("c", 3)
    )
)
val rdd2 = sc.makeRDD(
    List(
        ("a", 4), ("b", 5), ("c", 6)
    )
)
rdd1.cogroup(rdd2).collect().foreach(print)
sc.stop()
// 返回结果 
// (a,(CompactBuffer(1),CompactBuffer(4)))
// (b,(CompactBuffer(2),CompactBuffer(5)))
// (c,(CompactBuffer(3),CompactBuffer(6)))
```

##### foldByKey

```scala
// 当分区内计算规则和分区间计算规则相同时，aggregateByKey 就可以简化为 foldByKey
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("foldByKey")
val sc = new SparkContext(sparkConf)
val dataRDD1 = sc.makeRDD(
    List(
        ("a",1),("a",2),("b",3),
        ("b",4),("b",5),("a",6)
    ),2
)

dataRDD1.foldByKey(0)(_+_).collect().foreach(println)

sc.stop()
// ("a",9)
// ("b",12)
```

##### groupByKey

```scala
// 将数据源的数据根据 key 对 value 进行分组，形成一个对偶元祖
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("groupByKey")
val sc = new SparkContext(sparkConf)
val dataRDD1 =sc.makeRDD(List(("a",1),("a",2),("a",3),("b",4)))
val dataRDD2 = dataRDD1.groupByKey()
dataRDD2.collect().foreach(println)
sc.stop()
// 返回结果
// (a, CompactBuffer(1,2,3))
// (b, CompactBuffer(4))
```

##### join

```scala
// 在类型为(K,V)和(K,W)的 RDD 上调用
// 返回一个相同 key对应的所有元素连接在一起的
// (K,(V,W))的 RDD
// 如果两个数据源中key没有匹配上，那么数据不会出现在结果中
// 如果两个数据源中key有多个相同的，会依次匹配，可能会出现笛卡尔集
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("join")
val sc = new SparkContext(sparkConf)
val rdd = sc.makeRDD(Array(("a",1), ("b",2), ("c",3)))
val rdd1 = sc.makeRDD(Array(("a",4), ("b",5), ("c",6)))
rdd.join(rdd1).collect().foreach(println)
sc.stop()
// 返回结果
// (a,(1,4))
// (b,(2,5))
// (c,(3,6))
```

##### leftOutJoin

```scala
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("leftOutJoin")
val sc = new SparkContext(sparkConf)
val rdd1 = sc.makeRDD(
    List(
        ("a", 1), ("b", 2)
    )
)
val rdd2 = sc.makeRDD(
    List(
        ("a", 4), ("b", 5), ("c", 6)
    )
)
rdd1.leftOuterJoin(rdd2).collect().foreach(print)
sc.stop()
// 返回结果 (a,(Some(1),4))
//         (b,(Some(2),5))
//         (c,(None,6))
```

##### reduceByKey

```scala
// 可以将数据按照相同的 Key 对 Value 进行聚合
// reduceByKey中如果key的数据只有一个，是不会参与计算的
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("reduceByKey")
val sc = new SparkContext(sparkConf)
val rdd = sc.makeRDD (
    List(("a",1),("b",2),("c",3),("a",4))
)
val result = rdd.reduceByKey(_ + _)
result.collect().foreach(println)
println("=================")
sc.stop()
// 返回结果 
// ("a", 5)
// ("b", 2)
// ("c", 3)
```

##### sortByKey

```scala
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("partitionBy")
val sc = new SparkContext(sparkConf)
val dataRDD1 = sc.makeRDD(List(("a",1),("b",2),("c",3)))
dataRDD1.sortByKey(true).collect().foreach(println)
println("======================")
dataRDD1.sortByKey(false).collect().foreach(println)
// 返回结果
// (a,1)
// (b,2)
// (c,3)
// ======================
// (c,3)
// (b,2)
// (a,1)
```
