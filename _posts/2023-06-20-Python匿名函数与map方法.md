---
layout: post
title: Python匿名函数与map方法
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
# 有一些函数的定义具有清晰简单的映射关系，例如my_func函数，这时候可以用匿名函数的方法简洁表示
my_func = lambda x: 2*x
my_func(3)

multi_para_func = lambda a, b: a + b
multi_para_func(1,2)
```

```python
# 但上面的用法其实违背了匿名的含义，事实上它往往在无需多处调用的场合进行使用，例如上面列表推导式中的例子，用户不关心函数名字，只关心映射关系。
[(lambda x: 2*x)(i) for i in range(5)]

# 对于上述的这种列表推导式的匿名函数映射，Python中提供了map函数来完成，它返回的是一个map对象，需要通过list转为列表
list(map(lambda x: 2*x, range(5)))

# 对于多个输入值的映射函数，可以通过追加迭代对象实现
list(map(lambda x, y: str(x) + "_" + y, range(5), list("abcde")))
```

