---
layout: post
title: ClickHouse表引擎ReplacingMergeTree
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

ReplacingMergeTree是MergeTree 的一个变种，它存储特性完全继承 MergeTree，只是多了一个去重的功能。 尽管 MergeTree 可以设置主键，但是 primary key 其实没有唯一约束的功能。如果你想处理掉重复的数据，可以借助这个 ReplacingMergeTree。

去重时机：数据的去重只会在合并的过程中出现。合并会在未知的时间在后台进行，所以你无法预先作出计划。有一些数据可能仍未被处理。

去重范围：如果表经过了分区，去重只会在分区内部进行去重，不能执行跨分区的去重。 所以 ReplacingMergeTree 能力有限， ReplacingMergeTree 适用于在后台清除重复的数 据以节省空间，但是它不保证没有重复的数据出现。

```
create table t_order_rmt(
 id UInt32,
 sku_id String,
 total_amount Decimal(16,2) ,
 create_time Datetime 
) engine =ReplacingMergeTree(create_time)
 partition by toYYYYMMDD(create_time)
 primary key (id)
 order by (id, sku_id);
```

ReplacingMergeTree() 填入的参数为版本字段，重复数据保留版本字段值最大的。 

如果不填版本字段，默认按照插入顺序保留最后一条。

```
insert into t_order_rmt values
(101,'sku_001',1000.00,'2020-06-01 12:00:00') ,
(102,'sku_002',2000.00,'2020-06-01 11:00:00'),
(102,'sku_004',2500.00,'2020-06-01 12:00:00'),
(102,'sku_002',2000.00,'2020-06-01 13:00:00'),
(102,'sku_002',12000.00,'2020-06-01 13:00:00'),
(102,'sku_002',600.00,'2020-06-02 12:00:00');
```

执行第一次查询

```
select * from t_order_rmt;
```

手动合并

```
OPTIMIZE TABLE t_order_rmt FINAL;
```

再执行一次查询

```
select * from t_order_rmt;
```

通过测试得到结论：

1.实际上是使用 order by 字段作为唯一键 

2.去重不能跨分区 

3.只有同一批插入（新版本）或合并分区时才会进行去重 

4.认定重复的数据保留，版本字段值最大的 

5.如果版本字段相同则按插入顺序保留最后一笔
