---
layout: fragment
title: Kafka提高生产者吞吐量
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

import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.ProducerConfig;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.apache.kafka.common.serialization.StringSerializer;

import java.util.Properties;

/**
 * Project:  BigDataCode
 * Create date:  2023/5/17
 * Created by fujiahao
 */

/**
 * 提高生产者吞吐量
 * 可以通过设置缓冲区大小、批次大小、
 * linger.ms和压缩这四种方法来提高吞吐量
 */

public class CustomProducerParameters {
    public static void main(String[] args) {

        // 配置
        Properties properties = new Properties();
        properties.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "master:9092");
        properties.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
        properties.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());

        // 缓冲区大小
        // 默认32M
        properties.put(ProducerConfig.BUFFER_MEMORY_CONFIG, 33554432);

        // 批次大小
        // 默认16K
        properties.put(ProducerConfig.BATCH_SIZE_CONFIG, 16384);

        // linger.ms
        // 默认为0ms，生产环境一般为5-100ms
        properties.put(ProducerConfig.LINGER_MS_CONFIG, 1);

        // 压缩
        // 默认为none，可选gzip、snappy、lz4和zstd
        properties.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "snappy");

        // 1.创建生产者
        KafkaProducer<String, String> kafkaProducer = new KafkaProducer<>(properties);

        // 2.发送数据
        for (int i = 0; i < 5; i++) {
            kafkaProducer.send(new ProducerRecord<>("first", "lwPigKing" + i));
        }

        // 3.关闭资源
        kafkaProducer.close();
    }
}

```

