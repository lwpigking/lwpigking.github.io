---
layout: post
title: SparkCore的Shuffle和二次排序
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

本文作者：张老师！！！！！！！！！

# Shuffle

## HashShuffleManager

   对spark任务划分阶段，遇到宽依赖会断开，所以在stage 与 stage 之间会产生shuffle，大多数Spark作业的性能主要就是消耗在了shuffle环节，因为该环节包含了大量的磁盘IO、序列化、网络数据传输等操作。
​​ ​ ​ 负责shuffle过程的执行、计算和处理的组件主要就是ShuffleManager，也即shuffle管理器。而随着Spark的版本的发展，ShuffleManager也在不断迭代。
​ ​ ​ ​ ShuffleManager 大概有两个： HashShuffleManager 和 SortShuffleManager。
历史：
在spark 1.2以前，默认的shuffle计算引擎是HashShuffleManager；
在spark 1.2以后的版本中，默认的ShuffleManager改成了SortShuffleManager；
在spark 2.0以后，抛弃了 HashShuffleManager。
HashShuffleManager

假设：每个Executor只有1个CPU core，也就是说，无论这个Executor上分配多少个task线程，同一时间都只能执行一个task线程。
 上游 stage 有 2个 Executor，每个Executor 有 2 个 task。
 下游 stage 有 3个task。
shuffle write阶段：
 将相当于mapreduce的shuffle write，按照key的hash 分桶，写出中间文件。上游的每个task写自己的文件。
 写出中间文件个数 = maptask的个数 *reducetask的个数
 上图写出的中间文件个数 = 4* 3 = 12
 假设上游 stage 有 10 个Executor，每个 Executor有 5 个task，下游stage 有 4 个task，写出的中间文件数 = (10 *5)* 4 = 200 个，由此可见，shuffle write操作所产生的磁盘文件的数量是极其惊人的。
shuffle read 阶段：
 就相当于mapreduce 的 shuffle read， 每个reducetask 拉取自己的数据。
 由于shuffle write的过程中，task给下游stage的每个task都创建了一个磁盘文件，因此shuffle read的过程中，每个task只要从上游stage的所有task所在节点上，拉取属于自己的那一个磁盘文件即可。
弊端：
 shuffle write阶段占用大量的内存空间，会导致频繁的GC，容易导致OOM；也会产生大量的小文件，写入过程中会产生大量的磁盘IO，性能受到影响。适合小数据集的处理。

## HashShuffleManager 优化

开启consolidate机制。
设置参数：spark.shuffle.consolidateFiles。该参数默认值为false，将其设置为true即可开启优化机制。

假设：每个Executor只有1个CPU core，也就是说，无论这个Executor上分配多少个task线程，同一时间都只能执行一个task线程。
​ 上游 stage 有 2个 Executor，每个Executor 有 2 个 task，每个Executor只有1个CPU core。
​ 下游 stage 有 3个task。
shuffle write阶段：
​ 开启consolidate机制后，允许上游的多个task写入同一个文件，这样就可以有效将多个task的磁盘文件进行一定程度上的合并，从而大幅度减少磁盘文件的数量，进而提升shuffle write的性能。
​ 写出中间文件个数 = 上游的CPU核数 *下游task的个数
​ 上图写出的中间文件个数 = 2* 3 = 6
​ 假设上游 stage 有 10 个Executor，每个Executor只有1个CPU core，每个 Executor有 5 个task，下游stage 有 4 个task，写出的中间文件数 = 10 * 4 = 40个
shuffle read 阶段：
​ 就相当于mapreduce 的 shuffle read， 每个reducetask 拉取自己的数据。
​ 由每个reducetask只要从上游stage的所在节点上，拉取属于自己的那一个磁盘文件即可。
弊端：
​ 优化后的HashShuffleManager，虽然比优化前减少了很多小文件，但在处理大量数据时，还是会产生很多的小文件。

## SortShuffleManager

 Spark在引入Sort-Based Shuffle以前，比较适用于中小规模的大数据处理。为了让Spark在更大规模的集群上更高性能处理更大规模的数据，于是就引入了SortShuffleManager。
shuffle write 阶段：
​ SortShuffleManager不会为每个Reducer中的Task生成一个单独的文件，相反，会把上游中每个mapTask所有的输出数据Data只写到一个文件中。并使用了Index文件存储具体 mapTask 输出数据在该文件的位置。
​ 因此 上游 中的每一个mapTask中产生两个文件：Data文件 和 Index 文件，其中Data文件是存储当前Task的Shuffle输出的，而Index文件中存储了data文件中的数据通过partitioner的分类索引。
​ 写出文件数 = maptask的个数 * 2 （index 和 data ）
​ 可见，SortShuffle 的产生的中间文件的多少与 上个stage 的 maptask 数量有关。
shuffle read 阶段：
下游的Stage中的Task就是根据这个Index文件获取自己所要抓取的上游Stage中的mapShuffleMapTask产生的数据的；

```scala
package com.hk.spark

import com.hk.utils.HDFSUtils
import org.apache.spark.{SparkConf, SparkContext}

/**
 * @FileName ShuffleManagerDemo
 * @Description 测试shuffle
 * @Author ChunYang Zhang
 * @Date 2023/4/23 23:48
 * */
object ShuffleManagerDemo {

  implicit def str2HDFSUtil(s: String) = new HDFSUtils(s)

  def main(args: Array[String]): Unit = {
    // 初始化配置
    val conf = new SparkConf()
    conf.setMaster("local[*]")
    conf.setAppName("Word Count")
    conf.set("spark.shuffle.manager","hash")
    // 未优化
    conf.set("spark.shuffle.consolidateFiles","false")
    // 优化后
    // conf.set("spark.shuffle.consolidateFiles","true")

    // 初始化SparkContext对象
    val sc = new SparkContext(conf)

    // 读取文件
    val value = sc.textFile("data/wordcount")
      // 切割
      .flatMap(_.split(" "))
      // 转换 把单词变成 单词和1
      .map(t => {
        Thread.sleep(100000000)
        (t, 1)
      }
      )
      // 按照单词进行分组
      .groupBy(_._1)
      // 聚合
      .mapValues(_.map(_._2).sum)

    // 删除已经存在的输出路径
    "data/outPath".delete
    // 将计算好的数据保存到文件中
    value.saveAsTextFile("data/outPath")
  }
}
```

# 二次排序

```scala
package com.hk.spark

import com.hk.utils.ContextUtils

/**
 * @FileName UDOrdered
 * @Description ${Description}
 * @Author ChunYang Zhang
 * @Date 2023/4/23 22:52
 * */
object UDOrdered {
  def main(args: Array[String]): Unit = {
    val sc = ContextUtils.getCtx
    sc.textFile("data/b.txt").map(t => {
      val strs = t.split(" ")
      new Student(strs(0),strs(1).toInt,strs(2).toInt)
    }).sortBy(t=>t).foreach(print)
  }
}

case class Student(val name: String, val age: Int, val fv: Int) extends Ordered[Student] {
  override def compare(that: Student): Int = {

    if (this.age == that.age) {
      that.fv - this.fv
    } else {
      this.age - that.age
    }
  }

  override def toString = s"Student($name, $age, $fv)"
}
```
