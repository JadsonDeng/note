---
title: mysql3：事务
date: 2019-09-18 18:00:21
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

## InnoDB架构图

{% asset_img mysql-lock-5.png %}

{% asset_img mysql-lock-6.png %}

上图详细显示了InnoDB存储引擎的体系架构，从图可见，InnoDB存储引擎由**内存池、后台线程和磁盘文件**三大部分组成。下面简单了解一下内存相关的概念和原理。

### InnoDB内存结构

#### Buffer Pool缓冲池

- data page和index page ： 数据页和索引页。

  page是InnoDB存储的最基本单位，也是InnoDB磁盘管理的最小单位。

  做增删改操作时，缓存里的数据页和磁盘里的数据页不一致，该数据页为脏页。

- 插入缓冲 (insert buffer page)：进行插入操作时逻辑较为负责，如：主键排序、维持索引结构等，因此需要单独的一个插入缓冲区。

- 自适应哈希索引 (adaptive hash index) : 自适应哈希索引时K-V结构的hash结构，InnoDB会根据访问的频率和模式，为热点页建立hash索引，来提高查询效率。

##### Redo Log Buffer 重做日志缓冲

{% asset_img mysql-transaction-2.png %}

重做日志(redo log)：如果要存储数据则先存储到redo log日志中(顺序io，速度快)，一旦内存崩了则可以从内存中找。

重做日记保证了数据的可靠性，InnoDB采用的预写日志(Write Ahead Log) 策略，即当事务提交时，先写重做日志，然后再择时将脏页写入磁盘。如果发生宕机导致数据丢失，就通过重做日志进行数据恢复。

{% asset_img mysql-transaction-1.png %}

##### Double Write双写

Double Write带给InnoDB存储引擎的也是数据的可靠性。

{% asset_img mysql-transaction-3.png %}

如图所示，**Double Write由两部分组成，一部分是内存中的Double Write Buffer，大小为2M，另一部分是物理磁盘上共享表空间连续的128个页，大小也为2M。**在对缓冲池的脏页进行刷新时，并不直接写磁盘，而是通过memcpy函数将脏页先复制到内存中的Double Write Buffer区域，之后通过Double Write Buffer再分2次，每次1M顺序的写入共享表空间的物理磁盘上上，然后马上调用fsync函数，同步磁盘，避免操作系统缓冲带来的问题。在完成Double Write页的写入后，再将Double Write Buffer中的页写入各个表空间文件中。如果操作系统在将页写入磁盘的过程中发生了崩溃，在恢复过程中，InnoDB存储引擎可以从共享表空间中的Double Write中找到该页的一个副本，将其复制到表空间文件中，再应用重做日志。

##### CheckPoint（检查点）

检查点表示将脏页数据写入磁盘的时机，所以检查点也就意味着脏页数据的写入。

CheckPoint的目的：

+ 缩短数据库的恢复时间
+ buffer pool空间不够用时，将脏页刷新到磁盘
+ redolog不可用时，刷新脏页

检查点分类：

+ sharp checkpoint：完全检查点，数据库正常关闭时，会触发把所有的脏页都写入到磁盘上
+ fuzzy checkpoint：模糊检查点，正常使用时，部分页写入磁盘。分为4种
  1. master thread checkpoint: 以每秒或者每十秒的速度从缓冲池的脏页列表中刷新一定比例的页回磁盘，这个过程是异步的。
  2. flush_lru_checkpoint: 读取最近最少使用列表(Least Recently Used) list，找到脏页，写入磁盘。最近最少使用。
  3. async/sync flush checkpoint: log file快慢了，会批量的触发数据也的回写，这个事件触发的时候又分为异步和同步，不可被覆盖的redolog占log file的比值：75%-异步，90%-同步
  4. dirty page too much checkpoint:  默认是脏页占比75%的时候，就会触发刷盘，将脏页写入磁盘。

#### InnoDB磁盘文件

##### 系统表空间和用户表空间

{% asset_img mysql-transaction-5.png %}

Innodb_data_file_path的格式如下：

```
innodb_data_file_path=datafile1[,datafile2,datafile2...]
```

用户可以通过多个文件组成一个表空间，同时指定文件的属性：

```
innodb_data_file_path=/db/ibdata1:1000M,/db/ibdata2:1000M:autoentend
```

这里将/data/ibdata1和/data/ibdata2两个文件组成系统表空间。

+ 系统表空间（也叫共享表空间）包括：
  1. 数据字典(data dictionary)：记录数据库相关信息
  2. Double write buffer：解决部分写失败（页断裂）
  3. insert buffer：内存insert buffer数据，周期写入共享表空间，防止意外宕机。
  4. 回滚段(rollback segments)：
  5. undo空间：undo页
+ 用户表空间（也叫独立表空间）：
  1. 每个表的数据和索引都会存在自己的表空间中
  2. 每个表的结构
  3. undo空间：undo页（需要设置）

##### 重做日志文件和归档文件

{% asset_img mysql-transaction-6.png %}

redo log file的存储文件为：ib_logfile0和ib_logfile1

在日志组中每个重做日志文件的大小一致，并以**循环写入**的方式运行。

InnoDB存储引擎先写入重做日志文件1，当文件1被写满时，会切换到重做日志文件2，文件1进行落盘。再当重做日志文件2倍写满时，再切换到文件1。

## InnoDB的事务分析

{% asset_img mysql-transaction-7.png %}

数据库事务具有ACID4大特性：

+ 原子性(automicity)：事务最小工作单元，要么全成功，要么全失败。
+ 一致性(consistency)：事务开始和结束后，事务完整性不会被破坏。
+ 隔离性(isolation)：不同事务间互不影响。4中隔离级别为：RU（读未提交）、RC（读已提交）、RR（可重复度）、SERIALIZABLE（串行化）
+ 持久性(durability)：事务提交后，对数据的影响是永久性的，即使系统故障也不会丢失。

**事务的隔离性是由多版本并发控制机制和锁实现的，而原子性、一致性和持久性是通过InnoDB的redo log、undo log和Force Log at Commit机制来实现的**

#### 原子性、一致性、持久性

##### RedoLog

数据库日志和数据库落盘机制如下图：

{% asset_img mysql-transaction-6.png %}

redolog写入磁盘时，必须进行一次操作系统的fsync操作，防止redolog只是写入了操作系统的磁盘缓存中。参数innodb_flush_log_attrx_commit可以控制redolog日志刷新到磁盘的策略。

##### UndoLog

数据库崩溃重启后需要从redo log中把未落盘的脏页数据恢复出来，重新写入磁盘，保证用户的数据不丢失。当然，在崩溃恢复中还需要回滚没有提交的事务。由于回滚操作需要undo log日志的支持，undo log的完整性和可靠性需要redo log来保证，所以崩溃恢复先做redo log恢复数据，然后做undo log回滚。

在事务执行的过程中，除了记录redo log，还会记录一定量的undo log。undo log记录了数据在每个操作前的状态，如果事务执行过程中需要回滚，就可以根据undo log进行回滚操作。 

{% asset_img mysql-transaction-8.png %}

undo log的存储不同于redolog，它存放在数据库内部一个特殊的段(segment)中，这个段称为回滚段。回滚段位于共享表空间中。undo段中以undo page为最小单位。**undo page和存储数据库数据和索引的页类似。因为redo log是物理日志，记录的是数据库页的物理修改操作。所以undo log(也堪称数据库数据)的写入也会产生redo log，也就是undo log的产生会伴随着redo log的产生，这是因为undo log也需要redo log持久性的保护。**

如上图所示，表空间中有回滚段、叶节点段和非页节点段，而三者都有对应的页结构。

下图是undo page的记录格式：

{% asset_img mysql-transaction-9.png %}

下面是数据库事务的整个流程：

{% asset_img mysql-transaction-10.png %}

事务进行过程中，每次sql语句执行，都会记录undo log 和 redo log，然后更新数据形成脏页，然后redo log按照时间或者空间等条件进行落盘，undo log和脏页按照checkpoint进行落盘，落盘后相应的redo log就可以删除了。此时，事务还未commit，如果发生崩溃，则首先检查checkpoint记录，使用相应的redo log进行数据和undo log的恢复，然后查看undo log的状态发现事务尚未提交，然后就使用undo log进行回滚。事务执行commit操作时，会将本事务相关的所有redo log都进行落盘，只有所有redo log都落盘成功，事务才算commit成功。然后内存中的数据脏页继续按照checkpoint进行落盘。如果此时发生了崩溃，则只使用redo log进行数据恢复。

#### 隔离性

##### 事务并发问题

在事务的**并发操作**中可能会出现一些问题：

+ 丢失更新：两个事务同时修改同一条记录时，会存在丢失更新的问题
+ 脏读：一个事务读取到另一个事务未提交的数据
+ 不可重复读：一个事务因读取到另一个事务已提交的**update或delete**数据。导致对同一条记录读取两次以上记录的结果不一致。
+ 幻读：一个事务因读取到另一个事务已提交的**insert**数据。导致对同一张表读取两次以上的结果不一致。

##### 事务隔离级别

+ read uncommited(读未提交)：最低级别，任何情况都无法保证

+ read commited(读已提交)：可避免脏读的发生

+ repeatable read(可重复读)：可避免脏读、不可重复度的发生。

  **注：InnoDB的RR还可以解决幻读，主要原因是Next-key锁，只有RR才能使用Next-key锁**

+ serializable(串行化)：可以避免脏读、不可重复读、幻读发生（由MVCC降级为Locking-Base CC）。

考虑一个现实场景：

管理者要查询所有用户的存款总额，假设出了用户A和用户B之外其它用户的存款总额都为0，A和B用户各有存款1000，所以所有用户的存款总额为2000。但在查询过程中，用户A会向用户B进行转账操作。转账操作和查询总额操作的时序图如下：

{% asset_img mysql-transaction-11.png %}

如果没有任何的并发控制机制，查询总额事务先读取了用户A的账户存款，然后转账事务改变了用户A和用户B的存款，最后查询总额事务读取了转账后的用户B的存款，导致最终统计的存款总额多了100元，发生错误。

**read uncommitted**

```mysql
-- 创建账户表并初始化数据
create table account( id int primary key, aname varchar(100), account int );
alter table account add index idx_name(aname);
innsert into account values (1,'a',1000),(2,'b',1000);

-- Session1: 设置隔离级别为：读未提交
Session1: set session transaction isolation level read uncommitted;
-- 查询隔离级别
select @@TX_ISOLATION;

-- Session1 开启事务
Session1: begin;select * from account where aname='a';
+----+-------+---------+
| id | aname | account |
+----+-------+---------+
|  1 | a     |    1000 |
+----+-------+---------+
1 row in set (0.01 sec)

-- Session2开启事务，进行转账
Session2: begin;
Session2: update account set account=1100 where aname='b';

-- Session1查询B的存款金额
-- Session1读取到了Session2未提交的事务，出现了脏读
Session1: select * from account where aname='b';
+----+-------+---------+
| id | aname | account |
+----+-------+---------+
|  2 | b     |    1100 |
+----+-------+---------+
1 row in set (0.00 sec)

```

**使用锁机制可以解决上述的问题**。查询总额事务会对读取的行加锁，等到操作结束后再释放掉所有行上的锁。因为用户A的存款被锁，导致转账操作被阻塞，知道查询总额事务提交并将所有锁都释放掉。

{% asset_img mysql-transaction-12.png %}

但是这时可能会引入新的问题，当转账操作是从用户B向用户A进行转账时会导致死锁。转账事务会先锁住用户B的数据，等待用户A数据上的锁，但是查询总额的事务却先锁住了用户A的数据，等待用户B的数据上的锁。

**serializable**

```mysql
-- 设置Session1的隔离级别为serializable
set session transaction isolation level serializable;

-- Session1开启事务，查询A的数据
-- serializable隔离级别下，查询也会加锁 
Session1: begin;
Session1: select * from account where aname='a';
+----+-------+---------+
| id | aname | account |
+----+-------+---------+
|  1 | a     |    1000 |
+----+-------+---------+
1 row in set (0.00 sec)

-- Session2更新账户余额
Session2: update account set account=900 where aname='b';

-- Session1查询用户B的账户
-- 因为用户B的数据被Session2锁住，这里阻塞
Session1: select * from account where aname='b';

-- Session2更新用户A的数据
-- 因为A的数据被Session1锁住，B的数据被Session2锁住，并且在互相等待对方的锁注的资源，这里造成了死锁
Session2: update account set account=1100 where aname='a';
ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction

```

**repeatable read**

```mysql
-- 设置Session1的隔离级别为repeatable read
set session transaction isolation level repeatable read;

-- Session1开启事务，查询a的数据
Session1: begin;select * from account where aname='a';
+----+-------+---------+
| id | aname | account |
+----+-------+---------+
|  1 | a     |    1000 |
+----+-------+---------+
1 row in set (0.02 sec)

-- Session2开启事务，更新a的账户余额
Session2: begin;update account set account=1100 where aname='a';
Session2: select * from account where aname='a';
+----+-------+---------+
| id | aname | account |
+----+-------+---------+
|  1 | a     |    1100 |
+----+-------+---------+
1 row in set (0.00 sec)

-- Session1再次查询a的数据，发现还是1000
Session1: select * from account where aname='a';
+----+-------+---------+
| id | aname | account |
+----+-------+---------+
|  1 | a     |    1000 |
+----+-------+---------+

-- Session2提交事务，Session1再次查询a的账户，发现仍是1000
-- 这就是可重复读
Session2: commit;
Session1: select * from account where aname='a';
+----+-------+---------+
| id | aname | account |
+----+-------+---------+
|  1 | a     |    1000 |
+----+-------+---------+

```

**read committed**

```mysql
-- 设置Session1的隔离级别为read committed;
set session transaction isolation level read committed;

-- Session1开启事务，查询a的数据
Session1: begin;select * from account where aname='a';
+----+-------+---------+
| id | aname | account |
+----+-------+---------+
|  1 | a     |    1000 |
+----+-------+---------+
1 row in set (0.02 sec)

-- Session2开启事务，更新a的账户余额
Session2: begin;update account set account=1100 where aname='a';
Session2: select * from account where aname='a';
+----+-------+---------+
| id | aname | account |
+----+-------+---------+
|  1 | a     |    1100 |
+----+-------+---------+
1 row in set (0.00 sec)

-- Session1再次查询a的数据，发现还是1000
Session1: select * from account where aname='a';
+----+-------+---------+
| id | aname | account |
+----+-------+---------+
|  1 | a     |    1000 |
+----+-------+---------+

-- Session2提交事务，Session1再次查询a的账户，发现account已经变更
-- 这就是读已提交
Session2: commit;
Session1: select * from account where aname='a';
+----+-------+---------+
| id | aname | account |
+----+-------+---------+
|  1 | a     |    1100 |
+----+-------+---------+
```

##### InnoDB的MVCC实现

我们先看一下wiki上对MVCC的定义：

>**多版本并发控制**(Multiversion concurrency control， **MCC** 或 **MVCC**)，是[数据库管理系统](https://zh.wikipedia.org/wiki/数据库管理系统)常用的一种[并发控制](https://zh.wikipedia.org/wiki/并发控制)，也用于程序设计语言实现[事务内存](https://zh.wikipedia.org/wiki/事务内存)。

MVCC在mysql中的实现依赖的是**undo log**与**read view**

##### 当前读和快照读

在MVCC并发控制中，读操作可以分成2类：快照读(snapshot read)和当前读(current read)

> 快照读：读的是记录的可见版本，有可能是历史版本，不用加锁
>
> 当前读：读的是记录的最新版本，并且当前读返回的记录，都会加上锁，保证其它事务不会再并发修改这条记录

下面以mysql InnoDB为例，说明哪些操作是快照读，哪些操作是当前读。

**快照读**：简单的select操作，属于快照读，不加锁（会有例外，下面分析）不加读锁，读历史版本

**当前读**：特殊的读操作，insert/update/delete，属于当前读，需要加行写锁，读当前版本

{% asset_img mysql-transaction-15.png %}

##### 一致性非锁定读

一致性非锁定读(consistent nonlocking read)是指InnoDB存储引擎通过多版本控制读取当前数据库中行数据的方式。

如果读取的数据正在进行delete或update操作，这是读取操作不会去等待数据行上锁的释放。相反的，InnoDB回去读取行的一个最新可见快照。

{% asset_img mysql-transaction-16.png %}

{% asset_img mysql-transaction-17.png %}

如上图所示，当会话B提交事务后，会话A再次运行select * from test where id=1的sql语句时，两个事务隔离级别下得到的结果就不一样了。

##### UndoLog

InnoDB行记录有3个隐藏字段：分别对应**改行的rowid**、**事务号db_trx_id**和**回滚指针db_roll_ptr**，其中db_trx_id表示最近修改的事务的id，db_roll_ptr指向回滚段中的undo log.

根据行为的不同，undo log分为2种：insert undo log和update undo log

+ insert undo log：是在insert操作中产生的undo log。因为insert操纵产生的记录只在本事务可见，所以rollback后在该事务中直接删除，不需要进行purge操作。
+ update undo log：是在update或delete操作中产生的undo log。因为会对已经存在的记录产生影响。rollback时，MVCC会找它的历史版本进行恢复。

update或者delete操作中产生的undo log，因为会对已经存在的记录产生影响，为了提供MVCC机制，因此update undo log不能在事务提交时就进行删除，而是将事务提交时放入history list上，等待purge线程进行最后的删除操作。

如下图所示，初始状态：

{% asset_img mysql-transaction-13.png %}

当事务2使用update语句修改该行数据时，会首先使用排他锁锁定该行，将改行当钱的值复制到undo log中，然后再真正的修改该行的值，最后填写事务id，并使回滚指针指向undo log中修改前的行。 

如下图所示（第一次修改）：

{% asset_img mysql-transaction-14.png %}

{% asset_img mysql-transaction-18.png %}

##### 事务链表

MySQL中的事务在开始到提交这段过程中，都会被保存到一个叫trx_sys的事务链表中，这是一个基本的脸表结构：

ct-trx -> trx11 -> trx9 -> trx6 -> trx5 -> trx3

事务链表中保存的都是还未提交的事务，事务一旦被提交，则会从事务链表中删除。

repeatable read隔离级别下，在每个**事务**开始的时候，会将当前系统中的所有的活跃事务拷贝到一个列表中(read view)。

read committed隔离级别下，在每个**语句**开始的时候，会讲当前系统中所有的活跃事务拷贝到一个列表中(read view)。

查看事务列表：

```mysql
show engine innodb status
```



##### ReadView

read view解决了当前事务能读哪个历史版本的问题。

read view是事务开启时当前所有事务的一个集合，这个类中存储了当前read view中最大事务id及最小事务id。这就是当前活跃的事务列表，如下图所示：

ct-trx -> trx11 -> trx9 -> trx6 -> trx5 -> trx3

ct-trx表示当前事务的id，对应上面的read view数据结构如下：

```
read_view -> creator_trx_id = ct-trx
read_view -> up_limit_id = trx3 -- 低水位，最小的都要比这个高
read_view -> low_limit_id = trx11 -- 高水位
read_view -> trx_ids = [trx11, trx9, trx6, trx5, trx3]
```

low_limit_id是“高水位”，即当时活跃事务列表的最大id，如果读到row的db_trx_id >= low_limit_id, 说明这些id在此之前的数据都没有提交，如注视中的描述，这些数据都不可见。

```c
if (trx_id >= view -> low_limit_id) {
		return(FALSE);
}
# 注：read view部分源码
```

up_limie_id是“低水位”，即当时活跃事务列表的最小id，如果row的db_trx_id < up_limit_id，说明这些数据在事务创建的id时都已经提交，如注释中的描述，这些数据均可见。

```C
if (trx_id < view -> up_limit_id) {
		return(TRUE);
}
```

row的db_trx_id在low_limit_id和up_limit_id之间，则查找该记录的db_trx_id是否在自己事务的read_view -> trx_ids列表中，如果在则该记录的当前版本不可见，否则该记录的当前版本可见。

不同隔离级别read view实现方式：

1. read committed

```C
 # 函数 ha_innobase::external_lock
if (trx -> isolation_level <= TRX_ISO_READ_COMMITTED
			&& trx -> global_read_view) {
			/**
			 * At low transaction isolation level we let 
			 * each consistent read set its own snapshot
			 **/
  read_view_close_for_mysql(trx);
}
```

即：在每次语句执行的过程中，都关闭read_view，重新在row_search_for_mysql函数中创建当前的一份read_view。这样就会产生不可重复现象。

2. repeatable read

在repeatable read的隔离级别下，创建事务trx结构的时候，就生成了当前的global read view。使用trx_assign_read_view函数创建，一直维持到事务结束。在事务结束这段时间内，每一次查询都不会重新创建read_view，从而实现了可重复读。
