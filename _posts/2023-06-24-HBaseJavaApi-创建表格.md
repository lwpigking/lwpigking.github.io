---
layout: post
title: HBaseJavaApi-创建表格
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
     * 创建表格
     * @param namespace 命名空间名称
     * @param tableName 表格名称
     * @param columnFamilies 列族名称(可以有多个)
     */
    public static void createTable(String namespace, String tableName, String... columnFamilies) throws IOException {
        // 判断是否有至少一个列族
        if (columnFamilies.length == 0){
            System.out.println("创建表格至少有一个列族");
            return;
        }

        // 判断表格是否存在
        if (isTableExists(namespace, tableName)){
            System.out.println("表格已经存在");
            return;
        }

        // 1.获取admin
        Admin admin = connection.getAdmin();

        // 2.创建表格
        // 2.1 创建表格描述 => 建造者模式
        TableDescriptorBuilder builder = TableDescriptorBuilder.newBuilder(TableName.valueOf(namespace, tableName));

        // 2.2 添加参数
        for (String columnFamily : columnFamilies) {

            // 2.3 创建列族描述的建造者
            ColumnFamilyDescriptorBuilder columnFamilyDescriptorBuilder = ColumnFamilyDescriptorBuilder.newBuilder(Bytes.toBytes(columnFamily));

            // 2.4 对应当前的列族添加参数
            // 添加版本参数
            columnFamilyDescriptorBuilder.setMaxVersions(5);

            // 2.5 创建添加完参数的列族描述
            builder.setColumnFamily(columnFamilyDescriptorBuilder.build());
        }

        // 2.3 创建对应的表格描述
        try {
            admin.createTable(builder.build());
        } catch (IOException e) {
            e.printStackTrace();
        }

        // 3.关闭admin
        admin.close();
    }
```

