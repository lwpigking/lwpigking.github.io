---
layout: fragment
title: Flink自定义HBaseSink
tags: [Flink]
description: some word here
keywords: Flink, HBaseSink
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

```scala
package Calculation

import org.apache.flink.configuration.Configuration
import org.apache.flink.streaming.api.functions.sink.{RichSinkFunction, SinkFunction}
import org.apache.hadoop.conf
import org.apache.hadoop.hbase.client.{Connection, ConnectionFactory, Put, Table}
import org.apache.hadoop.hbase.util.Bytes
import org.apache.hadoop.hbase.{HBaseConfiguration, HConstants, TableName}

/**
 * Project:  BigdataCore
 * Create date:  2023/4/20
 * Created by fujiahao
 */
class customerHBaseSink extends RichSinkFunction[(String, Int, Long, String, String, Int)]{
  var conn: Connection = null

  override def open(parameters: Configuration): Unit = {
    val config: conf.Configuration = HBaseConfiguration.create
    config.set(HConstants.ZOOKEEPER_QUORUM,"master,slave1,slave2")
    config.set(HConstants.ZOOKEEPER_CLIENT_PORT,"2181")
    conn = ConnectionFactory.createConnection(config)
  }

  override def invoke(value: (String, Int, Long, String, String, Int), context: SinkFunction.Context): Unit = {
    val tableName: String = "customer_login_log"
    val rowKey: String = value._1
    val login_id: Int = value._2
    val customer_id: Long = value._3
    val login_time: String = value._4
    val login_ip: String = value._5
    val login_type: Int = value._6
    val table: Table = conn.getTable(TableName.valueOf(tableName))
    val put: Put = new Put(Bytes.toBytes(rowKey))
    put.addColumn(Bytes.toBytes("cf"), Bytes.toBytes("login_id"), Bytes.toBytes(login_id.toString))
    put.addColumn(Bytes.toBytes("cf"), Bytes.toBytes("customer_id"), Bytes.toBytes(customer_id.toString))
    put.addColumn(Bytes.toBytes("cf"), Bytes.toBytes("login_time"), Bytes.toBytes(login_time))
    put.addColumn(Bytes.toBytes("cf"), Bytes.toBytes("login_ip"), Bytes.toBytes(login_ip))
    put.addColumn(Bytes.toBytes("cf"), Bytes.toBytes("login_type"), Bytes.toBytes(login_type.toString))
    table.put(put)
  }

  override def close(): Unit = {
    if (conn != null) {
      conn.close()
    }
  }
}
```

