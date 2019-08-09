---
title: mysql索引与优化
date: 2019-08-09 12:55:47
tags: mysql
---
# 索引的使用场景

## 哪些情况需要创建索引

1. 主键自动创建唯一索引
2. 频繁作为查询条件的字段应该添加索引
3. 多表关联查询中，关联字段应该创建索引，on关键字两遍都要创建索引
4. 查询中排序的字段，应该创建索引，B+ tree有顺序
5. 覆盖索引的好处是：不需要回表查询，为了达到覆盖索引的效果，应该使用组合索引
6. 统计或者分组字段，应该创建索引

## 哪些情况不需要创建索引

1. 表记录太少，索引需要占用存储空间
2. 频繁更新，索引需要维护
3. 查询字段使用频率不高

## 为什么使用组合索引

**由多个字段组成的索引叫做组合索引**

> alter table table_name add index index_name(col1,col2,col3)

组合索引的数据结构：

{% asset_img mysql-index-optimize-1.png %}

在一刻索引树上有多个字段。

优势：节约存储空间、容易形成索引覆盖

使用：遵循最左前缀原则： 

+ 前缀索引： 

  1. like 'name%'使用索引，
  2. like '%name'不使用索引

+ 最左前缀：

  **从左向右匹配知道遇到范围查询**

  ```mysql
  alter table table_name add index index_name(col1,col2,col3)
  ```

  1. ... where col1=1 and col2=2 and col3=3: 使用索引
  2. ... where col=1 and col2>2 and col3=3: 只会使用到col1索引
  3. … where col1=1 and col3>3 and col2=2: 使用到col1和col2

  **能否使用到索引与where查询条件的顺序无关，与创建索引时列的顺序有关**



# 索引失效

## 查看执行计划

### 参数说明

explain出来的信息有10列，分别是：

```mysql
| id | select_type | table | type | possible_keys | key  | key_len | ref  | rows | Extra |
```

### 案例表

```mysql
mysql> create table user(
    -> id int primary key,
    -> loginname varchar(100),
    -> name varchar(100),
    -> age int,
    -> sex char(1),
    -> dep int,
    -> address varchar(100)
    -> );
Query OK, 0 rows affected (0.01 sec)

mysql> create table dep(
    -> id int primary key,
    -> name varchar(100)
    -> );
Query OK, 0 rows affected (0.01 sec)

mysql> create table addr(
    -> id int primary key,
    -> addr varchar(100)
    -> );
Query OK, 0 rows affected (0.01 sec)

-- 创建普通索引
mysql> alter table user add index index_dep(dep);
-- 创建唯一索引
mysql> alter table user add unique index index_loginname(loginname);
-- 创建组合索引
mysql> alter table user add index index_name_age_sex(name, age,sex);
-- 创建全文索引
mysql> alter table addr add fulltext ft_addr(addr);
```

#### id

+ 每个select语句都会自动分配一个唯一标识
+ 表示查询中操作表的顺序，有三种情况
  1. id相同，执行顺序由上到下
  2. id不同：如果是自查询，id号会增大，**id越大，优先级越高**
+ id列为null就表是这是一个结果集，不需要使用它来进行查询

#### select_type

查询类型，主要用于区别普通查询、联合查询(union\union all)、子查询等复杂查询。

##### simple

表示不需要union操作或不需要子查询的简单查询语句。有连接查询时，外层的为simple，且只有一个。

```mysql
mysql> explain select * from user;
+----+-------------+-------+------+---------------+------+---------+------+------+-------+
| id | select_type | table | type | possible_keys | key  | key_len | ref  | rows | Extra |
+----+-------------+-------+------+---------------+------+---------+------+------+-------+
|  1 | SIMPLE      | user  | ALL  | NULL          | NULL | NULL    | NULL |    1 | NULL  |
+----+-------------+-------+------+---------------+------+---------+------+------+-------+
1 row in set (0.00 sec)
```

##### primary

一个需要union操作，或者子查询的select，位于最外层的查询的select_type即为primary。且只有一个。

```mysql
mysql>  explain select (select name from user) from user;
+----+-------------+-------+-------+---------------+--------------------+---------+------+------+-------------+
| id | select_type | table | type  | possible_keys | key                | key_len | ref  | rows | Extra       |
+----+-------------+-------+-------+---------------+--------------------+---------+------+------+-------------+
|  1 | PRIMARY     | user  | index | NULL          | index_dep          | 5       | NULL |    1 | Using index |
|  2 | SUBQUERY    | user  | index | NULL          | index_name_age_sex | 312     | NULL |    1 | Using index |
+----+-------------+-------+-------+---------------+--------------------+---------+------+------+-------------+
2 rows in set (0.01 sec)
```

#####  subquery

表明该查询是一个子查询。

除了from子句中包含的子查询，其它地方出现的子查询都有可能是subquery

```mysql
mysql> explain select * from user where id=(select max(id) from user);
+----+-------------+-------+------+---------------+------+---------+------+------+-----------------------------------------------------+
| id | select_type | table | type | possible_keys | key  | key_len | ref  | rows | Extra                                               |
+----+-------------+-------+------+---------------+------+---------+------+------+-----------------------------------------------------+
|  1 | PRIMARY     | NULL  | NULL | NULL          | NULL | NULL    | NULL | NULL | Impossible WHERE noticed after reading const tables |
|  2 | SUBQUERY    | NULL  | NULL | NULL          | NULL | NULL    | NULL | NULL | No matching min/max row                             |
+----+-------------+-------+------+---------------+------+---------+------+------+-----------------------------------------------------+
2 rows in set (0.00 sec)
```

##### dependent subquery

与depengdent union类似，表示这个subquery的查询要收到外部表查询的影响

```mysql
mysql> explain select id,name,(select name from dep a where a.id=b.id) from user b;
+----+--------------------+-------+--------+---------------+--------------------+---------+-----------+------+-------------+
| id | select_type        | table | type   | possible_keys | key                | key_len | ref       | rows | Extra       |
+----+--------------------+-------+--------+---------------+--------------------+---------+-----------+------+-------------+
|  1 | PRIMARY            | b     | index  | NULL          | index_name_age_sex | 312     | NULL      |    1 | Using index |
|  2 | DEPENDENT SUBQUERY | a     | eq_ref | PRIMARY       | PRIMARY            | 4       | demo.b.id |    1 | NULL        |
+----+--------------------+-------+--------+---------------+--------------------+---------+-----------+------+-------------+
2 rows in set (0.00 sec)
```

##### union

union连接的2个select查询，第一个查询是primary，除了第一个表外，第二个表以后的select_type都是union

```mysql
mysql> explain select * from user where sex='1' union select * from user where sex='2';
+----+--------------+------------+------+---------------+------+---------+------+------+-----------------+
| id | select_type  | table      | type | possible_keys | key  | key_len | ref  | rows | Extra           |
+----+--------------+------------+------+---------------+------+---------+------+------+-----------------+
|  1 | PRIMARY      | user       | ALL  | NULL          | NULL | NULL    | NULL |    1 | Using where     |
|  2 | UNION        | user       | ALL  | NULL          | NULL | NULL    | NULL |    1 | Using where     |
| NULL | UNION RESULT | <union1,2> | ALL  | NULL          | NULL | NULL    | NULL | NULL | Using temporary |
+----+--------------+------------+------+---------------+------+---------+------+------+-----------------+
3 rows in set (0.00 sec)
```

##### dependent union

与union一样，出现在union或union all语句中，但是这个查询要受到外部查询的影响。

```mysql
mysql>  explain select * from user where sex in (select sex from user where sex='1' union select sex from user where sex='2');
+----+--------------------+------------+-------+---------------+--------------------+---------+------+------+--------------------------+
| id | select_type        | table      | type  | possible_keys | key                | key_len | ref  | rows | Extra                    |
+----+--------------------+------------+-------+---------------+--------------------+---------+------+------+--------------------------+
|  1 | PRIMARY            | user       | ALL   | NULL          | NULL               | NULL    | NULL |    1 | Using where              |
|  2 | DEPENDENT SUBQUERY | user       | index | NULL          | index_name_age_sex | 312     | NULL |    1 | Using where; Using index |
|  3 | DEPENDENT UNION    | user       | index | NULL          | index_name_age_sex | 312     | NULL |    1 | Using where; Using index |
| NULL | UNION RESULT       | <union2,3> | ALL   | NULL          | NULL               | NULL    | NULL | NULL | Using temporary          |
+----+--------------------+------------+-------+---------------+--------------------+---------+------+------+--------------------------+
4 rows in set (0.00 sec)
```

##### union result

包含union的结果集，在union和union all语句中，因为它不需要进行查询，所以id为null

##### derived

from子句中出现的子查询，也叫做派生表，其它数据库中可能叫做内联视图或者嵌套select

```mysql
mysql> explain select * from (select * from user where sex='1') b;
+----+-------------+------------+------+---------------+------+---------+------+------+-------------+
| id | select_type | table      | type | possible_keys | key  | key_len | ref  | rows | Extra       |
+----+-------------+------------+------+---------------+------+---------+------+------+-------------+
|  1 | PRIMARY     | <derived2> | ALL  | NULL          | NULL | NULL    | NULL |    2 | NULL        |
|  2 | DERIVED     | user       | ALL  | NULL          | NULL | NULL    | NULL |    1 | Using where |
+----+-------------+------------+------+---------------+------+---------+------+------+-------------+
2 rows in set (0.00 sec)
```

#### table

+ 显示查询的表名，如果使用了别名，则显示表的别名
+ 如果不设计对数据表的操作，那么这里显示null
+ 如果显示为尖括号括起来的<derivedN>，就表示这个是临时表，后面的N就是执行计划中的id，表示结果来自于这个查询产生。
+ 如果是尖括号括起来的<unionM,N>，与<derivedN>类似，表示这个结果来自union查询的id为M、N的结果。

####  type (重要)

- 依次从好到差：

  ```mysql
  system, const, eq_ref, ref, fulltext, ref_or_null, unique_subquery, index_subquery, range, index_merge, index, all
  ```

  **除了all之外，其它的type都可以使用到索引；除了index_merge之外，其它的type值可以用到一个索引。**

  优化器会选择一个最有的索引。

##### system

表中只有一条数据或者是空表

```mysql
mysql> explain select * from (select * from user where id=1) a;
+----+-------------+------------+--------+---------------+------+---------+------+------+--------------------------------+
| id | select_type | table      | type   | possible_keys | key  | key_len | ref  | rows | Extra                          |
+----+-------------+------------+--------+---------------+------+---------+------+------+--------------------------------+
|  1 | PRIMARY     | <derived2> | system | NULL          | NULL | NULL    | NULL |    0 | const row not found            |
|  2 | DERIVED     | NULL       | NULL   | NULL          | NULL | NULL    | NULL | NULL | no matching row in const table |
+----+-------------+------------+--------+---------------+------+---------+------+------+--------------------------------+
2 rows in set (0.00 sec)
```

##### const

使用唯一索引或者主键索引。

```mysql
mysql> explain select * from (select * from user where id=1) a;
+----+-------------+------------+--------+---------------+------+---------+------+------+--------------------------------+
| id | select_type | table      | type   | possible_keys | key  | key_len | ref  | rows | Extra                          |
+----+-------------+------------+--------+---------------+------+---------+------+------+--------------------------------+
|  1 | PRIMARY     | <derived2> | system | NULL          | NULL | NULL    | NULL |    0 | const row not found            |
|  2 | DERIVED     | NULL       | NULL   | NULL          | NULL | NULL    | NULL | NULL | no matching row in const table |
+----+-------------+------------+--------+---------------+------+---------+------+------+--------------------------------+
2 rows in set (0.00 sec)
```

##### eq_ref

关键字：连接字段是主键或者唯一索引

此类型通常出现在多表的join查询，表示对于前表的每一个结果，都只能匹配到后表的唯一一个结果，并且查询的比较操作通常是“=”，查询效率高。

```mysql
mysql> explain select a.id from user a left join dep b on a.dep=b.id;
+----+-------------+-------+--------+---------------+-----------+---------+------------+------+-------------+
| id | select_type | table | type   | possible_keys | key       | key_len | ref        | rows | Extra       |
+----+-------------+-------+--------+---------------+-----------+---------+------------+------+-------------+
|  1 | SIMPLE      | a     | index  | NULL          | index_dep | 5       | NULL       |    1 | Using index |
|  1 | SIMPLE      | b     | eq_ref | PRIMARY       | PRIMARY   | 4       | demo.a.dep |    1 | Using index |
+----+-------------+-------+--------+---------------+-----------+---------+------------+------+-------------+
2 rows in set (0.00 sec)
```

##### ref

针对非唯一性索引，或使用等值(=)非主键连接查询，或者是使用了最左前缀规则索引的查询。

+ 非唯一索引查询

```mysql
mysql> explain select * from user where dep=1;
+----+-------------+-------+------+---------------+-----------+---------+-------+------+-------+
| id | select_type | table | type | possible_keys | key       | key_len | ref   | rows | Extra |
+----+-------------+-------+------+---------------+-----------+---------+-------+------+-------+
|  1 | SIMPLE      | user  | ref  | index_dep     | index_dep | 5       | const |    1 | NULL  |
+----+-------------+-------+------+---------------+-----------+---------+-------+------+-------+
1 row in set (0.00 sec)
```

- 等值非主键连接查询

```mysql
-- user和dep的name字段都需要创建索引。
mysql> alter table dep add index index_name on (name);
mysql> explain select a.id from user a left join dep b on a.name = b.name;
+----+-------------+-------+-------+---------------+--------------------+---------+-------------+------+-------------+
| id | select_type | table | type  | possible_keys | key                | key_len | ref         | rows | Extra       |
+----+-------------+-------+-------+---------------+--------------------+---------+-------------+------+-------------+
|  1 | SIMPLE      | a     | index | NULL          | index_name_age_sex | 312     | NULL        |    1 | Using index |
|  1 | SIMPLE      | b     | ref   | index_name    | index_name         | 303     | demo.a.name |    1 | Using index |
+----+-------------+-------+-------+---------------+--------------------+---------+-------------+------+-------------+
2 rows in set (0.00 sec)
```

- 最左前缀

```
-- 命中了索引index_name_age_sex
mysql> explain select * from user where name='jadson';
+----+-------------+-------+------+--------------------+--------------------+---------+-------+------+-----------------------+
| id | select_type | table | type | possible_keys      | key                | key_len | ref   | rows | Extra                 |
+----+-------------+-------+------+--------------------+--------------------+---------+-------+------+-----------------------+
|  1 | SIMPLE      | user  | ref  | index_name_age_sex | index_name_age_sex | 303     | const |    1 | Using index condition |
+----+-------------+-------+------+--------------------+--------------------+---------+-------+------+-----------------------+
1 row in set (0.00 sec)
```

##### fulltext

全文索引检索。

要注意，全文索引检索的优先级很高，若全文索引和普通索引同时存在，mysql不管代价，优先使用全文索引。

**不建议使用**

```mysql
mysql> explain select * from addr where match(addr) against('bei');
+----+-------------+-------+----------+---------------+---------+---------+------+------+-------------+
| id | select_type | table | type     | possible_keys | key     | key_len | ref  | rows | Extra       |
+----+-------------+-------+----------+---------------+---------+---------+------+------+-------------+
|  1 | SIMPLE      | addr  | fulltext | ft_addr       | ft_addr | 0       | NULL |    1 | Using where |
+----+-------------+-------+----------+---------------+---------+---------+------+------+-------------+
1 row in set (0.00 sec)
```

##### red_or_null

与ref类似，只是增加了null值的比较，实际用的不多。

##### unique_subquery

用于where中的in形式子查询，子查询返回不重复唯一值。

##### Index_subquery

用于in形式子查询使用到了辅助索引或者in常数列表，子查询可能返回重复值，可以使用索引将子查询去重。

##### range

索引范围扫描，常见于使用<,>,is null, between,in,like等运算符的查询中。

```mysql
mysql> explain select * from user where id>1;
+----+-------------+-------+-------+---------------+---------+---------+------+------+-------------+
| id | select_type | table | type  | possible_keys | key     | key_len | ref  | rows | Extra       |
+----+-------------+-------+-------+---------------+---------+---------+------+------+-------------+
|  1 | SIMPLE      | user  | range | PRIMARY       | PRIMARY | 4       | NULL |    1 | Using where |
+----+-------------+-------+-------+---------------+---------+---------+------+------+-------------+
1 row in set (0.00 sec)

-- like前缀索引
mysql> explain select * from user where name like 'jadson%';
+----+-------------+-------+-------+--------------------+--------------------+---------+------+------+-----------------------+
| id | select_type | table | type  | possible_keys      | key                | key_len | ref  | rows | Extra                 |
+----+-------------+-------+-------+--------------------+--------------------+---------+------+------+-----------------------+
|  1 | SIMPLE      | user  | range | index_name_age_sex | index_name_age_sex | 303     | NULL |    1 | Using index condition |
+----+-------------+-------+-------+--------------------+--------------------+---------+------+------+-----------------------+
1 row in set (0.00 sec)
```

注：%jadson不使用索引。



##### index

要查询的列刚好覆盖索引.

```mysql
-- name列覆盖了索引
mysql> explain select name from user;
+----+-------------+-------+-------+---------------+--------------------+---------+------+------+-------------+
| id | select_type | table | type  | possible_keys | key                | key_len | ref  | rows | Extra       |
+----+-------------+-------+-------+---------------+--------------------+---------+------+------+-------------+
|  1 | SIMPLE      | user  | index | NULL          | index_name_age_sex | 312     | NULL |    1 | Using index |
+----+-------------+-------+-------+---------------+--------------------+---------+------+------+-------------+
1 row in set (0.00 sec)

-- 没有覆盖到索引，全文检索
mysql> explain select * from user;
+----+-------------+-------+------+---------------+------+---------+------+------+-------+
| id | select_type | table | type | possible_keys | key  | key_len | ref  | rows | Extra |
+----+-------------+-------+------+---------------+------+---------+------+------+-------+
|  1 | SIMPLE      | user  | ALL  | NULL          | NULL | NULL    | NULL |    1 | NULL  |
+----+-------------+-------+------+---------------+------+---------+------+------+-------+
1 row in set (0.00 sec)

-- order by的字段覆盖了索引（通过这个方法可以优化全表扫面的sql）
mysql> explain select * from user order by id;
+----+-------------+-------+-------+---------------+---------+---------+------+------+-------+
| id | select_type | table | type  | possible_keys | key     | key_len | ref  | rows | Extra |
+----+-------------+-------+-------+---------------+---------+---------+------+------+-------+
|  1 | SIMPLE      | user  | index | NULL          | PRIMARY | 4       | NULL |    1 | NULL  |
+----+-------------+-------+-------+---------------+---------+---------+------+------+-------+
1 row in set (0.00 sec)
```

##### all

全表扫描。

应避免全表扫面，原因：

1. 数据在在server的内存中过滤，而不是在存储引擎层，查询和过滤耗时长，占用内存多。

#### possible keys

此次查询中可能选用到的索引，一个或多个

#### key

查询真正使用到的索引，select_type为index_merge时，这里可能出现两个以上的索引，其它的select_type这里最多只会出现一个。

#### key_len

+ 用于处理查询的索引长度，如果是单列索引，那就整个索引长度算进去，如果是多列索引，那么查询不一定都能使用到所有的列，具体使用到了多少个列的索引就会算进去多少个，没有使用的不会算进去。
+ 根据这个值的长度可以计算出有没有使用到索引的全部列。
+ Key_len只计算where条件用到的索引长度，而排序和分组就算用到了索引，也不会计算到key_len中。

#### ref

+ 如果使用的是常数等值查询，这里会显示const
+ 如果使用的是连接查询，被驱动的表的这个字段会显示驱动表的关联字段
+ 如果条件使用了表达式或函数，或者条件列发生了内部隐式转换，这里可能显示为func

#### rows

这里是执行计划中估算的扫描行数，不是精确值（InnoDB不是精确值，MyIsam是精确值吗主要是因为InnoDB使用了MVCC并发机制）

#### extra

该列展示了不适合在其它列展示但是却十分重要的信息，这个列可以显示的信息非常多，有几十种，常用的有：

+ distinct：在select部分使用了distinct关键字。
+ no tables used