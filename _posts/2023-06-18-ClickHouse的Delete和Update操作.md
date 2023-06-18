---
layout: post
title: ClickHouse的Delete和Update操作
categories: [ClickHouse]
description: ClickHouse
keywords: ClickHouse
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: fals
---

ClickHouse 提供了 Delete 和 Update 的能力，这类操作被称为 Mutation查询，它可以看 做 Alter 的一种。 虽然可以实现修改和删除，但是和一般的数据库不一样，Mutation 语句是一种很 “重”的操作，而且不支持事务。

 “重”的原因主要是每次修改或者删除都会导致放弃目标数据的原有分区，重建新分区。 所以尽量做批量的变更，不要进行频繁小数据的操作。

```sql
# 删除
alter table t_order_smt delete where sku_id="sku_001";

# 修改
alter table t_order_smt update total_amount=toDecimal32(2000.00, 2) where id=102;
```

由于操作比较“重”，所以 Mutation 语句分两步执行。

第一步：进行新增数据新增分区和并，把旧分区打上逻辑上的失效标记。

第二步：触发分区合并，删除旧数据释放磁盘空间。

