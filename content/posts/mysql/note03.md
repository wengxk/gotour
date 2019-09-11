---
title: "InnoDB On-Disk Structures"
date: 2019-09-08
type:
- post
- posts
categories:
- mysql
---

# InnoDB 磁盘结构

InnoDB磁盘结构和Oracle数据库的磁盘结构具有一定相似性，比方说，都以一定的逻辑结构维护组织具体磁盘文件，
都有Tablespace（表空间）、Segment（段）、Extent（区）和Page（页）的逻辑概念，
其中，Page在Oracle数据库中多以Block（数据块）表示。

## 磁盘逻辑结构

### Tablespace

表空间是逻辑结构的最顶层，所以InnoDB中的所有数据必定是逻辑的归属于一个表空间，表空间又由Segment（段）、Extent（区）和Page（页）逻辑组成。

系统表空间ibdata1主要是用来组织存放doublewrite buffer和change buffer，还会用来共享存放一些数据，例如回滚信息、系统事物信息。

目前InnoDB存储引擎和数据库使用方都会默认开启 `innodb_file_per_table` 参数，即为每张用户表单独创建一个表空间及物理文件，
所以你经常会发现，索引的名称默认就是索引列的名称，不同用户表的相同列的索引就会有相同的索引名称，而这在Oracle数据库中几乎是不会存在的。
在使用Oracle数据库时，在创建实例后的第一件事就是创建业务表空间，然后再创建主要的数据库用户并为之分配默认表空间，
而后以这个用户创建的对象默认表空间都是这个用户所在的表空间，所以不能存在相同名称的同一类对象。

### Segment

段是InnoDB的次级逻辑结构，常见有数据段、索引段和回滚段，对于段，我们几乎不需要手工维护任何东西，有个概念即可。

### Extent

区是最小的扩展单位，是由连续页组成的空间，在任何情况下每个区的大小都为1MB。为了保证区中页的连续性，InnoDB存储引擎一次从磁盘申请4～5个区。
在默认情况下，InnoDB存储引擎页的大小为16KB，即一个区中一共有64个连续的页。

### Page

页是最小的存储单位，默认16K，由参数 `innodb_page_size` 控制，
同时，页也是最小的读取单位，InnoDB引擎在进行磁盘读取数据时都是整页的读取，然后将整页数据缓冲到buffer pool中。

## 表

### Index-Organized Tables

InnoDB存储引擎中的表相当于是Oracle数据库中的索引组织表（index-organized table），表中的数据按照主键顺序存储，索引就是表，表就是索引。
细微不同的是Oracle中的索引组织表在创建时必须同时声明主键，而InnoDB存储引擎创建表时不强制必须存在主键，存在非空唯一索引即可。
如果主键和非空唯一索引都不存在，InnoDB存储引擎自动创建一个6字节大小rowid列作为隐藏主键。
如果存在多个非空唯一索引，则以第一个定义的非空唯一索引为准。

> 虽然InnoDB允许表没有主键，但是强烈建议建表的同时主动声明主键

### AUTO_INCREMENT

#### Usage

当前在应用开发时一般都会对主键选择自增类型，Oracle提供了sequence，InnoDB则提供了AUTO_INCREMENT特性。

```SQL
CREATE TABLE students
(
id MEDIUMINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
name CHAR(30)
);

INSERT INTO students(name)VALUES('Tom');
INSERT INTO students(id,name)(NULL,'Hank');
INSERT INTO students(id,name)(0,'Jack'); -- error if NO_AUTO_VALUE_ON_ZERO sql_mode is enabled

SELECT * FROM students;
```

|id|name|
|:---|:---|
|1|Tom|
|2|Hank|
|3|Jack|

如果不小心为主键制定了合法的数值，那么以后还能不能安全的使用自增的值？答案是肯定的。

```SQL
INSERT INTO students(id,NAME) VALUES (100,'Nancy');
INSERT INTO students(NAME) VALUES ('Lucy');
SELECT * FROM students;
```

|id|name|
|:---|:---|
|1|Tom|
|2|Hank|
|3|Jack|
|100|Nancy|
|101|Lucy|

使用 `LAST_INSERT_ID()` 可以获得当前会话中最近的自增值。

```SQL
INSERT INTO students(name)VALUES('Mike');
SELECT LAST_INSERT_ID();
```

|LAST_INSERT_ID()|
|:---|
|102|

InnoDB存储引擎还提供了一个方便的自增类型 `SERIAL` ，它是 `BIGINT UNSIGNED NOT NULL AUTO_INCREMENT UNIQUE` 的同义词。

#### AUTO_INCREMENT MODE

TODO

### Constraints

InnoDB存储引擎虽未像Oracle那样提供了强大的约束，但一些主要的约束都有提供，共有以下五种：

- Primary Key
- Unique Key
- Foreign Key
- Default
- Not Null

对于外键，InnoDB存储引擎会自动为其创建索引，而Oracle数据库不会，所以在Oracle数据库中建外键时一定要记得加索引。

在Oracle数据库中check约束可以进行数据插入前的检验操作，InnoDB存储引擎则没有提供，
但是可以通过 `ENUM` 和 `SET` 数据类型及触发器实现类似的效果。

### Views

视图在数据库中使用广泛，但是视图本身很简单，相当于一张虚拟的表，核心是查询语句。
使用视图主要是有以下两个好处：

1. 方便。视图将常用的查询逻辑封装起来，有利于开发和代码维护。
2. 易于对外权限与安全的管理。当外围应用需要聚合查询各种数据时，我们可以为其创建特定的视图，并将视图的查询权限授权给它，
避免暴露数据库的基表，简化了授权的同时也提高了安全性。另外，视图不仅可以查询，还可以被更新，这特别适合于一些场景的
使用，比如我们只允许某外部应用更新本系统某张表的特定字段，而不想暴露其他字段，更不希望其更新其他字段，这时，
我们就可以为这张表创建特定字段的视图提供给外部应用。

### Partitions

InnoDB存储引擎和Oracle数据库都支持表分区，但是InnoDB存储引擎暂时不支持全局索引，Oracle数据库则可以。

表分区最大的一个特征就是在进行数据插入时会根据一定的条件，可以是列的散列值，区间值，函数，将数据进行分区存储与维护。
表分区适合于OLAP应用，特别是那些跟年份聚合数据有关的业务。表分区用的合适，的确可以提高一定的查询性能和降低维护成本，但是，
用的不好，副作用则远大于不分区。

目前，本人没有使用过表分区，且接触的几个系统也都没有使用分区，这主要是由这些业务决定，没有特别的需要要去使用分区。

工作中有接触和使用过一张Oracle数据表，有20多亿条数据，单表600G，这样一张大表，它都没有使用分区，查询性能也不差，查询一条数据最多1秒。

## 索引

索引分为聚簇索引（Clustered Index）和二级索引（Secondary Index），都是采用B+Tree的数据结构。

聚簇索引就是实际顺序存储数据的那个索引，其他索引都称为二级索引。

二级索引的叶子节点存储的是聚簇索引的索引键，然后再根据聚簇索引查询实际数据，因此聚簇索引的查询效率要高于二级索引。

关于排序索引构建的过程可以参考[MySQL InnoDB Sorted Index Builds](https://www.percona.com/blog/2019/05/08/mysql-innodb-sorted-index-builds/)，
附[中文翻译](https://blog.csdn.net/actiontech/article/details/99299841)

## Doublewrite Buffer

doublewrite（两次写）是牺牲了少部分的写入性能来提高的InnoDB存储引擎的可靠性，
之所以称为doublewrite，是因为从缓冲数据刷入到磁盘的过程中经历了两次写的过程。

![InnoDB Architecture](/images/mysql/logic_arch.png)

## Redo Log

## Undo Logs
