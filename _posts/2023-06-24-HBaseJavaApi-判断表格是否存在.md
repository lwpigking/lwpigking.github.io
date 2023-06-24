---
layout: post
title: HBaseJavaApi-判断表格是否存在
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
     * 判断表格是否存在
     * @param namespace 命名空间名称
     * @param tableName 表格名称
     * @return true表示存在
     */
    public static boolean isTableExists(String namespace, String tableName) throws IOException {
        // 1.获取admin
        Admin admin = connection.getAdmin();

        // 2.判断表格是否存在
        boolean b = false;
        try {
            b = admin.tableExists(TableName.valueOf(namespace, tableName));
        } catch (IOException e) {
            e.printStackTrace();
        }

        // 3.关闭连接
        admin.close();

        // 4.返回结果
        return b;
    }
```

