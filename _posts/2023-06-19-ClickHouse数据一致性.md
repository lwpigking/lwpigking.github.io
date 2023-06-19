---
layout: post
title: ClickHouse数据一致性
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

查询 CK 手册发现，即便对数据一致性支持最好的 Mergetree，也只是保证最终一致性。

在使用ReplacingMergeTree、SummingMergeTree这类表引擎的时候，会出现短暂数据不一致的情况。



## 手动optimize

在写入数据后，立刻执行 OPTIMIZE 强制触发新写入分区的合并动作。

生产环境中不建议这么做！！！！！！！！！！！！！

```sql
OPTIMIZE TABLE test_a FINAL;
```



## Group by去重

```sql
SELECT
 user_id ,
 argMax(score, create_time) AS score,  # argMax(field1，field2):按照field2的最大值取field1的值。
 argMax(deleted, create_time) AS deleted,
 max(create_time) AS ctime 
FROM test_a 
GROUP BY user_id
HAVING deleted = 0;  # 打标记
```
