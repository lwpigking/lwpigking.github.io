---
layout: post
title: HBaseJavaApi-删除一行中的一列数据
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
     * 删除一行中的一列数据
     * @param namespace 命名空间
     * @param tableName 表名
     * @param rowKey 行号
     * @param columnFamily 列族
     * @param columnName 列名
     */
    public static void deleteColumn(String namespace, String tableName, String rowKey, String columnFamily, String columnName) throws IOException {
        Table table = connection.getTable(TableName.valueOf(namespace, tableName));

        Delete delete = new Delete(Bytes.toBytes(rowKey));
        // 删除一个版本
        delete.addColumn(Bytes.toBytes(columnFamily), Bytes.toBytes(columnName));
        // 删除所有版本
        delete.addColumns(Bytes.toBytes(columnFamily), Bytes.toBytes(columnName));

        try {
            table.delete(delete);
        } catch (IOException e) {
            e.printStackTrace();
        }

        table.close();
    }
```

