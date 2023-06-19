---
layout: post
title: ClickHouse单表查询优化
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

## Prewhere代替where

Prewhere 和 where 语句的作用相同，用来过滤数据。不同之处在于 prewhere 只支持 *MergeTree 族系列引擎的表，首先会读取指定的列数据，来判断数据过滤，等待数据过滤之后再读取 select 声明的列字段来补全其余属性。

当查询列明显多于筛选列时使用Prewhere可十倍提升查询性能，Prewhere会自动优化执行过滤阶段的数据读取方式，降低 io 操作。在某些场合下，prewhere语句比where语句处理的数据量更少性能更高。

```sql
-- 默认开启where自动优化成prewhere
-- 先关闭自动优化进行测试
set optimize_move_to_prewhere=0;

-- 使用where查询
select WatchID, 
 JavaEnable, 
 Title, 
 GoodEvent, 
 EventTime, 
 EventDate, 
 CounterID, 
 ClientIP, 
 ClientIP6, 
 RegionID, 
 UserID, 
 CounterClass, 
 OS, 
 UserAgent, 
 URL, 
 Referer, 
 URLDomain, 
 RefererDomain, 
 Refresh, 
 IsRobot, 
 RefererCategories, 
 URLCategories, 
 URLRegions, 
 RefererRegions, 
 ResolutionWidth, 
 ResolutionHeight, 
 ResolutionDepth, 
 FlashMajor, 
 FlashMinor, 
 FlashMinor2
from datasets.hits_v1 where UserID='3198390223272470366';

-- 使用prewhere查询
select WatchID, 
 JavaEnable, 
 Title, 
 GoodEvent, 
 EventTime, 
 EventDate, 
 CounterID, 
 ClientIP, 
 ClientIP6, 
 RegionID, 
 UserID, 
 CounterClass, 
 OS, 
 UserAgent, 
 URL, 
 Referer, 
 URLDomain, 
 RefererDomain, 
 Refresh, 
 IsRobot, 
 RefererCategories, 
 URLCategories, 
 URLRegions, 
 RefererRegions, 
 ResolutionWidth, 
 ResolutionHeight, 
 ResolutionDepth, 
 FlashMajor, 
 FlashMinor, 
 FlashMinor2
from datasets.hits_v1 prewhere UserID='3198390223272470366';
```



## 数据采样

通过采样运算可以极大提升数据分析的性能。采样修饰符只有在 MergeTree engine 表中才有效。

```sql
SELECT Title,count(*) AS PageViews 
FROM hits_v1
SAMPLE 0.1 #代表采样 10%的数据,也可以是具体的条数
WHERE CounterID = 57
GROUP BY Title
ORDER BY PageViews DESC LIMIT 1000
```



## uniqCombined代替distinct

uniqCombined为近似去重，能接受2%左右的数据误差，这种去重方式可以提升查询性能。

不建议在千万级不同数据上执行 distinct 去重查询，改为近似去重 uniqCombined。

```sql
-- 反例：
select count(distinct rand()) from datasets.hits_v1;
-- 正例：
SELECT uniqCombined(rand()) from datasets.hits_v1;
```

