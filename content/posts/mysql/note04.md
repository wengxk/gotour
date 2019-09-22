---
title: "InnoDB Locking Mechanisms and Transaction Management"
date: 2019-09-15
type:
- post
- posts
categories:
- mysql
---

# InnoDB 锁机制与事务管理

现在的应用系统几乎都是多用户的，那在这样一个多用户多连接操作的情况下就会产生数据的并发性、一致性和完整性问题，InnoDB存储引擎应用锁机制来解决上述问题。
数据库事务是指数据库中一组不可分割的执行单元，事务具有ACID特性。
由于上述并发性问题，在事务并发执行期间会产生脏读、不可重复读和幻读等问题，为此，事务采用不同的事务级别来规避这些问题。
对于基于数据库应用的开发人员来说，充分理解锁机制和事务的管理是至关重要的，也只有此才能构建出安全与高效的数据库应用。

## 1. 锁

### 1.1 锁机制的整体行为表现

InnoDB存储引擎的锁机制实现类似于Oracle，整体都是基于行锁的实现，其整体行为表现大同小异。
在Oracle官方文档对于锁机制的一些讲解我认为写的非常好，而且对于我的工作也起到了很大的帮助，我认为这些基本原则同样也适用于InnoDB存储引擎。

1. row is locked only when modified by a writer.
2. writer of a row blocks a concurrent writer of the same row.
3. reader never blocks a writer.
4. writer never blocks a reader.

### 1.2 锁的分类

本文使用MySQL 8.0.16版本，`information_schema.INNODB_LOCKS` 和 `information_schema.INNODB_LOCK_WAITS` 已经被废弃，
由 `performance_schema.data_locks` 和 `performance_schema.data_lock_waits`替代。

#### 1.2.1 共享锁和排他锁

InnoDB存储引擎实现了两种标准的行级锁：

1. 共享锁：S Lock，允许事务读取一行数据
2. 排他锁：X Lock，允许事务删除和修改一行数据

关于两种锁的兼容性见意向锁总结列表。

#### 1.2.2 意向锁

意向锁简单的说就是指在对具体的行进行加锁前首先对更粗粒度的结构加上表明其意图的锁。
例如一个事务需要对一个行加上X锁，那么就需要对行所在的页或表等粗粒度的结构加上IX锁。
InnoDB存储引擎的意向锁设计简练，仅支持表级别的锁。

共有两种意向锁：

1. 意向共享锁：IS Lock，说明该事务想要对该表的行记录加上共享锁
2. 意向排他锁：IX Lock，说明该事务想要对该表的行记录加上排他锁

S Lock， X Lock， IS Lock， IX Lock之间的兼容性如下图所示：

||X|IX|S|IS|
|:---|:---|:---|:---|:---|
|**X**|冲突|冲突|冲突|冲突|
|**IX**|冲突|兼容|冲突|兼容|
|**S**|冲突|冲突|兼容|兼容|
|**IS**|冲突|兼容|兼容|兼容|

#### 1.2.3 示例

```SQL
CREATE TABLE `students` (
	`id` MEDIUMINT(9) NOT NULL AUTO_INCREMENT,
	`NAME` CHAR(30) NULL DEFAULT NULL COLLATE 'utf8mb4_general_ci',
	PRIMARY KEY (`id`)
)
COLLATE='utf8mb4_general_ci'
ENGINE=InnoDB;

INSERT INTO
	tour.students (`NAME`) VALUES ('Tom'),
('Hank'),
('Jack'),
('Nancy'),
('Lucy') ;
```

Session A:

```SQL
BEGIN;
UPDATE tour.students t SET t.NAME = CONCAT(t.NAME, '1') WHERE t.id = 1;
```

查询锁

```SQL
select
	t.`ENGINE_TRANSACTION_ID`,
	t.`THREAD_ID`,
	t.`OBJECT_SCHEMA`,
	t.`OBJECT_NAME`,
	t.`INDEX_NAME`,
	t.`LOCK_TYPE`,
	t.`LOCK_MODE`,
	t.`LOCK_STATUS`,
	t.`LOCK_DATA`
from
	performance_schema.data_locks t;
```

|ENGINE_TRANSACTION_ID|THREAD_ID|OBJECT_SCHEMA|OBJECT_NAME|INDEX_NAME|LOCK_TYPE|LOCK_MODE    |LOCK_STATUS|LOCK_DATA|
|---------------------|---------|-------------|-----------|----------|---------|-------------|-----------|---------|
|                 7818|       50|tour         |students   |          |TABLE    |IX           |GRANTED    |         |
|                 7818|       50|tour         |students   |PRIMARY   |RECORD   |X,REC_NOT_GAP|GRANTED    |1        |

查询锁等待

```SQL
select
	t.`REQUESTING_ENGINE_TRANSACTION_ID`,
	t.`REQUESTING_THREAD_ID`,
	t.`BLOCKING_ENGINE_TRANSACTION_ID`,
	t.`BLOCKING_THREAD_ID`
from
	performance_schema.data_lock_waits t;
```

Empty set (0.00 sec)

Session B:

```SQL
-- BEGIN; -- 此处不用显示开启事务也能达到效果。
SELECT * FROM tour.students t WHERE t.id = 1 LOCK IN SHARE MODE;
```

查询锁

|ENGINE_TRANSACTION_ID|THREAD_ID|OBJECT_SCHEMA|OBJECT_NAME|INDEX_NAME|LOCK_TYPE|LOCK_MODE    |LOCK_STATUS|LOCK_DATA|
|---------------------|---------|-------------|-----------|----------|---------|-------------|-----------|---------|
|                 7818|       50|tour         |students   |          |TABLE    |IX           |GRANTED    |         |
|                 7818|       50|tour         |students   |PRIMARY   |RECORD   |X,REC_NOT_GAP|GRANTED    |1        |
|      284053005736600|       59|tour         |students   |          |TABLE    |IS           |GRANTED    |         |
|      284053005736600|       59|tour         |students   |PRIMARY   |RECORD   |X,REC_NOT_GAP|WAITING    |1        |

查询锁等待

|REQUESTING_ENGINE_TRANSACTION_ID|REQUESTING_THREAD_ID|BLOCKING_ENGINE_TRANSACTION_ID|BLOCKING_THREAD_ID|
|--------------------------------|--------------------|--------------------------------|----------------|
|                 284053005736600|                  59|                            7818|              50|

> 50s会后出现如下错误，因为等待锁定超时，超时时间由参数 `innodb_lock_wait_timeout` 配置。  
> ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction

Session A:

`mysql> rollback;`

查询锁

Empty set (0.00 sec)

查询锁等待

Empty set (0.00 sec)

分析：

在会话A中显示开启一个事务，然后更新表tour.students主键id为1的数据，根据查询结果可以看到
此过程给表tour.students加了IX锁，且给主键id为1的记录加了X锁，此时没有任何资源等待。

另起会话B，以锁定方式读取表tour.students主键id为1的数据，此时可以看到表 `performance_schema.data_locks`多了两条记录，
分别为表tour.students上的IS锁和主键id为1记录上的S锁，但该S锁是 `WAITING` 状态，因为S与X锁不兼容。
表 `performance_schema.data_lock_waits` 有一条数据，BLOCKING_THREAD_ID: 50，REQUESTING_THREAD_ID: 59，
即上面会话A中事务对应的线程ID：50 阻塞了会话B中事务对应的线程ID：59。

若会话A在参数 `innodb_lock_wait_timeout` 设定的时间内既没提交也没回滚，则会话B会报错，等待资源锁超时。

### 1.3 行锁的三种分类

#### 1.3.1 介绍

- Record Lock：单行记录锁

- Gap Lock：间隙锁，锁定一个区间，但不包括记录本身

- Next-Key Lock：Record Lock + Gap Lock，锁定一个区间，并锁定记录本身

InnoDB存储引擎使用Next-Key Locking机制来解决幻读问题，所以Next-Key Locking会受到事务级别影响，同时还会受到where条件中的列是否具有索引及何种索引的影响。
InnoDB存储引擎的索引结构为B+Tree，在索引记录上查找给定记录时，InnoDB会在第一个不满足查询条件的记录上加gap lock,防止新的满足条件的记录插入。

一般的，如果一个索引列具有值10，13，8，17，可能的锁定区间为：

(-∞,8]

(8,10]

(10,13]

(13,17]

(17,+∞)

#### 1.3.2 示例

以下演示Next-Key Locking在不同场景下的使用情况

```SQL
CREATE TABLE `user` (
	`id` MEDIUMINT(9) NOT NULL AUTO_INCREMENT,
	`code` CHAR(5) NOT NULL COLLATE 'utf8mb4_general_ci',
	`name` VARCHAR(30) NULL DEFAULT NULL COLLATE 'utf8mb4_general_ci',
	`age` TINYINT(3) UNSIGNED NULL DEFAULT NULL, PRIMARY KEY (`id`), UNIQUE INDEX `code` (`code`), INDEX `age` (`age`)
) COLLATE='utf8mb4_general_ci' ENGINE=INNODB
;

INSERT INTO
	tour.user(CODE,NAME,age)
VALUES
	('00001','Tom',12),
	('00002','Jack',15),
	('00003','Nancy',8),
	('00004','Lucy',18),
	('00005','Jim',13);

```

表数据

|id |code |name |age|
|---:|---:|---:|---:|
| 1|00001|Tom  | 12|
| 2|00002|Jack | 15|
| 3|00003|Nancy|  8|
| 4|00004|Lucy | 18|
| 5|00005|Jim  | 13|

##### 1.3.2.1 主键

Session A:

- 单行查询

```SQL
BEGIN;
SELECT * FROM tour.user t WHERE t.id = 2 FOR UPDATE;
```

查询锁

```SQL
select
	t.`ENGINE_TRANSACTION_ID`,
	t.`THREAD_ID`,
	t.`OBJECT_SCHEMA`,
	t.`OBJECT_NAME`,
	t.`INDEX_NAME`,
	t.`LOCK_TYPE`,
	t.`LOCK_MODE`,
	t.`LOCK_STATUS`,
	t.`LOCK_DATA`
from
	performance_schema.data_locks t;
```

|ENGINE_TRANSACTION_ID|THREAD_ID|OBJECT_SCHEMA|OBJECT_NAME|INDEX_NAME|LOCK_TYPE|LOCK_MODE    |LOCK_STATUS|LOCK_DATA|
|---------------------|---------|-------------|-----------|----------|---------|-------------|-----------|---------|
|                 8684|       61|tour         |user       |          |TABLE    |IX           |GRANTED    |         |
|                 8684|       61|tour         |user       |PRIMARY   |RECORD   |X,REC_NOT_GAP|GRANTED    |2        |

说明：在唯一建上查找单条记录时不会产生任何gap lock，锁定类型为REC_NOT_GAP，只锁定一行数据，主键为2。

- 不存在的单行记录查询

```SQL
ROLLBACK;
BEGIN;
SELECT * FROM tour.user t WHERE t.id = 6 FOR UPDATE; -- 这里换成id=10后面的效果也一样

```

查询锁

|ENGINE_TRANSACTION_ID|THREAD_ID|OBJECT_SCHEMA|OBJECT_NAME|INDEX_NAME|LOCK_TYPE|LOCK_MODE|LOCK_STATUS|LOCK_DATA             |
|---------------------|---------|-------------|-----------|----------|---------|---------|-----------|----------------------|
|                 8701|       61|tour         |user       |          |TABLE    |IX       |GRANTED    |                      |
|                 8701|       61|tour         |user       |PRIMARY   |RECORD   |X        |GRANTED    |supremum pseudo-record|

说明：当查询记录不存在时会对next-key加X锁，而此时next-key就是+∞

Session B:

尝试插入记录

```SQL
BEGIN;
INSERT INTO
	tour.user(CODE,NAME,age)
VALUES
	('00006','Hank',12);
```

查询锁

|ENGINE_TRANSACTION_ID|THREAD_ID|OBJECT_SCHEMA|OBJECT_NAME|INDEX_NAME|LOCK_TYPE|LOCK_MODE         |LOCK_STATUS|LOCK_DATA             |
|---------------------|---------|-------------|-----------|----------|---------|------------------|-----------|----------------------|
|                 8713|       65|tour         |user       |          |TABLE    |IX                |GRANTED    |                      |
|                 8713|       65|tour         |user       |PRIMARY   |RECORD   |X,INSERT_INTENTION|WAITING    |supremum pseudo-record|
|                 8712|       61|tour         |user       |          |TABLE    |IX                |GRANTED    |                      |
|                 8712|       61|tour         |user       |PRIMARY   |RECORD   |X                 |GRANTED    |supremum pseudo-record|

查询锁等待

|ENGINE|REQUESTING_ENGINE_TRANSACTION_ID|REQUESTING_THREAD_ID|BLOCKING_ENGINE_TRANSACTION_ID|BLOCKING_THREAD_ID|
|------|--------------------------------|--------------------|------------------------------|------------------|
|INNODB|                            8713|                  65|                          8712|                61|

可以看到新插入数据时也需要对+∞进行加锁（X,INSERT_INTENTION），此时两个X锁不兼容，造成了阻塞与等待。

Session B:

```SQL
ROLLBACK;
```

Session C:

尝试更新记录

```SQL
BEGIN;
update tour.`user` t set t.name = concat(t.name,'1') where t.id = 3;
update tour.`user` t set t.name = concat(t.name,'1') where t.id = 10;
```

查询锁

|ENGINE_TRANSACTION_ID|THREAD_ID|OBJECT_SCHEMA|OBJECT_NAME|INDEX_NAME|LOCK_TYPE|LOCK_MODE    |LOCK_STATUS|LOCK_DATA             |
|---------------------|---------|-------------|-----------|----------|---------|-------------|-----------|----------------------|
|                 8727|       65|tour         |user       |          |TABLE    |IX           |GRANTED    |                      |
|                 8727|       65|tour         |user       |PRIMARY   |RECORD   |X,REC_NOT_GAP|GRANTED    |3                     |
|                 8727|       65|tour         |user       |PRIMARY   |RECORD   |X            |GRANTED    |supremum pseudo-record|
|                 8723|       61|tour         |user       |          |TABLE    |IX           |GRANTED    |                      |
|                 8723|       61|tour         |user       |PRIMARY   |RECORD   |X            |GRANTED    |supremum pseudo-record|

查询锁等待

Empty set (0.00 sec)

>疑问：为什么session a 里的插入会被阻塞，而session c里的更新却不会被阻塞呢？  
>不是都会对supremum pseudo-record伪列加X锁么，应该都不兼容会阻塞啊。。。

- 范围查询

```SQL
ROLLBACK;
BEGIN;
SELECT * FROM tour.user t WHERE t.id <= 2 FOR UPDATE;
```

查询锁

|ENGINE_TRANSACTION_ID|THREAD_ID|OBJECT_SCHEMA|OBJECT_NAME|INDEX_NAME|LOCK_TYPE|LOCK_MODE|LOCK_STATUS|LOCK_DATA|
|---------------------|---------|-------------|-----------|----------|---------|---------|-----------|---------|
|                 8694|       61|tour         |user       |          |TABLE    |IX       |GRANTED    |         |
|                 8694|       61|tour         |user       |PRIMARY   |RECORD   |X        |GRANTED    |1        |
|                 8694|       61|tour         |user       |PRIMARY   |RECORD   |X        |GRANTED    |2        |
|                 8694|       61|tour         |user       |PRIMARY   |RECORD   |X        |GRANTED    |3        |

此时产生了三个Record Lock，主键为1，2和3的记录被锁定。

>主键为1和2的记录被锁定，这个很容易理解，但是为什么主键为3的记录也会被锁定？  
>可以理解为3为2记录的next-key值。  
>此时不能对主键为3的记录进行诸如删改等不兼容X锁的操作。

- 修改事务级别为READ COMMITTED

```SQL
ROLLBACK;
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
BEGIN;
SELECT * FROM tour.user t WHERE t.id <= 2 FOR UPDATE;
```

查询锁

|ENGINE_TRANSACTION_ID|THREAD_ID|OBJECT_SCHEMA|OBJECT_NAME|INDEX_NAME|LOCK_TYPE|LOCK_MODE    |LOCK_STATUS|LOCK_DATA|
|---------------------|---------|-------------|-----------|----------|---------|-------------|-----------|---------|
|                 8696|       61|tour         |user       |          |TABLE    |IX           |GRANTED    |         |
|                 8696|       61|tour         |user       |PRIMARY   |RECORD   |X,REC_NOT_GAP|GRANTED    |1        |
|                 8696|       61|tour         |user       |PRIMARY   |RECORD   |X,REC_NOT_GAP|GRANTED    |2        |

间隙锁只会在高于READ COMMITTED级别中的事务出现。修改事务级别READ COMMITTED后可以发现没有任何间隙锁，而且只需要锁定主键为1和2的两条记录。

##### 1.3.2.2 唯一键

##### 1.3.2.3 非唯一索引

### 1.4 自增与锁

### 1.5 外键与锁

### 1.6 死锁

### 1.7 锁的转换与升级
