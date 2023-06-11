---
layout: post
title: Flink自定义ClickHouseSink
categories: [Flink]
description: Flink
keywords: Flink
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

```scala
package Calculation

import java.sql.{Connection, DriverManager, PreparedStatement}

import org.apache.flink.configuration.Configuration
import org.apache.flink.streaming.api.functions.sink.{RichSinkFunction, SinkFunction}

/**
 * Project:  Bigdata
 * Create date:  2023/4/13
 * Created by fujiahao
 */
class ClickHouseSink extends RichSinkFunction[Int]{
  private var conn: Connection = null
  private var insertStmt: PreparedStatement = null

  @throws[Exception]
  override def open(parameters: Configuration): Unit = {
    conn = DriverManager.getConnection("jdbc:clickhouse://master/PV")
  }

  @throws[Exception]
  override def invoke(value: Int, context: SinkFunction.Context): Unit = {
    insertStmt = conn.prepareStatement("INSERT INTO PVtable(pv) VALUES(?)")
    insertStmt.setInt(1,value)
    insertStmt.execute()
  }

  @throws[Exception]
  override def close(): Unit = {
    insertStmt.close()
  }

}

```

