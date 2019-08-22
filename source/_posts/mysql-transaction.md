---
title: mysqls事务
date: 2019-08-22 17:49:43
tags: mysql
---
# 事务

## 事务介绍

MySQL的事务是由存储引擎实现的，而且支持事务的存储引擎不多。这里主要讲InnoDB存储引擎的事务。

事务用来维护数据库数据的完整性，保证成批的sql语句要么全部执行，要么全部不执行。

事务用来管理DDL、DML、DCL操作，比如insert、delete、update语句，默认是自动提交的。



## 事务的四大特性(ACID)

- Atomicity（原子性）：构成事务的所有操作必须是一个逻辑单元，要么全部执行，要么全部不执行。

- Consistency（一致性）：数据库在事务执行前后状态都必须是稳定的或者一致的。

- Isolation（隔离性）：事务之间不会相互影响。

  由锁机制和MVCC机制来实现。

  MVCC（多版本并发控制）：优化读写性能（读不加锁，读写不冲突）

- Durability（持久性）：事务执行成功后必须全部写入磁盘。



## 事务开启

- begin或start transaction·： 开启事务
- commit或rollback： 提交或回滚

### InnoDB架构图

{% asset_img mysql-lock-5.png %}

{% asset_img mysql-lock-6.png %}

上图详细显示了InnoDB存储引擎的体系架构，从图可见，InnoDB存储引擎由**内存池、后台线程和磁盘文件**三大部分组成。下面简单了解一下内存相关的概念和原理。

#### InnoDB内存结构

##### Buffer Pool缓冲池

- data page和index page ： 数据页和索引页。

  page是InnoDB存储的最基本单位，也是InnoDB磁盘管理的最小单位。

  做增删改操作时，缓存里的数据页和磁盘里的数据页不一致，该数据页为脏页。

- 插入缓冲 (insert buffer page)：进行插入操作时逻辑较为负责，如：主键排序、维持索引结构等，因此需要单独的一个插入缓冲区。

- 自适应哈希索引 (adaptive hash index) : 自适应哈希索引时K-V结构的hash结构，InnoDB会根据访问的频率和模式，为热点页建立hash索引，来提高查询效率。

##### Redo Log Buffer 重做日志缓冲

{% asset_img mysql-transaction-2.png %}

重做日志(redo log)：如果要存储数据则先存储到redo log日志中(顺序io，速度快)，一旦内存崩了则可以从内存中找。

+ 重做日记保证了数据的可靠性，InnoDB采用的预写日志(Write Ahead Log) 策略，即当事务提交时，先写重做日志，然后再择时将脏页写入磁盘。如果发生宕机导致数据丢失，就通过重做日志进行数据恢复。

+ redo log file写入成功，则认为commit成功，否则commit失败。

+ redo log file:ib_logfile0, ib_logfile1	默认大小为8M，可以通过innodb_log_buffer_size修改大小。

+ Force Log at Commit机制实现事务的持久性。即当事务提交时，必须先将该事务的所有日志写入到重做日志文件进行持久化，然后事务才算提交完成。为了确保每次日志都写到重做日志文件，每次将重做日志缓冲写入重做日志后，必须调用一次fsync操作，将缓冲文件从文件系统缓存中真正写入到磁盘。

  > innodb_flush_log_at_trx_commit: 1
  >
  > 该参数默认值为1，

  

+ 重做日志的落盘机制如下图

  {% asset_img mysql-transaction-3.png %}

{% asset_img mysql-transaction-1.png %}

##### Double Write双写

#### InnoDB磁盘文件

