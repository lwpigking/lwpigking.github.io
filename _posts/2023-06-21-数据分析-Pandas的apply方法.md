---
layout: post
title: 数据分析-Pandas的apply方法
categories: [数据分析]
description: 数据分析
keywords: 数据分析
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

```python
# apply方法
# apply方法常用语DataFrame的行迭代或者列迭代。
# apply的参数往往是一个以序列为输入的函数
# 例如对于.mean()，使用apply可以如下写出：
df_demo = df[["Height", "Weight"]]
def my_mean(x):
    res = x.mean()
    return res
df_demo.apply(my_mean)

# 同样的，可以利用lambda表达式使得书写简介。
df_demo.apply(lambda x:x.mean())
```

