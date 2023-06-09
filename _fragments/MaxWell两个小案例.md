---
layout: fragment
title: MaxWell两个小案例
tags: [MaxWell]
description: some word here
keywords: MaxWell
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

## 监控MySQL数据

运行maxwell监控mysql

```
bin/maxwell --user='maxwell' --password='123456' --host='hadoop102' --producer=stdout
```

想mysql的text_maxwell库的test表插入一条数据，查看maxwell的控制台输出

```
mysql> insert into test values(1,'aaa');
```

## 输出到Kafka

一定要先启动zookeeper和kafka。

启动maxwell监控binlog。

```
bin/maxwell --user='maxwell' --password='123456' --host='master' --producer=kafka --kafka.bootstrap.servers=master:9092 --kafka_topic=maxwell
```

开启kafka消费者

```
kafka-console-consumer.sh --bootstrap-server master:9092 --topic maxwell
```

插入mysql数据

```
mysql> insert into test values (5,'eee');
```
