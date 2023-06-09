---
layout: fragment
title: Flink水位线计算GMV
tags: [Flink]
description: some word here
keywords: Flink, GMV
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

```scala
package Calculation

import java.time.Duration
import java.util.Properties

import org.apache.flink.api.common.eventtime.{SerializableTimestampAssigner, WatermarkStrategy}
import org.apache.flink.api.common.serialization.SimpleStringSchema
import org.apache.flink.streaming.api.scala._
import org.apache.flink.streaming.connectors.kafka.FlinkKafkaConsumer
import org.apache.flink.streaming.connectors.redis.RedisSink
import org.apache.flink.streaming.connectors.redis.common.config.FlinkJedisPoolConfig
import org.apache.flink.streaming.connectors.redis.common.mapper.{RedisCommand, RedisCommandDescription, RedisMapper}

import scala.collection.mutable.ListBuffer


/**
 * 该Scala代码为所选测试数据为moneyCml.txt
 */

case class OrderEvent(id:Long,
                      consignee:String,
                      consignee_tel: String,
                      final_total_amount: Double,
                      order_status: String,
                      user_id: Long,
                      delivery_address: String,
                      order_comment: String,
                      out_trade_no: String,
                      trade_body: String,
                      create_time: Long,
                      operate_time: Long,
                      province_id: Int,
                      feight_fee: Double)
object FlinkCalGMVTesta {
  def main(args: Array[String]): Unit = {
    // 创建环境
    val env: StreamExecutionEnvironment = StreamExecutionEnvironment.getExecutionEnvironment
    env.setParallelism(1)

    val watermark: WatermarkStrategy[OrderEvent] = WatermarkStrategy.forBoundedOutOfOrderness[OrderEvent](Duration.ofMinutes(2))
      .withTimestampAssigner(new SerializableTimestampAssigner[OrderEvent] {
        override def extractTimestamp(element: OrderEvent, recordTimestamp: Long): Long = scala.math.max(element.operate_time, element.create_time)
      })

    val properties: Properties = new Properties()
    properties.put("bootstrap.servers", "master:9092")
    val orderDS: DataStream[String] = env.addSource(new FlinkKafkaConsumer[String](
      "order",
      new SimpleStringSchema(),
      properties
    ))

    val CustomerList: ListBuffer[String] = ListBuffer[String]()
    var totalprice: Double = 0

    val GMV: DataStream[(String, Long)] = orderDS.filter(x => {
      x.split(":")(0).equals("order_info")
    }).map(data => {
      val id: String = data.split(":")(0)
      val consignee: String = data.split(":")(1)
      val consignee_tel: String = data.split(":")(2)
      val final_total_amount: String = data.split(":")(3)
      val order_status: String = data.split(":")(4)
      val user_id: String = data.split(":")(5)
      val delivery_address: String = data.split(":")(6)
      val order_comment: String = data.split(":")(7)
      val out_trade_no: String = data.split(":")(8)
      val trade_body: String = data.split(":")(9)
      val create_time: String = data.split(":")(10)
      val operate_time: String = data.split(":")(11)
      val province_id: String = data.split(":")(12)
      val feight_fee: String = data.split(":")(13)
      OrderEvent(id.toLong, consignee, consignee_tel, final_total_amount.toDouble, order_status, user_id.toLong, delivery_address, order_comment, out_trade_no, trade_body, create_time.toLong, operate_time.toLong, province_id.toInt, feight_fee.toLong)
    }).assignTimestampsAndWatermarks(watermark)
      .map(x => {
        totalprice += x.final_total_amount
        if (!CustomerList.contains(x.user_id.toString)) {
          CustomerList.append(x.user_id.toString)
        } else {
          totalprice -= x.final_total_amount
        }

        if (x.order_status == "已退款") {
          totalprice -= x.final_total_amount
        }
        ("totalprice", totalprice.toLong)
      })

    val flinkJedisPoolConfig: FlinkJedisPoolConfig = new FlinkJedisPoolConfig.Builder().setHost("master").setPort(6379).setPassword("123123").build()

    GMV.addSink(new RedisSink[(String, Long)](
      flinkJedisPoolConfig,
      new RedisMap
    ))
  }

  class RedisMap extends RedisMapper[(String, Long)] {
    override def getCommandDescription: RedisCommandDescription = {
      new RedisCommandDescription(RedisCommand.SET)
    }

    override def getKeyFromData(t: (String, Long)): String = {
      t._1
    }

    override def getValueFromData(t: (String, Long)): String = {
      t._2.toString
    }
  }
}
```

