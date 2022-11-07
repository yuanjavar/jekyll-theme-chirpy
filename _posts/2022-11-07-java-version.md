---
layout: post
title: Java 小知识：JDK版本这样多，该如何选择？
category: java
tags: [java]
excerpt: Java 小知识：JDK版本这样多，该如何选择？
keywords: Java版本选择,JDK版本选择
---

你好，我是猿java。

今天分享的内容是：如何在众多的 JDK版本中选择最合适你的版本。

## 背景

Java 是一种广泛使用的计算机编程语言，拥有跨平台、面向对象、泛型编程的特性，广泛应用于企业级Web应用开发和移动应用开发，下图为Java的 logo：

![img.png](https://yuanjava.cn/assets/md/java/java.png)

鉴于 Java的快速发展，目前 JDK的版本比较多，今天就一起来回顾一下 Java的发展历程，以及对于JDK众多版本，我们在生产环境上该如何选择。

## Java的诞生

1995年，Sun公司发布了 JDK Beta版本，这是 Java的第一个版本，标志着一场面向对象语言革命的到来。

Sun公司，简称 SUN，名字是由斯坦福大学网络（Stanford University Network）缩写而来，该公司主要产品包括工作站、服务器和UNIX操作系统等，公司于
1986年在美国纳斯达克上市，后于2009年被 Oracle（甲骨文）公司收购，结束长达27年的公司历史。

另外，提到 Java语言，就不得不提到 Java的创造者，被公认为"Java之父"的James Gosling（詹姆斯·高斯林），1990年，James Gosling 开发了
Java语言的雏形，最初被命名为Oak，后来在 Sun公司的推动下，改名为 Java。下图为 James Gosling的照片及简介：

![img.png](https://yuanjava.cn/assets/md/java/James-Gosling.png)
- 1955 年 5 月 19 日， James Gosling 出生于加拿大阿尔伯塔省卡尔加里；
- 1977 年，获得加拿大卡尔加里大学计算机科学学士学位；
- 1983 年，获得了美国卡内基梅隆大学计算机科学博士学位；毕业后到 IBM工作，设计 IBM第一代工作站 NeWS系统，但不受重视，后来转至 Sun公司；
- 1990 年，James Gosling 与Patrick Naughton、Mike Sheridan等人合作「绿色计划」，后来发展一套语言叫做「Oak」，后改名为 Java；
- 1994 年底，James Gosling在矽谷召开的「技术、教育和设计大会」上展示Java程式；
- 2009 年 4月 20 日，Oracle 公司以每股9.50美元，总额 74亿美金收购 Sun公司；
- 2010 年 4月，James Gosling从 Oracle公司离职；
- 2011 年 3 月 29 日，高斯林在个人博客上宣布将加入Google；
- 2011 年 8 月 30 日，加入Google数月之后的高斯林就在个人博客上宣布离开Google，加盟一家从事海洋机械人研究的创业公司Liquid
  Robotics，担任首席软件架构师；
- 2011 年 5 月，Scala 公司Typesafe Inc.，高斯林被列为公司顾问；


从 1995年 到 2009年，Java在 Sun公司渡过了启蒙和快速发展的14年，Java 也成为了全球最受欢迎的编程语言之一。

## Java的转折

2009 年 4月 20 日，Oracle 公司以每股9.50美元，总额 74亿美金收购 Sun公司，从此 Sun公司退出历史舞台，Java 也正式进入 Oracle的怀抱，在 Oracle公司，Java迎来了自己的高光时刻。下图为 Java在Oracle公司发布的版本：

![img.png](https://yuanjava.cn/assets/md/java/jdk-version-guan.png)

## 版本及功能

下图总结了 JDK从第一个发布版本到现在所有的版本信息：
![img.png](https://yuanjava.cn/assets/md/java/jdk-version.png)

从上图可以看出，最开始 Sun公司是以 JDK（Java SE Development Kits，Java 开发包）命名，从1.2 版本起，修改成以 J2SE（Java 2 Platform, Standard Edition）命名，从 Java SE 6后 Java不再带有"2" 这个号码，后面的版本全部以 Java SE + 数字版本 命名。

尽管在 Java发展历史上出现过好多名词，比如 J2SE，J2ME，J2EE，但还是可以笼统的认为：Java = Java SE = JDK。


针对几个长期维护版本，列举了其重要功能升级：

### JDK 8
- Lambda 和 函数式接口
- 方法推导
- 接口默认方法和静态方法
- 重复注解
- 类型注解
- 类型推断
- Optional
- Stream
- 日期时间 API
- Base64 支持
- 并行数组 ParallelSort


### JDK 11

- 基于嵌套的访问控制
- 标准 HTTP Client 升级
- Epsilon：低开销垃圾回收器
- 简化启动单个源代码文件的方法
- 用于 Lambda 参数的局部变量语法
- 低开销的 Heap Profiling
- 支持 TLS 1.3 协议
- ZGC：可伸缩低延迟垃圾收集器
- 飞行记录器
- 动态类文件常量

### JDK 17

- 语言特性增强
1. 密封的类和接口（正式版）
- 工具库的更新
1. JEP 306：恢复始终严格的浮点语义
2. JEP 356：增强的伪随机数生成器
3. JEP 382：新的macOS渲染管道
- 新的平台支持
1. JEP 391：支持macOS AArch64
- 旧功能的删除和弃用
1. JEP 398：弃用 Applet API
2. JEP 407：删除 RMI 激活
3. JEP 410：删除实验性 AOT 和 JIT 编译器
4. JEP 411：弃用安全管理器以进行删除
- 新功能的预览和孵化API
1. JEP 406：新增switch模式匹配（预览版）
2. JEP 412：外部函数和内存api （第一轮孵化）
3. JEP 414：Vector API（第二轮孵化）
4. JEP 389：外部链接器 API（孵化器）
5. JEP 393：外部存储器访问 API（第三次孵化）


> 备注： LTS： long time support，长时间支持


## 如何选择版本

通过上面的介绍可以看出，JDK 8，JDK 11，JDK 17 是 3个长期维护的版本，但因为 JDK 17是 2021年发布的 GA版本，所以，生产上尽量选择 JDK 8 或者 JDK 11。

在 JDK版本的选择上，尽量选择长期维护的版本，不要使用最新版本的。因为新版本的 JDK，新功能没有经过生产环境的验证，如果想成为第一个吃螃蟹的人，一定要三思能否 hold得住。


## 参考
[官方JDK维护路线图](https://www.oracle.com/java/technologies/java-se-support-roadmap.html)


## 鸣谢

如果你觉得本文章对你有帮助，感谢转发给更多的好友，关注我：猿java，为你呈现更多的硬核文章。

<img src="https://yuanjava.cn/assets/img/pub.jpg" alt="drawing" style="width:300px;"/>

