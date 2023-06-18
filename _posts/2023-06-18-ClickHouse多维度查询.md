---
layout: post
title: ClickHouse多维度查询
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

clickhouse查询操作基本和标准sql差不多，但是group by增加了with rollup\with cube\with totals用来进行多维度分析，具体如下：

插入数据

```sql
alter table t_order_mt delete where 1=1;
insert into t_order_mt values
(101,'sku_001',1000.00,'2020-06-01 12:00:00'),
(101,'sku_002',2000.00,'2020-06-01 12:00:00'),
(103,'sku_004',2500.00,'2020-06-01 12:00:00'),
(104,'sku_002',2000.00,'2020-06-01 12:00:00'),
(105,'sku_003',600.00,'2020-06-02 12:00:00'),
(106,'sku_001',1000.00,'2020-06-04 12:00:00'),
(107,'sku_002',2000.00,'2020-06-04 12:00:00'),
(108,'sku_004',2500.00,'2020-06-04 12:00:00'),
(109,'sku_002',2000.00,'2020-06-04 12:00:00'),
(110,'sku_003',600.00,'2020-06-01 12:00:00');
```

with rollup：上钻

```sql
select id, sku_id, sum(total_amount) from t_order_mt group by id,sku_id with rollup;
```

with cube：上下钻

```sql
select id, sku_id, sum(total_amount) from t_order_mt group by id,sku_id with cube;
```

with totals：最上层上钻

```sql
select id , sku_id,sum(total_amount) from t_order_mt group by id,sku_id with totals;
```

