---
layout: post
title: ClickHouse表引擎MergeTree
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

## MergeTree

ClickHouse 中最强大的表引擎当属 MergeTree（合并树）引擎及该系列（*MergeTree）中的其他引擎，支持索引和分区，地位可以相当于 innodb 之于 Mysql。而且基于 MergeTree， 还衍生除了很多小弟，也是非常有特色的引擎。

```
create table t_order_mt(
 id UInt32,
 sku_id String,
 total_amount Decimal(16,2),
 create_time Datetime
) engine =MergeTree
  partition by toYYYYMMDD(create_time)
  primary key (id)
  order by (id,sku_id);
```

### partition by（可选参数）

分区的目的主要是降低扫描的范围，优化查询速度。

如果不填，就会使用一个分区。

分区目录：MergeTree 是以列文件+索引文件+表定义文件组成的，但是如果设定了分区那么这些文 件就会保存到不同的分区目录中。

并行：分区后，面对涉及跨分区的查询统计，ClickHouse 会以分区为单位并行处理。

数据写入与分区合并：任何一个批次的数据写入都会产生一个临时分区，不会纳入任何一个已有的分区。写入 后的某个时刻（大概 10-15 分钟后），ClickHouse会自动执行合并操作（等不及也可以手动通过 optimize 执行），把临时分区的数据，合并到已有分区中。

```
optimize table xxxx final;
```

### primary key（可选参数）

ClickHouse 中的主键，和其他数据库不太一样，它只提供了数据的一级索引，但是却不是唯一约束。这就意味着是可以存在相同 primary key 的数据的。 

主键的设定主要依据是查询语句中的 where 条件。根据条件通过对主键进行某种形式的二分查找，能够定位到对应的 index granularity,避免了全表扫描。 

index granularity： 直接翻译的话就是索引粒度，指在稀疏索引中两个相邻索引对应数据的间隔。ClickHouse 中的 MergeTree 默认是 8192。官方不建议修改这个值，除非该列存在 大量重复值，比如在一个分区中几万行才有一个不同数据。

稀疏索引：稀疏索引的好处就是可以用很少的索引数据，定位更多的数据，代价就是只能定位到索 引粒度的第一行，然后再进行进行一点扫描。

### order by（必选）

order by 设定了分区内的数据按照哪些字段顺序进行有序保存。 

order by 是 MergeTree 中唯一一个必填项，甚至比 primary key 还重要，因为当用户不 设置主键的情况，很多处理会依照 order by 的字段进行处理（比如后面会讲的去重和汇总）。 

要求：主键必须是 order by 字段的前缀字段。 

比如 order by 字段是 (id,sku_id) 那么主键必须是 id 或者(id,sku_id)

### 二级索引

目前在 ClickHouse 的官网上二级索引的功能在 v20.1.2.4 之前是被标注为实验性的，在 这个版本之后默认是开启的。

是否允许使用实验性的二级索引（v20.1.2.4 开始，这个参数已被删除，默认开启）

```
set allow_experimental_data_skipping_indices=1;
```

```
create table t_order_mt2(
 id UInt32,
 sku_id String,
 total_amount Decimal(16,2),
 create_time Datetime,
INDEX a total_amount TYPE minmax GRANULARITY 5
) engine =MergeTree
  partition by toYYYYMMDD(create_time)
  primary key (id)
  order by (id, sku_id);
```

其中 GRANULARITY N 是设定二级索引对于一级索引粒度的粒度。

```
insert into t_order_mt2 values
(101,'sku_001',1000.00,'2020-06-01 12:00:00') ,
(102,'sku_002',2000.00,'2020-06-01 11:00:00'),
(102,'sku_004',2500.00,'2020-06-01 12:00:00'),
(102,'sku_002',2000.00,'2020-06-01 13:00:00'),
(102,'sku_002',12000.00,'2020-06-01 13:00:00'),
(102,'sku_002',600.00,'2020-06-02 12:00:00');
```

那么在使用下面语句进行测试，可以看出二级索引能够为非主键字段的查询发挥作用。

```
clickhouse-client --send_logs_level=trace <<< 'select 
* from t_order_mt2 where total_amount > toDecimal32(900., 2)';
```

### 数据TTL

TTL 即 Time To Live，MergeTree 提供了可以管理数据表或者列的生命周期的功能。

```
create table t_order_mt3(
 id UInt32,
 sku_id String,
 total_amount Decimal(16,2) TTL create_time+interval 10 SECOND,
 create_time Datetime 
) engine =MergeTree
partition by toYYYYMMDD(create_time)
 primary key (id)
 order by (id, sku_id);
```

```
insert into t_order_mt3 values
(106,'sku_001',1000.00,'2020-06-12 22:52:30'),
(107,'sku_002',2000.00,'2020-06-12 22:52:30'),
(110,'sku_003',600.00,'2020-06-13 12:00:00');
```

手动合并，查看效果到期后，指定的字段数据归 0

下面的这条语句是数据会在 create_time 之后 10 秒丢失

```
alter table t_order_mt3 MODIFY TTL create_time + INTERVAL 10 SECOND;
```

涉及判断的字段必须是 Date 或者 Datetime 类型，推荐使用分区的日期字段。
