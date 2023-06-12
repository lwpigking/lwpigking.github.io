---
layout: post
title: ClickHouse表引擎SummingMergeTree
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

对于不查询明细，只关心以维度进行汇总聚合结果的场景。如果只使用普通的MergeTree 的话，无论是存储空间的开销，还是查询时临时聚合的开销都比较大。 

ClickHouse 为了这种场景，提供了一种能够“预聚合”的引擎 SummingMergeTree。

```
create table t_order_smt(
 id UInt32,
 sku_id String,
 total_amount Decimal(16,2) ,
 create_time Datetime 
) engine =SummingMergeTree(total_amount)
 partition by toYYYYMMDD(create_time)
 primary key (id)
 order by (id,sku_id );
```

```
insert into t_order_smt values
(101,'sku_001',1000.00,'2020-06-01 12:00:00'),
(102,'sku_002',2000.00,'2020-06-01 11:00:00'),
(102,'sku_004',2500.00,'2020-06-01 12:00:00'),
(102,'sku_002',2000.00,'2020-06-01 13:00:00'),
(102,'sku_002',12000.00,'2020-06-01 13:00:00'),
(102,'sku_002',600.00,'2020-06-02 12:00:00');
```

```
select * from t_order_smt;
```

```
OPTIMIZE TABLE t_order_smt FINAL;
```

```
select * from t_order_smt;
```

通过结果可以得到以下结论：

1.以 SummingMergeTree（）中指定的列作为汇总数据列 

2.可以填写多列必须数字列，如果不填，以所有非维度列且为数字列的字段为汇总数据列 

3.以 order by 的列为准，作为维度列 

4.其他的列按插入顺序保留第一行 

5.不在一个分区的数据不会被聚合 

6.只有在同一批次插入(新版本)或分片合并时才会进行聚合

开发建议：设计聚合表的话，唯一键值、流水号可以去掉，所有字段全部是维度、度量或者时间戳。

问题：

能不能直接执行以下 SQL 得到汇总值

```
select total_amount from XXX where province_name=’’ and create_date=’xxx’
```

不行，可能会包含一些还没来得及聚合的临时明细

如果要是获取汇总值，还是需要使用 sum 进行聚合，这样效率会有一定的提高，但本 身 ClickHouse 是列式存储的，效率提升有限，不会特别明显。

```
select sum(total_amount) from province_name=’’ and create_date=‘xxx’
```

