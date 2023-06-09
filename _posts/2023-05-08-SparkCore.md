---
layout: post
title: SparkCore从入门到放弃~~
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

## SparkCore

### RDD简介

`RDD` 全称为 Resilient Distributed Datasets（弹性分布式数据集），是 Spark 最基本的数据处理模型，基于内存进行计算，它是只读的、分区记录的集合，支持并行操作，可以由外部数据集或其他 RDD 转换而来，它具有以下特性：

+ 一个 RDD 由一个或者多个分区（Partitions）组成。对于 RDD 来说，每个分区会被一个计算任务所处理，用户可以在创建 RDD 时指定其分区个数，如果没有指定，则默认采用程序所分配到的 CPU 的核心数；
+ RDD 拥有一个用于计算分区的函数 compute；
+ RDD 会保存彼此间的依赖关系，RDD 的每次转换都会生成一个新的依赖关系，这种 RDD 之间的依赖关系就像流水线一样。在部分分区数据丢失后，可以通过这种依赖关系重新计算丢失的分区数据，而不是对 RDD 的所有分区进行重新计算；
+ Key-Value 型的 RDD 还拥有 Partitioner(分区器)，用于决定数据被存储在哪个分区中，目前 Spark 中支持 HashPartitioner(按照哈希分区) 和 RangeParationer(按照范围进行分区)；
+ 一个优先位置列表 (可选)，用于存储每个分区的优先位置 (prefered location)。对于一个 HDFS 文件来说，这个列表保存的就是每个分区所在的块的位置，按照“移动数据不如移动计算“的理念，Spark 在进行任务调度的时候，会尽可能的将计算任务分配到其所要处理数据块的存储位置。

`RDD[T]` 抽象类的部分相关代码如下：

```scala
// 由子类实现以计算给定分区
def compute(split: Partition, context: TaskContext): Iterator[T]

// 获取所有分区
protected def getPartitions: Array[Partition]

// 获取所有依赖关系
protected def getDependencies: Seq[Dependency[_]] = deps

// 获取优先位置列表
protected def getPreferredLocations(split: Partition): Seq[String] = Nil

// 分区器 由子类重写以指定它们的分区方式
@transient val partitioner: Option[Partitioner] = None
```

### RDD算子

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

#### RDD_Transformation(转换算子)

转换算子：功能的补充和封装（把旧的RDD转换成新的RDD的操作）

##### coalesce

```scala
// coalesce是将RDD中的分区数量减少到numpartition
// 大数据集过滤后，可以将过滤的数据弄到一个分区，减少资源调度

// coalesce方法默认情况下不会将分区的数据打乱重新组合
// 这种情况下的缩减分区可能会导致数据不均衡，出现数据倾斜
// 如果想要让数据均衡，可以进行shuffle处理
// coalesce第二个参数shuffle：true和false

val sparkConf = new SparkConf().setMaster("local[*]").setAppName("transformation")
val sc = new SparkContext(sparkConf)
val rdd = sc.makeRDD (
    List(1, 2, 3, 4)
)
println("当前分区数："+rdd.partitions.size)
println("====================")
val rdd2 = rdd.coalesce(1)
println("合并分区后，现在的分区数："+rdd2.partitions.size)

sc.stop()
// 返回结果：
// 当前分区数：8
// ====================
// 合并分区后，现在的分区数：1
```

##### distinct

```scala
// distinct 去重
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("transformation")
val sc = new SparkContext(sparkConf)
val rdd = sc.makeRDD(
    List(1, 2, 2, 4, 4, 5, 5, 3, 9, 10)
)
rdd.distinct().collect().foreach(println)
// 返回结果：1 9 10 2 3 4 5
sc.stop()
```

##### filter

```scala
// filter 过滤掉不符合规则的元素
// 当数据进行筛选过滤后，分区不变，但是分区内的数据可能不均衡，生产环境下可能会出现数据倾斜。
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("transformation")
val sc = new SparkContext(sparkConf)
val rdd = sc.makeRDD(
    List(1, 2, 3, 4)
)
rdd.filter(
    _ != 3
).collect().foreach(println)
// 返回结果：1 2 4
sc.stop()
```

```scala
// 从服务器日志数据apache.log中获取2015年5月17日的请求路径
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("transformation")
val sc = new SparkContext(sparkConf)
val rdd = sc.textFile("apache.log")

rdd.filter(
	line => {
        val datas = line.split(" ")
        val time = datas(3)
        time.startsWith("17/05/2015")
    }
).collect().foreach(println)

sc.stop()
```

##### flatMap 

```scala
// 扁平映射
// flatMap会先执行map的操作，再将所有对象合并为一个对象，返回值是一个Sequence
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("flatMap")
val sc = new SparkContext(sparkConf)
val rdd = sc.makeRDD(
    List(1, 2, 3, 4)
)
rdd.flatMap(data => data).collect().foreach(println)
// 返回结果：1 2 3 4
sc.stop()
```

```scala
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("flatMap")
val sc = new SparkContext(sparkConf)
val rdd = sc.makeRDD(List(
    List(1, 2), List(3,4)
))

val flatRDD = rdd.flatMap(
	list => {
        list
    }
)

flatRDD.collect().foreach(println)

sc.stop()
```

```scala
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("flatMap")
val sc = new SparkContext(sparkConf)
val rdd = sc.makeRDD(List(
    "Hello Scala", "Hello Spark"
))

val flatRDD = rdd.flatMap(
	s => {
        s.split(" ")
    }
)

flatRDD.collect().foreach(println)

sc.stop()
```

```scala
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("flatMap")
val sc = new SparkContext(sparkConf)
val rdd = sc.makeRDD(List(List(1,2), 3, List(4,5)))

val flatRDD = rdd.flatMap(
	data => {
        data match {
            case list:List[_] => list
            case dat => List(dat)
        }
    }
)

flatRDD.collect().foreach(println)

sc.stop()
```



##### map

```scala
// map是对数据逐条进行操作（映射转换），这里的转换可以是类型的转换，也可以是值的转换。
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("map")
val sc = new SparkContext(sparkConf)

val rdd = sc.makeRDD(
    List(1, 2, 3, 4)
)

val mapRDD = rdd.map(
    _ + 1
)

mapRDD.collect().foreach(println)
// 返回结果：2 3 4 5

sc.stop()
```

```scala
// 小功能：从服务器日志数据apache.log中获取用户请求URL资源路径
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("map")
val sc = new SparkContext(sparkConf)

val rdd = sc.textFile("apache.log")

val mapRDD = rdd.map(
	line => {
        val datas = line.split(" ")
        datas(6)
    }
)

mapRDD.collect().foreach(println)

sc.stop()
```

```scala
// 并行计算演示
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("map")
val sc = new SparkContext(sparkConf)
val rdd = sc.makeRDD(List(1,2,3,4),1)
val mapRDD = rdd.map(
	num => {
        println(">>>>>>" + num)
        num
    }
)

val mapRDD1 = mapRDD.map(
	num => {
        println("######" + num)
        num
    }
)

mapRDD1.collect()

sc.stop()
```

##### map与flatmap 

```scala
// map操作后会返回到原来的集合中
// flatmap操作后会返回一个新的集合
// map与flatmap一般一起使用**
// 合并使用：
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("flatMap")
val sc = new SparkContext(sparkConf)
val rdd = sc.makeRDD(
    List(List(1,2),List(3,4)),1
)
rdd.flatMap(data =>{
    data.map(_*3)
}).collect().foreach(println)

sc.stop()
// 返回结果3 6 9 12
// 先进行map操作，将数据逐条*3，然后放回到data中
// flatmap再对data进行扁平映射，将list里面的数据逐条拿出来
```

##### glom 

```scala
// glom将同一个分区中的所有元素合并成一个数组并返回
// 分区1:1,2 glom => List(1,2)
// 分区2:3,4 glom => List(3,4)
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("transformation")
val sc = new SparkContext(sparkConf)
val rdd = sc.makeRDD (
    List(1, 2, 3, 4),2
)
rdd.glom().collect().foreach(println)
println("=========================")

//分区求和
rdd.glom().map(_.sum).collect().foreach(println)

sc.stop()
```

```scala
// 分区内取最大值，分区间最大值求和
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("transformation")
val sc = new SparkContext(sparkConf)
val rdd = sc.makeRDD (
    List(1, 2, 3, 4),2
)

val glomRDD = rdd.glom()

val maxRDD = glomRDD.map(
	array => {
        array.max
    }
)

println(maxRDD.collect().sum())

sc.stop()
```

##### groupby 

```scala
// 进行分组，而分区不变,数据被重新打乱，这个过程叫做shuffle
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("transformation")
val sc = new SparkContext(sparkConf)
val rdd = sc.makeRDD (
    List(1, 2, 3, 4),2
)
rdd.groupBy(_%2==0).collect().foreach(println)
// 返回结果：
//(false,CompactBuffer(1, 3))
//(true,CompactBuffer(2, 4))

sc.stop()
```

```scala
// 从服务器日志数据apache.log中获取每个时间段访问量
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("transformation")
val sc = new SparkContext(sparkConf)

val rdd = sc.textFile("apache.log")

val timeRDD = rdd.map(
	line => {
        val datas = line.split(" ")
        val time = datas(3)
        val sdf = new SimpleDateFormat("dd/MM/yyyy:HH:mm:ss")
        val date = sdf.parse(time)
        val sdf1 = new SimpleDateFormat("HH")
        val hour = sdf1.formate(date)
        (hour, 1)
    }
).groupBy(_._1)

timeRDD.map{
    case ( hour, iter ) => {
        (hour, iter.size)
    }
}.collect.foreach(println)

sc.stop()
```



##### mapPartitions

```scala
// 以分区为单位进行操作
// 各个分区进行自我操作
// 但是会将整个分区的数据加载到内存进行引用
// 如果处理完的数据是不会被释放的，存在对象引用
// 在内存较小，数据量较大的场合下，容易出现内存溢出
// 传入迭代器，返回一个迭代器
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("mapPartitions")
val sc = new SparkContext(sparkConf)
val rdd = sc.makeRDD(
    Array(1,2,3,4,5,6,7,8,9,10),2
)
rdd.mapPartitions(
    iter=>Iterator(iter.toArray)
).collect.foreach(item => println(item.toList))

sc.stop()
// 返回结果
// List(1, 2, 3, 4, 5)
// List(6, 7, 8, 9, 10)
```

```scala
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("mapPartitions")
val sc = new SparkContext(sparkConf)

val rdd = sc.makeRDD(List(1,2,3,4),2)

// 把一个分区的数据都拿到了以后做操作
val mpRDD = rdd.mapPartitions(
	iter => {
        println(">>>>>>>")
        iter.map(_*2)
    }
)

mpRDD.collect().foreach(println)

sc.stop()
```

```scala
// 获取每个分区数据的最大值
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("mapPartitions")
val sc = new SparkContext(sparkConf)

val rdd = sc.makeRDD(List(1,2,3,4),2)

val mapRDD = rdd.mapPartitions(
	iter => {
        List(iter.max).iterator
    }
)

mapRDD.collect().foreach(println)

sc.stop()
```

##### mapPartitionsWithIndex

```scala
/**
 * 通过对这个RDD的每个分区应用一个函数来返回一个新的RDD，
 * 同时跟踪原始分区的索引。
 * mapPartitionsWithIndex类似于mapPartitions()，
 * 但它提供了第二个参数索引，用于跟踪分区。
 */
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("mapPartitionsWithIndex")
val sc = new SparkContext(sparkConf)
val rdd = sc.makeRDD(
    List(1, 2, 3, 4, 5, 6, 7, 8, 9, 10), 2
)
def f(partitionIndex:Int, i:Iterator[Int])= {
    (partitionIndex, i.sum).productIterator
}
rdd.mapPartitionsWithIndex(f).collect().foreach(println)

sc.stop()
// 返回结果:0 15 1 40
```

```scala
// 获取第二个分区的数据
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("mapPartitionsWithIndex")
val sc = new SparkContext(sparkConf)
val rdd = sc.makeRDD(List(1,2,3,4),2)

val mapiRDD = rdd.mapPartitionsWithIndex(
	(index,iter) => {
        if( index == 1) {
            iter
        }else{
            Nil.iterator
        }
    }
)

mapiRDD.collect().foreach(println)

sc.stop()
```

```scala
// 查看数据所在分区
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("mapPartitionsWithIndex")
val sc = new SparkContext(sparkConf)
val rdd = sc.makeRDD(List(1,2,3,4),2)

val mapiRDD = rdd.mapPartitionsWithIndex(
	(index,iter) => {
        iter.map(
            num => {
                (index, num)
            }
      	)
    }
)

mapiRDD.collect().foreach(println)

sc.stop()
```

##### repartition

```scala
// repartition()用于增加或减少RDD分区
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("transformation")
val sc = new SparkContext(sparkConf)
val rdd = sc.makeRDD (
    List(1, 2, 3, 4)
)
println("当前分区数："+rdd.partitions.size)
println("=====================")
//得先repartition
val rdd2 = rdd.repartition(4)
println("重分区后，现在的分区数："+rdd2.partitions.size)

sc.stop()
// 返回结果
// 当前分区数：8
// =====================
// 重分区后，现在的分区数：4
```

##### sample

```scala
//从数据中抽取数据
/**
 * 第一个参数：抽取的数据是否放回，false：不放回
 * 第二个参数：抽取的几率，范围在[0,1]之间,0：全不取；1：全取；
 * 第三个参数：随机数种子
 */
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("map")
val sc = new SparkContext(sparkConf)
val rdd = sc.makeRDD(
    List(1, 2, 3, 4)
)
val rdd2 = rdd.sample(false, 0.5)
rdd2.collect().foreach(println)
sc.stop()
// 返回结果 1 3
```

##### sortBy 

```scala
/**
 * 排序数据。（排序大小）
 * 在排序之前，可以将数据通过 f 函数进行处理，之后按照 f 函数处理
 * 的结果进行排序
 */
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("transformation")
val sc = new SparkContext(sparkConf)
val rdd = sc.makeRDD (
    List(1, 2, 3, 4,5,9,6,46)
)
//False为降序
rdd.sortBy(x => x,false).collect().foreach(print)
println("========================")
//默认升序
rdd.sortBy(x => x).collect().foreach(print)
sc.stop()
// 返回结果
// 46 9 6 5 4 3 2 1
// ========================
// 1 2 3 4 5 6 9 46
```

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

16.统计出每一个省份每个广告被点击数量排行的Top3

```scala
val sparkConf = new SparkConf().setMaster("local[*]").setAppName("Exam16")
val sc = new SparkContext(sparkConf)

// 1.获取原始数据：时间戳，省份，城市，用户，广告
val dataRDD = sc.textFile("agent.log")

// 2.将原始数据进行结构转换，方便统计
// 时间戳，省份，城市，用户，广告 -> ( (省份, 广告), 1 )
val mapRDD = dataRDD.map(
	line => {
        val datas = line.split(" ")
        (( datas(1), datas(4) ), 1)
    }
)

// 3.将转换结构后的数据进行分组聚合
// ( (省份, 广告), 1 ) -> ( (省份, 广告), sum )
val reduceRDD = mapRDD.reduceByKey(_+_)

// 4.将聚合的结果进行结构的转换
// ( (省份, 广告), sum ) -> ( 省份, (广告, sum) )
val newMapRDD = reduceRDD.map{
    case( (prv, ad), sum ) => {
        (prv, (ad, sum))
    }
}

// 5.将转换结构后的数据根据省份进行分组
val groupRDD = newMapRDD.groupByKey()

// 6.将分组后的数据组内排序，取前3名
val resultRDD = groupRDD.mapValues(
	iter => {
        iter.toList.sortBy(_._2)(Ordering.Int.reverse).take(3)
    }
)

resultRDD.collect().foreach(println)
sc.stop()
```

### 缓存RDD

#### 缓存级别

Spark 速度非常快的一个原因是 RDD 支持缓存。成功缓存后，如果之后的操作使用到了该数据集，则直接从缓存中获取。虽然缓存也有丢失的风险，但是由于 RDD 之间的依赖关系，如果某个分区的缓存数据丢失，只需要重新计算该分区即可。

Spark 支持多种缓存级别 ：

| Storage Level<br/>（存储级别）                 | Meaning（含义）                                              |
| ---------------------------------------------- | ------------------------------------------------------------ |
| `MEMORY_ONLY`                                  | 默认的缓存级别，将 RDD 以反序列化的 Java 对象的形式存储在 JVM 中。如果内存空间不够，则部分分区数据将不再缓存。 |
| `MEMORY_AND_DISK`                              | 将 RDD 以反序列化的 Java 对象的形式存储 JVM 中。如果内存空间不够，将未缓存的分区数据存储到磁盘，在需要使用这些分区时从磁盘读取。 |
| `MEMORY_ONLY_SER`<br/>                         | 将 RDD 以序列化的 Java 对象的形式进行存储（每个分区为一个 byte 数组）。这种方式比反序列化对象节省存储空间，但在读取时会增加 CPU 的计算负担。仅支持 Java 和 Scala 。 |
| `MEMORY_AND_DISK_SER`<br/>                     | 类似于 `MEMORY_ONLY_SER`，但是溢出的分区数据会存储到磁盘，而不是在用到它们时重新计算。仅支持 Java 和 Scala。 |
| `DISK_ONLY`                                    | 只在磁盘上缓存 RDD                                           |
| `MEMORY_ONLY_2`, <br/>`MEMORY_AND_DISK_2`, etc | 与上面的对应级别功能相同，但是会为每个分区在集群中的两个节点上建立副本。 |
| `OFF_HEAP`                                     | 与 `MEMORY_ONLY_SER` 类似，但将数据存储在堆外内存中。这需要启用堆外内存。 |

> 启动堆外内存需要配置两个参数：
>
> + **spark.memory.offHeap.enabled** ：是否开启堆外内存，默认值为 false，需要设置为 true；
> + **spark.memory.offHeap.size** : 堆外内存空间的大小，默认值为 0，需要设置为正值。

#### 使用缓存

缓存数据的方法有两个：`persist` 和 `cache` 。`cache` 内部调用的也是 `persist`，它是 `persist` 的特殊化形式，等价于 `persist(StorageLevel.MEMORY_ONLY)`。示例如下：

```scala
// 所有存储级别均定义在 StorageLevel 对象中
fileRDD.persist(StorageLevel.MEMORY_AND_DISK)
fileRDD.cache()
```

#### 移除缓存

Spark 会自动监视每个节点上的缓存使用情况，并按照最近最少使用（LRU）的规则删除旧数据分区。当然，你也可以使用 `RDD.unpersist()` 方法进行手动删除。



### 理解shuffle

#### shuffle介绍

在 Spark 中，一个任务对应一个分区，通常不会跨分区操作数据。但如果遇到 `reduceByKey` 等操作，Spark 必须从所有分区读取数据，并查找所有键的所有值，然后汇总在一起以计算每个键的最终结果 ，这称为 `Shuffle`。

![spark-reducebykey](/picture/pictures/spark-reducebykey.png)


#### Shuffle的影响

Shuffle 是一项昂贵的操作，因为它通常会跨节点操作数据，这会涉及磁盘 I/O，网络 I/O，和数据序列化。某些 Shuffle 操作还会消耗大量的堆内存，因为它们使用堆内存来临时存储需要网络传输的数据。Shuffle 还会在磁盘上生成大量中间文件，从 Spark 1.3 开始，这些文件将被保留，直到相应的 RDD 不再使用并进行垃圾回收，这样做是为了避免在计算时重复创建 Shuffle 文件。如果应用程序长期保留对这些 RDD 的引用，则垃圾回收可能在很长一段时间后才会发生，这意味着长时间运行的 Spark 作业可能会占用大量磁盘空间，通常可以使用 `spark.local.dir` 参数来指定这些临时文件的存储目录。

#### 导致Shuffle的操作

由于 Shuffle 操作对性能的影响比较大，所以需要特别注意使用，以下操作都会导致 Shuffle：

+ **涉及到重新分区操作**： 如 `repartition` 和 `coalesce`；
+ **所有涉及到 ByKey 的操作**：如 `groupByKey` 和 `reduceByKey`，但 `countByKey` 除外；
+ **联结操作**：如 `cogroup` 和 `join`。



### 宽依赖和窄依赖

RDD 和它的父 RDD(s) 之间的依赖关系分为两种不同的类型：

- **窄依赖 (narrow dependency)**：父 RDDs 的一个分区最多被子 RDDs 一个分区所依赖；
- **宽依赖 (wide dependency)**：父 RDDs 的一个分区可以被子 RDDs 的多个子分区所依赖。

如下图，每一个方框表示一个 RDD，带有颜色的矩形表示分区：

![spark-窄依赖和宽依赖](/picture/pictures/spark-窄依赖和宽依赖.png)


区分这两种依赖是非常有用的：

+ 首先，窄依赖允许在一个集群节点上以流水线的方式（pipeline）对父分区数据进行计算，例如先执行 map 操作，然后执行 filter 操作。而宽依赖则需要计算好所有父分区的数据，然后再在节点之间进行 Shuffle，这与 MapReduce 类似。
+ 窄依赖能够更有效地进行数据恢复，因为只需重新对丢失分区的父分区进行计算，且不同节点之间可以并行计算；而对于宽依赖而言，如果数据丢失，则需要对所有父分区数据进行计算并再次 Shuffle。



### DAG的生成

RDD(s) 及其之间的依赖关系组成了 DAG(有向无环图)，DAG 定义了这些 RDD(s) 之间的 Lineage(血统) 关系，通过血统关系，如果一个 RDD 的部分或者全部计算结果丢失了，也可以重新进行计算。那么 Spark 是如何根据 DAG 来生成计算任务呢？主要是根据依赖关系的不同将 DAG 划分为不同的计算阶段 (Stage)：

+ 对于窄依赖，由于分区的依赖关系是确定的，其转换操作可以在同一个线程执行，所以可以划分到同一个执行阶段；
+ 对于宽依赖，由于 Shuffle 的存在，只能在父 RDD(s) 被 Shuffle 处理完成后，才能开始接下来的计算，因此遇到宽依赖就需要重新划分阶段。

![spark-DAG](/picture/pictures/spark-DAG.png)

