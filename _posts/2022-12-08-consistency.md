---
layout: post
title: 分布式系统的一致性有哪些？
category: java
tags: [java,面试]
excerpt: 分布式系统的一致性有哪些？
keywords: 一致性、数据一致性
---

Hello，大家好，我是猿java。

今天我们分享的内容是：分布式系统的一致性有哪些？

## 一致性问题的来源

查阅了很多资料发现：最早研究一致性问题的场景不是分布式系而是计算机多处理器。


## 线性一致性

线性一致性，也叫强一致性（Strong Consistency）、原子一致性（Atomic Consistency）、立即一致性（Immediate Consistency）和外部一致性（External Consistency），它是由 Maurice P. Herlihy  和 Jeannette M. Wing 在 1987 年的论文 “Axioms for Concurrent Objects” 中提出的。

线性一致性是指，数据的写顺序或者读顺序都是按照时间线来从旧往新发展，不会出现旧覆盖新的问题。

对于写操作， 两个任一的写操作write1，write2：

- 如果 write1，write2无法保证线性，可能会出现 write1 覆盖 write2，也可能出现 write2 覆盖 write1；
- 如果 write1 先于 write2完成，那么 write2 一定覆盖 write1；

对于读操作：

- write 写操作完成后，所有的 read操作能拿到最新的值；
- 所有的 read必须读取到一样的顺序；


为了更好的说明线性一致性，下面通过一张示意图来进行说明：

![img.png](https://yuanjava.cn/assets/md/java/Strong-Consistency.png)

write1，write2 按照时间线操作，这样就保证了线性一致性，因此，在 write2 之后所有的 read 操作读到的 a = 2。



关于线性一致性的和作者说明如下：

论文连接：[Axioms for Concurrent Objects.pdf](https://yuanjava.cn/assets/md/java/HerlihyWing87a.pdf)

> 作者介绍
>
> [Maurice Herlihy](https://cs.brown.edu/~mph/): 生于 1954 年 1 月 4 日，是一位活跃于多处理器同步领域的计算机科学家，拥有哈佛大学数学学士学位和博士学位。
> 他在麻省理工学院获得计算机科学博士学位，曾在卡内基梅隆大学任教，并在 DEC 剑桥研究实验室任职。
> 他是 2003 年 Dijkstra 分布式计算奖、2004 年理论计算机科学哥德尔奖、2008 年 ISCA 影响力论文奖、2012 年Edsger W. Dijkstra 奖和 2013 年 Wallace McDowell 奖的获得者。他获得了 2012 年富布赖特自然科学和工程讲座奖学金，他是ACM 院士、美国国家发明家院士、美国国家工程院院士和美国国家艺术与科学学院院士。2022年，他第三次获得Dijkstra奖。

作者照片如下：

![img.png](https://yuanjava.cn/assets/md/java/Maurice-Herlihy.png)

>[Jeannette M. Wing](https://www.cs.columbia.edu/~wing/)，美国人，于 1979 年 6 月在麻省理工学院获得电气工程和计算机科学专业的 SB 和 SM，
> 哥伦比亚大学计算机科学，系研究执行副总裁
> 哥伦比亚大学计算机科学系
> 卡内基梅隆大学计算机科学系计算机科学兼职教授

作者照片如下：

![img.png](https://yuanjava.cn/assets/md/java/Jeannette-M-Wing.png)


## 顺序一致性

顺序一致性模型（Sequential Consistency），它是 Leslie Lamport 在 1979 年发表的论文 [How to Make a Multiprocessor Computer That Correctly Executes Multiprocess Program](https://lamport.azurewebsites.net/pubs/lamport-how-to-make.pdf) 中提出的，在论文中具体的定义如下：


> [Leslie Lamport](http://lamport.org/)， 1941 年 2 月 7 日（81 岁）出生于莱斯利湾兰波特，美国人，2013 年图灵奖的获得者，他是分布式共识算法 [Paxos](https://www.yuanjava.cn/posts/paxos-protocol/) 的提出者，在分布式起着举足轻重的作用。作者照片如下：

![img.png](https://yuanjava.cn/assets/md/java/Leslie-Lamport.png)

## 因果一致性

因果一致性模型（Causal Consistency）是 Mustaque Ahamad, Gil Neiger, James E. Burns, Prince Kohli, Phillip W. Hutto 在 1991 年发表的论文 “Causal memory: definitions, implementation, and programming” 中提出的一种一致性强度低于顺序一致性的模型。 在这里，我们依然从读写操作的维度来进行描述。

对于写操作来说，任意两个写操作 x1 和 x2：
- 如果两个写操作没有因果关系，那么写 x1 操作在写 x2 开始前完成，有的节点是 x1 覆盖 x2，有的节点则 x2 可能覆盖 x1；
- 如果两个写操作有因果关系，即同一台机器节点先写 x1，或者先看到 x1 然后再写 x2，则所有节点必须用 x2 覆盖 x1。

对于读操作来说：
- 如果写操作 x2 覆盖 x1 完成，那么如果一个客户端到 x2 后，它就无法读取到 x1 了，但是这个时候，其他的客户端还可以观察到 x1。

相对于顺序一致性来说，因果一致性在一致性方面有两点放松：
- 对于写操作，对没有因果关系的非并发写入操作，不仅不要求按时间排序，还不再要求节点之间的写入顺序一致了；
- 对于读操作，由于对非并发写入顺序不再要求一致性，所以自然也无法要求多个客户端必须观察到一样的顺序。

## 最终一致性

最终一致性模型（Eventual Consistency）是 Amazon 的 CTO Werner Vogels 在 2009 年发表的一篇论文 “Eventual Consistency” 里提出的，它是 Amazon 基于 Dynamo 等系统的实战经验所总结的一种很务实的实现，它不同于前面几种由大学计算机科学的教授提出的一致性模型，所以也没有非常学院派清晰的定义，但是我们依然可以从读写操作的维度来描述它。

对于同一台机器的两个写操作 x1 和 x2 来说：如果写 x1 操作在写 x2 开始前完成，那么所有节点在最终某时间点后，都会用 x2 覆盖 x1。

对于读操作来说：
- 在数据达到最终一致性的过程中，客户端的多次观察可以看到的结果是 x1 和 x2 中的任意值；
- 在数据达到最终一致性的过程后，所有客户端都将只能观察到 x2。我们可以看出来，“最终”是一个模糊的、不确定的概念，它是没有明确上限的，Vogels 提出这个不一致的时间窗口可能是由通信延迟、负载和复制次数造成的，但是最终所有进程的观点都一致，这个不一致的时间窗口可能是几秒也可能是几天。

所以，最终一致性是一个一致性非常低的模型，但是它能非常高性能地实现，在一些业务量非常大，但是对一致性要求不高的场景，是非常推荐使用的。


## 总结

首先，我们讨论了一致性问题最早出现在多路处理器的场景，现在在分布式系统中广泛出现。同时，我们还得出了一个结论：对一致性模型进行分级是正确性和性能之间的一个权衡。

接着，我们从一致性模型强弱的维度，讨论了四个经典一致性模型的定义与差异，这里我们再从其他的维度描述一下，让你对一致性模型有一个更立体的理解。

第一，现在可以实现的一致性级别最强的是线性一致性，它是指所有进程看到的事件历史一致有序，并符合时间先后顺序, 单个进程遵守 program order，并且有 total order。

第二，是顺序一致性，它是指所有进程看到的事件历史一致有序，但不需要符合时间先后顺序, 单个进程遵守 program order，也有 total order。

第三，是因果一致性，它是指所有进程看到的因果事件历史一致有序，单个进程遵守 program order，不对没有因果关系的并发排序。

第四，是最终一致性，它是指所有进程互相看到的写无序，但最终一致。不对跨进程的消息排序。


## 鸣谢
如果你觉得本文章对你有帮助，感谢转发给更多的好友，关注我：猿java，为你呈现更多的硬核文章。

<img src="https://yuanjava.cn/assets/img/pub.jpg" alt="drawing" style="width:300px;"/>
