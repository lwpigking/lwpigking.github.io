---
layout: post
title: ClickHouse表引擎MaterializeMySQL
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

ClickHouse 20.8.2.3 版本新增加了MaterializeMySQL的database引擎，该database能映射到MySQL中的某个database，并自动在ClickHouse中创建对应的ReplacingMergeTree。

ClickHouse服务做为MySQL副本，读取Binlog 并执行DDL和DML请求，实现了基于MySQL Binlog机制的业务数据库实时同步功能。



```
打开/etc/my.cnf,在[mysqld]下添加：

server-id=1 
log-bin=mysql-bin
binlog_format=ROW

gtid-mode=on
enforce-gtid-consistency=1 # 设置为主从强一致性
log-slave-updates=1 # 记录日志
```

GTID是MySQL复制增强版，从MySQL5.6版本开始支持，目前已经是MySQL主流复制模式。它为每个event分配一个全局唯一ID和序号，我们可以不用关心MySQL集群主从拓扑结构，直接告知MySQL这个GTID即可。



```sql
-- 在mysql中创建表并写入数据
CREATE DATABASE testck;

CREATE TABLE `testck`.`t_organization` (
 `id` int(11) NOT NULL AUTO_INCREMENT,
 `code` int NOT NULL,
 `name` text DEFAULT NULL,
 `updatetime` datetime DEFAULT NULL,
 PRIMARY KEY (`id`),
 UNIQUE KEY (`code`)
) ENGINE=InnoDB;

INSERT INTO testck.t_organization (code, name,updatetime) VALUES(1000,'Realinsight',NOW());
INSERT INTO testck.t_organization (code, name,updatetime) VALUES(1001, 'Realindex',NOW());
INSERT INTO testck.t_organization (code, name,updatetime) VALUES(1002,'EDT',NOW());


CREATE TABLE `testck`.`t_user` (
 `id` int(11) NOT NULL AUTO_INCREMENT,
 `code` int,
 PRIMARY KEY (`id`)
) ENGINE=InnoDB;

INSERT INTO testck.t_user (code) VALUES(1);
```

```
-- 开启ClickHouse物化引擎（必须开）
set allow_experimental_database_materialize_mysql=1;
```

```sql
-- 在clickhouse中创建数据库
CREATE DATABASE test_binlog ENGINE = MaterializeMySQL('master:3306','testck','root','123456');
```

```sql
-- 查看clickhouse的数据
use test_binlog;
show tables;
select * from t_organization;
select * from t_user;
```

