---
layout: fragment
title: Spark全量抽取和动态分区
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
object SparkDynamicETL {
  def main(args: Array[String]): Unit = {
    val warehouse: String = "hdfs://master:9000/hive/warehouse"
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
     * 3、抽取ds_pub库中user_info的全量数据进入Hive的ods库中表user_info。
     * 字段名称、类型不变，同时添加动态分区，分区字段类型为String，
     * 值为birthday列数据，格式为（yyyyMM）。
     * 使用hive cli执行select count(distinct(etldate)) from  ods.user_info命令，
     * 将结果截图粘贴至对应报告中；
     */

    sparkSession.read.jdbc(mysql, "user_info", properties).createTempView("mysql_user_info")
    sparkSession.sql(
      s"""
         |from mysql_user_info
         |select *,
         |date_format(birthday, 'yyyyMM') as etldate
         |""".stripMargin).createTempView("hive_user_info")

    sparkSession.sql(
      s"""
         |insert overwrite table ods.user_info
         |partition(etldate)
         |select * from hive_user_info
         |""".stripMargin)

  }
}

```

