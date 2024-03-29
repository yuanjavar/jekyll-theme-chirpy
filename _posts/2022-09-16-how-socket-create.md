---
layout: post
title: Socket是如何创建的？
category: interview
tags: [Socket,面试]
description: Socket是如何创建的？
keywords: Socket,Socket创建
---

你好，我是猿java。

说起网络通信，就不得不提到 Socket，不管使用的是Java语言，还是C/C++，Go，PHP，只要你跟网络打交道，基本上离不开Socket。那么 Socket到底是什么？ 它又是如何被创建的？ 这篇文章，我们就来讲清楚。

> 申明：本文基于 Linux内核

在正式分析 Socket之前，我们先铺垫下 Linux操作系统下的内核态和用户态，方便下文更好理解 Socket。

## Linux 用户态和内核态

Linux设计的初衷是不同的操作给与不同的“权限”，因此将权限分为 2个等级：内核态（内核空间）和用户态（用户空间）。

- 用户态：提供应用程序运行的空间，一般我们编写的业务代码就是运行在空间。

- 内核态：提供给操作系统内核使用，比如：cpu可以访问内存、外围设备等。

为什么要划分用户态和内核态？

简单来说：
- 禁止用户程序和底层硬件平台直接交互。
- 禁止用户程序直接访问任意内存地址空间。


用户态和内核态主要可以通过 3种方式来进行转换：

- 系统调用： 调用系统的库函数或者shell调用；
- 异常：如果当前进程运行在用户态，如果这个时候发生了异常事件，就会触发切换。例如：缺页异常。
- 外设中断：当外设完成用户的请求时，会向CPU发送中断信号。

![img.png](https://yuanjava.cn/assets/md/interview/user-kernel.png)

接下来我们开始分析 Socket以及创建过程

## 什么是 Socket

Socket：中文翻译有很多，所有的翻译整理如下：
> n. （电源）插座；（电器）插口，插孔；（人体的）窝，槽；（高尔夫插球杆的）棒头承口；（用以插入某物使其转动的）承窝，轴孔
> v. 插入，使装入插座

在技术界，很多地方把 Socket翻译成”套接字“，或许这个中文名词可以很生动形象的传达 Socket所要表达的意思，但是我查阅了很多资料，一直没有找到为什么翻译成”套接字“的有力文档说明，所以个人还是喜欢直接用英文 Socket来表达，不过这个不影响整体对 Socket的理解。

在 Linux操作系统中，替代传输层以上协议实体的标准接口，称为Socket，它负责实现传输层以上所有的功能，可以说 Socket 是 TCP/IP 协议栈对外的窗口。

## Socket的数据结构

在 Linux内核中，Socket的数据结构由 struct socket 与 struct sock 2部分组成。

- struct socket

每个 Socket在内核中都唯一对应一个 struct socket结构，其结构如下：

```text
struct socket {
  socket_state            state;  // socket的状态
  unsigned long           flags;  // socket的设置标志。存放socket等待缓冲区的状态信息，其值的形式如SOCK_ASYNC_NOSPACE等
  struct fasync_struct    *fasync_list;  // 等待被唤醒的socket列表，该链表用于异步文件调用
  struct file             *file;  // socket所属的文件描述符
  struct sock             *sk;  // 指向存放socket属性的结构指针
  wait_queue_head_t       wait;  // socket的等待队列
  short                   type;  // socket的类型。其取值为SOCK_XXXX形式
  const struct proto_ops *ops;  // socket层的操作函数块
}
```

- struct sock

struct sock是 Socket在网络中的最小描述，它包含了内核管理 Socket最重要的信息集合，其结构如下：

```text
struct sock_common {
  unsigned short          skc_family;         // 地址族
  volatile unsigned char  skc_state;          // 连接状态
  unsigned char           skc_reuse;          // SO_REUSEADDR设置
  int                     skc_bound_dev_if;
  struct hlist_node       skc_node;
  struct hlist_node       skc_bind_node;      // 哈希表相关
  atomic_t                skc_refcnt;         // 引用计数
};
```

## Socket 存在方式

在 Linux中 Socket存在的方式是：文件。

当 Socket连接建立后，用户进程就可以使用常规文件操作访问 Socket，每个 Socket都分配了一个 VFS inode，inode 结构如下：

```text
struct inode{
    struct file_operation *i_fop // 指向默认文件操作函数块
}
```

## 如何创建 Socket

上文我们讲解了 Socket的定义以及数据结构，接下来就要分析基于 TCP协议的 Socket是如何创建的，一般来说，创建 Socket需要经过下面 6个步骤：
1. 创建套接字；
2. 将套接字与地址绑定，设置套接字选项；
3. 建立套接字之间的连接；
4. 监听套接字；
5. 接收、发送数据；
6. 关闭、释放套接字；

因为 Socket是双通道的，所以我们从服务端和客户端两个部分来说明 Socket的创建过程。

**服务端创建 Socket**
1. 调用 socket()函数，创建一个 socket，该 Socket成为主动Socket（Active Socket）；
2. 调用 bind()函数，给第 1步的主动Socket绑定一个 ip和 port；
3. 调用 listen()函数，将主动Socket转成监听Socket，开始监听客户端的连接请求；
4. 调用 accept()函数，从已完成连接的队列中拿出一个连接进行处理，如果还没有完成，就要阻塞等待有连接完成；
5. 若服务端accept()函数获取到了一个已连接Socket(Connected Socket)，则服务端可以往已连接Socket 读数据或者写数据；

说明：在 Linux内核中，会为每个 Socket维护两个队列：一个是已经建立了连接的队列（三次握手已经完毕），处于 established状态；一个是还没有完全建立连接的 （未完成三次握手），处于 syn_rcvd的状态。

**客户端创建 Socket**
1. 调用 socket()函数，创建一个 socket，该 Socket成为主动Socket（Active Socket）；
2. 当服务端调用 accept()时，客户端可以调用 connect()向服务器发起连接请求，内核会给客户端分配一个临时的端口，一旦握手成功，服务端的 accept()就会返回另一个 Socket；
3. 客户端可以向 已连接Socket读数据或者写数据；

Socket整个创建过程可以描述成如下图：

![img.png](https://yuanjava.cn/assets/md/interview/socket.png)

**主动socket&被动socket&已连接socket&监听socket**

- 主动socket（Active Socket）：通过系统库函数 socket()生成的就是主动socket，主要是用于客户端主动向服务器发送连接；
- 被动socket（Passive Socket）：通过调用系统库函数 listen()，可以将主动socket标记为被动socket，用于被动监听客户端的请求连接，也叫监听socket。被动socket是服务端独有的，将伴随服务端的整个生命周期。
- 监听socket（Listened Socket）： 同被动socket（Passive Socket）一样。
- 已连接socket（Connected Socket）：通过系统库函数 accept()获取的已建立连接的socket，该 socket是用于客户端和服务端数据读写的通道，已连接socket是服务器独有的，生命周期为 客户端和服务端的维持的连接时长，当断开连接，生命周期结束。

讲述了 socket这么多的理论知识，最后我们看下 Socket在应用层语言是怎么使用的，这里以 java为例。

服务端源码

```java
// java.net.ServerSocket
public ServerSocket() throws IOException{}

public ServerSocket(int port) throws IOException{}

public ServerSocket(int port, int backlog) throws IOException {}

public ServerSocket(int port, int backlog, InetAddress bindAddr) throws IOException {}

public void bind(SocketAddress endpoint, int backlog) throws IOException {
    getImpl().bind(epoint.getAddress(), epoint.getPort());
    getImpl().listen(backlog);
}

// java.net.PlainSocketImpl
native void socketBind(InetAddress address, int port) throws IOException;

native void socketListen(int count) throws IOException;
```

java源码 ServerSocket类中封装了多个构造器，源码调试可以看下，构造器最终都指向 bind(SocketAddress endpoint, int backlog)方法，该方法里面包含重要两个步骤 bind()和 listen()，俩方法又指向 PlainSocketImpl类中的的 native socketBind()和 native socketListen()，native方法其实最终操作系统的 bind()和 listen()函数对应。

所以通过上面的java源码可以看出，java的 ServerSocket类只是对操作系统的函数做了一层简单的包装，下面我们再看看 java对客户端的代码实现：

```java
// java.net.Socket
public Socket(){}

public Socket(Proxy proxy){}

public Socket(String host, int port) throws UnknownHostException, IOException{}

public Socket(InetAddress address, int port) throws IOException {}

public Socket(String host, int port, InetAddress localAddr, int localPort) throws IOException {}

public void connect(SocketAddress endpoint) throws IOException {
    connect(endpoint, 0);
}

// java.net.PlainSocketImpl
native void socketConnect(InetAddress address, int port, int timeout) throws IOException;
```

java源码对客户端的包装和服务端很类似，最后都指向native connect()方法和操作系统的connect()函数对应。

通过对 java.net包中 ServerSocket类和 Socket类的源码分析，我们看出java只是在jdk里面做了一层使用封装，最终都是指向 native的方法，和操作系统的函数绑定，这个流程也再次和上面基于 TCP协议的Socket程序函数调用过程图 吻合。

## 参考文献

[Linux系统帮助文档](Index of /linux/man-pages/man2)

[Linux内核源码文档](https://elixir.bootlin.com/linux/v5.10.23/source/include/linux/socket.h)

[java socket官方文档 What Is a Socket?](https://docs.oracle.com/javase/tutorial/networking/sockets/definition.html#:~:text=A%20socket%20is%20one%20endpoint,address%20and%20a%20port%20number)

[IBM socket官方文档](https://www.ibm.com/docs/en/zos/2.3.0?topic=functions-socket-create-socket)

## 鸣谢
如果你觉得本文章对你有帮助，感谢转发给更多的好友，关注我：猿java，为你呈现更多的硬核文章。

<img src="https://yuanjava.cn/assets/img/pub.jpg" alt="drawing" style="width:300px;"/>

