---
layout: post
title: 肝了一周，这下彻底把 MySQL的锁搞懂了
category: mysql
tags: [mysql]
description:  MySQL 数据库有哪些锁，他们是如何工作的？
keywords: MySQL,MySQL锁,MySQL数据库
---

你好，我是Weiki，欢迎来到猿java。

最近，同事在生产上遇到一个 MySQL死锁的问题，于是在帮忙解决问题后，特意花了一周的时间，把 MySQL所有的锁都整理了一遍，今天就来一起聊聊 MySQL锁。

> 申明：本文基于 MySQL 8.0.30 版本，InnoDB引擎

MySQL数据库锁设计的初衷是处理并发问题，保证数据安全。MySQL 数据库锁可以从下面 3个维度进行划分：

- 按照锁的使用方式，MySQL锁可以分成共享锁、排它锁两种；
- 根据加锁的范围，MySQL锁大致可以分成全局锁、表级锁和行锁三类；
- 从思想层面上看，MySQL锁可以分为悲观锁、乐观锁两种；

我们会先讲解共享锁和排它锁，然后讲解全局锁、表级锁和行锁，因为这三种类别的锁中，有些是共享锁，有些是排他锁，最后，我们再讲解 悲观锁和乐观锁。

![img.png](https://yuanjava.cn/assets/md/mysql/mysql-lock-outline.png)

## 共享锁&排他锁

### 1.共享锁

共享锁，Share lock，也叫读锁。它是指当对象被锁定时，允许其它事务读取该对象，也允许其它事务从该对象上再次获取共享锁，但不能对该对象进行写入。
加锁方式是：
```shell
# 方式1
select ... lock in share mode;
# 方式2
select ... for share;
```

如果事务T1 在某对象持有共享(S)锁，则事务T2 需要再次获取该对象的锁时，会出现下面两种情况：
- 如果T2 获取该对象的共享(S)锁，则可以立即获取锁；
- 如果T2 获取该对象的排他(X)锁，则无法获取锁；

为了更好的理解上述两种情况，可以参照下面的执行顺序流和实例图：

**给user表加共享锁**

| 加锁线程   sessionA                                         | 线程B    sessionB                                                |
|---------------------------------------------------------|----------------------------------------------------------------|
| #开启事务 <br> begin;                                       |                                                                |
| #对user整张表加共享锁<br>select * from user lock in share mode; |                                                                |
|                                                         | #获取user表上的共享锁ok，select操作成功执行<br>select * from user;            |
|                                                         | #获取user表上的排他锁失败，操作被堵塞<br>delete from user where id = 1;        |
| #提交事务<br>#user表上的共享锁被释放<br> commit;                     |                                                                |
|                                                         | #获取user表上的排他锁成功，delete操作执行ok<br>delete from user where id = 1; |

![img.png](https://yuanjava.cn/assets/md/mysql/share-lock.png)

**给user表id=3的行加共享锁**

| 加锁线程  sessionA                                                            | 线程B  sessionB                                                          | 线程C  sessionC                                                          |
|---------------------------------------------------------------------------|------------------------------------------------------------------------|------------------------------------------------------------------------|
| #开启事务 <br>begin;                                                          |                                                                        |                                                                        |
| #给user表id=3的行加共享锁<br>select * from user<br> where id = 3 lock in share mode; |                                                                        |                                 |
|                                                                           | #获取user表id=3行上的共享锁ok<br>#select操作执行成功<br>select * from user where id=3; | #获取user表id=3行上的共享锁ok<br>#select操作执行成功<br>select * from user where id=3; |
|                                                                           | #获取user表id=3行上的排它锁失败<br>#delete操作被堵塞<br>delete from user where id = 3;  | #获取user表id=4行上的排它锁成功<br>#delete操作执行成功<br>delete from user where id = 4; |
| #提交事务<br>#user表id=3的行上共享锁被释放<br> commit;                                  |                                                                        |
|                                                                           | #获取user表id=3行上的排它锁成功<br>#被堵塞的delete操作执行ok<br> delete from user where id = 3;                              |

![img.png](https://yuanjava.cn/assets/md/mysql/share-lock-row.png)

通过上述两个实例可以看出：
- 当共享锁加在user表上，则其它事务可以再次获取user表的共享锁，其它事务再次获取user表的排他锁失败，操作被堵塞；
- 当共享锁加在user表id=3的行上，则其它事务可以再次获取user表id=3行上的共享锁，其它事务再次获取user表id=3行上的排他锁失败，操作被堵塞，但是事务可以再次获取user表id!=3行上的排他锁；


### 2. 排它锁

排它锁，Exclusive Lock，也叫写锁或者独占锁，主要是防止其它事务和当前加锁事务锁定同一对象。同一对象主要有两层含义：
- 当排他锁加在表上，则其它事务无法对该表进行insert,update,delete,alter,drop等更新操作；
- 当排他锁加在表的行上，则其它事务无法对该行进行insert,update,delete,alter,drop等更新操作；

排它锁加锁方式为：
```shell
select ... for update;
```

为了更好的说明排他锁，可以参照下面的执行顺序流和实例图：

**给user表对象加排他锁**

| 加锁线程   sessionA                                 | 线程B    sessionB                                          |
|-------------------------------------------------|----------------------------------------------------------|
| #开启事务 begin;                                    |                                                          |
| #对user整张表加排他锁<br>select * from user for update; |                                                          |
|                                                 | #获取user表上的共享锁ok，select执行成功<br>select * from user;        |
|                                                 | #获取user表上的排他锁失败，操作被堵塞<br>delete from user where id=3;    |
| #提交事务<br>#user表上的排他被释放<br> commit;              |                                                          |
|                                                 | #获取user表上的排他锁成功，操作执行ok<br>delete from user where id = 1; |

![img.png](https://yuanjava.cn/assets/md/mysql/excusive-lock-1.png)

**给user表id=3的行对象加排他锁**

| 加锁线程  sessionA                                                       | 线程B  sessionB                                          | 线程C  sessionC                                          |
|----------------------------------------------------------------------|--------------------------------------------------------|--------------------------------------------------------|
| #开启事务 <br>begin;                                                     |                                                        |                                                        |
| #给user表id=3的行加排他锁<br>select * from user<br> where id = 3 for update; |                                                        ||
|                                                                      | #获取user表id=3行上的共享锁ok<br>select * from user where id=3; | #获取user表id=3行上的共享锁ok<br>select * from user where id=3; |
|                                                                      | #获取user表id=3行上的排它锁失败<br>delete from user where id = 3; | #获取user表id=4行上的排它锁成功<br>delete from user where id = 4; |
| #提交事务<br>#user表id=3的行上排他锁被释放<br> commit;                     |                                                        |
|                                                                      | #获取user表id=3行上的排它锁成功<br>#被堵塞的delete操作执行ok<br>delete from user where id = 3;              |


![img.png](https://yuanjava.cn/assets/md/mysql/excusive-lock-2.png)

## 全局锁&表级锁&行锁

## 1. 全局锁

### 1.1 定义

全局锁，顾名思义，就是对整个数据库实例加锁。它是粒度最大的锁。

### 1.2 加锁

在MySQL中，通过执行 flush tables with read lock 指令加全局锁：

```shell
flush tables with read lock
```
指令执行完，<font color="red">整个数据库就处于只读状态了</font>，其他线程执行以下操作，都会被阻塞：
- 数据更新语句被阻塞，包括 insert, update, delete语句；
- 数据定义语句被阻塞，包括建表 create table,alter table、drop table 语句；
- 更新操作事务commit语句被阻塞；

### 1.3 释放锁

MySQl释放锁有2种方式：
1. 执行 unlock tables 指令
```mysql
unlock tables
```
2. 加锁的会话断开，全局锁也会被自动释放

为了更好的说明全局锁，可以参照下面的执行顺序流和实例图：

| 加锁线程 sessionA                     | 线程B sessionB   |
|-----------------------------------|----------------|
| flush tables with read lock; 加全局锁 |                |
| select user表ok                    | select user表ok |
| insert user表堵塞                    | insert user表堵塞 |
| delete user表堵塞                    | delete user表堵塞 |
| drop user 表堵塞                     | drop user 表堵塞  |
| alter user表 堵塞                    | alter user表 堵塞 |
| unlock tables； 解锁                 |                |
| 被堵塞的修改操作执行ok                      | 被堵塞的修改操作执行ok   |

![img.png](https://yuanjava.cn/assets/md/mysql/global-lock.png)

通过上述的实例可以看出，当加全局锁时，库下面所有的表都处于只能状态，不管是当前事务还是其他事务，对于库下面所有的表只能读，不能执行insert，update，delete，alter，drop等更新操作。

### 1.4 使用场景

全局锁的典型使用场景是做全库逻辑备份，在备份过程中整个库完全处于只读状态。如下图：

![img.png](https://yuanjava.cn/assets/md/mysql/global-lock-use.png)


- 假如在主库上备份，备份期间，业务服务器不能对数据库执行更新操作，因此涉及到更新操作的业务就瘫痪了；
- 假如在从库上备份，备份期间，从库不能执行主库同步过来的 binlog，会导致主从延迟越来越大，如果做了读写分离，那么从库上获取数据就会出现延时，影响业务；

从上述分析可以看出，使用全局锁进行数据备份，不管是在主库还是在从库上进行备份操作，对业务总是不太友好。那不加锁行不行？我们可以通过下面还钱转账的例子，看看不加锁会不会出现问题：

![img.png](https://yuanjava.cn/assets/md/mysql/back_lock.png)

- 备份前：账户A 有1000，账户B 有500
- 此时，发起逻辑备份
- 假如数据备份时不加锁，此时，客户端A 发起一个还钱转账的操作：账户A 往账户B 转200
- 当账户A 转出200完成，账户B 转入200 还未完成时，整个数据备份完成
- 如果用该备份数据做恢复，会发现账户A 转出了200，账户B 却没有对应的转入记录，这样就会产生纠纷：A 说我账户少了 200， B 说我没有收到，最后，A，B谁都不干。

既然不加锁会产生错误，加全局锁又会影响业务，那么有没有两全其美的方式呢？

有，MySQL官方自带的逻辑备份工具 mysqldump，具体指令如下：

```shell
mysqldump –single-transaction
```
执行该指令，在备份数据之前会先启动一个事务，来确保拿到一致性视图， 加上 MVCC 的支持，保证备份过程中数据是可以正常更新。但是，single-transaction方法只适用于库中所有表都使用了事务引擎，如果有表使用了不支持事务的引擎，备份就只能用 FTWRL 方法。


## 2. 表级锁

MySQL 表级锁有两种：
1. 表锁
2. 元数据锁（metadata lock，MDL)

### 2.1 表锁

表锁就是对整张表加锁，包含读锁和写锁，由MySQL Server实现，表锁需要显示加锁或释放锁，具体指令如下：

```shell
# 给表加写锁
lock tables tablename write;

# 给表加读锁
lock tables tablename read;

# 释放锁
unlock tables;
```

**读锁**：代表当前表为只读状态，读锁是一种共享锁。需要注意的是，读锁除了会限制其它线程的操作外，也会限制加锁线程的行为，具体限制如下：
1. 加锁线程只能对当前表进行读操作，不能对当前表进行更新操作，不能对其它表进行所有操作；
2. 其它线程只能对当前表进行读操作，不能对当前表进行更新操作，可以对其它表进行所有操作；

为了更好的说明读锁，可以参照下面的执行顺序流和实例图：

| 加锁线程 sessionA                        | 线程B   sessionB    |
|--------------------------------------|-------------------|
| #给user表加读锁<br>lock tables user read; |                   |
| select user表 ok                      | select user表 ok   |
| insert user表被拒绝                      | insert user表堵塞    |
| insert address表被拒绝                   | insert address表ok |
| select address表被拒绝                   | alter user表堵塞     |
| unlock tables; 释放锁                   |                   |
|                                      | 被堵塞的修改操作执行ok      |

![img.png](https://yuanjava.cn/assets/md/mysql/read-lock.png)

**写锁**：写锁是一种独占锁，需要注意的是，写锁除了会限制其它线程的操作外，也会限制加锁线程的行为，具体限制如下：
1. 加锁线程对当前表能进行所有操作，不能对其它表进行任何操作；
2. 其它线程不能对当前表进行任何操作，可以对其它表进行任何操作；

为了更好的说明写锁，可以参照下面的执行顺序流和实例图：

| 加锁线程 sessionA                         | 线程B sessionB                     |
|---------------------------------------|--------------------------|
| #给user表加写锁<br>lock tables user write; |                          |
| select user表 ok                       | select user表 ok          |
| insert user表被拒绝                       | insert user表堵塞           |
| insert address表被拒绝                    | insert address表ok        |
| select address表被拒绝                    | alter user表堵塞            |
| unlock tables; 释放锁                    |                          |
|                                       | 堵塞在user表的上更新操作执行ok |

![img.png](https://yuanjava.cn/assets/md/mysql/write-lock.png)

### 2.2 MDL元数据锁

元数据锁：metadata lock，简称MDL，它是在MySQL 5.5版本引进的。元数据锁不用像表锁那样显式的加锁和释放锁，而是在访问表时被自动加上，以保证读写的正确性。加锁和释放锁规则如下：

- MDL读锁之间不互斥，也就是说，允许多个线程同时对加了 MDL读锁的表进行CRUD(增删改查)操作；
- MDL写锁，它和读锁、写锁都是互斥的，目的是用来保证变更表结构操作的安全性。也就是说，当对表结构进行变更时，会被默认加 MDL写锁，因此，如果有两个线程要同时给一个表加字段，其中一个要等另一个执行完才能开始执行。
- MDL读写锁是在事务commit之后才会被释放；

为了更好的说明 MDL读锁规则，可以参照下面的顺序执行流和实例图：

| 加锁线程 sessionA                | 其它线程  sessionB              |
|------------------------------|-----------------------------|
| 开启事务<br>begin;               |                             |
| select user表，user表会默认加上MDL读锁 |                             |
| select user表ok               | select user表ok              |
| insert user表ok               | insert user表ok              |
| update user表ok               | update user表ok              |
| delete user表ok               | delete user表ok              |
|                              | alter user表，获取MDL写锁失败，操作被堵塞 |
| commit;提交事务，MDL读锁被释放         |                             |
|                              | 被堵塞的修改操作执行ok                |

![img.png](https://yuanjava.cn/assets/md/mysql/MDL-read-lock.png)

为了更好的说明 MDL写锁规则，可以参照下面的顺序执行流和实例图：

| 加锁线程  sessionA                    | 线程B  sessionB                     | 线程C  sessionC                    |
|-----------------------------------|-----------------------------------|----------------------------------|
| #开启事务<br> begin;                  |                                   |                                  |
| #user表会默认加上MDL读锁<br>select user表， |                                   |
| select user表ok                    | select user表ok                    | select user表ok                   |
|                                   | #获取MDL写锁失败<br>alter user表操作被堵塞    | #获取MDL读锁失败<br>select * from user; |
| 提交事务，MDL读锁被释放                     |                                   |                                  |
|                                   | #MDL写锁被释放<br>被堵塞的alter user操作执行ok |    |
|                                   |                                   | #被堵塞的select 操作执行ok               |


![img.png](https://yuanjava.cn/assets/md/mysql/MDL-write-lock.png)


### 2.3 意向锁

由于InnoDB引擎支持多粒度锁定，允许行锁和表锁共存，为了快速的判断表中是否存在行锁，InnoDB推出了意向锁。

意向锁，Intention lock，它是一种表锁，用来标识事务打算在表中的行上获取什么类型的锁。
不同的事务可以在同一张表上获取不同种类的意向锁，但是第一个获取表上意向排他(IX) 锁的事务会阻止其它事务获取该表上的任何 S锁 或 X 锁。反之，第一个获得表上意向共享锁(IS) 的事务可防止其它事务获取该表上的任何 X 锁。

意向锁通常有两种类型：
- 意向共享锁(IS)，表示事务打算在表中的各个行上设置共享锁。
- 意向排他锁(IX)，表示事务打算对表中的各个行设置排他锁。

意向锁是InnoDB自动加上的，加锁时遵从下面两个协议：
- 事务在获取表中行的共享锁之前，必须先获取表上的IS锁或更强的锁。
- 事务在获取表中行的排他锁之前，必须先获取表上的IX锁。

为了更好的说明意向共享锁，可以参照下面的顺序执行流和实例图：

| 加锁线程 sessionA                                                                 | 线程B  sessionB                                             |
|-------------------------------------------------------------------------------|-----------------------------------------------------------|
| #开启事务 <br>begin;                                                              |                                                           |
| #user表id=6加共享行锁 ，默认user表会 加上IS锁<br>select * from user where id = 6 for share; |                                                           |
|                                                                               | # 观察IS锁 <br>select * from performance_schema.data_locks\G |

![img.png](https://yuanjava.cn/assets/md/mysql/intention_lock_share.png)


| 加锁线程 sessionA                                                                | 线程B  sessionB                                           |
|------------------------------------------------------------------------------|---------------------------------------------------------|
| #开启事务<br>begin;                                                              |                                                         |
| #user表id=6加排他锁，默认user表会 加上IX锁<br>select * from user where id = 6 for update; |                                                         |
|                                                                              | # 观察IX锁 <br>select * from performance_schema.data_locks\G |

![img.png](https://yuanjava.cn/assets/md/mysql/intention_lock_excusive.png)


### 2.4 AUTO-INC锁

AUTO-INC锁是一种特殊的表级锁，当表中有AUTO_INCREMENT的列时，如果向这张表插入数据时，InnoDB会先获取这张表的AUTO-INC锁，等插入语句执行完成后，AUTO-INC锁会被释放。

AUTO-INC锁可以使用innodb_autoinc_lock_mode变量来配置自增锁的算法，innodb_autoinc_lock_mode变量可以选择三种值如下表：

| innodb_autoinc_lock_mode | 含义                                  |
|--------------------------|-------------------------------------|
| 0                        | 传统锁模式，采用 AUTO-INC 锁                 |
| 1                        | 连续锁模式，采用轻量级锁                        |
| 2                        | 交错锁模式(MySQL8默认)，AUTO-INC和轻量级锁之间灵活切换 |


为了更好的说明意AUTO-INC锁，可以参照下面的顺序执行流和实例图：



### 2.5 锁的兼容性

下面的图表总结了表级锁类型的兼容性

|  | X | IX | S | IS |
|----|----|----|----|----|
| X | 冲突 |冲突 |  冲突|冲突        |
| IX |  冲突|  兼容|   冲突 |    兼容  |
| S | 冲突 | 冲突| 兼容 | 兼容 |
| IS | 冲突 | 兼容 | 兼容 | 兼容 |



## 3. 行锁

行锁是针对数据表中行记录的锁。MySQL 的行锁是在引擎层实现的，并不是所有的引擎都支持行锁，比如，InnoDB引擎支持行锁而 MyISAM引擎不支持。

InnoDB 引擎的行锁主要有三类：
1. Record Lock： 记录锁，是在索引记录上加锁；
2. Gap Lock：间隙锁，锁定一个范围，但不包含记录；
3. Next-key Lock：Gap Lock + Record Lock，锁定一个范围(Gap Lock实现)，并且锁定记录本身(Record Lock实现)；


### 3.1 Record Lock

Record Lock：记录锁，是针对索引记录的锁，锁定的总是索引记录。

例如，select id from user where id = 1 for update; for update 就显式在索引id上加行锁(排他锁)，防止其它任何事务 update或delete id=1 的行，但是对user表的insert、alter、drop操作还是可以正常执行。

为了更好的说明 Record Lock锁，可以参照下面的执行顺序流和实例图：

| 加锁线程  sessionA                                                    | 线程B  sessionB                                     | 线程B  sessionC                                     |
|-------------------------------------------------------------------|---------------------------------------------------|---------------------------------------------------|
| #开启事务<br> begin;                                                  |                                                   |                                                   |
| 给user表id=1加写锁<br>select id from user<br> where id = 1 for update; |                                                   |
|                                                                   | update user set name = 'name121'<br> where id = 1; |
|                                                                   |                                                   | 查看 InnoDB监视器中记录锁数据<br>show engine innodb status\G |
| commit提交事务<br>record lock 被释放                                     |                                                   |
|                                                                   | 被堵塞的update操作执行ok                                  |

![img.png](https://yuanjava.cn/assets/md/mysql/record-lock.png)



### 3.2 Gap Lock

Gap Lock：间隙锁，锁住两个索引记录之间的间隙上，由InnoDB隐式添加。比如(1,3) 表示锁住记录1和记录3之间的间隙，这样记录2就无法插入，间隙可能跨越单个索引值、多个索引值，甚至是空。

![img.png](https://yuanjava.cn/assets/md/mysql/gap-lock-table.png)

为了更好的说明 Gap Lock间隙锁，可以参照下面的顺序执行流和实例图：

| 加锁线程  sessionA                                         | 线程B  sessionB                          | 线程C  sessionC                                      |
|--------------------------------------------------------|----------------------------------------|----------------------------------------------------|
| #开启事务<br> begin;                                       |                                        |                                                    |
| 加锁<br>select * from user<br> where age = 10 for share; |                                        |
|                                                        | insert into user(id,age) values(2,20); |
|                                                        |                                        | #查看 InnoDB监视器中记录锁数据<br>show engine innodb status\G |
| commit提交事务<br>Gap Lock被释放                            |                                        |
|                                                        | 被堵塞的insert操作执行ok                       |

![img.png](https://yuanjava.cn/assets/md/mysql/gap-lock.png)

上图中，事务A(sessionA)在加共享锁的时候产生了间隙锁(Gap Lock)，事务B(sessionB)对间隙中进行insert/update操作，需要先获取排他锁(X)，导致阻塞。事务C(sessionC)通过"show engine innodb status\G" 指令可以查看到间隙锁的存在。需要说明的，间隙锁只是锁住间隙内部的范围，在间隙外的insert/update操作不会受影响。

Gap Lock锁，只存在于可重复读隔离级别，目的是为了解决可重复读隔离级别下幻读的现象。

### 3.3 Next-Key Lock

Next-Key锁，称为临键锁，它是Record Lock + Gap Lock的组合，用来锁定一个范围，并且锁定记录本身锁，它是一种左开右闭的范围，可以用符号表示为：(a,b]。

![img.png](https://yuanjava.cn/assets/md/mysql/next-key-lock-table.png)

为了更好的说明 Next-Key Lock间隙锁，可以参照下面的顺序执行流和实例图：


| 加锁线程  sessionA                                         | 线程B  sessionB                                                      | 线程C  sessionC                                | 线程D  sessionD                                      |
|--------------------------------------------------------|--------------------------------------------------------------------|----------------------------------------------|----------------------------------------------------|
| #开启事务<br> begin;                                       |                                                                    |                                              |                                                    |
| 加锁<br>select * from user<br> where age = 10 for share; |                                                                    |
|                                                        | #获取锁失败，insert操作被堵塞<br>insert into user(id,age)  <br> values(2,20); |
|                                                        |                                                                    | update user set name='name1'  <br> where age = 10; | #查看 InnoDB监视器中记录锁数据<br>show engine innodb status\G |
| 提交事务Gap Lock被释放  <br> commit                         |                                                                    |
|                                                        | 被堵塞的insert操作执行ok                                                   | 被堵塞的update操作执行ok                             | |


![img.png](https://yuanjava.cn/assets/md/mysql/next-key-lock.png)

上图中，事务A(sessionA)在加共享锁的时候产生了间隙锁(Gap Lock)，事务B(sessionB)对间隙中进行insert操作，需要先获取排他锁(X)，导致阻塞。
事务C(sessionC)对间隙中进行update操作，需要先获取排他锁(X)，导致阻塞。
事务D(sessionD)通过"show engine innodb status\G" 指令可以查看到间隙锁的存在。需要说明的，间隙锁只是锁住间隙内部的范围，在间隙外的insert/update操作不会受影响。

### 3.4 Insert Intention Lock

插入意向锁，它是一种特殊的间隙锁，特指插入操作产生的间隙锁。

为了更好的说明 Insert Intention Lock锁，可以参照下面的顺序执行流和实例图：

| 加锁线程  sessionA                                         | 线程B  sessionB                                                 | 线程C  sessionC                                      |
|--------------------------------|---------------------------------------------------------------|------------------------|
| #开启事务<br> begin;                                       |                                                               |                                                    |
| 加锁<br>select * from user<br> where age = 10 for share; |                                                               |
|                                                        | #获取锁失败，insert操作被堵塞 <br>insert into user(id,age) values(2,20); |
|                                                        |                                                               | #查看 InnoDB监视器中记录锁数据<br>show engine innodb status\G |
| commit提交事务<br>Gap Lock被释放                            |                                                               | |
|                                                        | #被堵塞的insert操作执行ok <br> insert into user(id,age) values(2,20); | |

![img.png](https://yuanjava.cn/assets/md/mysql/gap-lock.png)


## 乐观锁&悲观锁

在MySQL中，无论是悲观锁还是乐观锁，都是人们对概念的一种思想抽象，它们本身还是利用 MySQL提供的锁机制来实现的。其实，除了在MySQL数据，像 Java语言里面也有乐观锁和悲观锁的概念。

- 悲观锁，可以理解成：在对任意记录进行修改前，先尝试为该记录加上排他锁(exclusive locking)，采用的是先获取锁再操作数据的策略，可能会产生死锁；
- 乐观锁，是相对悲观锁而言，一般不会利用数据库的锁机制，而是采用类似版本号比较之类的操作，因此乐观锁不会产生死锁的问题；


## 死锁和死锁检测

当并发系统中不同线程出现循环资源依赖，涉及的线程都在等待别的线程释放资源时，就会导致这几个线程都进入无限等待的状态，称为死锁。可以通过下面的指令查看死锁

```shell
show engine innodb status\G
```

当出现死锁以后，有两种策略：
- 一种策略是，直接进入等待，直到超时。这个超时时间可以通过参数 innodb_lock_wait_timeout 来设置，InnoDB 中 innodb_lock_wait_timeout 的默认值是 50s。
- 另一种策略是，发起死锁检测，发现死锁后，主动回滚死锁链条中的某一个事务，让其它事务得以继续执行。将参数 innodb_deadlock_detect 设置为 on，表示开启死锁检测。


## 总结

本文基于 MySQL 8.0.30 版本和InnoDB引擎，对MySQL中的锁进行了讲解，每种锁都有其特定的使用场景。
作为经常和 MySQL 打交道的Java程序员来说，对MySQL锁了解的越深，越可以帮助我们更好的去写出高性能的SQL语句。


## 参考

[InnoDB Locking官方资料](https://dev.mysql.com/doc/refman/8.0/en/innodb-locking.html#:~:text=A%20record%20lock%20is%20a,is%20defined%20with%20no%20indexes.)

《MySQL实战45讲》

## 鸣谢
如果你觉得本文章对你有帮助，感谢转发给更多的好友，关注我：猿java，为你呈现更多的硬核文章。

<img src="https://yuanjava.cn/assets/img/pub.jpg" alt="drawing" style="width:300px;"/>
