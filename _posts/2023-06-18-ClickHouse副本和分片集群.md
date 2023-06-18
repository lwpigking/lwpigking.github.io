---
layout: post
title: ClickHouse副本和分片集群
categories: [ClickHouse]
description: ClickHouse
keywords: ClickHouse
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

副本的目的主要是保障数据的高可用性，即使一台 ClickHouse 节点宕机，那么也可以从 其他服务器获得相同的数据。

副本写入流程：

1.客户端写入数据到ClickHouse-a

2.ClickHouse-a提交写入日志到ZooKeeper集群

3.ClickHouse-b收到来自ZooKeeper的写入日志

4.ClickHouse-b从目标副本（ClickHouse-a）下载新数据



副本虽然能够提高数据的可用性，降低丢失风险，但是每台服务器实际上必须容纳全量数据，对数据的横向扩容没有解决。 

要解决数据水平切分的问题，需要引入分片的概念。通过分片把一份完整的数据进行切分，不同的分片分布到不同的节点上，再通过 Distributed 表引擎把数据拼接起来一同使用。 Distributed 表引擎本身不存储数据，有点类似于 MyCat 之于 MySql，成为一种中间件， 通过分布式逻辑表来写入、分发、路由来操作多台节点不同分片的分布式数据。



一个分片集群可以有多个副本。

节点数 = 分片数 x 副本数
