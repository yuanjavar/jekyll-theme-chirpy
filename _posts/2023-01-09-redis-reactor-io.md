---
layout: post
title: Redis 是如何完美驾驭 Reactor模型和 IO多路复用机制？
category: arch
tags: [Redis,面试]
excerpt: Redis 是如何完美融合 Reactor模型和 IO多路复用机制？
keywords:  Redis、Reactor模型、IO多路复用机制
---

Hello，大家好，我是猿java。



## Reactor模型

Reactor 模型，它是一种优秀的编程模型，主要是用于服务器端处理高并发网络 IO 请求。Reactor 模型的核心思想包含下面 2个"3种"：

- 3种事件：连接事件、写事件、读事件；
- 3种角色：reactor、acceptor、handler；

在讲解 Reactor模型和2个"3种"的关系之前，我们先看一个真实的网上购物场景：
- 用户在电脑的购物网站上输入商品名，然后搜索商品；
- 网站的服务器端经过各种逻辑处理来筛选商品；
- 用户电脑端看到了自己查询的商品，然后下单购买；

![img.png](http://127.0.0.1:4000/assets/md/redis/redis-shop.png)

上述购物流程应该是大家日常相当熟悉的过程，因此我们就用这个熟悉得不能再熟悉的过程来阐述Reactor 模型。

### 事件

- 用户要想搜索商品，首先浏览器要和服务器建立一个连接，也就是客户端要向服务器端发送连接请求，该请求对应了服务器的一个**连接事件**；
- 连接建立后，用户输入的关键字等内容需要发送到服务端，因此客户端需要给服务器端发送读请求，服务器端收到请求，需要从请求中读取内容，这就对应了服务器的**读事件**；
- 用户能够看到搜索结果，其实是服务器经过处理后，把结果值返回给客户端，这就对应了服务器的**读事件**；

Reactor 模型种的 3种事件可以抽象成下图：

![img.png](http://127.0.0.1:4000/assets/md/redis/redis-event.png)


### 角色

上面我们分析了事件和 Reactor 的关系，对于每种事件都需要有一个专门的负责人，在 Reactor 模型中，这个负责人就是角色，其说明如下：

- 连接事件由 acceptor 来处理，只负责接收连接；acceptor 在接收连接后，会创建 handler，用于网络连接上对后续读写事件的处理；
- 读写事件由 handler 处理，处理完后响应客户端；
- 在连接事件、读写事件会同时发生的高并发场景中，需要reactor 角色专门监听和分配事件：连接事件交由 acceptor 处理；读写请求交由 handler 处理；

Reactor 模型种的 3种角色可以抽象成下图：

![img.png](http://127.0.0.1:4000/assets/md/redis/redis-reactor.png)

### Reactor 线程模型


## 总结




## 鸣谢
如果你觉得本文章对你有帮助，感谢转发给更多的好友，关注我：猿java，为你呈现更多的硬核文章。

<img src="https://yuanjava.cn/assets/img/pub.jpg" alt="drawing" style="width:300px;"/>
