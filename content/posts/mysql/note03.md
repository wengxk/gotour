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

## 1. 磁盘逻辑结构

### 1.1 Tablespace

表空间是逻辑结构的最顶层，所以InnoDB中的所有数据必定是逻辑的归属于一个表空间，表空间又由Segment（段）、Extent（区）和Page（页）逻辑组成。

系统表空间ibdata1主要是用来组织存放doublewrite buffer和change buffer，还会用来共享存放一些数据，例如回滚信息、系统事物信息。

目前InnoDB存储引擎和数据库使用方都会默认开启 `innodb_file_per_table` 参数，即为每张用户表单独创建一个表空间及物理文件，
所以你经常会发现，索引的名称默认就是索引列的名称，不同用户表的相同列的索引就会有相同的索引名称，而这在Oracle数据库中几乎是不会存在的。
在使用Oracle数据库时，在创建实例后的第一件事就是创建业务表空间，然后再创建主要的数据库用户并为之分配默认表空间，
而后以这个用户创建的对象默认表空间都是这个用户所在的表空间，所以不能存在相同名称的同一类对象。

### 1.2 Segment

段是InnoDB的次级逻辑结构，常见有数据段、索引段和回滚段，对于段，我们几乎不需要手工维护任何东西，有个概念即可。

### 1.3 Extent

区是最小的扩展单位，是由连续页组成的空间，在任何情况下每个区的大小都为1MB。为了保证区中页的连续性，InnoDB存储引擎一次从磁盘申请4～5个区。
在默认情况下，InnoDB存储引擎页的大小为16KB，即一个区中一共有64个连续的页。

### 1.4 Page

页是最小的存储单位，默认16K，由参数 `innodb_page_size` 控制，
同时，页也是最小的读取单位，InnoDB引擎在进行磁盘读取数据时都是整页的读取，然后将整页数据缓冲到buffer pool中。

## 2. 表

### 2.1 Index-Organized Tables

InnoDB存储引擎中的表相当于是Oracle数据库中的索引组织表（index-organized table），表中的数据按照主键顺序存储，索引就是表，表就是索引。
细微不同的是Oracle中的索引组织表在创建时必须同时声明主键，而InnoDB存储引擎创建表时不强制必须存在主键，存在非空唯一索引即可。
如果主键和非空唯一索引都不存在，InnoDB存储引擎自动创建一个6字节大小rowid列作为隐藏主键。
如果存在多个非空唯一索引，则以第一个定义的非空唯一索引为准。

> 虽然InnoDB允许表没有主键，但是强烈建议建表的同时主动声明主键

### 2.2 AUTO_INCREMENT

#### 2.2.1 Usage

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

如果不小心为主键指定了合法的数值，那么以后还能不能安全的使用自增的值？答案是肯定的。

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

#### 2.2.2 AUTO_INCREMENT MODE

TODO

### 2.3 Constraints

InnoDB存储引擎虽未像Oracle那样提供了强大的约束，但一些主要的约束都有提供，共有以下五种：

- Primary Key
- Unique Key
- Foreign Key
- Default
- Not Null

对于外键，InnoDB存储引擎会自动为其创建索引，而Oracle数据库不会，所以在Oracle数据库中建外键时一定要记得加索引。

在Oracle数据库中check约束可以进行数据插入前的检验操作，InnoDB存储引擎则没有提供，
但是可以通过 `ENUM` 和 `SET` 数据类型及触发器实现类似的效果。

### 2.4 Views

视图在数据库中使用广泛，但是视图本身很简单，相当于一张虚拟的表，核心是查询语句。
使用视图主要是有以下两个好处：

1. 方便。视图将常用的查询逻辑封装起来，有利于开发和代码维护。
2. 易于对外权限与安全的管理。当外围应用需要聚合查询各种数据时，我们可以为其创建特定的视图，并将视图的查询权限授权给它，
避免暴露数据库的基表，简化了授权的同时也提高了安全性。另外，视图不仅可以查询，还可以被更新，这特别适合于一些场景的
使用，比如我们只允许某外部应用更新本系统某张表的特定字段，而不想暴露其他字段，更不希望其更新其他字段，这时，
我们就可以为这张表创建特定字段的视图提供给外部应用。

### 2.5 Partitions

InnoDB存储引擎和Oracle数据库都支持表分区，但是InnoDB存储引擎暂时不支持全局索引，Oracle数据库则可以。

表分区最大的一个特征就是在进行数据插入时会根据一定的条件，可以是列的散列值，区间值，函数，将数据进行分区存储与维护。
表分区适合于OLAP应用，特别是那些跟年份聚合数据有关的业务。表分区用的合适，的确可以提高一定的查询性能和降低维护成本，但是，
用的不好，副作用则远大于不分区。

目前，本人没有使用过表分区，且接触的几个系统也都没有使用分区，这主要是由这些业务决定，没有特别的需要要去使用分区。

工作中有接触和使用过一张Oracle数据表，有20多亿条数据，单表600G，这样一张大表，它都没有使用分区，查询性能也不差，查询一条数据最多1秒。

## 3. 索引

索引分为聚簇索引（Clustered Index）和二级索引（Secondary Index），都是采用B+Tree的数据结构。

聚簇索引就是实际顺序存储数据的那个索引，其他索引都称为二级索引。

二级索引的叶子节点存储的是聚簇索引的索引键，然后再根据聚簇索引查询实际数据，因此聚簇索引的查询效率要高于二级索引。

关于排序索引构建的过程可以参考[MySQL InnoDB Sorted Index Builds](https://www.percona.com/blog/2019/05/08/mysql-innodb-sorted-index-builds/)，
附[中文翻译](https://blog.csdn.net/actiontech/article/details/99299841)

## 4. Doublewrite Buffer

doublewrite（两次写）是牺牲了少部分的写入性能来提高InnoDB存储引擎的可靠性，
之所以称为doublewrite，是因为从缓冲数据刷入到磁盘的过程中经历了两次写的过程。

![Doublewrite Buffer](/images/mysql/double_write.png)

## 5. Redo Log

重做日志主要是为了能够在系统故障时能够恢复已经提交事务但尚未完整更新到磁盘的事务数据。

如果每次在事务提交时都需要将修改后的数据完整更新到对应的磁盘文件中，那么整体操作是比较费时的，因此InnoDB存储引擎采用了
`Write-Ahead Logging` 策略，buffer pool中的脏页先缓冲到重做日志缓冲中，然后再以一定的策略定时刷入到磁盘中的重做日志文件，
最后则在事务提交时将所有的重做日志缓冲刷入到磁盘的重做日志文件中（见[log buffer](https://wengxk.netlify.com/2019/note02/)），
这保证了事务的D特性，最后buffer pool中的脏页会适时的更新到磁盘文件中，避免在磁盘I/O高峰期增加额外负担。

![Redo Log](/images/mysql/redo_log.png)

重做日志文件大小由参数 `innodb_log_file_size` 控制，默认48MB，重做日志文件的个数由参数 `innodb_log_files_in_group` 控制，
默认2，即有两个重做日志文件，分别为 `ib_logfile0` 和 `ib_logfile1` 。

## 6. Undo Log

DML操作期间不仅会产生redo log，还会产生 `Undo Log` ，其主要用于当前事务的 `Rollback` 操作和其他事务的非锁定一致性读，即 `MVCC` 。

undo log存在于 `Undo Segment` 中，而 `Rollback Segment` 又由1024个undo segment组成。
系统可以通过参数 `innodb_rollback_segments` 配置rollback segment的个数，默认128个。
对用户自定义表进行DML操作时产生的undo log存在于  `Undo Tablespace` ，
对用户自定义临时表进行DML操作时所产生的undo log存在于  `global temporary tablespace` 。

undo相关参数

`mysql> SHOW VARIABLES LIKE '%undo%';`

|Variable_name|Value|Description|
|:---|:---|:---|
|innodb_max_undo_log_size|1073741824|1GB，每个undo tablespace所包含undo log的最大容量|
|innodb_undo_directory|.\\|undo log的存储路径，默认当前路径|
|innodb_undo_log_encrypt|OFF|控制是否加密，默认不开启|
|innodb_undo_log_truncate|ON|当undo tablespace所包含的undo log超过innodb_max_undo_log_size配置的值时，是否发生截断，ON代表开启|
|innodb_undo_tablespaces|2|2代表会有两个undo tablespace，对应就会有两个数据文件undo_001和undo_002|

全局临时表空间

`mysql> SHOW VARIABLES LIKE '%innodb_temp_data_file_path%';`

|Variable_name|Value|Description|
|:---|:---|:---|
|innodb_temp_data_file_path|ibtmp1:12M:autoextend|全局临时表空间，数据文件名为ibtmp1，初始12M，自增|
