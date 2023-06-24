---
layout: post
title: Pandas透视表pivot和pivot_table
categories: [Pandas]
description: Pandas
keywords: pandas
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

# pivot

```python
df = pd.DataFrame({'Class':[1,1,2,2],
				 'Name':['San Zhang','San Zhang','Si Li','Si Li'],
                   'Subject':['Chinese','Math','Chinese','Math'],
                   'Grade':[80,75,90,85]})
```

上表存储了张三和李四的语文和数学分数，现在想要把语文和数学分数作为列来展示。

```python
df.pivot(index='Name', columns='Subject', values='Grade')
```

变形后的行索引、需要转到列索引的列， 以及这些列和行索引对应的数值，它们分别对应了 pivot 方法中的 index, columns, values 参数。新生成表的列索引是 columns 对应列的 unique 值，而新表的行索引是 index对应列的unique值，而values对应了想要展示的数值列。

```
  Class Name      Subject Grade
0 1     San Zhang Chinese 80
1 1     San Zhang Math    75
2 2     Si Li     Chinese 90
3 2     Si Li     Math    85

#pivot后
Subject        Chinese Math
Name
San Zhang      80      75
Si Li          90      85
```

# pivot_table

pivot 的使用依赖于唯一性条件，那如果不满足唯一性条件，那么必须通过聚合操作使得相同行列组合对应 的多个值变为一个值。例如，张三和李四都参加了两次语文考试和数学考试，按照学院规定，最后的成绩是 两次考试分数的平均值，此时就无法通过 pivot 函数来完成。

```python
df = pd.DataFrame({'Name':['San Zhang', 'San Zhang','San Zhang', 'San Zhang','Si Li', 'Si Li', 'Si Li', 'Si Li'],
                   'Subject':['Chinese', 'Chinese', 'Math', 'Math','Chinese', 'Chinese', 'Math', 'Math'],
                   'Grade':[80, 90, 100, 90, 70, 80, 85, 95]})
```

```
df.pivot_table(index = 'Name',columns = 'Subject',values = 'Grade',aggfunc = 'mean')
```

 aggfunc参数是使用的聚合函数

```
  Name      Subject    Grade
0 San Zhang Chinese    80
1 San Zhang Chinese    90
2 San Zhang Math       100
3 San Zhang Math       90
4 Si Li     Chinese    70
5 Si Li     Chinese    80
6 Si Li     Math       85
7 Si Li     Math       95


#pivot_table后
Subject   Chinese Math
Name
San Zhang 85      95
Si Li     75      90
```

# 总结

两者之间就是数据是否要聚合。

pivot如果不聚合，就得确保数据的唯一性（index和columns）。

pivot_table必须聚合，自带参数aggfunc。
