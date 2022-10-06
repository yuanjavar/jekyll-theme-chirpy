---
layout: post
title: 如何用mac自带终端连接多个远程Linux服务器
category: common
tags: [common]
excerpt: 如何在mac自带的终端通用ssh秘钥连接多个远程Linux服务
keywords: mac ssh秘钥生成,mac连接多个linux
---

你好，我是Weiki，欢迎来到猿java。

现如今，Mac电脑已经成了很多公司开发人员的标配，作为开发人员进行线上Linux服务器查错是在所难免的，很多公司有堡垒机(跳板机)，可以帮助开发人员
快速管理和进入线上机器，但是相对运维比较落后的公司，要么安装一些三方工具，要么电脑上不停的切换服务器IP进行远程连接操作。今天，小编就分享如何利用mac自带
终端连接多个远程Linux服务。

## 使用ssh秘钥连接

### 设置mac本地ssh秘钥

```shell
cd ~
# -t 指定密钥类型，默认即 rsa；-C 设置注释文字，比如 邮箱
ssh-keygen -t rsa -C '替换成你的备注内容'
```

下面是小编君的操作记录，Enter passphrase操作是设置私钥的密码，大家根据具体情况填写，建议不填写

```shell
weiki@WeikideMacBook-Pro ~ % cd ~
weiki@WeikideMacBook-Pro ~ %
weiki@WeikideMacBook-Pro ~ % sudo ssh-keygen -t rsa -C ''
Password:
Generating public/private rsa key pair.
Enter file in which to save the key (/var/root/.ssh/id_rsa):
Created directory '/var/root/.ssh'.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /var/root/.ssh/id_rsa
Your public key has been saved in /var/root/.ssh/id_rsa.pub
The key fingerprint is:
SHA256:oFpxxEJFWfXIc4a73rD65QpQ5wKaRwtGabRqmcwkbaA
The key's randomart image is:
+---[RSA 3072]----+
|.  o+=+o...      |
|.o .+oo  . +     |
|E +.=.= . * +    |
| * = O = o =     |
|  B = + S o      |
| . o . . . .     |
|  .     . o .    |
|         o *     |
|        .o=.o    |
+----[SHA256]-----+
```

查看公钥，私钥的安装情况，操作写入：

```shell
weiki@WeikideMacBook-Pro ~ % cd .ssh
weiki@WeikideMacBook-Pro .ssh %
weiki@WeikideMacBook-Pro .ssh % ll
total 40
drwx------   7 weiki  staff   224  9  6 10:12 ./
drwxr-x---+ 55 weiki  staff  1760  9  6 09:48 ../
-rw-------   1 weiki  staff   180  6 24 16:05 config
-rw-------   1 weiki  staff  2635  9  6 10:12 id_rsa
-r--------@  1 weiki  staff   584  6  9 14:45 id_rsa.pub
-rw-------   1 weiki  staff  4022  6 24 16:05 known_hosts
-rw-------   1 weiki  staff  3278  6 24 16:05 known_hosts.old
```

### 拷贝本地公钥到远程服务器

将本地生成的公钥(~/.ssh/id_rsa.pub)拷贝到你需要连接的服务器的/home目录下，远程拷贝指令如下：
```shell
# port, user, ip 替换成自己的服务器相关的信息
scp -P port ~/.ssh/id_rsa.pub user@ip:/home/id_rsa.pub
```

查看远程服务公钥是否拷贝成功，指令如下：

```shell
[root@yuanjava ~]# cd /home/
# ll指令是列举目录下所有文件列表信息
[root@yuanjava home]# ll
总用量 4
-r-------- 1 root root 584 9月   6 10:30 id_rsa.pub
[root@yuanjava home]#
```

### 配置mac的config信息

编辑 ~/.ssh/config 文件，增加服务器相关信息：
```shell
cd ~/.ssh

# 编辑 config文件
vim config
```

往config文件中增加服务器信息，服务器信息修改成自己真实的信息，模板内容如下：

```text
# Host后面的内容是服务器别名，用于登录使用，你可以为每个服务器设置一个容易记住的别名
Host ubuntu
    HostName 150.230.58.131
    User ubuntu
    IdentityFile ~/.ssh/id_rsa

Host yuan
    HostName 192.168.1.112
    port 22
    User root
    IdentityFile ~/.ssh/id_rsa
```

在Mac终端指令操作"ssh 服务器别名"，就可以登录远程服务器，不过此处还是需要输入远程服务器的密码，详情如下：

```text
# ssh 后面的内容是 ~/.ssh/config 文件中Host的别名
weiki@WeikideMacBook-Pro .ssh % ssh yuan
# 输入服务器密码
root@192.168.1.112's password:
Last login: Tue Sep  6 10:30:44 2022

Welcome to Service !
```

如果想操作"ssh 服务器别名"指令后不需要输入密码，可以登录远程服务器，执行如下指令：

```shell
# 将 /home/id_rsa.pub 的公钥内容复制到 ~/.ssh/authorized_keys
[root@yuanjava ~]#cat /home/id_rsa.pub >> ~/.ssh/authorized_keys
```

在mac上新开一个终端，重新连接服务器，如下指令：

```shell
# 注意：输入指令后会提醒输入mac生成ssh秘钥时的密码
weiki@WeikideMacBook-Pro ~ % ssh yuan
Enter passphrase for key '~/.ssh/id_rsa':
Last failed login: Tue Sep  6 11:15:18 CST 2022
Welcome to Service !
```

到处，通过mac通过ssh秘钥连接远程端服务器环节就分享完成，如果你有多台服务器，只需要在mac的 ~/.ssh/config 文件中配置多个Host，这样就可以在mac终端通过 ssh host别名 切换登录服务器，可以集中管理所有的服务器。


**那么，问题来了...**

如果不想设置复杂的ssh秘钥去连接远程服务，有没有办法实现？

## 不使用ssh秘钥连接

答案是：有。操作参考如下：

### 配置 ~/.ssh/config 文件

在文件中增加如下内容(服务器信息修改成自己真实的信息)，相比上面ssh秘钥的方法，我们再config中未配置 IdentityFile ~/.ssh/id_rsa 这块的内容

```text
# Host后面的内容是服务器别名，用于登录使用，你可以为每个服务器设置一个容易记住的别名
Host ubuntu
    HostName 150.230.58.131
    User ubuntu

Host yuan
    HostName 192.168.1.112
    port 22
    User root
```

### ssh 服务器别名 登录

通过在mac终端执行 ssh 服务器别名，连接远程服务器

```shell
# 注意：此处需要远程服务器
weiki@WeikideMacBook-Pro ~ % ssh yuan
root@192.168.1.112's password:
Last failed login: Tue Sep  6 11:15:18 CST 2022 from 60.177.97.166 on ssh:notty
There was 1 failed login attempt since the last successful login.
Last login: Tue Sep  6 11:14:42 2022 from 60.177.97.166

Welcome to Service !
```


## 总结

- 本文分享了mac 通过ssh秘钥和不使用ssh秘钥 2种方式连接远程服务器，两种方式各有使用场景
- 通过ssh秘钥，可以使用ssh秘钥设置时的密码登录，一般用于服务器秘钥敏感的场景，比如公司服务器对开发人员
- 不使用ssh秘钥，使用账号/密码登录远程服务器，一般用于服务器秘钥不敏感的场景，比如私人的服务器
- 通过config配置，可以灵活的通过服务器别名管理和连接多个远程服务器


## 鸣谢
如果你觉得本文章对你有帮助，感谢转发给更多的好友，关注我：猿java，为你呈现更多的硬核文章。

<img src="https://yuanjava.cn/assets/img/pub.jpg" alt="drawing" style="width:300px;"/>

