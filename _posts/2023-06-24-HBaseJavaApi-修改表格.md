---
layout: post
title: HBaseJavaApi-修改表格
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
     * 修改表格
     * @param namespace 命名空间
     * @param tableName 表名
     * @param columnFamily 列族
     * @param version 版本
     */
    public static void modifyTable(String namespace, String tableName, String columnFamily, int version) throws IOException {

        // 判断表格是否存在
        if (!isTableExists(namespace, tableName)){
            System.out.println("表格不存在");
            return;
        }

        // 1.获取admin
        Admin admin = connection.getAdmin();

        try {
            // 2.修改表
            // 2.0 获取之前的表格描述!!!!!!!!!(这个很重要!!!!!!!)
            TableDescriptor descriptor = admin.getDescriptor(TableName.valueOf(namespace, tableName));
            // 2.1 创建一个表格描述
            // 如果直接填写tableName，相当于创建了一个新的表格描述建造者，没有之前的信息。
            // 如果想要修改之前的信息，必须调用方法填写一个旧的表格描述
            TableDescriptorBuilder tableDescriptorBuilder = TableDescriptorBuilder.newBuilder(descriptor);

            // 2.2 对应建造者进行表格数据的修改
            // 需要填写旧的列族描述
            ColumnFamilyDescriptor columnFamily1 = descriptor.getColumnFamily(Bytes.toBytes(columnFamily));
            ColumnFamilyDescriptorBuilder columnFamilyDescriptorBuilder = ColumnFamilyDescriptorBuilder.newBuilder(columnFamily1);
            // 修改对应版本
            columnFamilyDescriptorBuilder.setMaxVersions(version);
            // 此处修改的时候，如果填写新创建的，那么别的参数会初始化
            tableDescriptorBuilder.modifyColumnFamily(columnFamilyDescriptorBuilder.build());

            admin.modifyTable(tableDescriptorBuilder.build());
        } catch (IOException e) {
            e.printStackTrace();
        }

        // 关闭admin
        admin.close();
    }
```

