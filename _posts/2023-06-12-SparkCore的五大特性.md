---
layout: post
title: SparkCore的五大特性
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

本文作者：张老师！！！！！！！！！！！

# RDD的五大特性

- **一系列的分区**
  每个RDD上面都包含一系列分区：每个分区代表一个Task任务，每个Task任务至少需要1个核心和1GB内存，所以Task越多计算能力越强，读取HDFS的时候一个Blokc块是一个分区（前提块足够大的时候）
- **computing方法**
  每个RDD上面都包含一个computing方法：帮你迭代数据，会从两个位置拿数据，要么cache缓存中的数据，要么就是计算的数据

**一系列的依赖关系**
RDD上面存在两种依赖关系：ShuffleDependency和NarrowDependency，遇到Shuffle就切分阶段，窄依赖都在一个数据管道中进行计算。其中窄依赖有两种 分别是OneToOneDependency和RangeDependency

- **一系列的分区器**
  在KV键值对类型的RDD上面都包含一个可选的分区器：分区器就是让上一个RDD中的数据怎么去往下一个RDD对应的分区中，要么就是有Shuffle类算子（GroupBy、GroupByKey、ReduceBy、ReduceByKey、SortBy、SortByKey等），要么就是人为自定义分区器UDPartitioner
- **优先位置计算**
  移动的计算要比移动数据快的多

# 自定义SparkContext工具类

```scala
package com.hk.utils

import org.apache.spark.{SparkConf, SparkContext}

/**
 * @FileName ContextUtils
 * @Description 用于获取SparkContext对象
 * @Author ChunYang Zhang
 * @Date 2023/4/23 21:48
 * */
object ContextUtils {
  def getCtx = {
    val conf = new SparkConf()
    conf.setAppName("test").setMaster("local[*]")
    val sc = new SparkContext(conf)
    sc
  }
}
```

# 自定义分区器

```scala
package com.hk.spark

import com.hk.utils.ContextUtils
import org.apache.spark.Partitioner

/**
 * @FileName UDPartitioner
 * @Description ${Description}
 * @Author ChunYang Zhang
 * @Date 2023/4/23 22:06
 * */
object UDPartitioner {
  def main(args: Array[String]): Unit = {
    val sc = ContextUtils.getCtx

    val rdd = sc.textFile("data/wordcount")
    rdd.flatMap(_.split(" "))
      .map((_, 1))
      .reduceByKey(_ + _)
      .partitionBy(new MyPartitioner)
      .saveAsTextFile("data/outPath/res02")
  }

}

class MyPartitioner extends Partitioner {
  override def numPartitions: Int = 2

  override def getPartition(key: Any): Int = {
    if (key.toString.contains("h")){
      0
    }else{
      1
    }
  }
}
```

