---
layout: post
title: HadoopHDFS详解
categories: [Hadoop, HDFS]
description: HadoopHDFS
keywords: Hadoop, HDFS
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

## HDFS

### 介绍

`HDFS（Hadoop Distributed File System）`是 Hadoop 下的`分布式文件系统`，具有`高容错`、`高吞吐量`等特性，可以部署在低成本的硬件上。



### HDFS 设计原理

![hdfsarchitecture](/picture/pictures/hdfsarchitecture.png)

#### 2.1 HDFS 架构

HDFS 遵循主/从架构，由`单个 NameNode(NN)` 和`多个 DataNode(DN)` 组成：

- **NameNode** : 负责执行有关 ` 文件系统命名空间 ` 的操作，例如`打开，关闭、重命名文件和目录`等。它同时还负责集群`元数据的存储`，记录着文件中各个数据块的位置信息。
- **DataNode**：负责提供来自文件系统客户端的`读写请求`，`执行块的创建，删除`等操作。



#### 2.2 文件系统命名空间

HDFS 的 ` 文件系统命名空间 ` 的层次结构与大多数文件系统类似 (如 Linux)， 支持目录和文件的创建、移动、删除和重命名等操作，支持配置用户和访问权限，但不支持硬链接和软连接。`NameNode` 负责维护文件系统名称空间，记录对名称空间或其属性的任何更改。



#### 2.3 数据复制

由于 Hadoop 被设计运行在廉价的机器上，这意味着硬件是不可靠的，为了保证容错性，HDFS 提供了`数据复制机制`。HDFS `将每一个文件存储为一系列块，每个块由多个副本来保证容错`，块的大小和复制因子可以自行配置（默认情况下，块大小是 128M，默认复制因子是 3）。

![hdfsdatanodes](/picture/pictures/hdfsdatanodes.png)

#### 2.4 数据复制的实现原理

大型的 HDFS 实例在通常分布在多个机架的多台服务器上，不同机架上的两台服务器之间通过交换机进行通讯。在大多数情况下，同一机架中的服务器间的网络带宽大于不同机架中的服务器之间的带宽。因此 HDFS 采用`机架感知副本放置策略`，对于常见情况，当复制因子为 3 时，HDFS 的放置策略是：

在写入程序位于 `datanode` 上时，就优先将写入文件的一个副本放置在该 `datanode` 上，否则放在随机 `datanode` 上。之后在另一个远程机架上的任意一个节点上放置另一个副本，并在该机架上的另一个节点上放置最后一个副本。此策略可以减少机架间的写入流量，从而提高写入性能。

![hdfs-机架](/picture/pictures/hdfs-机架.png)

如果复制因子大于 3，则随机确定第 4 个和之后副本的放置位置，同时保持每个机架的副本数量低于上限，上限值通常为 `（复制系数 - 1）/机架数量 + 2`，需要注意的是不允许同一个 `dataNode` 上具有同一个块的多个副本。



#### 2.5  副本的选择

为了最大限度地减少带宽消耗和读取延迟，HDFS 在执行读取请求时，优先读取距离读取器最近的副本。如果在与读取器节点相同的机架上存在副本，则优先选择该副本。如果 HDFS 群集跨越多个数据中心，则优先选择本地数据中心上的副本。



#### 2.6 架构的稳定性

##### 1. 心跳机制和重新复制

`每个 DataNode 定期向 NameNode 发送心跳消息`，如果超过指定时间没有收到心跳消息，则将 DataNode 标记为死亡。NameNode 不会将任何新的 IO 请求转发给标记为死亡的 DataNode，也不会再使用这些 DataNode 上的数据。 由于数据不再可用，可能会导致某些块的复制因子小于其指定值，NameNode 会跟踪这些块，并在必要的时候进行重新复制。

##### 2. 数据的完整性

由于存储设备故障等原因，存储在 DataNode 上的数据块也会发生损坏。为了避免读取到已经损坏的数据而导致错误，HDFS 提供了`数据完整性校验机制`来保证数据的完整性，具体操作如下：

当客户端创建 HDFS 文件时，它会计算文件的每个块的 ` 校验和 `，并将 ` 校验和 ` 存储在同一 HDFS 命名空间下的单独的隐藏文件中。当客户端检索文件内容时，它会验证从每个 DataNode 接收的数据是否与存储在关联校验和文件中的 ` 校验和 ` 匹配。如果匹配失败，则证明数据已经损坏，此时客户端会选择从其他 DataNode 获取该块的其他可用副本。

##### 3.元数据的磁盘故障

`FsImage` 和 `EditLog` 是 HDFS 的`核心数据`，这些数据的意外丢失可能会导致整个 HDFS 服务不可用。为了避免这个问题，可以配置 NameNode 使其支持 `FsImage` 和 `EditLog` 多副本同步，这样 `FsImage` 或 `EditLog` 的任何改变都会引起每个副本 `FsImage` 和 `EditLog` 的同步更新。

##### 4.支持快照

快照支持在特定时刻存储数据副本，在数据意外损坏时，可以通过回滚操作恢复到健康的数据状态。



### HDFS 的特点

#### 3.1 高容错

由于 HDFS 采用数据的多副本方案，所以部分硬件的损坏不会导致全部数据的丢失。

#### 3.2 高吞吐量

HDFS 设计的重点是支持高吞吐量的数据访问，而不是低延迟的数据访问。

#### 3.3  大文件支持

HDFS 适合于大文件的存储，文档的大小应该是 GB 到 TB 级别的。

#### 3.3 简单一致性模型

HDFS 更适合于一次写入多次读取 (write-once-read-many) 的访问模型。支持将内容追加到文件末尾，但不支持数据的随机访问，不能从文件任意位置新增数据。

#### 3.4 跨平台移植性

HDFS 具有良好的跨平台移植性，这使得其他大数据计算框架都将其作为数据持久化存储的首选方案。



### 附：图解HDFS存储原理

#### HDFS写数据原理

![hdfs-write-1](/picture/pictures/hdfs-write-1.jpg)

![hdfs-write-2](/picture/pictures/hdfs-write-2.jpg)

![hdfs-write-3](/picture/pictures/hdfs-write-3.jpg)



#### HDFS读数据原理

![hdfs-read-1](/picture/pictures/hdfs-read-1.jpg)

#### HDFS故障类型和其检测方法

![hdfs-tolerance-1](/picture/pictures/hdfs-tolerance-1.jpg)

![hdfs-tolerance-2](/picture/pictures/hdfs-tolerance-2.jpg)



#### **读写故障的处理**

![hdfs-tolerance-3](/picture/pictures/hdfs-tolerance-3.jpg)



#### **DataNode 故障处理**

![hdfs-tolerance-4](/picture/pictures/hdfs-tolerance-4.jpg)



#### **副本布局策略**：

![hdfs-tolerance-5](/picture/pictures/hdfs-tolerance-5.jpg)



## HDFS常用shell命令

**1. 显示当前目录结构**

```shell
# 显示当前目录结构
hadoop fs -ls  <path>
# 递归显示当前目录结构
hadoop fs -ls  -R  <path>
# 显示根目录下内容
hadoop fs -ls  /
```

**2. 创建目录**

```shell
# 创建目录
hadoop fs -mkdir  <path> 
# 递归创建目录
hadoop fs -mkdir -p  <path>  
```

**3. 删除操作**

```shell
# 删除文件
hadoop fs -rm  <path>
# 递归删除目录和文件
hadoop fs -rm -R  <path> 
```

**4. 从本地加载文件到 HDFS**

```shell
# 二选一执行即可
hadoop fs -put  [localsrc] [dst] 
hadoop fs - copyFromLocal [localsrc] [dst] 
```


**5. 从 HDFS 导出文件到本地**

```shell
# 二选一执行即可
hadoop fs -get  [dst] [localsrc] 
hadoop fs -copyToLocal [dst] [localsrc] 
```

**6. 查看文件内容**

```shell
# 二选一执行即可
hadoop fs -text  <path> 
hadoop fs -cat  <path>  
```

**7. 显示文件的最后一千字节**

```shell
hadoop fs -tail  <path> 
# 和Linux下一样，会持续监听文件内容变化 并显示文件的最后一千字节
hadoop fs -tail -f  <path> 
```

**8. 拷贝文件**

```shell
hadoop fs -cp [src] [dst]
```

**9. 移动文件**

```shell
hadoop fs -mv [src] [dst] 
```


**10. 统计当前目录下各文件大小**  

+ 默认单位字节  
+ -s : 显示所有文件大小总和，
+ -h : 将以更友好的方式显示文件大小（例如 64.0m 而不是 67108864）

```shell
hadoop fs -du  <path>  
```

**11. 合并下载多个文件**

+ -nl  在每个文件的末尾添加换行符（LF）
+ -skip-empty-file 跳过空文件

```shell
hadoop fs -getmerge
# 示例 将HDFS上的hbase-policy.xml和hbase-site.xml文件合并后下载到本地的/usr/test.xml
hadoop fs -getmerge -nl  /test/hbase-policy.xml /test/hbase-site.xml /usr/test.xml
```

**12. 统计文件系统的可用空间信息**

```shell
hadoop fs -df -h /
```

**13. 更改文件复制因子**

```shell
hadoop fs -setrep [-R] [-w] <numReplicas> <path>
```

+ 更改文件的复制因子。如果 path 是目录，则更改其下所有文件的复制因子
+ -w : 请求命令是否等待复制完成

```shell
# 示例
hadoop fs -setrep -w 3 /user/hadoop/dir1
```

**14. 权限控制**  

```shell
# 权限控制和Linux上使用方式一致
# 变更文件或目录的所属群组。 用户必须是文件的所有者或超级用户。
hadoop fs -chgrp [-R] GROUP URI [URI ...]
# 修改文件或目录的访问权限  用户必须是文件的所有者或超级用户。
hadoop fs -chmod [-R] <MODE[,MODE]... | OCTALMODE> URI [URI ...]
# 修改文件的拥有者  用户必须是超级用户。
hadoop fs -chown [-R] [OWNER][:[GROUP]] URI [URI ]
```

**15. 文件检测**

```shell
hadoop fs -test - [defsz]  URI
```

可选选项：

+ -d：如果路径是目录，返回 0。
+ -e：如果路径存在，则返回 0。
+ -f：如果路径是文件，则返回 0。
+ -s：如果路径不为空，则返回 0。
+ -r：如果路径存在且授予读权限，则返回 0。
+ -w：如果路径存在且授予写入权限，则返回 0。
+ -z：如果文件长度为零，则返回 0。

```shell
# 示例
hadoop fs -test -e filename
```
