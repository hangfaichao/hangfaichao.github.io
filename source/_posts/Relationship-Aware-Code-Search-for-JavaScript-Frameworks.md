---
title: Relationship-Aware Code Search for JavaScript Frameworks
date: 2019-11-10 10:04:58
tags: Information-Retrieval
category: Paper
---

提到的一些代码搜索工具Ohloh Code (https://code.openhub.net/), Krugle (http://www.krugle.com/), PARSEWeb, MAPO, SNIFF

# 1 Javascript frameworks的主要特点

- 方法的调用关系很复杂，有很多异步的API方法，经常使用回调函数来保证执行顺序
- JS常常用在浏览器的客户端，经典的query经常包含简单动作和动作之间的关系描述词（例如when，after），这些关系词可以对应API之间的顺序执行或回调执行
- Javascript frameworks经常要选择和操纵DOM元素

RACS关注三种类型的关系，sequencing（顺序）, callback（回调）, and condition（条件）

# 2 一个query例子

show a busy image while the actual image is downloading, and when
image is downloaded, the busy image is removed and actual image is be
shown there

# 3 方法

1. 爬取API使用模式。这一步线下进行，爬取使用API的JS代码片段，并用方法调用关系（MCR）图来表示。
2. 抽象自然语言query。将自然语言的query抽象成动作关系（AR）图。
3. 搜索代码片段。将所有的MCR图与AR图进行匹配，并返回前几个结果。这部分利用API文档来作为连接自然语言和API调用的代码片段的桥梁。

## 3.1 爬取API使用模式

### 3.1.1 抽取MCR图

对每一个函数，RACS首先抽取出其中的API调用语句（假设一共n个），然后按照顺序重新组织成代码片段，再将新的代码片段拆分成n*(n+1)/2个小片段（n个单句的片段，n-1个两句的片段…一个n句的片段）

对每一个代码片段，利用Rhino JavaScript engine (https://www.mozilla.org/rhino/) 进行AST解析

### 3.1.2 合并相同MCR图

将得到的若干个代码片段按照它们的模式进行分类存储，以提供后续的查询速度。

## 3.2 抽象自然语言query

### 3.2.1 预处理

用空格代替API中的'.'，并将用'_'，'-'连接的变量以及驼峰命名法的变量分开

基于领域相关的字典将缩写补全（例如'attr'变为'attribute'）

### 3.2.2 动作定位

利用工具Stanford Parser，得到POS（词性标注）和解析树，标注为VP、NP、PP的都可以认为是一个动作。

### 3.2.3 关系定位

对于相邻的两个动作，通过规则来判断他们的关系是三个中的哪一个

### 3.2.4 后处理

将动作描述中较为具体的名词（image，div）替换为更抽象普遍的名词（element），并且去掉限定词和形容词，例如the busy image变为element。同时，这些更改过的词记录下来作为keywords，以便后面使用。

## 3.3 搜索代码片段

### 3.3.1 从AR图生成A-MCR图

将AR图中的自然语言转换为API方法。通过比较AR图中的自然语言与API方法的文档描述之间的文本相似度，选择最高的K个进行替换。

### 3.3.2 通过A-MCR图搜索MCR图

通过图相似度：$sim(G_1, G_2)=\frac{mrn(G_1,G_2)}{rn(G_1)+rn(G_2)-mrn(G_1,G_2)}$，选择前K'个作为候选。其中$rn(G_i)$是$G_i$包含的关系个数，$mrn(G_1,G_2)$是$G_1,G_2$中匹配的关系个数。

### 3.3.3 选择代码片段

从上面得到的K'个候选中，各选择一个代码片段。基于两个指标：代码片段中匹配到的自然语言中的keyword个数，代码片段的长度。代码片段匹配的keyword越多，长度越少，排名越靠前。

# 4 实验

实验结果：https://sites.google.com/site/racsproject/