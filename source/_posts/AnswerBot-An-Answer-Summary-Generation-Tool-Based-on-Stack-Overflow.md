---
title: 'AnswerBot: An Answer Summary Generation Tool Based on Stack Overflow'
date: 2019-11-05 16:55:19
tags: Q&A
category: Paper
mathjax: true
---

# 1 摘要

提出了AnswerBot，可以为技术问题自动生成答案总结

Demo tool website: http://answerbot.cn.

Demo video: https://youtu.be/EfHp_Cbeg2w.

Replication package: https://github.com/XBWer/AnswerBot.

# 2 方法

分为三个步骤：相关问题检索、答案片段选择、答案总结生成

## 2.1 相关问题检索

计算query $q$ 到Question $Q$的相关性：

$$rel(W_q \rightarrow W_Q) = \frac{\sum_{w_q \in W_q}rel(w_q, W_Q)*idf(w_q)}{\sum_{w_q \in W_q}idf(w_q)}$$

其中$idf(w_q)$是单词$w_q$的IDF，$rel(w_q, W_Q)=max_{w_Q \in W_Q}rel(w_q, w_Q)$，而$rel(w_q,w_Q)$是单词$w_q$和单词$w_Q$对应word embedding的余弦相似度

相似的也可以计算query $Q$ 到Question $q$的相关性，从而

query $q$ 和Question $Q$的相关性为：

$$rel(q,Q)=\frac{rel(W_q \rightarrow W_Q)+rel(W_Q \rightarrow W_q)}{2}$$

## 2.2 答案片段选择

以段落为单位，通过三方面指标来选择合适的段落

- 有关query的特征
  - answer与query的相关程度：设为question与query的相关程度
  - 实体重叠大小：将StackOverflow的tag作为实体，可通过$|E_q \cap E_{ap}| / |E_q|(|E_q| \neq 0)$来计算，当$|E_q| = 0$时，认为重叠大小也就是0

- 段落内容的特征
  - 信息熵：将段落中每个单词的IDF相加
  - 语义pattern：如果包含12种我们发现的语义模式之一，就将这个特征设为1，否则设为0
  - 形式pattern：如果段落包含\<strong\>and\< strike\>，就将这个特征设为1，否则设为0

- 用户导向的特征
  - 段落位置：$1/pos$，$pos$是段落的编号，当$pos>3$时，这个值为0
  - vote值：等于回答的vote值

将上述七个特征归一化后相乘作为最终的得分，取top10作为候选答案。

## 2.3 答案总结生成

利用maximal marginal relevance (MMR) 最大边界相关算法。MRR首先算出10个候选答案两两之间的相似度，然后迭代的选择有最大边界相关的五个段落形成最终答案。

# 3 评价方式

- baseline：Google、StackOverflow

- 2个博士后和6个博士分别查看100个query，并对答案从相关性、有用性和多样性三个方面进行打分（1-5分）

- 利用Wilcoxon signed-rank test检验和baseline的差异是否显著