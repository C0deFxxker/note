---
title: "DRF调度算法"
category: 算法
mathjax: true
---

DRF算法是 Dominant Resource Fairness 的简称，意思是主资源公平分配。它是一个多纬度资源分配算法，__主资源__指的是参与调度的任务资源需求量占集群资源容量百分比最大的那个纬度的资源。

<!--more-->

# Max-Min Fairness
要介绍DRF算法前，不得不先介绍 Max-Min Fairness 算法，Max-Min Fairness 算法解决了单纬度资源公平分配的问题，而DRF算法则是基于 Max-Min Fairness 延伸出的多维度资源调度算法。

该算法会为每个用户定义了最低资源要求(min)和资源上限(max)。如果每个用户提交任务的资源需求量还未达到上限，则照常分配任务即可。当某个用户提交任务的资源需求总量大于分配给该用户的资源上限时，我们需要使用 Max-Min Fairness 算法来调度其它空闲用户的资源暂时借给这个忙用户使用，从而提高集群资源使用率。

Max-Min Fairness 算法资源分配遵循以下规则：

* 分配资源顺序与资源申请的提交顺序一致。
* 出现多个不能满足资源需求的用户时，分配给这些用户的资源量相等。
* 用户获取到的资源不能大于该用户提交的资源需求量。

假设把集群资源分配给

$a \ne 0$

# 参考文献
* CSDN博文: https://blog.csdn.net/pelick/article/details/19326865
* [Dominant Resource Fairness: Fair Allocation of Multiple Resource Types](http://static.usenix.org/event/nsdi11/tech/full_papers/Ghodsi.pdf)
* [Computer Networks : Performance and Quality of Service](https://www.ece.rutgers.edu/~marsic/books/CN/book-CN_marsic.pdf) - 第五章