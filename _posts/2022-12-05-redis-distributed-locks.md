---
layout: post
title: Redis 分布式锁
category: java
tags: [java,面试]
excerpt: Redis 分布式锁
keywords: redis、锁、分布式锁、redis分布式锁
---

你好，我是猿java。

今天我们分享的内容是：Redis 分布式锁。

## Redis 部署方式

### 单机部署

单机部署，顾名思义，只部署一个Redis节点，其优点是简单，简单，简单；缺点也很明显：无法保证高可用。部署图如下：

![img.png](http://127.0.0.1:4000/assets/md/redis/redis-single.png)


### 主从部署

主从部署如下图，只能有一个 master节点和 n个 slave节点，其中slave也可以有更多的slave节点，为了保证高可用，通常会采用哨兵部署方式，当 master出现异常时，可以自动从slave中选举新的 master。

![img.png](http://127.0.0.1:4000/assets/md/redis/redis-master-slave.png)

### 集群部署

集群部署方式如下图，多个 master节点保存整个集群中的全部数据，而 Redis cluster集群中每个master节点负责不同的slot范围，整个集群包含 16384个slot。

![img.png](http://127.0.0.1:4000/assets/md/redis/redis-cluster.png)

## Redis分布式锁原理

Redis锁主要利用 Redis 的 SETNX（**SET** if **N**ot e**X**ists）命令，表示：如果 key 不存在，才会设置 value，否则不做任何操作。

- 加锁命令：SETNX key value，当 key不存在时，对 key进行设置操作并返回成功，否则返回失败。
- 解锁命令：DEL key，通过删除键值对释放锁，以便其他线程可以通过 SETNX 命令来获取锁。
- 锁超时：EXPIRE key timeout，设置 key 的超时时间，以保证即使锁没有被显式释放，锁也可以在一定时间后自动释放，避免资源被永远锁住。

加锁，解锁核心伪代码如下：
```java
if (SETNX(key, value) == 1) {
    EXPIRE(key, 30)
    try {
        // 业务逻辑

    } finally {
        DEL(key)
    }
}
```

## 单机部署-Redis分布式锁


## 主从部署-Redis分布式锁

## 集群部署-Redis分布式锁




## 鸣谢
如果你觉得本文章对你有帮助，感谢转发给更多的好友，关注我：猿java，为你呈现更多的硬核文章。

<img src="https://yuanjava.cn/assets/img/pub.jpg" alt="drawing" style="width:300px;"/>
