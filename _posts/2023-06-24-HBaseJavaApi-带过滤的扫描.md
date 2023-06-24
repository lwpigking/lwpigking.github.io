---
layout: post
title: HBaseJavaApi-带过滤的扫描
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
     * 带过滤的扫描
     * @param namespace 命名空间
     * @param tableName 表格名
     * @param startRow 开始
     * @param stopRow 结束
     * @param columnFamily 列族
     * @param columnName 列明
     * @param value 值
     * @throws IOException
     */
    public static void filterScan(String namespace, String tableName, String startRow,
                                  String stopRow, String columnFamily, String columnName, String value) throws IOException {
        // 1.获取table
        Table table = connection.getTable(TableName.valueOf(namespace, tableName));

        // 2.创建scan对象
        Scan scan = new Scan();
        // 如果直接调用，就扫描整张表
        // 添加参数来控制扫描数据
        scan.withStartRow(Bytes.toBytes(startRow));
        scan.withStopRow(Bytes.toBytes(stopRow));

        // 可以添加多个过滤
        FilterList filterList = new FilterList();

        // 创建过滤器
        // (1) 结果只保留当前列的数据
        ColumnValueFilter columnValueFilter = new ColumnValueFilter(
                // 列族名称
                Bytes.toBytes(columnFamily),
                // 列名
                Bytes.toBytes(columnName),
                // 比较关系
                CompareOperator.EQUAL,
                // 值
                Bytes.toBytes(value)
        );

        // (2) 结果保留整行数据
        // 结果会同时保留没有当前列的数据
        // 比如："bigdata:student"中"rowKey"为"1001"的"info:age"值是18,但是"info:name"为空值
        // 也会保留该行数据
        SingleColumnValueFilter singleColumnValueFilter = new SingleColumnValueFilter(
                // 列族名称
                Bytes.toBytes(columnFamily),
                // 列名
                Bytes.toBytes(columnName),
                // 比较关系
                CompareOperator.EQUAL,
                // 值
                Bytes.toBytes(value)
        );

        // 本身可以添加多个过滤器
        filterList.addFilter(singleColumnValueFilter);

        // 添加过滤
        scan.setFilter(filterList);

        try {
            // 读取多行数据
            ResultScanner scanner = null;
            scanner = table.getScanner(scan);
            // result来记录一行数据
            // ResultScanner来记录多行数据
            for (Result result : scanner) {
                Cell[] cells = result.rawCells();
                for (Cell cell : cells) {
                    System.out.print(new String(CellUtil.cloneRow(cell)) + "-"
                            + new String(CellUtil.cloneFamily(cell)) + "-"
                            + new String(CellUtil.cloneQualifier(cell)) + "-"
                            + new String(CellUtil.cloneValue(cell)) + "\t");
                }
                System.out.println();
            }
        } catch (IOException e) {
            e.printStackTrace();
        }

        table.close();
    }
```

