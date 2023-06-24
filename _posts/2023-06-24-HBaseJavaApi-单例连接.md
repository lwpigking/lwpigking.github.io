---
layout: post
title: HBaseJavaApi-单例连接
categories: [HBase]
description: HBase
keywords: HBase
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

```java
import org.apache.flink.table.filesystem.FileSystemOutputFormat;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.hbase.client.AsyncConnection;
import org.apache.hadoop.hbase.client.Connection;
import org.apache.hadoop.hbase.client.ConnectionFactory;

import java.io.IOException;
import java.util.concurrent.CompletableFuture;

/**
 * Project:  BigDataCode
 * Create date:  2023/6/20
 * Created by fujiahao
 */

/**
 * 单线程使用连接
 */

public class HBaseConnection {

    // 声明静态属性
    public static Connection connection = null;
    static {
        // 创建连接
        // 默认使用同步连接
        try {
            // 使用读取本地文件的形式添加参数 hbase-site.xml
            connection = ConnectionFactory.createConnection();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    public static void closeConnection() throws IOException {
        // 判断连接是否为空值
        if (connection != null){
            connection.close();
        }
    }

    public static void main(String[] args) throws IOException {

        /*// 创建连接配置对象
        Configuration conf = new Configuration();
        conf.set("hbase.zookeeper.quorum", "master,slave1,slave2");

        // 创建连接
        // 默认使用同步连接
        Connection connection = ConnectionFactory.createConnection(conf);

        // 可以使用异步连接
        // 不推荐
        // CompletableFuture<AsyncConnection> asyncConnection = ConnectionFactory.createAsyncConnection(conf);

        // 使用连接
        System.out.println(connection);

        // 关闭连接
        connection.close();*/

        // 直接使用创建好的链接
        // 不要在main线程里面单独创建
        System.out.println(HBaseConnection.connection);

        // 在main线程的最后记得关闭连接
        HBaseConnection.closeConnection();
    }
}

```

