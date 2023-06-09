---
layout: fragment
title: Spark数据合并
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

import org.apache.spark.sql._
import org.apache.spark.sql.expressions.{Window, WindowSpec}
import org.apache.spark.sql.functions._

/**
 * Project:  Bigdata
 * Create date:  2023/4/26
 * Created by fujiahao
 */
object SparkCleanMerge {
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
      .config("spark.sql.warehouse,dir", warehouse)
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
     * 1、抽取ods库中user_info表中昨天的分区（任务一生成的分区）数据，
     * 并结合dim_user_info最新分区现有的数据，
     * 根据id合并数据到dwd库中dim_user_info的分区表
     * （合并是指对dwd层数据进行插入或修改，
     * 需修改的数据以id为合并字段，
     * 根据operate_time排序取最新的一条），
     * 分区字段为etl_date且值与ods库的相对应表该值相等，
     * 同时若operate_time为空，
     * 则用create_time填充，
     * 并添加dwd_insert_user、
     * dwd_insert_time、
     * dwd_modify_user、
     * dwd_modify_time四列,
     * 其中dwd_insert_user、dwd_modify_user均填写“user1”。
     * 若该条记录第一次进入数仓dwd层则dwd_insert_time、dwd_modify_time均存当前操作时间，
     * 并进行数据类型转换。
     * 若该数据在进入dwd层时发生了合并修改，
     * 则dwd_insert_time时间不变，
     * dwd_modify_time存当前操作时间，其余列存最新的值。
     * 使用hive cli执行show partitions dwd.dim_user_info命令，将结果截图粘贴至对应报告中；
     */

    val MaxPartition: Int = sparkSession.sql(
      s"""
         |select max(cast(etldate as INT))
         |from dwd.dim_user_info
         |""".stripMargin).collect()(0).get(0).toString.toInt

    val dim_user_info: DataFrame = sparkSession.sql(
      s"""
         |select * from dwd.dim_user_info
         |where cast(etldate as INT)=${MaxPartition}
         |""".stripMargin)

    val columns: Array[Column] = dim_user_info.columns.map(col(_))

    val user_info: DataFrame = sparkSession.sql(
      s"""
         |select * from ods.user_info
         |where etldate = 20230426
         |""".stripMargin)
      .withColumn("dwd_insert_user", lit("user1"))
      .withColumn("dwd_insert_time", lit(date_format(current_timestamp(), "yyyy-MM-dd HH:mm:ss")).cast("timestamp"))
      .withColumn("dwd_modify_user", lit("user1"))
      .withColumn("dwd_modify_time", lit(date_format(current_timestamp(), "yyyy-MM-dd HH:mm:ss")).cast("timestamp"))
      .select(columns: _*)

    val unioned: Dataset[Row] = user_info.union(dim_user_info)
    val w1: WindowSpec = Window.partitionBy("id").orderBy(col("operate_time").desc)
    val w2: WindowSpec = Window.partitionBy("id")

    val unioned2: DataFrame = unioned.withColumn("_row_number", row_number().over(w1))
      .withColumn("dwd_insert_time", min("dwd_insert_time").over(w2))
      .withColumn("dwd_modify_time", max("dwd_modify_time").over(w2))

    val merged: DataFrame = unioned2.where("_row_number=1").drop("_row_number")

    merged.write.mode(SaveMode.Overwrite).saveAsTable("dwd.tmp_dim_user_info")

    sparkSession.table("dwd.tmp_dim_user_info")
      .write.mode(SaveMode.Overwrite)
      .partitionBy("etldate")
      .saveAsTable("dwd.dim_user_info")

    sparkSession.sql(
      s"""
         |drop table dwd.tmp_dim_user_info
         |""".stripMargin)
  }
}

```

