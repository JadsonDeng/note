---
title: mysql4：行锁原理分析
date: 2019-09-18 19:05:15
tags: mysql
---
# 行锁原理分析

## 谈锁前提

```mysql
select * from t1 where id=10;
delete from t1 where id=10;
```

+ id列是不是主键？
+ 当前系统的隔离级别是什么
+ id列如果不是主键，那么id列上有索引吗

## 简单sql的加锁分析

### 组合一：id主键 + RC

这个组合是最简单、最容易分析的组合。id是主键，read committed隔离级别，给定sql：delete from t1 where id=10;只需要将主键上id=10的记录加上X锁即可。

如下图所示：

{% asset_img mysql-row-lock-1.png %}

### 组合二：id唯一索引 + RC

这个组合，id不是主键，而是一个unique的二级索引键值。那么在rd隔离级别下，需要加什么锁呢？见下图：

```mysql
delete from t1 where id=10;
```

{% asset_img mysql-row-lock-2.png %}

### 组合三：id非唯一索引 + RC

{% asset_img mysql-row-lock-3.png %}

相对于组合一二，组合三又发生了变化，隔离级别仍旧是RC不变，但是id列上的约束又降低了，id列不再唯一，只有一个普通的索引。如上图，会对name=b和d的2列加行锁。

### 组合四：id无索引 + RC

id列上没有索引，where id=10这个过滤条件，没法通过索引进行过滤，那么只能走圈表扫描进行过滤

{% asset_img mysql-row-lock-4.png %}

若id列无索引，则全表扫描过滤，则锁表。

### 组合五：id主键 + RR

同组合一。

### 组合六：id唯一索引 + RR

同组合二。

### 组合七：id非唯一索引 + RR

{% asset_img mysql-row-lock-5.png %}

{% asset_img mysql-row-lock-6.png %}

不能在间隙insert，防止幻读（RR），

### 组合八：id无索引 + RR

全表扫描，锁表。

### 组合九：Serializable（LBCC）

只要有sql（包括select），就锁。

如果有索引，则是行锁。

如果无索引，则是表锁。

## 一条复杂sql的加锁分析

{% asset_img mysql-row-lock-7.png %}

如图所示sql，在**repeatable read**隔离级别下，会加什么锁呢？

在详细分析这条sql的加锁情况前，需要有一个知识储备，那就是一个sql中的where条件如何拆分？这里直接给出结果：

+ Index key：pub time > 1 and pub time < 20. 此条件用于确定sql在idx_t1_pu索引上的查询范围。
+ Index filter：userid = 'hdc'. 此条件可以在idx_t1_pu上过滤，但不属于Index key。
+ Table filter：comment is not null. 此条件在idx_t1_pu上无法过滤，只能在聚簇索引上过滤。

在where条件过滤时，先过滤index key（索引列卫范围查询，起始条件为index first key，截止条件为index last key），再过滤Index filter（索引列），最后过滤table filter（非索引列）。在ICP过程中，下推index key和index filter。

{% asset_img mysql-row-lock-8.png %}

结论：

+ index key确定范围，需要加上GAP锁。
+ index filter过滤条件，若mysql支持ICP(mysql 5.6 >= 5.6)，则不满足条件的记录，不加X锁（索引下推），否则需要加X锁。
+ table filter过滤条件，无论是否满足，都需要加X锁
