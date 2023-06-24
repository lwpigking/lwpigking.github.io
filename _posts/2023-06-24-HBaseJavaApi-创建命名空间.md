---
layout: post
title: HBaseJavaApi-创建命名空间
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
     * 创建命名空间
     * @param namespace 命名空间名称
     */
    public static void createNamespace(String namespace) throws IOException {
        // 1.获取admin
        // 此处异常先不抛出，最后统一处理
        // admin连接：轻量级，不是线程安全的，不推荐池化或者缓存该连接
        Admin admin = connection.getAdmin();

        // 2.调用方法创建命名空间
        // 代码相对shell更底层，所以shell能够实现的功能，代码一定能实现
        // 所以需要填写完整的命名空间描述

        // 2.1 创建命名空间描述的建造者 => 设计师
        NamespaceDescriptor.Builder builder = NamespaceDescriptor.create(namespace);

        // 2.2 给命名空间添加需求
        builder.addConfiguration("user", "atguigu");

        // 2.3 使用builder构造出对应的添加完参数的对象
        // 创建命名空间出现的问题，都属于本方法自身问题
        try {
            admin.createNamespace(builder.build());
        } catch (IOException e) {
            System.out.println("命名空间已经存在");
            e.printStackTrace();
        }

        // 3.关闭admin
        admin.close();

    }
```

