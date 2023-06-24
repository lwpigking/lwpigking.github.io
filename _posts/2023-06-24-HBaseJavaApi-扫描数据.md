---
layout: post
title: HBaseJavaApi-扫描数据
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
     * 扫描数据
     * @param namespace 命名空间
     * @param tableName 表名
     * @param startRow 开始行(包含)
     * @param stopRow 结束行(不包含)
     */
    public static void scanRows(String namespace, String tableName, String startRow, String stopRow) throws IOException {
        // 1.获取table
        Table table = connection.getTable(TableName.valueOf(namespace, tableName));

        // 2.创建scan对象
        Scan scan = new Scan();
        // 如果直接调用，就扫描整张表
        // 添加参数来控制扫描数据
        scan.withStartRow(Bytes.toBytes(startRow));
        scan.withStopRow(Bytes.toBytes(stopRow));

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

