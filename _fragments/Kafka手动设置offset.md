---
layout: fragment
title: Kafka手动设置offset
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
package consumer;

/**
 * Project:  BigDataCode
 * Create date:  2023/5/23
 * Created by fujiahao
 */

import org.apache.kafka.clients.consumer.ConsumerConfig;
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import org.apache.kafka.common.serialization.StringDeserializer;

import java.time.Duration;
import java.util.ArrayList;
import java.util.Properties;

/**
 * 同步和移动提交、手动设置offset
 */

public class CustomConsumerByHandSync {
    public static void main(String[] args) {

        // 0.配置
        Properties properties = new Properties();
        properties.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG,"master:9092");
        properties.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        properties.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class.getName());
        properties.put(ConsumerConfig.GROUP_ID_CONFIG, "test");

        // 手动提交
        properties.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);

        // 1.创建一个消费者 "","hello"
        KafkaConsumer<String, String> kafkaConsumer = new KafkaConsumer<>(properties);

        // 2.订阅主题:lwPigKing
        ArrayList<String> topics = new ArrayList<>();
        topics.add("lwPigKing");
        kafkaConsumer.subscribe(topics);

        // 3.消费数据
        while (true) {

            ConsumerRecords<String, String> consumerRecords = kafkaConsumer.poll(Duration.ofSeconds(1));
            for (ConsumerRecord<String, String> consumerRecord : consumerRecords) {
                System.out.println(consumerRecord);
            }

            // 手动同步提交offset
            // kafkaConsumer.commitSync();

            // 手动异步提交offset
            kafkaConsumer.commitAsync();
        }

    }
}
```

