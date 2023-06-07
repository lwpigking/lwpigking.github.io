---
layout: fragment
title: Flink计算商品PV和UV
tags: [Flink]
description: 了解一下PV和UV 
keywords: Flink, PV, UV, 商品
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

```scala
package Calculation

import org.apache.flink.streaming.api.functions.co.CoMapFunction
import org.apache.flink.streaming.api.scala._

import scala.collection.mutable.ListBuffer

/**
 * Project:  BigdataCore
 * Create date:  2023/4/24
 * Created by fujiahao
 */

/**
 * 需求：1.计算商品的PV值
 *      2.计算商品的UV值
 * pv:每个product_id被不同的customer_id所点击的总次数
 * uv:每个product_id一共被多少个customer_id所点击
 * 该Scala代码为测试，所选数据为fakePvandUv.txt
 */

object FlinkPvAndUv {
  def main(args: Array[String]): Unit = {
    // 创建环境
    val env: StreamExecutionEnvironment = StreamExecutionEnvironment.getExecutionEnvironment
    env.setParallelism(1)

    // 输入测试数据
    val FileDS: DataStream[String] = env.readTextFile("D:\\BigDataCode\\Code\\Data\\fakePvandUv.txt")

    // 字段与字段值如下
    /**
     * log_id, product_id, customer_id, gen_order, order_sn, modified_time
     * 1,      752,        9097,        0,         0,        2022-03-15 00:54:50
     * 2,      12218,      9097,        0,         0,        2022-03-15 00:24:53
     * 3,      10756,      9097,        0,         0,        2022-03-15 14:26:55
     * 4,      14307,      18577,       1, 2022031625520775, 2022-03-15 04:59:50
     * 5,      8047,       18577,       0,         0,        2022-03-15 16:38:53
     * 6,      5775,       18577,       0,         0,        2022-03-15 00:16:56
     */

    /**
     * productIdList功能：计算PV值，当一条DS过来后增加到该List中
     * 最后该商品的PV结果用count函数和匿名函数进行判断
     */
    val productIdList: ListBuffer[String] = ListBuffer[String]()

    // 计算UV值，对ProductID进行keyby聚合，再进行Agg相加
    val uv: DataStream[(String, Int)] = FileDS.map(x => {
      val product_id: String = x.split(", ")(1)
      (product_id, 1)
    }).keyBy(0)
      .sum(1)

    // 计算PV值
    val pv: DataStream[(String, Int)] = FileDS.map(x => {
      val product_id: String = x.split(", ")(1)
      productIdList.append(product_id)
      val PvCount: Int = productIdList.count(_ == product_id)
      (product_id, PvCount)
    })

    // 双流合并，继承CoMapFunction，输出合并流
    pv.connect(uv)
      .map(new CoMapFunction[(String, Int), (String, Int), String] {
        override def map1(in1: (String, Int)): String = {
          (s"商品${in1._1}的pv值是${in1._2}")
        }

        override def map2(in2: (String, Int)): String = {
          (s"商品${in2._1}的uv值是${in2._2}")
        }
      }).print()

    // 执行环境
    env.execute()
  }
}
```

