---
layout: post
title: 分布式系统2大基石之 一致性
category: java
tags: [java,面试]
excerpt: 分布式系统2大基石之 一致性
keywords: 一致性、数据一致性
---

你好，我是猿java。

今天我们分享的内容是：新一代 Java垃圾回收神器：ZGC。

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

![img.png](http://127.0.0.1:4000/assets/md/java/Strong-Consistency.png)

write1，write2 按照时间线操作，这样就保证了线性一致性，因此，在 write2 之后所有的 read 操作读到的 a = 2。



关于线性一致性的和作者说明如下：

论文连接：[Axioms for Concurrent Objects.pdf](http://127.0.0.1:4000/assets/md/java/HerlihyWing87a.pdf)

> 作者介绍
>
> [Maurice Herlihy](https://cs.brown.edu/~mph/): 生于 1954 年 1 月 4 日，是一位活跃于多处理器同步领域的计算机科学家，拥有哈佛大学数学学士学位和博士学位。
> 他在麻省理工学院获得计算机科学博士学位，曾在卡内基梅隆大学任教，并在 DEC 剑桥研究实验室任职。
> 他是 2003 年 Dijkstra 分布式计算奖、2004 年理论计算机科学哥德尔奖、2008 年 ISCA 影响力论文奖、2012 年Edsger W. Dijkstra 奖和 2013 年 Wallace McDowell 奖的获得者。他获得了 2012 年富布赖特自然科学和工程讲座奖学金，他是ACM 院士、美国国家发明家院士、美国国家工程院院士和美国国家艺术与科学学院院士。2022年，他第三次获得Dijkstra奖。

作者照片如下：

![img.png](http://127.0.0.1:4000/assets/md/java/Maurice-Herlihy.png)

>[Jeannette M. Wing](https://www.cs.columbia.edu/~wing/)，美国人，于 1979 年 6 月在麻省理工学院获得电气工程和计算机科学专业的 SB 和 SM，
> 哥伦比亚大学计算机科学，系研究执行副总裁
> 哥伦比亚大学计算机科学系
> 卡内基梅隆大学计算机科学系计算机科学兼职教授

作者照片如下：

![img.png](http://127.0.0.1:4000/assets/md/java/Jeannette-M-Wing.png)


## 顺序一致性

顺序一致性模型（Sequential Consistency），它是 Leslie Lamport 在 1979 年发表的论文 [How to Make a Multiprocessor Computer That Correctly Executes Multiprocess Program](https://lamport.azurewebsites.net/pubs/lamport-how-to-make.pdf) 中提出的，在论文中具体的定义如下：


> [Leslie Lamport](http://lamport.org/)， 1941 年 2 月 7 日（81 岁）出生于莱斯利湾兰波特，美国人，2013 年图灵奖的获得者，他是分布式共识算法 [Paxos](https://www.yuanjava.cn/posts/paxos-protocol/) 的提出者，在分布式起着举足轻重的作用。作者照片如下：

![img.png](http://127.0.0.1:4000/assets/md/java/Leslie-Lamport.png)

## 因果一致性


## 最终一致性






## 鸣谢
如果你觉得本文章对你有帮助，感谢转发给更多的好友，关注我：猿java，为你呈现更多的硬核文章。

<img src="https://yuanjava.cn/assets/img/pub.jpg" alt="drawing" style="width:300px;"/>
