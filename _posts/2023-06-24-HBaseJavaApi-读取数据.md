---
layout: post
title: HBaseJavaApi-读取数据
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
     * 读取数据
     * @param namespace 命名空间
     * @param tableName 表名
     * @param rowKey 行号
     * @param columnFamily 列族
     * @param columnName 列明
     */
    public static void getCells(String namespace, String tableName, String rowKey, String columnFamily, String columnName) throws IOException {
        // 1.获取table对象
        Table table = connection.getTable(TableName.valueOf(namespace, tableName));

        // 2.创建get对象
        Get get = new Get(Bytes.toBytes(rowKey));

        // 如果直接调用get方法读取数据，此时读一整行数据
        // 如果想要读取某一列数据，需要添加对应的参数
        get.addColumn(Bytes.toBytes(columnFamily), Bytes.toBytes(columnName));
        // 设置读取数据的版本
        get.readAllVersions();

        // 读取数据，得到result对象
        try {
            Result result = null;
            result = table.get(get);
            // 处理数据
            Cell[] cells = result.rawCells();
            for (Cell cell : cells) {
                // cell存储数据比较底层
                // 会乱码
                String value = new String(CellUtil.cloneValue(cell));
                System.out.println(value);
            }
        } catch (IOException e) {
            e.printStackTrace();
        }

        table.close();
    }
```

