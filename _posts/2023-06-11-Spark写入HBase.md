---
layout: post
title: Spark写入HBase
categories: [Spark, HBase]
description: Spark, HBase
keywords: Spark, HBase
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

import org.apache.hadoop.conf.Configuration
import org.apache.hadoop.hbase.HBaseConfiguration
import org.apache.hadoop.hbase.client.Put
import org.apache.hadoop.hbase.io.ImmutableBytesWritable
import org.apache.hadoop.hbase.mapred.TableOutputFormat
import org.apache.hadoop.hbase.util.Bytes
import org.apache.hadoop.mapred.JobConf
import org.apache.spark.rdd.RDD
import org.apache.spark.sql.{DataFrame, SparkSession}

/**
 * Project:  Bigdata
 * Create date:  2023/4/26
 * Created by fujiahao
 */
object SparkWriteHBase {
  def main(args: Array[String]): Unit = {
    val warehouse: String = "hdfs://master:9000/hive/warehouse"
    val metastore: String = "thrift://master:9083"
    val MySQLUrl: String = "jdbc:mysql://master:3306/ds_pub?useUnicode=true&characterEncoding=utf-8"
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

    val frame: DataFrame = sparkSession.sql(
      s"""
         |
         |""".stripMargin)

    var key = 0
    val config: Configuration = HBaseConfiguration.create()
    config.set("hbase.zookeeper.quorum", "master:2181,slave1:2181,slave2:2181")
    val jobConf: JobConf = new JobConf(config)
    jobConf.setOutputFormat(classOf[TableOutputFormat])
    jobConf.set(TableOutputFormat.OUTPUT_TABLE, "biao")

    val rdd: RDD[(ImmutableBytesWritable, Put)] = frame.rdd.map(col => {
      val col1: Int = col(0).asInstanceOf[Int]
      val put = new Put(Bytes.toBytes(key))
      key += 1
      put.addColumn(Bytes.toBytes("cf"), Bytes.toBytes("col1"), Bytes.toBytes(col1))
      (new ImmutableBytesWritable(), put)
    })

    rdd.saveAsHadoopDataset(jobConf)
  }
}

```

