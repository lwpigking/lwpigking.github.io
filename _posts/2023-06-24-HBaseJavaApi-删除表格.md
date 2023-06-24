---
layout: post
title: HBaseJavaApi-删除表格
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
     * 删除表格
     * @param namespace 命名空间
     * @param tableName 表名
     * @return 是否删除
     */
    public static boolean deleteTable(String namespace, String tableName) throws IOException {
        // 判断表格是否存在
        if (isTableExists(namespace, tableName)){
            System.out.println("表格不存在，无法删除");
            return false;
        }

        // 1.获取admin
        Admin admin = connection.getAdmin();

        // 2.删除表格
        try {
            // HBase删除表格之前一定要先标记表格为不可用
            admin.disableTable(TableName.valueOf(namespace, tableName));
            admin.deleteTable(TableName.valueOf(namespace, tableName));
        } catch (IOException e) {
            e.printStackTrace();
        }

        // 3.关闭连接
        admin.close();

        return true;
    }
```

