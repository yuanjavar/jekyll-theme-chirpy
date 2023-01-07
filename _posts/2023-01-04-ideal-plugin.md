---
layout: post
title: 免费且超实用的 IntelliJ 插件，强烈推荐！！！
category: arch
tags: [java,面试]
excerpt: 免费且超实用的 IntelliJ 插件，强烈推荐！！！
keywords:  IntelliJ 提效插件
---

你好，我是猿java。

作为一名 Java程序员，IntelliJ 应该是非常主流的开发工具，今天推荐几款个人工作中经常使用的插件，真的是免费又超实用。

## Translation

阅读纯英文资料对于大多数技术人来说是件头疼的事情，因此如果能在 idea 中进行翻译，可以高效阅读文档，因此强烈推荐插件：Translation

### 安装

![img.png](https://yuanjava.cn/assets/md/common/plugin-transaction.png)

### 使用


进入类，右键 -> Translate Documentation， 操作如下：

![img.png](https://yuanjava.cn/assets/md/common/plugin-transaction-1.png)


## GsonFormatPlus

在平时的日常开发中，经常会对接一些 Open API的场景，API文档会定义好入参以及返回结果 json数据格式，按照正常的逻辑，我们需要根据 json来手动创建对应的 Java对象，如果 json返回的字段比较多，那么这个工作量还是比较大的，今天就分享一款可以直接用 json 文本生成 Java对象的插件：GsonFormatPlus

### 安装

![img.png](https://yuanjava.cn/assets/md/common/plugin-GsonFormatPlus.png)

### 使用

先创建一个 Person 类，然后 Code -> Generate -> GsonFormatPlus，填写对应的 json文本，插件会生成对应的字段和注释，操作如下图：

![img.png](https://yuanjava.cn/assets/md/common/plugin-GsonFormatPlus-1.png)

![img.png](https://yuanjava.cn/assets/md/common/plugin-GsonFormatPlus-2.png)

![img.png](https://yuanjava.cn/assets/md/common/plugin-GsonFormatPlus-3.png)


## Java Bean to Json

如果你想在不运行任何代码的情况下直接把一个 Java类对象转换成一个 json，Java Bean to Json 插件可以帮到你，安装如下：
### 安装

![img.png](https://yuanjava.cn/assets/md/common/plugin-javatojson.png)

## 使用

进入类，右键 -> ConvertToJson，这样生成的json就已经生成被复制了，只需要拷贝到需要的地方就OK了。

![img.png](https://yuanjava.cn/assets/md/common/plugin-javatojson-1.png)




## SequenceDiagram

工作中难免需要画时序图，因此我们很自然的想到能不能基于现有的 Java类直接生成时序图，这样就能节省很多画图的时间

### 安装

![img.png](https://yuanjava.cn/assets/md/common/plugin-SequenceDiagram.png)


### 使用

如下图，进入某个类，右键 -> Sequence Diagram -> 选中方法，这样时序图就生成

![img.png](https://yuanjava.cn/assets/md/common/plugin-SequenceDiagram1.png)

![img_1.png](https://yuanjava.cn/assets/md/common/plugin-SequenceDiagram2.png)


## 最后

如果你还有更好的插件，欢迎评论区留言吗，分享给更多的小伙伴。


## 鸣谢
如果你觉得本文章对你有帮助，感谢转发给更多的好友，关注我：猿java，为你呈现更多的硬核文章。

<img src="https://yuanjava.cn/assets/img/pub.jpg" alt="drawing" style="width:300px;"/>
