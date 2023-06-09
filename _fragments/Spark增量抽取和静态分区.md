---
layout: fragment
title: Spark增量抽取和静态分区
tags: [Spark]
description: some word here
keywords: Spark
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

```scala
package ETL

import java.util.Properties

import org.apache.spark.sql.SparkSession


/**
 * Project:  Bigdata
 * Create date:  2023/4/26
 * Created by fujiahao
 */
object SparkIncETL {
  def main(args: Array[String]): Unit = {
    val warehouse: String = "hdfs://master:9000/user/warehouse"
    val metastore: String = "thrift://master:9083"
    val mysql: String = "jdbc:mysql://master:3306/ds_pub?useUnicode=true&characterEncoding=utf-8"
    val properties: Properties = new Properties()
    properties.put("user", "root")
    properties.put("password", "123456")
    properties.put("driver", "com.mysql.jdbc.Driver")
    val sparkSession: SparkSession = SparkSession
      .builder()
      .config("spark.sql.warehouse.dir", warehouse)
      .config("hive.metastore.uris", metastore)
      .config("spark.sql.shuffle.partitions", 1000)
      .config("hive.exec.dynamic.partition", true)
      .config("hive.exec.dynamic.partition.mode", "nonstrict")
      .config("hive.exec.max.dynamic.partitions", 10000)
      .config("spark.sql.writeLegacyFormat", true)
      .appName("spark")
      .enableHiveSupport()
      .getOrCreate()
    /**
     * 1、抽取ds_pub库中order_info的增量数据进入Hive的ods库中表order_info，
     * 根据ods.order_info表中operate_time或create_time作为增量字段
     * (即MySQL中每条数据取这两个时间中较大的那个时间作为增量字段去和ods
     * 里的这两个字段中较大的时间进行比较)，只将新增的数据抽入，字段名称、
     * 类型不变，同时添加静态分区，分区字段类型为String，且值为当前比赛日
     * 的前一天日期（分区字段格式为yyyyMMdd）。
     * 使用hive cli执行show partitions ods.order_info命令，将结果截图粘贴至对应报告中；
     */

    val MaxOperateTime: Any = sparkSession.sql(
      s"""
         |select max(operate_time)
         |from ods.order_info
         |""".stripMargin).collect()(0).get(0)

    val MaxCreateTime: Any = sparkSession.sql(
      s"""
         |select max(create_time)
         |from ods.order_info
         |""".stripMargin).collect()(0).get(0)

    sparkSession.read.jdbc(mysql, "order_info", properties).createTempView("mysql_order_info")
    sparkSession.sql(
      s"""
         |select * from mysql_order_info
         |where create_time > '${MaxCreateTime}'
         |or operate_time > '${MaxOperateTime}'
         |""".stripMargin).createTempView("hive_order_info")

    sparkSession.sql(
      s"""
         |insert into table ods.order_info
         |partition(etldate=20230425)
         |select * from hive_order_info
         |""".stripMargin)
  }
}

```

