---
layout: post
title: 分布式算法：Raft 是如何达成共识的？
category: arch
tags: [架构,distributed-algorithm,分布式算法]
description: Raft是如何达成共识的？
keywords: 分布式算法,Raft算法,共识算法
---

你好，我是Weiki，欢迎来到猿java。

在 [分布式算法：Paxos 是如何达成共识的？](https://www.yuanjava.cn/posts/paxos-protocol/) 这篇文章中，我们深入的讲解了 Paxos算法，知道了 Basic Paxos是如何达成共识以及存在哪些问题，
Multi-Paxos在通过引入Leader节点解决了Basic Paxos多Proposer带来的问题，但是对于如何选举Leader却没有详细的说明。因此，今天我们就来分析一种Multi-Paxos算法的经典实现-Raft 算法。

## 什么是Raft算法

Raft 是英文"**R**eliable, **R**eplicated, **R**edundant, **A**nd **F**ault-**T**olerant"（“可靠、可复制、可冗余、可容错”）的首字母缩写，
它是一种用于替代 Paxos的共识算法。相比于 Paxos，Raft 的目标是提供更清晰的逻辑分工使得算法本身能被更好地理解，同时它安全性更高，并能提供一些额外的特性。Raft 能为在计算机集群之间部署有限状态机提供一种通用方法，并确保集群内的任意节点在某种状态转换上保持一致。

## 3种角色

在Raft算法 中有 Leader(领导者)，Candidate(候选人)，Follower(追随者) 3种角色，其说明和关系如下：

**Leader(领导者)**

- 负责处理所有外部的请求，如果不是 Leader机器收到请求时，请求会被转到 Leader机器；
- 负责向 Follower(追随者) 同步请求日志；
- 负责向 其他节点 发送心跳信息；
- 同一任期，最多只能有一个 Leader；

**Candidate(候选人)**

- 主动发起让自己成为 Leader(领导者)的投票；
- 重置选举超时时间；
- 获取 大多数 Follower(追随者) 的投票后成为 Leader(领导者)；


**Follower(追随者)**

- 响应 Leader(领导者) 和 Candidate(候选人) 的RPC 请求；
- 当选举心跳超时，如果既不需要响应 RPC 又不需要给 Candidate(候选人)投票，则主动切换成 Candidate(候选人)角色；



三者的关系如下图：

![img.png](http://127.0.0.1:4000/assets/md/framework/raft-roles.png)

## 如何选举Leader

在 讲解 Raft算法是如何选举 Leader(领导者) 之前，我们先来看个班级选班长的真实实例，帮助大家理解：







## 如何复制日志


## 如何保证安全



## 总结

- 提问前一定要自己思考，主动去寻找解决方案
- 向他人提问，一定要注意态度的谦卑，因为这可能是获得回答的一个前提哦
- 向陌生人提问时，注意信息安全
- 不要幻想回答者能完全解决问题，回复只是多一些 idea，拓展解决问题的思路
- 要敢于提问，不要害怕被拒绝，成长的路上一定会遇到困难
- 要多总结提问的技巧，让提问能勾起他人的回答欲望

## 鸣谢
如果你觉得本文章有帮助，感谢转发给更多的好友，我们定将呈现更多的干货， 欢迎关注公众号：猿java

