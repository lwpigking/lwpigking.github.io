---
layout: fragment
title: centos7扩容根目录
tags: [Linux]
description: some word here
keywords: Linux
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: fals
---

在省赛比赛前一段时间，我的磁盘容量突然爆炸了，突然意识到学会扩容根目录是件很重要的事情，找到了亲测有效的方法。

源文章地址：[centos7扩容根目录（/dev/mapper/centos-root） - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/450057653)

使用虚拟机扩展磁盘后，在系统内进行分区。

1.查看分区状况

```
df -h
```

2.新建分区，并将id改为8e

```
fdisk /dev/sda
n
p
t
w
```

3.刷新并查看sda3是否存在

```
partprobe
lsblk
```

4.使用lvm命令新建卷/dev/sda3，并将其加载到卷组centos中。

```
lvm
pvcreate /dev/sda3
pvdisplay
vgdisplay
vgextend centos /dev/sda3
vgdisplay
lvextend -l +100%FREE /dev/centos/root
exit
```

5.之前只是对逻辑卷扩容，还要同步到文件系统，实现对根目录的扩容。

```
xfs_growfs /dev/centos/root
df -h
```

