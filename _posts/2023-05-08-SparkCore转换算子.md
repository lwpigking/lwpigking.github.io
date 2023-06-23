---
layout: post
title: SparkCore转换算子
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

