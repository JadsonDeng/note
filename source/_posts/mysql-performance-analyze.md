---
title: mysql5：性能分析与优化
tags:
---
# 性能分析和优化

## 性能分析的思路

1. 首先需要使用【**慢查询日志**】功能，去获取所有查询时间比较长的sql语句。
2. 其次查看【**执行计划**】，查看有问题的sql的执行计划（explain）。
3. 最后可以使用【**show profile**】查看有问题的sql的性能使用情况。

### 慢查询日志

#### 慢查询日志介绍

查询慢查询日志的状态：

```mysql
show variables like '%slow_query%';
+---------------------+-------------------------------+
| Variable_name       | Value                         |
+---------------------+-------------------------------+
| slow_query_log      | ON                            |
| slow_query_log_file | /var/lib/mysql/slow_query.log |
+---------------------+-------------------------------+
2 rows in set (0.00 sec)
```

#### 开启慢查询功能

开启

```shell
slow_query_log=ON
long_query_log=3
slow_query_log_file=/usr/local/var/mysql/slow-query.log
```

#### 分析慢查询日志的工具

##### 使用mysqldumpslow工具

mysqldumpslow是mysql自带的慢查询日志工具，可以使用mysqldumpslow搜索慢查询日志中的sql语句。

得到按照时间排序的前10条里面含有左连接的查询语句：

```bash
mysqldumpslow -s t -t 10 -g "left join" /usr/local/var/mysql/slow-query.log
```

参数说明：

-s：是表示按照何种方式排序

---

c：访问计数

l：锁定时间

r：返回记录

t：查询时间

al：平均锁定时间

ar：平均返回记录数

at：平局查询时间



-t：是top n的意思，即为返回前面多少条数据

-g：后面可以写一个正则匹配模式

##### 使用专业的percona-toolkit工具

这里不做详细介绍。

#### profile分析语句

Query Profile是mysql自带的一种query诊断分析工具，通过它可以分析出一条sql语句的**硬件性能瓶颈**在什么地方。

默认情况下，mysql该功能没有开启，必须手动开启。

```mysql
mysql> select @@profiling;
+-------------+
| @@profiling |
+-------------+
|           0 |
+-------------+
1 row in set, 1 warning (0.00 sec)
-- 为0表示没有开启
```

开启profile：

```mysql
mysql> set profiling=1;
Query OK, 0 rows affected, 1 warning (0.00 sec)
```

查询profiles:

```mysql
mysql> show profiles;
+----------+------------+---------------------------+
| Query_ID | Duration   | Query                     |
+----------+------------+---------------------------+
|        1 | 0.00013075 | select count(1) from user |
|        2 | 0.00017675 | SELECT DATABASE()         |
|        3 | 0.00016625 | SELECT DATABASE()         |
|        4 | 0.00108425 | show databases            |
|        5 | 0.00122625 | show tables               |
|        6 | 0.00030950 | select count(1) from user |
|        7 | 0.00029600 | select * from user        |
+----------+------------+---------------------------+
7 rows in set, 1 warning (0.00 sec)
```

查看query_id=6的查询的**大致**信息:

```mysql
mysql> show profile for query 6;
+----------------------+----------+
| Status               | Duration |
+----------------------+----------+
| starting             | 0.000079 |
| checking permissions | 0.000012 |
| Opening tables       | 0.000024 |
| init                 | 0.000021 |
| System lock          | 0.000011 |
| optimizing           | 0.000007 |
| statistics           | 0.000017 |
| preparing            | 0.000015 |
| executing            | 0.000004 |
| Sending data         | 0.000057 |
| end                  | 0.000007 |
| query end            | 0.000011 |
| closing tables       | 0.000009 |
| freeing items        | 0.000018 |
| cleaning up          | 0.000017 |
+----------------------+----------+
15 rows in set, 1 warning (0.00 sec)
```

查看query_id=6的查询的**详细**信息:

```mysql
mysql> show profile cpu, block io, swaps for query 6;
+----------------------+----------+----------+------------+--------------+---------------+-------+
| Status               | Duration | CPU_user | CPU_system | Block_ops_in | Block_ops_out | Swaps |
+----------------------+----------+----------+------------+--------------+---------------+-------+
| starting             | 0.000079 | 0.000000 |   0.000000 |            0 |             0 |     0 |
| checking permissions | 0.000012 | 0.000000 |   0.000000 |            0 |             0 |     0 |
| Opening tables       | 0.000024 | 0.000000 |   0.000000 |            0 |             0 |     0 |
| init                 | 0.000021 | 0.000000 |   0.000000 |            0 |             0 |     0 |
| System lock          | 0.000011 | 0.000000 |   0.000000 |            0 |             0 |     0 |
| optimizing           | 0.000007 | 0.000000 |   0.000000 |            0 |             0 |     0 |
| statistics           | 0.000017 | 0.000000 |   0.000000 |            0 |             0 |     0 |
| preparing            | 0.000015 | 0.000000 |   0.000000 |            0 |             0 |     0 |
| executing            | 0.000004 | 0.000000 |   0.000000 |            0 |             0 |     0 |
| Sending data         | 0.000057 | 0.000000 |   0.000000 |            0 |             0 |     0 |
| end                  | 0.000007 | 0.000000 |   0.000000 |            0 |             0 |     0 |
| query end            | 0.000011 | 0.000000 |   0.000000 |            0 |             0 |     0 |
| closing tables       | 0.000009 | 0.000000 |   0.000000 |            0 |             0 |     0 |
| freeing items        | 0.000018 | 0.000000 |   0.000000 |            0 |             0 |     0 |
| cleaning up          | 0.000017 | 0.000000 |   0.000000 |            0 |             0 |     0 |
+----------------------+----------+----------+------------+--------------+---------------+-------+
15 rows in set, 1 warning (0.00 sec)
```

## 服务器层面的优化

### 将数据保存在内存中，保证从内存读取数据

buffer pool默认是128M

**扩大buffer pool空间，理论上可以扩大到内存的3/4或4/5**

怎样确定innodb_buffer_pool_size足够大，数据从内存中读取数据而不是硬盘？

```mysql
mysql> show global status like 'innodb_buffer_pool_pages_%';
+----------------------------------+-------+
| Variable_name                    | Value |
+----------------------------------+-------+
| Innodb_buffer_pool_pages_data    | 323   |
| Innodb_buffer_pool_pages_dirty   | 0     |
| Innodb_buffer_pool_pages_flushed | 70    |
| Innodb_buffer_pool_pages_free    | 7868  |
| Innodb_buffer_pool_pages_misc    | 0     |
| Innodb_buffer_pool_pages_total   | 8191  |
+----------------------------------+-------+
6 rows in set (0.01 sec)

Innodb_buffer_pool_pages_free 表示剩余可用页的数量，为0表示已经用完了 
```

Innodb_buffer_pool_pages_free=750M 

### 内存预热

提前将select语句执行一遍，数据已被夹在到内存中，再次执行时会直接从内存中查询数据。

### 降低磁盘写入次数

1. 增加redolog大小

   redolog的空间占用75%后异步落盘，90%后同步落盘。

   所以分配给redolog的空间越大，落盘次数越少。

   如下可以修改redolog大小，通常设置为Innodb_buffer_pool_pages_free * 0.25 

   ```bash
   innodb_log_file_size=180M
   ```

2. 能不开的日志不开，如：通用日志、慢查询日志。

   **binlog一定要开**。 

3. 写redolog的策略：innodb_flush_log_at_trx_commut: 0,1,2

### 提高磁盘读写速度

SSD

## sql设计层面优化

+ 设计中间表，一般针对于统计分析功能，或者实时性要求不高的功能。
+ 为了减少关联查询，创建合理的冗余字段。
+ 对于字段太多的大表，考虑拆表。
+ 对于表中经常不被使用的字段或者存储数据比较多的字段，考虑拆表。
+ 每张表建议都要有一个主键（主键索引），而且主键类型最好是int类型，建议自增主键。

## sql语句优化

### 索引优化

where字段、组合索引（最左前缀）、索引下推（非选择行不加锁）、索引覆盖（不回表）、on两边字段加索引、排序、分组统计

不要用* 

### limit优化

若offset过大，则先排序，再根据条件过滤。如：

原s q l:

```sql
select * from user limit 10000000,1
```

优化后：

```sql
select * from user where id>10000000 limit 1
```

**limit可以提前停止全表扫描**

### 其它优化

+ 查询条件不要使用内置函数
+ 不要使用not in

