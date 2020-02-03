---
title: "DRF调度算法"
category: 算法
mathjax: true
---

DRF算法是 Dominant Resource Fairness 的简称，意思是主资源公平分配，它是多纬度资源分配算法。

<!--more-->

# Max-Min Fairness(MMFS)
要介绍DRF算法前，不得不先介绍 Max-Min Fairness 算法，Max-Min Fairness 算法解决了单纬度资源公平分配的问题，而DRF算法则是基于 Max-Min Fairness 延伸出的多维度资源调度算法。

该算法会为每个用户定义了最低资源要求(min)和资源上限(max)。如果每个用户提交任务的资源需求量还未达到上限，则照常分配任务即可。当某个用户提交任务的资源需求总量大于分配给该用户的资源上限时，我们需要使用 Max-Min Fairness 算法来调度其它空闲用户的资源暂时借给这个忙用户使用，从而提高集群资源使用率。

Max-Min Fairness 算法资源分配遵循以下规则：

* 分配资源顺序与资源申请的提交顺序一致。
* 出现多个不能满足资源需求的用户时，分配给这些用户的资源量相等。
* 用户获取到的资源不能大于该用户提交的资源需求量。

* 单一资源公平性：对于单一资源，解决方案应降低到最大 - 最小公平性。
* 瓶颈公平性：如果每个用户都有一个百分比要求的资源，那么解决方案应该降低到该资源的最大 - 最小公平性。
* 人口单调性：当用户离开系统并放弃其资源时，其余用户的分配都不会减少。
* 资源单调性：如果将更多资源添加到系统，则现有用户的任何分配都不应减少。

假设把集群资源分配给

# Dominant Resource Fairness(DRF)
__主资源__指的是用户提出的资源需求占集群资源容量百分比最大的那个纬度的资源，不同用户的主资源纬度可能不一样。假设集群资源有4核CPU+12GB内存，有两个用户向集群申请资源，用户A申请"1核CPU+6GB内存"，用户B申请"3核CPU+1GB内存"，那么用户A来说主资源纬度就是内存，主资源值是6/12=0.5，用户B的主资源纬度是CPU，主资源值为1/4=0.25。先计算出各个用户资源申请的主资源值，然后直接对主资源值执行Max-Min Fairness算法就是DRF算法。

# 参考文献
* CSDN博文: https://blog.csdn.net/pelick/article/details/19326865
* [Dominant Resource Fairness: Fair Allocation of Multiple Resource Types](http://static.usenix.org/event/nsdi11/tech/full_papers/Ghodsi.pdf)
* [Computer Networks : Performance and Quality of Service](https://www.ece.rutgers.edu/~marsic/books/CN/book-CN_marsic.pdf) - 第五章