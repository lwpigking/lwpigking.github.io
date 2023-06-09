---
layout: fragment
title: Kafka自定义分区器
tags: [Kafka]
description: some word here
keywords: Kafka
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

```java
package producer;

import org.apache.kafka.clients.producer.Partitioner;
import org.apache.kafka.common.Cluster;

import java.util.Map;

/**
 * Project:  BigDataCode
 * Create date:  2023/5/17
 * Created by fujiahao
 */
public class MyPartitioner implements Partitioner {
    @Override
    // s为topic主题、o为key、bytes为序列化key、byte1为序列化value、o1为value数据
    public int partition(String s, Object o, byte[] bytes, Object o1, byte[] bytes1, Cluster cluster) {

        // 获取数据
        String msgValues = o1.toString();

        int partition;
        if (msgValues.contains("lwPigKing")) {
            partition = 0;
        } else {
            partition = 1;
        }

        return partition;
    }

    @Override
    public void close() {

    }

    @Override
    public void configure(Map<String, ?> map) {

    }
}

```

