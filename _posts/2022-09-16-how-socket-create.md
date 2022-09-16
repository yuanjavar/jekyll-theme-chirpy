---
layout: post
title: Socket是如何创建的？
category: interview
tags: [Socket,面试]
description: Socket是如何创建的？
keywords: Socket,Socket创建
---

你好，我是Weiki，欢迎来到猿java。

说起网络通信，就不得不提到 Socket，不管使用的是Java语言，还是C/C++，Go，PHP，只要你跟网络打交道，基本上离不开Socket。那么 Socket到底是什么？ 它是如何被创建的呢？ 这篇文章，我们就来把这些问题讲清楚。

> 申明：本文讲解的 是基于linux内核，基于TCP/IP 协议的 Intenet socket

## Linux用户态和内核态

Linux操作系统就将权限等级分为了2个等级，分别就是内核态(内核空间)和用户态(用户空间)。

- 用户态：提供应用程序运行的空间，只能受限的访问内存，且不允许访问外围设备，占用cpu的能力被剥夺，cpu资源可以被其他程序获取。

- 内核态：cpu可以访问内存的所有数据，包括外围设备，例如硬盘，网卡，cpu也可以将自己从一个程序切换到另一个程序。

**用户态和内核态的转换**

- 系统调用： 调用系统的库函数或者shell调用；
- 异常：如果当前进程运行在用户态，如果这个时候发生了异常事件，就会触发切换。例如：缺页异常。
- 外设中断：当外设完成用户的请求时，会向CPU发送中断信号。

![img.png](https://yuanjava.cn/assets/md/interview/user-kernel.png)

## Socket的种类

根据底层网络机制的差异，计算机网络世界中定义了不同协议族的socket，常见的socket有：
DARPA Internet 地址(Internet socket)、本地节点的路径名(Unix socket)、CCITT X.25地址(X.25 socket)等。


## Socket的定义
Socket：中文翻译有很多，分名词和动词，所有的翻译整理如下：
> n. （电源）插座；（电器）插口，插孔；（人体的）窝，槽；（高尔夫插球杆的）棒头承口；（用以插入某物使其转动的）承窝，轴孔
> v. 插入，使装入插座

在技术界，很多地方把Socket翻译成”套接字“，或许这个中文名词可以很生动形象的传达Socket所要表达的意思，但是我查阅了很多资料，一直没有找到为什么翻译成”套接字“的有力文档说明，所以个人还是喜欢直接用英文Socket来表达，不过这个不影响整体对Socket的理解。

在 Linux操作系统中，替代传输层以上协议实体的标准接口，称为Socket，它负责实现传输层以上所有的功能，可以说 Socket 是 TCP/IP 协议栈对外的窗口。

## Socket的数据结构
Linux 内核在 Socket 层定义了包含 socket 通用属性的数据结构，分别是 struct socket 与 struct sock。
struct socket，每个socket在内核中都唯一对应的 struct socket 结构

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

struct sock 是套接字在网络中的最小描述，它包含了内核管理套接字最重要信息的集合

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

## Socket在Linux中以什么方式存在

Linux中socket存在的方式是：文件，socket 连接建立后，用户进程就可以使用常规文件操作访问socket。
在 Linux 虚拟文件系统层(VFS)中，每个文件都有一个 VFS inode 结构，每个Socket都分配了一个该类型的 inode，inode 代码如下
```text
struct inode{
    struct file_operation *i_fop // 指向默认文件操作函数块
}
```


## 如何创建Socket
有了上面知识铺垫，我们就来看下一个基于TCP协议的Socket程序函数调用过程，创建socket应用程序一般要经过下面6 个步骤。
1. 创建套接字；
2. 将套接字与地址绑定，设置套接字选项；
3. 建立套接字之间的连接；
4. 监听套接字；
5. 接收、发送数据；
6. 关闭、释放套接字；

**服务端**
1. 调用socket()函数，创建一个socket，该Socket就是主动Socket(Active Socket)；
2. 调用bind()函数，给第1步的主动Socket绑定一个ip和port；
3. 调用listen()函数，将主动Socket转成监听Socket，开始监听客户端的连接请求；
4. 调用accept()函数，从已完成连接的队列中拿出一个连接进行处理，如果还没有完成，就要阻塞等待有连接完成；
5. 若服务端accept()函数获取到了一个已连接Socket(Connected Socket)，则服务端可以往已连接Socket 读数据或者写数据；

说明：在内核中，为每个Socket维护两个队列。一个是已经建立了连接的队列(三次握手已经完毕)，处于established状态；一个是还没有完全建立连接的(未完成三次握手)，处于syn_rcvd的状态。

**客户端**
1. 调用socket()函数，创建一个socket，该Socket就是主Socket(Active Socket)；
2. 当服务端调用accept()时，客户端可以调用connect()向服务器发起连接请求，内核会给客户端分配一个临时的端口，一旦握手成功，服务端的accept()就会返回另一个Socket；
3. 客户端可以往已连接Socket 读数据或者写数据；

整个过程可以描述成如下图：

![img.png](https://yuanjava.cn/assets/md/interview/socket.png)

**主动socket&被动socket&已连接socket&监听socket**

- 主动socket(Active Socket)：通过系统库函数socket()生成的就是主动socket，主要是用于客户端主动向服务器发送连接；
- 被动socket(Passive Socket)：通过系统库函数listen()，可以将主动socket标记为被动socket，用于被动监听客户端的请求连接，也叫监听socket。被动socket是服务端独有的，将伴随服务端的整个生命周期。
- 监听socket(Listened Socket)： 同被动socket(Passive Socket)一样。
- 已连接socket(Connected Socket)：通过系统库函数accept()获取的已建立连接的socket，该socket是用于客户端和服务端数据读写的通道，已连接socket是服务器独有的，生命周期为 客户端和服务端的维持的连接时长，当断开连接，生命周期结束。

讲述了socket这么多的理论知识，最后我们看下Socket在应用层语言是怎么使用的，这里以java为例。
java如何使用socket
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

java源码ServerSocket类中封装了多个构造器，源码调试可以看下，构造器最终都指向bind(SocketAddress endpoint, int backlog)方法，该方法里面包含重要两个步骤 bind()和listen()， 俩方法又指向PlainSocketImpl类中的的native socketBind()和snative ocketListen()，native方法其实最终操作系统的bind()和listen()函数对应。
所以通过上面的java源码可以看出，java在网络包里ServerSocket只是对操作系统的函数做了一层使用的包装，下面我们再看看java对客户端的代码实现
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

java源码对客户端的包装和服务端很类似，最后都指向native connect()方法和操作系统的connnect()函数对应。
通过对java.net包中对ServerSocket类和Socket类的源码分析，我们看出java只是在jdk里面做了一层使用封装，最终都是需要指向native的方法，和操作系统的函数绑定，这个流程也再次和上面基于TCP协议的Socket程序函数调用过程图 吻合。

## 参考文献

[Linux系统帮助文档](Index of /linux/man-pages/man2)

[Linux内核源码文档](https://elixir.bootlin.com/linux/v5.10.23/source/include/linux/socket.h)

[java socket官方文档 What Is a Socket?](https://docs.oracle.com/javase/tutorial/networking/sockets/definition.html#:~:text=A%20socket%20is%20one%20endpoint,address%20and%20a%20port%20number)

[IBM socket官方文档](https://www.ibm.com/docs/en/zos/2.3.0?topic=functions-socket-create-socket)

## 鸣谢
如果你觉得本文章有帮助，感谢转发给更多的好友，我们定将呈现更多的干货， 欢迎关注公众号：猿java

