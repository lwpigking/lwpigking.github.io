---
layout: post
title: Flume整合Kafka
categories: [Flume]
description: Flume
keywords: Flume
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

Flume 发送数据到 Kafka 上主要是通过 `KafkaSink` 来实现的。

# 启动Zk和Kf

```bash
# 启动Zookeeper
zkServer.sh start

# 启动kafka
bin/kafka-server-start.sh config/server.properties
```

# 创建主题

```bash
# 创建主题
bin/kafka-topics.sh --create \
--zookeeper master:2181 \
--replication-factor 1   \
--partitions 1 --topic flume-kafka

# 查看创建的主题
bin/kafka-topics.sh --zookeeper master:2181 --list
```

# 启动Kf消费者

启动一个消费者，监听我们刚才创建的 `flume-kafka` 主题：

```bash
bin/kafka-console-consumer.sh --bootstrap-server master:9092 --topic flume-kafka
```

# 配置Flume

新建配置文件 `exec-memory-kafka.properties`，文件内容如下。这里我们监听一个名为 `kafka.log` 的文件，当文件内容有变化时，将新增加的内容发送到 Kafka 的 `flume-kafka` 主题上。

```bash
a1.sources = s1
a1.channels = c1
a1.sinks = k1

a1.sources.s1.type=exec
a1.sources.s1.command=tail -F /tmp/kafka.log
a1.sources.s1.channels=c1 

#设置Kafka接收器
a1.sinks.k1.type= org.apache.flume.sink.kafka.KafkaSink
#设置Kafka地址
a1.sinks.k1.brokerList=hadoop001:9092
#设置发送到Kafka上的主题
a1.sinks.k1.topic=flume-kafka
#设置序列化方式
a1.sinks.k1.serializer.class=kafka.serializer.StringEncoder
a1.sinks.k1.channel=c1     

a1.channels.c1.type=memory
a1.channels.c1.capacity=10000
a1.channels.c1.transactionCapacity=100   
```

# 启动Flume

```bash
flume-ng agent \
--c conf \
--f /opt/module/flume/job/exec-memory-kafka.properties \
--n a1 -Dflume.root.logger=INFO,console
```

# 测试

向监听的 `/tmp/kafka.log     ` 文件中追加内容，查看 Kafka 消费者的输出：

```bash
# 向文件中追加数据
echo "hello flume" >> kafka.log
echo "hello kafka" >> kafka.log
echo "hello flink" >> kafka.log
echo "hello storm" >> kafka.log
echo "hello SparkStreaming" >> kafka.log
```

可以看到 `flume-kafka` 主题的消费端已经收到了对应的消息。
