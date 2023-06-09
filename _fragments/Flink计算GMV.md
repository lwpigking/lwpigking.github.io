---
layout: fragment
title: Flink计算GMV
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

import org.apache.flink.streaming.api.scala._
import org.apache.flink.streaming.connectors.redis.RedisSink
import org.apache.flink.streaming.connectors.redis.common.config.FlinkJedisPoolConfig
import org.apache.flink.streaming.connectors.redis.common.mapper.{RedisCommand, RedisCommandDescription, RedisMapper}

import scala.collection.mutable.ListBuffer

/**
 * Project:  Bigdata
 * Create date:  2023/4/24
 * Created by fujiahao
 *
 */

/**
 * 需求：消费Kafka中topic为order的数据，进行
 * GMV指标计算（如果是同一条订单就只计算一次，
 * 如果有状态为退款的订单也不进行累加），最后将输出
 * 结果输出到Redis中，用Redis Cli查询（key设置
 * 为totalprice）。测试文件为moneyCml.txt。
 */

object FlinkCalGMVTest {
  def main(args: Array[String]): Unit = {
    // 创建环境
    val env: StreamExecutionEnvironment = StreamExecutionEnvironment.getExecutionEnvironment
    env.setParallelism(1)

    // 读取Kafka中topic为order的数据
    /*val properties: Properties = new Properties()
    properties.put("bootstrap.servers", "master:9092")

    val orderDS: DataStream[String] = env.addSource(new FlinkKafkaConsumer[String](
      "order",
      new SimpleStringSchema(),
      properties
    ))*/

    // 测试环境
    val orderDS: DataStream[String] = env.readTextFile("D:\\BigDataCode\\src\\main\\Data\\moneyCml.txt")

    /**
     * orderSnList模块和totalprice两者配合，功能如下：
     * 1.首先将所有的实收金额进行相加
     * 2.对orderSnList进行判断，如果orderSnList中没有order_sn，说明之前
     * 没有该订单号，那么就将该订单号加入到orderSnList；如果有order_sn，
     * 说明已经有该订单号的金额被统计相加了，那么就将总金额减去该条DS的实收金额
     * 3.如果有已退款的订单，那么就该减去该条DS的实收金额
     */
    val orderSnList: ListBuffer[String] = ListBuffer[String]()
    var totalprice: Double = 0
    // count用于验证数据条数
    var count = 0

    val Res: DataStream[(String, Double)] = orderDS.map(x => {
      val order_sn: String = x.split(", ")(1)
      val order_status: String = x.split(", ")(19)
      val order_money: Double = x.split(", ")(9).toDouble
      val name: String = x.split(", ")(3)
      totalprice += order_money

      if (!orderSnList.contains(order_sn)) {
        orderSnList.append(order_sn)
      } else {
        totalprice -= order_money
      }

      if (order_status == "已退款") {
        totalprice -= order_money
      }
      count += 1
      // 用于测试
      ("totalprice", totalprice, order_status, order_sn, name, order_money, count)
      // 返回数据(key,totalprice)
      ("totalprice", totalprice)
    })

    // Jedis配置
    val flinkJedisPoolConfig: FlinkJedisPoolConfig = new FlinkJedisPoolConfig.Builder().setHost("master").setPort(6379).setPassword("123123").build()
    // 写入Redis
    Res.addSink(new RedisSink[(String, Double)](
      flinkJedisPoolConfig,
      new MyRedisMap
    ))

    // 执行环境
    env.execute()

  }

  // 重写RedisMapper方法
  class MyRedisMap extends RedisMapper[(String, Double)] {
    override def getCommandDescription: RedisCommandDescription = {
      new RedisCommandDescription(RedisCommand.SET)
    }

    override def getKeyFromData(t: (String, Double)): String = {
      t._1
    }

    override def getValueFromData(t: (String, Double)): String = {
      t._2.toString
    }
  }

}
```

