---
layout: post
title: HBaseJavaApi-插入数据
categories: [HBase]
description: HBase
keywords: HBase
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

```java
/**
     * 插入数据
     * @param namespace 命名空间
     * @param tableName 表名
     * @param rowKey 行号
     * @param columnFamily 列族
     * @param columnName 列明
     * @param value 值
     */
    public static void putCell(String namespace, String tableName, String rowKey, String columnFamily, String columnName, String value) throws IOException {
        // 1.获取table
        Table table = connection.getTable(TableName.valueOf(namespace, tableName));

        // 2.创建put对象
        Put put = new Put(Bytes.toBytes(rowKey));

        // 3.添加列
        put.addColumn(Bytes.toBytes(columnFamily), Bytes.toBytes(columnName), Bytes.toBytes(value));
        try {
            table.put(put);
        } catch (IOException e) {
            e.printStackTrace();
        }

        // 关闭table
        table.close();
    }
```

