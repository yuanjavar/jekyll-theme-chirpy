---
layout: post
title: 面试官：能聊聊 MySQL count()的各种用法吗？
category: mysql
tags: [mysql,面试,interview]
excerpt:  揭秘 count(*)、 count(1)、 count(主键)、 count(非主键) 该如何选择
keywords: count(*)、 count(1)、 count(主键)、 count(非主键)
---

你好，我是猿java。

实际工作中我们难免会使用到 MySQL的 count()函数进行统计操作，但是，对于 count(1)、count(*)、count(主键)、count(非主键) 等多种 count()操作，我们往往比较疑惑该如何选择，今天我们就来聊聊这些 count()的效率以及背后的实现原理。

> 申明：本文基于 MySQL 8.0.30，数据库引擎为 InnoDB引擎 和 MyISAM引擎；
>
> 本文的count()操作都是基于不加 where条件
>
> 如果需要mac本地安装 MySQL，参考：[macOS M1 源码安装 MySQL8 版本](https://www.yuanjava.cn/posts/mac-mysql8/)


为了更好的展开本文的讲解，我们首先创建 user和 person两张表，user表使用 InnoDB引擎，person表使用 MyISAM引擎，同时查看它们在磁盘上的文件信息，具体信息如下截图：

![img.png](https://yuanjava.cn/assets/md/mysql/mysql-count-table.png)

从上述截图可以看出：

在使用 MyISAM引擎的 person表中，表定义，数据，索引是分三个文件存储，如下：

- person_365.sdi，存储 person表定义，sdi（**S**erialized **D**ictionary **I**nformation，序列化字典信息），MySQL 8.0引入，以前的版本是 .frm；
- person.MYD，存储 person表数据，MYD（**My**ISAM **D**ata）；
- person.MYI，存储 person表索引，MYI（**My**ISAM **I**ndex）；

在使用 InnoDB引擎的 user表中，表定义，数据，索引都存放在一个文件中，如下：
- user.ibd，ibd（**i**nnod**b** **d**ata）

接下来正式分析各个count()操作

## count(*)

- MyISAM 引擎会把表的总行数存在了磁盘上（存放在 information_schema 库中的 PARTITIONS 表中），因此执行 count( * )时会直接返回这个总数，所以 count( * )效率很高，这里指的是不加 where条件。如果增加 where条件，会先根据 where条件查出数据，然后再统计，性能取决于遍历索引树的时间；
- InnoDB 引擎并没有像 MyISAM引擎那样把表的总行数存储在磁盘，而是在执行 count( * )时，MySQL Server层需要把数据从引擎里面读出来，然后逐行累加，得出总数。

![img.png](https://yuanjava.cn/assets/md/mysql/mysql-count-myisam.png)

为了更好的理解，我们可以先看下 user表和 person表 两张表 count(*) 的执行计划，结果如图：

![img.png](https://yuanjava.cn/assets/md/mysql/mysql-count*.png)


从上述执行计划的截图可以看出：

- InnoDB引擎 count( * )时 rows = 2（全表只有2行数据），需要扫描全表，Extra里面的内容是 "Using index"，说明该 count( *)操作使用了索引。

- MyISAM引擎 count( * )时 rows = NULL，Extra里面的内容是 "Select tables optimized away"，它包含的意思是：MyISAM 表以单独的行数存储总数，执行 count查询时，MySQL不需要查看任何表行数据，而是将预先计算的行数立即返回， 因此查询速度快。

那么，为什么 InnoDB引擎不能像 MyISAM引擎一样，把表的数据总数存起来，而是需要扫描全表呢？

这是因为 InnoDB引擎可以支持事务，默认的隔离级别是 Repeatable Read（可重复读，指的是一个事务执行过程无法看到其它事务未提交的数据），而可重复读又是通过多版本并发控制（MVCC）来实现的，MVCC更直白的表述是：一行记录在不同的事务中表现的结果值是不一样的，呈现出一行记录多种版本数据。
关于 MVCC可以参照下面的图来理解：
![img.png](https://yuanjava.cn/assets/md/mysql/count-mvcc.png)

为了更好的说明原因，我们通过一个案例进行解析，如下：

有两个事务会话 sessionA，sessionB，
- sessionA 先启动一个事务，然后 select count(*) from user 统计总行数；
- sessionB 也启动事务，先执行一次 select count(*) from user，然后插入一行数据，再 select count(*) from user，统计总行数；

执行顺序流以及截图如下：

| sessionA                                             | sessionB                                          |
|------------------------------------------------------|---------------------------------------------------|
| #开启事务 <br> begin;                                    |                                                   |
| select count(*) from user; |                                                   |
|                                                      | select count(*) from user;                        |
|                                                      | #插入一条数据<br>insert into user(id,age) values(3,30); |
| select count(*) from user;                  | select count(*) from user;                        |

![img.png](https://yuanjava.cn/assets/md/mysql/mysql-count*-sql.png)

通过运行结果截图，我们可以看出，在事务 sessionA和 sessionB都未commit提交事务的前提下，事务 sessionB在插入一条数据后，sessionA的 count( * )操作并没有把这条数据统计进去，符合可重复读隔离级别的要求，假如 InnoDB也像 MyISAM一样把行的总数存在磁盘上，那么 sessionA 和sessionB count( * )操作应该拿到同一条数据，
也就是说 sessionA 和 sessionB 最后一次 count( * )的结果值都是3，这显然就违背了可重复读隔离级别的要求。

所以，通过案例的分析也刚好验证了上述 count( * ) user表的执行计划中需要全表扫描 user表。


**有人可能会说，执行"show table status"指令，结果中的 ROWS 就是表的总行数，快捷方便。**

方法是否有效，我们还是用事实说话，我们可以执行"show table status"指令，执行顺序流和结果截图如下：

| sessionA                   | sessionB                                          |
|----------------------------|---------------------------------------------------|
| #开启事务 <br> begin;          |                                                   |
| select count(*) from user; |                                                   |
|                            | select count(*) from user;                        |
|                            | #插入一条数据<br>insert into user(id,age) values(3,30); |
| select count(*) from user; | select count(*) from user;                        |
| show table status\G        | show table status\G                               |

![img.png](https://yuanjava.cn/assets/md/mysql/mysql-count*-sql2.png)

通过运行结果截图可以看出，sessionA的 count(*)结果和 "show table status"指令结果中的 ROWS值相等，但是在 sessionB中两个值就不一样，因此说，通过 "show table status"来统计总数，结果值是不准确的。

按照 MySQL官方的说法： "show table status"命令显示行数的误差率在 40% ~ 50%。

> show table status 查询的是系统 information_schema 库中的 TABLES 表，关于表字段可以参考官方文档：[TABLES 官方文档](https://docs.oracle.com/cd/E17952_01/mysql-8.0-en/information-schema-partitions-table.html)

需要说明的是：尽管 InnoDB引擎的 count( * )操作需要扫描全表，但是 MySQL还是有做过优化处理，具体优化如下：

因为 InnoDB引擎采用的是聚簇索引机制，主键索引的叶子节点存放了数据，而普通索引的叶子节点存放的是主键值。因此，不管遍历哪一棵索引树，count( * )的结果都是一致的，所以，MySQL优化器会找到最小的那棵索引树进行遍历，这样尽管扫描的行数没有减少，但是针对每行记录获取的数据量减少了，因此性能就提升了。


有了上述对 count( * )的讲解，我们分析和理解其他几种 count()操作就会轻松很多，在 InnoDB引擎中，count()是一个聚合函数，对于引擎返回的结果集，MySQL Server会逐行判断，count(参数)函数最终就是统计"参数不是 NULL"的总数作为结果值。

## count(主键)

count(主键)的执行逻辑为：InnoDB引擎会遍历整张表，然后取出每一行的主键值并返回给 MySQL Server层，Server层拿到引擎的结果值后，统计主键总数量。

## count(1)

count(1)的执行逻辑为：InnoDB引擎遍历整张表，但是不会取数据值，MySQL Server层对于引擎返回的每一行记录都放置一个数字“1”，最终再统计包含 1的总行数。

所以，count(1)操作要比 count(主键)快。因为count(主键)需要从引擎返回主键值，过程中会涉及到数据行的解析，字段值的拷贝等I/O操作。


## count(非主键字段)

count(非主键字段)操作有些特殊，我们先看一张截图：

![img.png](https://yuanjava.cn/assets/md/mysql/mysql-count*-sql-filed.png)

如上述截图：

InnoDB引擎的 user表中最开始有 3条数据，然后执行 insert into user(id,age) values (4,NULL); 插入一条 age=NULL的数据，各种count()的结果为：
- count(*) = 4；
- count(1) = 4；
- count(id) = 4;
- count(age) = 3;

MyISAM引擎的 person表中最开始有 3条数据，然后执行 insert into person(id,name) values (4,NULL); 插入一条 name=NULL的数据，各种count()的结果为：
- count(*) = 4；
- count(1) = 4；
- count(id) = 4;
- count(name) = 3;

从结果可以发现，user表通过 count(age)方式进行统计，表的总行数就会比其他几种 count()方式少 1条，为什么呢？

这是因为：
- 如果"非主键字段"（比如 user表的age字段），在表字段定义时不允许为 NULL，也就是该字段一定有值，则统计时只要按行累加即可；
- 如果"非主键字段"（比如 user表的age字段），在表字段定义时允许为 NULL，也就是该字段可能为NUll，则统计时不会把NULL的累加进来；


## 总结
通过上面的对比和分析，我们可以得出 count()函数按照执行效率从低到高依次排序为：

count(非主键字段) < count(主键) < count(1) ≈ count( * )

因此，count(1) 或者 count( * )的效率最高。对于这两种方式的选择，建议尽量使用 count( * )，因为 MySQL优化器会选择最小的索引树进行统计，我们把这个优化的问题交给 MySQL优化器去解决。


count( * )、count(主键)、count(1) 都是返回满足条件的结果总行数；而 count(非主键字段），统计"非主键字段"不为 NULL 的总行数。

在生产中，如果对数据不要求特别精确，可以使用 "show table status" 方式获取。

## 参考文献

[MySQl8.0官方文档](https://dev.mysql.com/doc)

[MySQl8.0 MyISAM引擎官方文档](https://dev.mysql.com/doc/refman/8.0/en/myisam-storage-engine.html)

[MySQL实战 45讲]


## 鸣谢
如果你觉得本文章对你有帮助，感谢转发给更多的好友，关注我：猿java，为你呈现更多的硬核文章。

<img src="https://yuanjava.cn/assets/img/pub.jpg" alt="drawing" style="width:300px;"/>

