---
layout: post
title: Python的zip对象与enumerate方法
categories: [Python]
description: Python
keywords: Python
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

```python
# zip函数能够把多个可迭代对象打包成一个元组构成的可迭代对象，它返回了一个zip对象，通过tuple,list可以得到相应的打包结果
L1, L2, L3 = list("abc"), list("def"), list("hij")
list(zip(L1, L2, L3))
# 返回结果
# [('a', 'd', 'h'), ('b', 'e', 'i'), ('c', 'f', 'j')]

# 往往会在循环迭代的时候使用到zip函数
for i, j, k in zip(L1, L2, L3):
    print(i, j, k)
# 返回结果
# a d h
# b e i
# c f j
```

```python
# enumerate是一种特殊的打包，它可以在迭代时绑定迭代元素的遍历序号：
L = list("abcd")
for index, value in enumerate(L):
    print(index, value)
# 返回结果
# 0 a
# 1 b
# 2 c
# 3 d

# 用zip对象也能够简单地实现这个功能
for index, value in zip(range(len(L)), L):
    print(index, value)
# 返回结果
# 0 a
# 1 b
# 2 c
# 3 d
```

```python
# 当需要对两个列表建立字典映射时，可以利用zip对象
dict(zip(L1, L2))

# 既然有了压缩函数，那么python也提供了*操作符合zip联合使用来进行解压操作
zipped = list(zip(L1, L2, L3))
list(zip(*zipped))
```

