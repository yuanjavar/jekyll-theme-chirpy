---
layout: post
title: 从生活实例，理解 Reactor 和 Proactor 网络通信模型！
category: arch
tags: [arch,面试]
excerpt: 从生活实例，理解 Reactor 和 Proactor 网络通信模型！
keywords:  Redis、Reactor模型、IO多路复用机制
---

Hello，大家好，我是猿java。

我们都知道，科技领域的很多思维其实都是来自于现实生活，比如今天要分享的高性能网络通信模型 Reactor 和 Proactor。

## 为什么要学习 Reactor 和 Proactor

万丈高楼平地起，要想在技术上有所成就，就一定要修练好内功。随着互联网的快速发展，技术更迭的速度也是超乎想象，也许花大力气掌握的技能几年就过时了（比如 JSP，sturts/sturts2）。但是有一些东西却是历久弥新，比如：架构思想，设计思维，掌握了这些精髓，可以帮助你快速适应技术更迭。 Reactor 和 Proactor 是网络 IO 处理
中两个经典的高性能模型，学习它们可以在网络 IO 处理上有个不一样的认知。两个模型可以高度抽象为下图：

![img.png](https://yuanjava.cn/assets/md/arch/reactor-proactor.png)


## Reactor 模型

### 定义

Reactor，中文翻译为"反应器"，它是一个被动过程，可以理解为"当接收到客户端事件后，Reactor 会根据事件类型来调用相应的代码进行处理"。Reactor 模型也叫 Dispatcher 模式，底层是 I/O 多路复用结合线程池，主要是用于服务器端处理高并发网络
IO 请求。Reactor 模型的核心思想可以总结成 2个"3种"：
- 3种事件：连接事件、读事件、写事件；
- 3种角色：reactor、acceptor、handler；

### 事件

- 客户端向服务器端发送连接请求，该请求对应了服务器的一个**连接事件**；
- 连接建立后，客户端给服务器端发送写请求，服务器端收到请求，需要从请求中读取内容，这就对应了服务器的**读事件**；
- 服务器经过处理后，把结果值返回给客户端，这就对应了服务器的**写事件**；

事件的描述可以参考下图：

![img.png](https://yuanjava.cn/assets/md/redis/redis-event.png)

### 角色

上述描述了 Reactor 的事件，每个事件都需要有一个专门的负责人，在 Reactor 模型中，这个负责人就是角色，其说明如下：

- 连接事件由 acceptor 来处理，只负责接收连接；acceptor 在接收连接后，会创建 handler，用于网络连接上对后续读写事件的处理；
- 读写事件由 handler 处理，处理完后响应客户端；
- 在连接事件、读写事件会同时发生的高并发场景中，需要reactor 角色专门监听和分配事件：连接事件交由 acceptor 处理；读写请求交由
  handler 处理；

事件和角色的对应可以参考下文的 Reactor 线程模型。

### Reactor 线程模型

Reactor 线程模型有单 Reactor 单线程模型、单 Reactor 多线程模型、多 Reactor 多线程模型 三种。

**单Reactor单线程模型**

单 Reactor 单线程模型，很容易理解：接受请求、业务处理、响应请求都在一个线程中处理。

**模型抽象**

![img.png](https://yuanjava.cn/assets/md/arch/single-reactor.png)

**工作原理**

1. Reactor 通过 select函数监听事件，收到事件后通过 dispatch 分发给 Acceptor 或 Handler；
2. 如果监听到 client 的连接事件，则分发给 Acceptor 处理，Acceptor 通过 accept 接受连接，并创建一个 Handler 来处理连接后续的各种事件；
3. 如果不是连接建立事件，则 Reactor 会调用连接对应的 Handler（第 2 步中创建的 Handler）来进行响应；
4. Handler 通过： read-> 业务处理 ->send 流程完成完整业务流程；


**优缺点**

优点是简单，不存在线程竞争，缺点是无法充分利用和发挥多核 CPU 的性能，当业务耗时很长时，容易造成阻塞。

**案例**

- Redis6.0以下的版本使用的是 单 Reactor 单线程模型，因为 Redis使用的是内存，CPU不是性能瓶颈，所以单 Reactor 单线程模型可以支持 Redis 单机服务的高性能；
- Netty4 通过参数配置，可以使用单 Reactor 单线程模型；


**单Reactor多线程模型**

鉴于单 Reactor 单线程模式无法充分利用和发挥多核 CPU 的性能，于是就诞生了单 Reactor 多线程模型。

**模型抽象图**

![img.png](https://yuanjava.cn/assets/md/arch/single-reactor-more-thread.png)

**工作原理**

1. 在主线程中，Reactor 对象通过 select 监控事件，收到事件后通过 dispatch 分发给 Acceptor 或 Handler；
2. 如果监听到 client 的连接事件，则分发给 Acceptor 处理，Acceptor 通过 accept 接受连接，并创建一个 Handler 来处理连接后续的各种事件；
3. 如果不是连接建立事件，则 Reactor 会调用连接对应的 Handler（步骤2中创建的 Handler）来进行响应。注意，此模型的 Handler
   只负责响应事件，不进行业务处理；
4. Handler 通过 read 读取到数据后，会发给 Processor 进行业务处理；
5. Processor 会在独立的子线程中完成真正的业务处理，然后将响应结果发给主线程的 Handler 处理；Handler 收到响应后通过 send
   将响应结果返回给 client；

**优点**

采用了线程池来处理业务逻辑，能够充分利用多 CPU 的处理能力

**缺点**

1. 多线程数据共享和访问比较复杂。例如，子线程完成业务处理后，要把结果传递给主线程的 Reactor 进行发送，这里涉及共享数据的互斥和保护机制；
2. 尽管引进了多线程处理业务逻辑，但是事件的监听和响应还是需要 Reactor 来处理，因此，瞬间高并发可能会造成 Reactor 的性能瓶颈；

**案例**

- Netty4 通过参数配置，可以使用单 Reactor 多线程模型；

**多Reactor多线程模型**

单 Reactor 多线程模型的性能瓶颈在于单个 Reactor 的处理能力，于是我们很自然的想到：能不能增加多个 Reactor来提升性能？于是，多 Reactor 多线程模型就运应而生。

**模型抽象图**

![img.png](https://yuanjava.cn/assets/md/arch/more-reactor-more-thread.png)

**工作原理**
1. 父线程中 mainReactor 对象通过 select 监控连接建立事件，收到事件后通过 Acceptor 接收，将新的连接分配给某个子线程；
2. 子线程的 subReactor 把 mainReactor 分配的连接加入到连接队列中并进行监听，同时创建一个 Handler 用于处理连接的各种事件；
3. 当有新的事件发生时，subReactor 会调用连接对应的 Handler（步骤2中创建的 Handler）来进行响应；
4. Handler 通过： read-> 业务处理 ->send 流程完成完整业务流程；

**优点**
- 父线程和子线程的职责明确，父线程只负责接收新连接，子线程负责完成后续的业务处理；
- 父线程和子线程的交互简单，父线程只需要把新连接传给子线程，子线程无须返回数据；

**案例**

- Nginx 采用的是多 Reactor 多进程模型，但方案与标准的多 Reactor 多进程有差异；
- 开源软件 Memcache 采用的是多 Reactor 多线程模型；
- Netty4 通过参数参数配置可以使用多 Reactor 多线程模型；


到此， Reactor模型就分析完了，上文讲述的 Reactor 3种线程模型，同样可以以进程的方式部署，可能在逻辑处理上和线程有些差异。接下来再分析和 Reactor模型很类似的 Proactor 模型。

## Proactor 模型

Proactor，中文翻译为"前摄器"。乍一看，这个翻译还是挺懵的，个人觉得"主动器"更符合 Proactor 模型的本意。Proactor 可以理解为“当有连接、读写等IO事件时，操作系统内核在处理完事件后主动通知我们的程序代码”。

**模型抽象图**

![img.png](https://yuanjava.cn/assets/md/arch/Proactor.png)

**工作原理**

- Proactor Initiator 负责创建 Proactor 和 Handler，并将 Proactor 和 Handler 都通过 Asynchronous Operation Processor 注册到内核；
- Asynchronous Operation Processor 负责处理注册请求，并完成 I/O 操作；
- Asynchronous Operation Processor 完成 I/O 操作后通知 Proactor；
- Proactor 根据不同的事件类型回调不同的 Handler 进行业务处理；
- Handler 完成业务处理，Handler 也可以注册新的 Handler 到内核进程；

**优缺点**

- Proactor 在处理高耗时 IO 时的性能要高于 Reactor，但对于低耗时 IO 的执行效率提升并不明显；
- Proactor 的异步性使其并发处理能力要强于 Reactor；
- Proactor 的实现逻辑复杂，编码成本较 Reactor 要高很多；
- Proactor 的异步高度依赖于操作系统对于异步的支持。若操作系统对异步的支持不好，Proactor 的性能还不如 Reactor；

**案例**

- Netty5， 它是采用 AIO，其网络通信模型就是 Proactor，但该版本已经被不再维护，主要原因是 Linux 目前对于异步的支持不完善，导致 Netty5 花了大代价，性能相对 Netty4 不但没有提升，甚至还会降低。

## 总结

- Reactor 是同步非阻塞网络模型，Proactor 是异步非阻塞网络模型；
- Reactor 是 I/O 多路复用和线程池的完美结合；
- Reactor模型看似高深，其实是生活中很多真实案例的写照，比如：
1. 夜市一个老板一辆推车的单人炒粉模式，从点菜，出餐，结算都是老板一人完成，这个就对应了 单 Reactor单线程模型；
2. 医院叫号看病就对应了 单 Reactor多线程模型，一个叫号机负责叫号，多名医生负责接待病人；
3. 大型餐饮就餐对应了 多 Reactor多线程模型，一个接待员负责接客送客，多名服务员，每名服务员负责几桌客人，然后有专门的端菜人员负责给客人端菜，比如：海底捞；

- Reactor思维在日常开发中也会经常使用，最常用的是单线程处理，当并发量比较大时引进线程池，把业务细分，专门的线程处理专门的事情，这样就和 Reactor 模型的演变有异曲同工之妙；
- Proactor 主要是采用异步的方式来处理 IO 事件（比如：叫外卖，下单支付后不需要关注，直接处理自己的事情，等外卖好了之后，外卖小哥会把主动把外卖送到你手上），不过目前 Linux 对 AIO支持的不太友好，使用该模型的 Netty5 最终也为此夭折了；



## 鸣谢

如果你觉得本文章对你有帮助，感谢转发给更多的好友，关注我：猿java，为你呈现更多的硬核文章。

<img src="https://yuanjava.cn/assets/img/pub.jpg" alt="drawing" style="width:300px;"/>
