---
layout: post
title: ClickHouse执行计划
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

```sql
EXPLAIN [AST | SYNTAX | PLAN | PIPELINE] [setting = value, ...] SELECT ... [FORMAT ...]
```

PLAN：用于查看执行计划，默认值。 

AST ：用于查看语法树。

SYNTAX：用于优化语法。

PIPELINE：用于查看 PIPELINE 计划。



案例实操

1.查看plan

```
explain plan select arrayJoin([1,2,3,null,null]);
```

```sql
explain select database,table,count(1) cnt from system.parts where database in ('datasets','system') group by database,table order by database,cnt desc limit 2 by database;
```

2.AST语法树

```sql
EXPLAIN AST SELECT number from system.numbers limit 10;
```

3.SYNTAX语法优化

```sql
-- 先做一次查询
SELECT number = 1 ? 'hello' : (number = 2 ? 'world' : 'atguigu') FROM numbers(10);
-- 查看语法优化
EXPLAIN SYNTAX SELECT number = 1 ? 'hello' : (number = 2 ? 'world' : 'atguigu') FROM numbers(10);
-- 开启三元运算符优化
SET optimize_if_chain_to_multiif = 1;
-- 再次查看语法优化
EXPLAIN SYNTAX SELECT number = 1 ? 'hello' : (number = 2 ? 'world' : 'atguigu') FROM numbers(10);
-- 返回优化后的语句
SELECT multiIf(number = 1, \'hello\', number = 2, \'world\', \'xyz\') FROM numbers(10)
```

4.查看PIPELINE

```sql
EXPLAIN PIPELINE SELECT sum(number) FROM numbers_mt(100000) GROUP BY number % 20;

EXPLAIN PIPELINE header=1,graph=1 SELECT sum(number) FROM numbers_mt(10000) GROUP BY number%20;
```

执行计划中最重要的是语法优化，可以把优化后的sql语句拿出来用，起到一个装逼的效果。
