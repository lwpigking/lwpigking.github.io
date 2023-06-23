---
layout: post
title: SparkCore练习
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

#### SparkCore练习

1.创建一个1-10数组的RDD，将所有元素*2形成新的RDD

```scala
val sparkConf = new SparkConf().setMaster("local[*]").setAppname("Exam1")
val sc = new SparkContext(sparkConf)
val inputRDD = sc.makeRDD(1 to 10)
val newRDD = inputRDD.map(_*2)
newRDD.collect().foreach(println)
```

2.创建一个10-20数组的RDD，使用mapPartitions将所有元素*2形成新的RDD

```scala
val sparkConf = new SparkConf().setMaster("local[*]").setAppname("Exam2")
val sc = new SparkContext(sparkConf)
val inputRDD = sc.makeRDD(10 to 20)
val newRDD = inputRDD.mapPartitions(
	iter => {
        iter.map(_*2)
    }
)
newRDD.collect().foreach(println)
```

3.创建一个元素为 1-5 的RDD，运用 flatMap创建一个新的 RDD，新的 RDD 为原 RDD 每个元素的 平方和三次方 来组成 1,1,4,8,9,27…

```scala
val sparkConf = new SparkConf().setMaster("local[*]").setAppname("Exam3")
val sc = new SparkContext(sparkConf)
val inputRDD = sc.makeRDD(1 to 5)
val newRDD = inputRDD.flatMap(n =>{
    List(Math.pow(n,2).toInt,Math.pow(n,3).toInt)
})
newRDD.collect().foreach(println)
```

4.创建一个 4 个分区的 RDD数据为Array(10,20,30,40,50,60)，使用glom将每个分区的数据放到一个数组

```scala
val sparkConf = new SparkConf().setMaster("local[*]").setAppname("Exam4")
val sc = new SparkContext(sparkConf)
val inputRDD = sc.makeRDD(
	Array(10,20,30,40,50,60),4
)
val newRDD = inputRDD.glom()
newRDD.foreach(x => println(x.mkString(",")))
```

5.创建一个 RDD数据为Array(1, 3, 4, 20, 4, 5, 8)，按照元素的奇偶性进行分组

```scala
val sparkConf = new SparkConf().setMaster("local[*]").setAppname("Exam5")
val sc = new SparkContext(sparkConf)
val inputRDD = sc.makeRDD(
	Array(1,3,4,20,4,5,8)
)
val newRDD = inputRDD.groupBy(_*2==0)
newRDD.collect().foreach(println)
```

6.创建一个 RDD（由字符串组成）Array(“xiaoli”, “laoli”, “laowang”, “xiaocang”, “xiaojing”, “xiaokong”)，过滤出一个新 RDD（包含“xiao”子串）

```scala
val sparkConf = new SparkConf().setMaster("local[*]").setAppname("Exam6")
val sc = new SparkContext(sparkConf)
val inputRDD = sc.makeRDD(
	Array(“xiaoli”, “laoli”, “laowang”, “xiaocang”, “xiaojing”, “xiaokong”)
)
val newRDD = inputRDD.filter(_.contains("xiao"))
newRDD.collect().foreach(println)
```

7.创建一个 RDD数据为1 to 10，请使用sample不放回抽样

```scala
val sparkConf = new SparkConf().setMaster("local[*]").setAppname("Exam7")
val sc = new SparkContext(sparkConf)
val inputRDD = sc.makeRDD(1 to 10)
val newRDD = inputRDD.sample(false,0.5)
newRDD.collect().foreach(println)
```

8.创建一个 RDD数据为Array(10,10,2,5,3,5,3,6,9,1),对 RDD 中元素执行去重操作

```scala
val sparkConf = new SparkConf().setMaster("local[*]").setAppname("Exam8")
val sc = new SparkContext(sparkConf)
val inputRDD = sc.makeRDD(
	Array(10,10,2,5,3,5,3,6,9,1)
)
val newRDD = inputRDD.distinct()
newRDD.collect().foreach(println)
```

9.创建一个分区数为5的 RDD，数据为0 to 100，之后使用coalesce再重新减少分区的数量至 2

```scala
val sparkConf = new SparkConf().setMaster("local[*]").setAppname("Exam9")
val sc = new SparkContext(sparkConf)
val inputRDD = sc.makeRDD(0 to 100,5)
val newRDD = inputRDD.coalesce(2)
println(newRDD.partitions.length)
```

10.创建一个分区数为5的 RDD，数据为0 to 100，之后使用repartition再重新减少分区的数量至 3

```scala
val sparkConf = new SparkConf().setMaster("local[*]").setAppname("Exam10")
val sc = new SparkContext(sparkConf)
val inputRDD = sc.makeRDD(0 to 100,5)
val newRDD = inputRDD.repartition(3)
println(newRDD.partitions.length)
```

11.创建一个 RDD数据为1,3,4,10,4,6,9,20,30,16,请给RDD进行分别进行升序和降序排列

```scala
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("Exam11")
val sc = new SparkContext(sparkConf)
val inputRDD = sc.makeRDD(
    Array(1,3,4,10,4,6,9,20,30,16)
)
val newRDD1 = inputRDD.sortBy(x => x,true)
newRDD1.collect().foreach(print)
println()
println("========================")
val newRDD2 = inputRDD.sortBy(x => x,false)
newRDD2.collect().foreach(print)
```

12.创建两个RDD，分别为rdd1和rdd2数据分别为1 to 6和4 to 10，求并集

```scala
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("Exam12")
val sc = new SparkContext(sparkConf)
val inputRDD1 = sc.makeRDD(1 to 6)
val inputRDD2 = sc.makeRDD(4 to 10)
val newRDD = inputRDD1.union(inputRDD2)
newRDD.collect().foreach(println)
```

13.创建一个RDD数据为List((“female”,1),(“male”,5),(“female”,5),(“male”,2))，请计算出female和male的总数分别为多少

```scala
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("Exam13")
val sc = new SparkContext(sparkConf)
val inputRDD = sc.makeRDD(
    List(("female",1),("male",5),("female",5),("male",2))
)
val newRDD = inputRDD.reduceByKey(_+_)
newRDD.collect().foreach(println)
```

14.创建一个有两个分区的 RDD数据为List((“a”,3),(“a”,2),(“c”,4),(“b”,3),(“c”,6),(“c”,8))，取出每个分区相同key对应值的最大值，然后相加

```scala
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("Exam14")
val sc = new SparkContext(sparkConf)
val inputRDD = sc.makeRDD(
    List(("a",3),("a",2),("c",4),("b",3),("c",6),("c",8))
)
inputRDD.aggregateByKey(0)(
  (tmp,item) => {
     println(tmp,item,"===")
     Math.max(tmp,item)
   },
   (tmp,result) => {
      println(tmp,result,"===")
      tmp + result
    }
).foreach(println(_))
```

15.创建一个有两个分区的 pairRDD数据为Array((“a”, 88), (“b”, 95), (“a”, 91), (“b”, 93), (“a”, 95), (“b”, 98))，根据 key 计算每种 key 的value的平均值

```scala
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("Exam15")
val sc = new SparkContext(sparkConf)
val inputRDD = sc.makeRDD(
	Array(("a", 88), ("b", 95), ("a", 91), ("b", 93), ("a", 95), ("b", 98)),2
)
inputRDD.groupByKey()
        .map(x => x._2.sum / x._2.size)
		.foreach(println)
```
