---
layout: post
title: ClickHouse物化视图
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

## 概述

ClickHouse的物化视图是一种查询结果的持久化，提升了查询效率。

用户查起来跟表没有区别，它就是一张表，它也像是一张时刻在预计算的表，创建的过程它是用了一个特殊引擎，加上后来 as select，就是CTAS的写法。 

“查询结果集”的范围很宽泛，可以是基础表中部分数据的一份简单拷贝，也可以是多表 join之后产生的结果或其子集，或者原始数据的聚合指标等等。物化视图不会随着基础表的变化而变化，所以也称为快照（snapshot）。

物化视图保存了数据；而普通视图不保存数据，只保存查询语句。



## 用法

```sql
CREATE [MATERIALIZED] VIEW [IF NOT EXISTS] [db.]table_name [TO[db.]name] [ENGINE = engine] [POPULATE] AS SELECT ...
```

1.必须指定物化视图的engine用于数据存储。

2.TO [db].[table]语法的时候，不得使用 POPULATE。 

3.查询语句(select）可以包含下面的子句： DISTINCT, GROUP BY, ORDER BY, LIMIT… 

4.物化视图的 alter 操作有些限制，操作起来不大方便。 

5.若物化视图的定义使用了 TO [db.]name 子语句，则可以将目标表的视图卸载（DETACH）再装载（ATTACH）。



## 实操

```sql
-- 建表语句
CREATE TABLE hits_test
(
 EventDate Date, 
 CounterID UInt32, 
 UserID UInt64, 
 URL String, 
 Income UInt8
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(EventDate)
ORDER BY (CounterID, EventDate, intHash32(UserID))
SAMPLE BY intHash32(UserID)
SETTINGS index_granularity = 8192
```

```sql
INSERT INTO hits_test 
 SELECT 
 EventDate,
 CounterID,
 UserID,
 URL,
 Income 
FROM hits_v1 
limit 10000;
```

```sql
-- 建立物化视图
CREATE MATERIALIZED VIEW hits_mv 
ENGINE=SummingMergeTree
PARTITION BY toYYYYMM(EventDate) ORDER BY (EventDate, intHash32(UserID)) 
AS SELECT
UserID,
EventDate,
count(URL) as ClickCount,
sum(Income) AS IncomeSum
FROM hits_test
WHERE EventDate >= '2014-03-20'
GROUP BY UserID,EventDate;
```

```sql
-- 导入增量数据
INSERT INTO hits_test 
SELECT 
 EventDate,
 CounterID,
 UserID,
 URL,
 Income 
FROM hits_v1 
WHERE EventDate >= '2014-03-23' 
limit 10;

-- 查询物化视图
SELECT * FROM hits_mv;
```

```sql
-- 导入历史数据
INSERT INTO hits_mv
SELECT
 UserID,
 EventDate,
 count(URL) as ClickCount,
 sum(Income) AS IncomeSum
FROM hits_test
WHERE EventDate = '2014-03-20'
GROUP BY UserID,EventDate

-- 查询物化视图
SELECT * FROM hits_mv;
```

