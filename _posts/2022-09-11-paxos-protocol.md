---
layout: post
title: 分布式算法：Paxos 是如何达成共识的？
category: arch
tags: [架构,distributed-algorithm,分布式算法]
description: 分布式算法：Paxos 是如何达成共识的？
keywords: 分布式算法,Paxos算法,共识算法,Basic Paxos,Multi-Paxos
---

你好，我是Weiki，欢迎来到猿java。

提到分布式算法，就不得不说 Paxos算法，曾在一段时间里，Paxos 几乎成了分布式共识的代名词，现如今很多流行的算法，
比如：ZAB，Raft 都是基于 Paxos 改进而来，这足以看出 Paxos的重要性。
但是，由于 Paxos算法比较晦涩难懂，令很多人望而却步。今天我们就来讨论这个为分布式共识算法奠定了基石的开山之作。


> 申明：
>
> 本文的理解是基于作者的原作：https://lamport.azurewebsites.net/pubs/paxos-simple.pdf
>
> 因为 Paxos算法涉及到很多专有名词，所以本文尽量保持专有名词的英文格式而非翻译后的中文格式。

## 1. Paxos算法是什么

Paxos 发音：[ˈpæksoʊs]，它是由2013年图灵奖获得者，美国科学家 Leslie Lamport(莱斯利·兰伯特)发明的。

首先，Paxos是一种很抽象的学术性算法，作者 Leslie Lamport 本人也是连发三篇文章，以不同的形式来向读者传达其精髓，但是结果并不理想。最终，因为 Google 的行业影响力，才把 Paxos推向了大众的视野。

其次：Paxos是一种共识算法，主要是用于解决分布式系统中，多个节点如何达成一致的问题。比如：从5个人里面选一个Leader，一致性算法就是要保证要么顺利选出一个Leader，
要么因为异常选不出 Leader，而不会选出2个或多个 Leader。

最后：本文章是抽丝剥茧，尽量把学术性的晦涩以更生动形象的图文方式展示。

莱斯利·兰伯特 描述的 Paxos算法包括两个部分：

- Basic Paxos，也叫做 single-decree Paxos，解决的对单值提案达成共识问题；
- Multi-Paxos，主要是分析如何通过执行多个Basic Paxos，解决一系列值达成共识问题；

## 2. Basic Paxos

### 2.1 三种角色

Basic Paxos算法存在三种角色：Proposer(提议者)、Acceptor(决策者/接受者)、Learner(学习者)。3种角色之间的关系如下图：

- Proposer(提议者)：Proposal(提案)的发起者。

- Acceptor(接受者)：接收和响应 Proposer(提议者)的PrepareRequest 和 AcceptRequest 请求，参与 Proposer(提议者) 提案的决策。

- Learner(学习者)：接受和保存 Proposal(提案)结果，不参与提案和决策过程。

![img.png](https://yuanjava.cn//assets/md/framework/paxos-nodes.png)


### 2.2  如何达成共识

在分析如何达成共识之前，先看一张 Paxos算法核心工作流程图：

![img.png](https://yuanjava.cn//assets/md/framework/paxos-algorithm.png)

从核心工作流程图可以看出，Basic Paxos算法是分3个阶段来工作：前 2 个阶段围绕某个值达成共识，第 3 个阶段是将共识结果通知所有的副本。
作者 莱斯利·兰伯特使用 Proposal[n,v] 代表一个提案,其中 n 为提案编号，v 为提议值。整个流程可以描述如下：

<font color=red size=3>目标：Proposer 需要对 V(default) 达成共识</font>

- **Prepare phase(准备阶段)**:

Proposer(提议者) 生成提案编号n，然后向所有的 Acceptor(接受者) 广播 PrepareRequest[n] 准备请求，
Acceptor(接受者) 接收到 PrepareRequest[n] 准备请求后，会做如下处理：
```java
   if (n > max(rx'd PrepareRequest)){
       if(has accept proposal[m,w]){
           promise(承诺)不再响应小于编号n的 PrepareRequest;
           promise(承诺)不再 accept(接受)小于等于编号m的提案值;
           return PrepareResponse[Proposal[m, w]];
       }else {
           promise(承诺)不再响应小于编号n的 PrepareRequest;
           promise(承诺)不再 accept(接受)小于等于编号n的的提案值;
           return PrepareResponse[None];
       }
   }else {
      丢弃请求,不做响应
   }
```
- **Accept phase(接受阶段)**

当 Proposer(提议者) 收到了 大多数 Acceptor(接受者)的 PrepareRequestResponse 响应后，会做如下的操作：

```java
if(PrepareResponse has acceptValue]){
    set v = max number's acceptValue
}else {
  set v = Vdefault
}
broadcast AcceptRequest[n,v];
```

Acceptor(接受者) 在收到大多数 Acceptor的 AcceptResponse[m,w] 响应后，会做如下的操作：

```java
if(AcceptResponse has any rejections){
    set v = max rejections number's acceptValue
    Re-establish consensus with v; // 重新对reject中的值进行共识
}else {
    consensus success and commit consensus value; // 共识成功，通知其他的副本
}

```

- **Commit Phase(提交阶段)**

 当集群对提案值v 达成共识后，需要通知其他所有副本。


下面通过 一个由节点1，节点2，节点3，节点4，节点5 五个节点组成的集群 进行详细描述：

**场景1：只有一个Proposer(提议者)**

客户端A 向节点1发起"设置name=猿java"的请求。因此，节点1被提为 Proposer(提议者)，集群中的所有节点(包括节点1)称为“决策者”，如下图：

![img.png](https://yuanjava.cn//assets/md/framework/paxos-one-proposer.png)



**准备阶段**

Proposer(提议者) 也就是节点1，生成提案编号n，并向所有的 Acceptor(接受者) 广播 PrepareRequest[n] 请求，当 Acceptor(接受者) 接收到 PrepareRequest[n]请求后，会作出如下响应：

- 由于节点1、节点2、节点3之前未响应任何 PrepareResponse 和 AcceptResponse 请求。因此，在收到 PrepareRequest[n] 请求后，会直接给出一个 PrepareResponse[n,none] 的响应，并且承诺不会接受比编号n更小的提案。
- 此时，节点4、节点5 未对 PrepareRequest[n] 做出响应。

准备阶段后，各节点的提案编号和提案值分别为：

| 节点名  | 节点1 | 节点2 | 节点3 | 节点4 | 节点5 |
|---|-----|-----|-----|-----|-----|
| 提议编号  | n   | n   | n   | 无   | 无   |
| 提议值  | 无   | 无   | 无   | 无   | 无   |

> 注意： 准备阶段，只需要向决策者发送提案编号，不需要发送提案值；


**接受阶段**

- Proposer(提议者) 在收到大多数 Acceptor 的 PrepareResponse 响应后，判断 响应的编号都是n，因此 向所有的 Acceptor 广播 AcceptRequest[n,猿java] 的请求；
- 由于节点1、节点2、节点3之前未响应任何的 AcceptResponse请求，因此收到 PrepareRequest[n,猿java] 请求后，会直接给出一个 AcceptResponse[n,猿java] 的响应，并且承诺，不再accept 比编号n更小的值；
- 此时，Proposer(提议者) 收到了 节点4的 PrepareResponse[n,none] 响应，因为 Proposer(提议者) 已经对编号n 广播了AcceptRequest，所以该响应会被直接忽略；
- Proposer(提议者) 收到了 大多数 Acceptor(接受者) 的 AcceptResponse[n,猿java] 响应后，会对响应中的 number 进行判断，发现number == n，因此 对"value = 猿java" 达成共识；

因此接受阶段过后，各个节点的提议编号和提议值如下表：

| 节点名  | 节点1       | 节点2       | 节点3  | 节点4 | 节点5 |
|---|-----------|-----------|------|----|-----|
| 提议编号  | n         | n         | n    | n  | n   |
| 提议值  | [n,猿java] | [n,猿java] | [n,猿java] | 无  | 无   |

**提交阶段**

前面2个阶段已经对"value = 猿java" 达成共识，因此需要将共识结果告知所有的副本，因此提交阶段过后，各节点的提议编号和提议值如下表：

| 节点名  | 节点1       | 节点2       | 节点3  | 节点4 | 节点5 |
|---|-----------|-----------|------|---|---|
| 提议编号  | n         | n         | n    | n | n |
| 提议值  | [n,猿java] | [n,猿java] | [n,猿java] | [n,猿java]  | [n,猿java]  |


**场景2：多个Proposer(提议者)**

客户端A 向节点1发起"设置name=猿java" 请求，同时客户端B 向节点5发起"设置name=猿" 的请求。因此，节点1和节点5都被提为 Proposer(提议者)，集群中的所有节点(包括节点1)称为“决策者”，如下图：

![img.png](https://yuanjava.cn//assets/md/framework/paxos-several-proposer.png)

**准备阶段**

Proposer(提议者) 节点1生成提案编号n，节点5生成提案编号m，m > n。并分别向所有的 Acceptor(接受者) 广播 PrepareRequest[n]请求，当 Acceptor(接受者) 接收到 PrepareRequest[n]请求后，会作出如下响应：

- 由于节点1、节点2之前未做 PrepareResponse 和 AcceptResponse 响应，在收到 PrepareRequest[n] 请求后，会直接给出一个 Proposer(提议者)节点1 PrepareResponse[n,none] 的响应，并且承诺不会接受比编号n更小的提案。
- 由于节点3、节点4、节点5之前未做 PrepareResponse 和 AcceptResponse 响应，在收到 PrepareRequest[m] 请求后，会直接给出一个 Proposer(提议者)节点5 PrepareResponse[m,none] 的响应，并且承诺不会接受比编号m更小的提案。
- 此时，节点3在收到 PrepareRequest[n] 请求后，因为响应并承诺过不会响应小于m的请求，所以该请求会被直接丢弃。

准备阶段后各个节点的提案编号和提案值分别为：

| 节点名  | 节点1 | 节点2 | 节点3 | 节点4 | 节点5 |
|---|-----|-----|-----|-----|-----|
| 提议编号  | n   | n   | m   | m   | m   |
| 提议值  | 无   | 无   | 无   | 无   | 无   |

> 注意： 准备阶段，只需要向决策者发送提案编号，不需要发送提案值；


**接受阶段**

Proposer(提议者)节点5 收到大多数的 PrepareResponse 响应后，会对响应里面的 number、value进行判断。发现所有的 number=n，value=none。
因此 把value设置成v=Vdefault=猿，向所有的 Acceptor(接受者) 广播 AcceptRequest[m,v] 请求，Acceptor(接受者) 收到请求后，作如下操作：

- 由于节点3、节点4、节点5之前未做 AcceptResponse 响应，在收到 PrepareRequest[m,v] 请求后，会直接给出一个 AcceptResponse[m,v] 的响应。
- Proposer(提议者) 收到了 大多数 Acceptor(接受者) 的 AcceptResponse后，会对响应中的 number 进行判断，如果 number == m，达成共识，否则，重新 对新的value进行共识。

因此接受阶段过后，各个节点的提议编号和提议值如下表：

| 节点名  | 节点1 | 节点2 | 节点3  | 节点4  | 节点5  |
|---|-----|-----|------|------|------|
| 提议编号  | n   | n   | n    | n    | n    |
| 提议值  | 无   | 无   | [n,猿] | [n,猿] | [n,猿] |

**提交阶段**

前面2个阶段已经对"value = 猿java" 达成共识，因此需要将共识结果告知所有的副本，因此提交阶段过后，各节点的提议编号和提议值如下表：

| 节点名  | 节点1   | 节点2   | 节点3   | 节点4 | 节点5 |
|---|-------|-------|-------|--|---|
| 提议编号  | m     | m     | m     | m | m |
| 提议值  | [n,猿] | [n,猿] | [n,猿] | [n,猿] | [n,猿]  |



## 3. Multi-Paxos

在上面的「多个Proposer」案例中，当集群中出现2个或者2个以上的Proposer时，会导致共识失败，需要重新共识，针对这些问题，就需要 Multi-Paxos，它是基于 Basic Paxos 改进而来，主要的改进点为：

- 只能有一个提议者，角色为Leader，解决了多个提议者同时提交提案带来的冲突甚至活锁的问题；
- 可以对一系列值达成一致共识；

核心流程图如下：

![img.png](https://yuanjava.cn//assets/md/framework/mulit-praxos.png)

通过上图，我们可以看到，引入Leader(主节点)后，整个共识过程就变得很简单，整个集群只有 Leader节点 能够发起提案，此时系统若要对某个值达成一致，只需要进行一次批准的交互即可。
可是，Leader(主节点)是如何选择出来的呢？在作者的文章中并没有详细的说明，不过，Raft算法是对 Multi-Paxos思想经典的实现，会在后续的文章中进行深入讲解。


## 5. 总结

- Paxos 算法分 Basic Paxos算法 和 Multi-Paxos思想 两种；
- Basic Paxos 只能对单个值进行共识来保证过程是安全的(Safety)；
- Basic Paxos 有Proposer、Acceptor、Learner 3种角色；
- Basic Paxos 可以存在多个Proposer，可能存在冲突甚至活锁；
- Basic Paxos 是通过二阶段提交来达成共识的；
- Basic Paxos 需要2轮 RPC 通讯，往返消息多、耗性能、延迟大；
- Multi-Paxos 只有一个提议者，即Leader；
- Multi-Paxos 有 Leader 和 Follower 2种角色；
- Multi-Paxos 可以对一系列值达成一致共识；
- 生产中的大多数是（n - 1）/2 个节点且n 一般为3，5，7...奇数个节点；



## 6. Paxos的生产使用案例

这里值列列举了4个大厂的代表性案例，还有很多其他的案例并未列举

- Google的 Chubby分布式锁服务中使用了Paxos算法
- Microsoft在Bing的Autopilot集群管理服务和 Windows Server Failover Clustering 中使用 Paxos
- Apache Cassandra NoSQL 数据库仅将 Paxos 用于 轻量级事务功能
- Amazon Elastic Container Services 使用 Paxos 维护集群状态的一致视图


## 7. 参考文献

[Paxos Algorithm](https://lamport.azurewebsites.net/tla/paxos-algorithm.html)

[Paxos Made Simple](https://lamport.azurewebsites.net/pubs/paxos-simple.pdf)

[The Part-Time Parliament](http://lamport.azurewebsites.net/pubs/pubs.html#lamport-paxos)

[马丁福勒 paxos](https://martinfowler.com/articles/patterns-of-distributed-systems/paxos.html)

[wiki-Paxos](https://en.wikipedia.org/wiki/Paxos_(computer_science))

[paxos-systems](https://paxos.systems/)



## 鸣谢
如果你觉得本文章对你有帮助，感谢转发给更多的好友，关注我：猿java，为你呈现更多的硬核文章。

<img src="https://yuanjava.cn/assets/img/pub.jpg" alt="drawing" style="width:300px;"/>

