---
layout: post
title: Redis在各种操作系统下的安装方式
category: redis
tags: [redis]
excerpt: Redis各种方式安装大全
keywords: redis安装,源码安装,macos安装,linux安装
---

你好，我是Weiki，欢迎来到猿java。

Redis作为nosql的典型代表，在当下互联网行业绝对是不可或缺的缓存神奇。作为刚入门的小伙伴，肯定想自己安装一台Redis服务器开启自己的Redis学习之路，今天我们就把Redis各种安装方式全部整理成文档，希望帮助到你。

## Redis的安装方法

Redis主要有4种安装方式：

1. 源码安装

2. Linux安装

3. MacOS安装

4. Windows安装

## 源码安装 Redis

源码安装是在Linux或者类Linux系统比较友好的安装方式，我们可以通过下载源码，解压，编译，安装等步骤进行操作，下面给出了Redis源码安装的指令

```shell
# 下载源码
wget https://download.redis.io/redis-stable.tar.gz

# 解压源码包
tar -xzvf redis-stable.tar.gz
# 进入解压包
cd redis-stable

# 编译源码
make
```
如果 make 编译成功了，在 redis-stable/src 目录下会出现下面两个文件，如果 make 编译 失败，则根据具体出现的异常具体解决。
> redis-server     // Redis服务器
> redis-cli        // Redis客户端

```shell
# 安装redis， 基于 make指令成功的前提
make install

# 终端启动，关闭终端，redis-server就关闭了
redis-server

# 守护进程启动 redis-server
redis-server &

# 查看 redis-server进程
ps -ef|grep redis-server
```


## Linux 安装Redis
Linux 常见的有源码安装和软件包管理器安装 2种方式，Linux包括 Ubuntu, RHEL, CentOS等类型

源码安装参考 ## 源码安装

软件包管理器安装 可以有 apt 和 yum。这种方式的好处是，可以自动下载，配置，安装二进制或者源代码格式的软件包，简化我们的安装流程。

> apt 是 Advanced Packaging Tools 的缩写，是Debian及其派生的Linux软件包管理器。APT可以自动下载，配置，安装二进制或者源代码格式的软件包，因此简化了Unix系统上管理软件的过程
>
> yum 是Yellow dog Updater,Modified 的缩写，是一个在 Fedora 和 RedHat 以及 SUSE 中的 Shell 前端软件包管理器。

```shell
# apt 方式安装
apt install redis

# yum 方式安装
yum install redis
```

## MacOS安装Redis
MacOS是类Linux系统，常见的有源码安装和软件包管理器安装 2种方式

源码安装参考 ## 源码安装

软件包管理器安装，brew

```shell
# 查看 brew的版本，如果mac上为安装brew，可以参考 https://brew.sh/ 进行安装
brew --version

# 安装 Redis
brew install redis

# 终端启动 redis-server，该方式当终端关闭了，server服务就停止了，也可以 Ctrl+C 终止服务
redis-server

# 守护进程启动 redis-server
brew services start redis

# 查看 redis-server的信息
brew services info redis

# 关闭 redis-server
brew services stop redis
```

## Windows安装Redis

首先申明下，Redis的Windows版本是不受Redis官方的欢迎的，下面是Redis官方的原文：
> Redis is not officially supported on Windows.
>
> Redis 在 Windows 上不受官方支持

但是，Redis官方还是提供了Windows的安装方式

**安装或启用 WSL2**

要在 Windows 上安装 Redis，首先需要启用WSL2（适用于 Linux 的 Windows 子系统）。
WSL2 允许在 Windows 上本地运行 Linux 二进制文件。要使此方法起作用，您需要运行 Windows 10 版本 2004 及更高版本或 Windows 11。
[安装 WSL 的详细说明](https://docs.microsoft.com/en-us/windows/wsl/install)

**安装 Redis**

在 Windows 上运行 Ubuntu 后，可以按照在Ubuntu/Debian上安装中详述的步骤从官方packages.redis.ioAPT存储库安装Redis的最新稳定版本。
将存储库添加到apt索引，更新它，然后安装，指令如下：

```shell
curl -fsSL https://packages.redis.io/gpg | sudo gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] https://packages.redis.io/deb $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/redis.list

# 安装 redis-server
sudo apt-get update
sudo apt-get install redis

# 启动 redis-server
sudo service redis-server start
```


## 总结

上面给出了4种Redis的安装方式，仔细观察可以发现，Redis官方是支持Linux或者类Linux系统。

初级程序员，如果有条件，可以搞一台云主机，作为学习使用

## 鸣谢
如果你觉得本文章对你有帮助，感谢转发给更多的好友，关注我：猿java，为你呈现更多的硬核文章。

<img src="https://yuanjava.cn/assets/img/pub.jpg" alt="drawing" style="width:300px;"/>

