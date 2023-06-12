---
layout: post
title: ClickHouse表引擎的使用
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

表引擎是 ClickHouse 的一大特色。可以说， 表引擎决定了如何存储表的数据。包括： 

1.数据的存储方式和位置，写到哪里以及从哪里读取数据。 

2.支持哪些查询以及如何支持。 

3.并发数据访问。 

4.索引的使用（如果存在。 

5.是否可以执行多线程请求。 

6.数据复制参数。 表引擎的使用方式就是必须显式在创建表时定义该表使用的引擎，以及引擎使用的相关 参数。 

特别注意：引擎的名称大小写敏感
