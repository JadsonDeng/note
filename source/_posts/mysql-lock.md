---
title: MySQL锁
date: 2019-08-22 17:47:07
tags: mysql
---
#  MySQL锁介绍

{% asset_img mysql-lock-1.png %}

# MySQL表级锁

## 表级锁介绍

### 由MySQL layer层实现

+ MySQL 表级锁有2种：表锁和原数据锁

+ MySQL实现的表级锁的状态变量：

  ```mysql
  mysql> show status like 'table%';
  +----------------------------+-------+
  | Variable_name              | Value |
  +----------------------------+-------+
  | Table_locks_immediate      | 70    |
  | Table_locks_waited         | 0     |
  | Table_open_cache_hits      | 0     |
  | Table_open_cache_misses    | 3     |
  | Table_open_cache_overflows | 0     |
  +----------------------------+-------+
  5 rows in set (0.00 sec)
  ```

  ```
  Table_locks_immediate: 产生表级锁定的次数
  Table_locks_waited：出现表级锁争用而发生的等待次数
  ```

## 表锁介绍

+ 表锁有2种形式：

  ```
  表共享读锁(Table Read Lock)
  表排他写锁(Table Write Lock)
  ```

+ 手动增加表锁

  ```
  lock table 表名称 read/write, 表名称2 read/write, 其它;
  ```

+ 查看表锁状态

  ```mysql
  show open tables
  ```

+ 删除表锁

  ```mysql
  unlock tables
  ```

## 表锁演示

### 环境准备

```mysql
-- 新建表
mysql> create table mylock (
    -> id int not null auto_increment,
    -> name varchar(20) default null,
    -> primary key(id)
    -> );
Query OK, 0 rows affected (0.02 sec)

insert into mylock(name) values('a'),('b'),('c'),('d');
```

### 读锁演示

1. 表读锁 

   {% asset_img mysql-lock-2.png %}

   Session1、Session2

   ```
   Session1: lock table mylock read; -- 给mylock表加读锁
   Session1: select * from mylock; -- 可以查询
   Session1: select * from dep; -- 不能访问非锁定表
   Session2: select * from mylock; -- 可以查询，没有锁
   Session2: update mylock set name='2'; -- 修改阻塞，自动加行写锁
   Session1: unlock tables; -- 释放表锁
   Session2: Rows matched: 1  Changed: 1  Warnings: 0 --修改执行完成
   Session1: select * from dep; -- 可以访问
   
   Session1: lock table mylock read; -- Session1可以给mylock表加读锁
   Session2: lock table mylock read; -- Session2也给mylock表加读锁
   ```

2. 表写锁

   Session1、Session2

   ```mysql
   Session1: lock table mylock write; -- 给mylock表加写锁
   Session1: select * from mylock; -- 可以查询
   Session1: select * from dep; -- 不能访问非锁定表
   Session1: update mylock set name='aa' where id=1; -- 可以更新
   Session2: select * from mylock; -- 阻塞查询
   Session1: unlock tables; -- 释放锁
   Session2: 4 rows in set (7.66 sec) -- Session1释放锁后可以查询出结果
   Session1: select * from dep; -- 可以访问
   
   Session1: lock table mylock write; -- session1获得了写锁
   Session2: lock table mylock read; -- 阻塞，不能获得读锁
   Session2: lock table mylock write; -- 阻塞，不能获得写锁
   ```

   **读锁和写锁在实际应用中应禁止使用**

# 元数据锁

## 元数据锁概念

MDL(metadata lock)元数据：表结构

在MySQL5.5版本中引入了MDL，当对一个表做增删改查的时候，加DML读锁；当修改表结构的时候，加MDL写锁。

在做增删改查的时候不能修改表结构，修改表结构的时候不能做增删改查操作。

## 元数据锁演示

Session1、Session2

```mysql
Session1: begin; -- 开启事务
					select * from mylock; -- 自动加MDL读锁
Session2: alter table mylock add f int; -- 修改阻塞；
Session1: commit; -- commit提交事务(或者rollback释放锁)
Session2: Query OK, 0 rows affected (7.53 sec)
Records: 0  Duplicates: 0  Warnings: 0 -- 1释放锁，2执行成功
```



# 行级锁

## 行级锁介绍

InnoDB存储引擎实现

InnoDB的行级锁，按照锁定范围来说，分为3种：

1. 记录锁(Record Locks)：锁定索引中一条记录。主键指定： where id=1;
2. 间隙锁(Gap Locks)：锁定记录前、记录中、记录后的行。
3. Next-key锁：只出现在Repeatable Read隔离级别中，相当于记录锁+间隙锁

按照功能来说，分为2种：

1. 共享读锁(S)：允许一个事务去读一行，阻止其他事务获得相同数据集的排他写锁。

   添加方式：

   ```mysql
   select * from tablename where ... lock in share mode -- 共享读锁 手动添加
   
   select * from tablename where ...  -- 无锁
   ```

   

2. 排他写锁(X)：允许获得排他写锁的事务更新数据阻止其他事务取得相同数据集的共享读锁(不是读)和排他写锁。

   + 自动加DML

     对于update、delete、insert，InnoDB会自动给涉及的数据集加排他写锁。

   + 手动加：

     ```mysql
     select * from tablename where .... for update
     ```

InnoDB也实现了表级锁，也就是意向锁，意向锁是mysql内部使用的，不需要用不手动干预。

+ 意向共享读锁(IS)：事务打算给数据行加共享读锁，事务在给一个数据行加共享锁前必须先取得该表的IS锁
+ 意向排他写锁(IX)：事务打算给数据行加排他写锁，事务在给一个数据行加排他锁前必须先取得该表的IX锁。

|            | 共享锁 | 排他锁 | 意向共享锁 | 意向排他锁 |
| ---------- | ------ | ------ | ---------- | :--------: |
| 共享锁     | 兼容   | 冲突   | 兼容       |    冲突    |
| 排他锁     | 冲突   | 冲突   | 冲突       |    冲突    |
| 意向共享锁 | 兼容   | 冲突   | 兼容       |    兼容    |
| 意向排他锁 | 冲突   | 冲突   | 兼容       |    兼容    |



## 两阶段锁(2PL)

{% asset_img mysql-lock-3.png %}

锁操作分为2个阶段：加锁阶段和解锁阶段。

在加锁阶段不能接锁，在接锁阶段不能加锁。

加锁阶段与解锁阶段互不相交。

### 行锁演示

**InnoDB上的行级锁是通过给索引上的项加锁来实现的，也就是说，只有通过索引来检索数据，InnoDB才使用行级锁，否则升级为表锁。**

> where 索引 —使用行锁
>
> where 非索引 —使用表锁

#### 行读锁

```mysql
-- 查看行锁状态: show status like 'innodb_row_lock%'
Innodb_row_lock_current_waits：当前正在等待锁定的数量
Innodb_row_lock_time：从系统启动到现在锁定总时间长度
Innodb_row_lock_time_avg：每次等待所花平均时间
Innodb_row_lock_time_max：从系统启动到现在等待最长的一次所花的时间
Innodb_row_lock_waits：系统启动后到现在总共等待的次数
```

```mysql
Session1: begin; 
					select * from mylock where id=1 lock in share mode;-- 手动加id=1的行读锁，使用索引
Session2: update mylock set name='y' where id=2;  -- 未锁定该行，可以修改
Session2: update mylock set name='y' where id=1; -- 锁定该行，修改阻塞
	ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction -- 锁定超时
Session1: commit; -- 提交事务，或者rollback释放锁
Session2: update mulock set name='y' where id=1; -- 修改成功

注： 使用索引加行锁，未锁定的行可以访问
```

#### 行读锁升级为表锁

Session1、Session2

```mysql
Sesson1: begin; -- 开启事务
				-- 手动加name='c'的行读锁，未使用索引
				select * from mylock where name='c' lock in share mode;
Session2: update mylock set name='y' where id=2; -- 修改阻塞，未使用索引，航速升级为表锁
Session1: commit; -- 提交事务，或者rollback释放锁
Session2: update mylock set name='y' where id=2; -- 修改成功

注： 未使用索引，行锁将会升级为表锁
```

#### 行写锁

Session1、Session2

```mysql
Session1: begin; -- 开启事务
					-- 手动加id=1的行写锁
					select * from mylock where id=1 for update;
Session2: select * from mylock where id=2; -- 可以访问
Session2: select * from mylock where id=1; -- 可以读，不加锁
Session2: select * from mylock where id=1 lock in share mode; -- 加读锁被阻塞
Session1: commi; -- 提交或rollback释放写锁
Session2: 执行成功

主键索引产生记录锁
```

#### 间隙锁

{% asset_img mysql-lock-4.png %}

数据准备:

```mysql
-- 创建news表
create table news (
id int primary key,
number int
);

-- 创建非唯一索引
alter table news add index idx_number(number);

-- 插入测试数据
insert into news values(1,2),(3,4),(6,5),(8,5),(10,5),(13,5);
```

案例演示:

```mysql
Session1: start transaction; -- 开启事务
					select * from news where number=4 for update; -- 手动加读锁，锁的是1<= id <= 6 并且 2 <= number <= 5范围内的所有记录。
					
Session2: start transaction;
Session2: insert into news values(2, 4); -- 阻塞
Session2: insert into news values(2, 2); -- 阻塞
Session2: insert into news values(4, 4); -- 阻塞
Session2: insert into news values(4, 5); -- 阻塞
Session2: insert into news values(7, 5); -- 执行成功
Session2: insert into news values(9, 5); -- 执行成功
Session2: insert into news values(11, 5); -- 执行成功
注： id和number都在间隙内则阻塞
-------------------------------------------------------

Session1: start transaction;
					-- 指定number=13这条记录(13,5)，该记录后面没有数据了
					select * from news where number=13 for update;
					
Session2: start transaction;
Session2: insert into news values(13,1); -- 执行成功
Session2: insert into news values(13, 10); -- 阻塞
Session2: insert into news values(14, 10); -- 阻塞
注： number=13后面没有记录了，则会对所有id>=13&&number>=5的数据加锁
-------------------------------------------------------

Session1: start transaction;
					-- 通过主键加锁(3,4)
					select * from news where id=3 for update;
					
Session2: start transaction;
Session2: insert into news values(2, 4); -- 执行成功
Session2: insert into news values(2, 2); -- 执行成功
Session2: insert into news values(4, 4); -- 执行成功
Session2: insert into news values(4, 5); -- 执行成功
Session2: insert into news values(7, 5); -- 执行成功
Session2: insert into news values(9, 5); -- 执行成功
Session2: insert into news values(11, 5); -- 执行成功
注：通过主键=来更新数据，产生记录锁
-------------------------------------------------------

Session1: start transaction;
					-- 主键指定范围，产生间隙锁：范围是：(1,6]
					select * from news where id>1 and id<4 for update;
					
Session2: start transaction;
Session2: insert into news values(2,4); -- 阻塞
Session2: update news set number=10 where id=6; -- 阻塞
Session2: insert into news values(7,4); -- 成功
注：当通过主键范围来更新数据时，加间隙锁
```



#### 死锁

2个session互相等待对方的资源释放后，才能释放自己的资源，造成的死锁。

Session1，Session2

```mysql
Session1: begin;
Session1: update mylock set name='qqq' where id=1; -- 手动加行写锁，id=1,使用索引
Session2: begin;
Session2: update mylock set name='111' where id=2; -- 手动加行写锁，id=2,使用索引
Session1: update mylock set name='qqq' where id=2; -- 加写锁被阻塞
Session2: update mylock set name='222' where id=1; -- 加写锁会死锁，不允许操作
ERROR 1213 (40001): Deadlock found when trying to get lock; try restarting transaction
```





