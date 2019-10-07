---
title: "InnoDB Locking Mechanisms"
date: 2019-09-15
type:
- post
- posts
categories:
- mysql
---

# InnoDB 锁机制

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
    `name` CHAR(30) NULL DEFAULT NULL COLLATE 'utf8mb4_general_ci',
    PRIMARY KEY (`id`)
)
COLLATE='utf8mb4_general_ci'
ENGINE=InnoDB;

INSERT INTO tour.students (name) VALUES ('Tom'),('Hank'),('Jack'),('Nancy'),('Lucy');
```

Session A:

```SQL
BEGIN;
UPDATE tour.students t SET t.name = CONCAT(t.NAME, '1') WHERE t.id = 1;
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

另起会话B，以share方式读取表tour.students主键id为1的数据，此时可以看到表 `performance_schema.data_locks`多了两条记录，
分别为表tour.students上的IS锁和主键id为1记录上的X锁，但该X锁是 `WAITING` 状态，因为X锁与X锁不兼容。
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

官方文档8.0版本的文档

(-∞,8]

(8,10]

(10,13]

(13,17]

(17,+∞)

MySQL技术内幕：InnoDB存储引擎

(-∞,8)

[8,10)

[10,13)

[13,17)

[17,+∞)

>经过测试，第二种方式符合实际结果。

#### 1.3.2 示例

还是使用以上数据演示Next-Key Locking在不同场景下的使用情况

##### 1.3.2.1 主键&唯一键

- 单行查询

在上面的例子已经体现了在唯一键上查询单条数据时的锁情况：不会产生任何gap lock，只会产生单行记录锁，类型为X,REC_NOT_GAP。

- 不存在的单行记录查询

Session A:

```SQL
ROLLBACK;
BEGIN;
SELECT * FROM tour.students t WHERE t.id = 6 FOR UPDATE; -- 这里换成id=10后面的效果也一样

```

查询锁

|ENGINE_TRANSACTION_ID|THREAD_ID|OBJECT_SCHEMA|OBJECT_NAME|INDEX_NAME|LOCK_TYPE|LOCK_MODE|LOCK_STATUS|LOCK_DATA             |
|---------------------|---------|-------------|-----------|----------|---------|---------|-----------|----------------------|
|                 8701|       61|tour         |students   |          |TABLE    |IX       |GRANTED    |                      |
|                 8701|       61|tour         |students   |PRIMARY   |RECORD   |X        |GRANTED    |supremum pseudo-record|

说明：当查询记录不存在时会对next-key加X锁，而此时next-key就是+∞

Session B:

尝试插入记录

```SQL
BEGIN;
INSERT INTO tour.user(NAME) VALUES ('Hank');
```

查询锁

|ENGINE_TRANSACTION_ID|THREAD_ID|OBJECT_SCHEMA|OBJECT_NAME|INDEX_NAME|LOCK_TYPE|LOCK_MODE         |LOCK_STATUS|LOCK_DATA             |
|---------------------|---------|-------------|-----------|----------|---------|------------------|-----------|----------------------|
|                 8713|       65|tour         |students   |          |TABLE    |IX                |GRANTED    |                      |
|                 8713|       65|tour         |students   |PRIMARY   |RECORD   |X,INSERT_INTENTION|WAITING    |supremum pseudo-record|
|                 8712|       61|tour         |students   |          |TABLE    |IX                |GRANTED    |                      |
|                 8712|       61|tour         |students   |PRIMARY   |RECORD   |X                 |GRANTED    |supremum pseudo-record|

查询锁等待

|ENGINE|REQUESTING_ENGINE_TRANSACTION_ID|REQUESTING_THREAD_ID|BLOCKING_ENGINE_TRANSACTION_ID|BLOCKING_THREAD_ID|
|------|--------------------------------|--------------------|------------------------------|------------------|
|INNODB|                            8713|                  65|                          8712|                61|

可以看到此时出现了阻塞，这是因为新插入的数据处在区间[6,+∞)上，会被阻塞。

Session B:

```SQL
ROLLBACK;
```

Session C:

尝试更新记录

```SQL
BEGIN;
update tour.students t set t.name = concat(t.name,'1') where t.id = 3;
update tour.students t set t.name = concat(t.name,'1') where t.id = 10;
delete from tour.students t where t.id = 11;
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

>sessoin b中的事务会被阻塞，但session c中的事务并没有被阻塞，这里说明了gap锁可以共存。  
>gap锁会阻塞gap内的insert操作，但是不会阻塞gap内的更新或是删除操作

##### 1.3.2.2 非唯一索引

使用非唯一索引更能体现出gap锁的锁定机制细节。

准备测试表及数据。

```SQL
CREATE TABLE `t01` (
    `num` INT(10) NULL DEFAULT NULL,
    INDEX `num` (`num`)
)COLLATE='utf8mb4_general_ci' ENGINE=InnoDB;

INSERT INTO t01(num)VALUES(-3),(10),(15),(20),(30),(70);
```

上述二级索引有区间(-∞,-3],(3,10],(10,15],(15,20],(20,30],(30,70],(70,+∞)，通过以下示例可以探究到这些区间在不同查询条件下的具体细节。

- 等于

Session A:

```SQL
BEGIN;
SELECT * FROM tour.t01 t WHERE t.num = 20 FOR UPDATE;
```

查询锁

|ENGINE_TRANSACTION_ID|THREAD_ID|OBJECT_SCHEMA|OBJECT_NAME|INDEX_NAME     |LOCK_TYPE|LOCK_MODE    |LOCK_STATUS|LOCK_DATA         |
|---------------------|---------|-------------|-----------|---------------|---------|-------------|-----------|------------------|
|                 9010|      124|tour         |t01        |               |TABLE    |IX           |GRANTED    |                  |
|                 9010|      124|tour         |t01        |num            |RECORD   |X            |GRANTED    |20, 0x000000000501|
|                 9010|      124|tour         |t01        |GEN_CLUST_INDEX|RECORD   |X,REC_NOT_GAP|GRANTED    |0x000000000501    |
|                 9010|      124|tour         |t01        |num            |RECORD   |X,GAP        |GRANTED    |30, 0x000000000503|

说明：

1. 第一行为表级别锁IX，第二行为二级索引上的X锁，第三行为隐藏主键上的单行记录锁，第四行为gap锁。
2. 此时实际上会话a对表t01的锁定区间为[20,30)，具体影响为其他事务不能对20这一行数据进行删改，即不能进行不兼容X锁的操作，同时不能在[20,30)
这个区间内插入任何数据。
3. 除了20这条记录，其他的随便删，不会被阻塞。

验证：验证的sql就不放了，不然篇幅太长，影响阅读。

让我们再修改下上述语句。

```SQL
BEGIN;
SELECT * FROM tour.t01 t WHERE t.num = 21 FOR UPDATE;
```

查询锁

|ENGINE_TRANSACTION_ID|THREAD_ID|OBJECT_SCHEMA|OBJECT_NAME|INDEX_NAME|LOCK_TYPE|LOCK_MODE|LOCK_STATUS|LOCK_DATA         |
|---------------------|---------|-------------|-----------|----------|---------|---------|-----------|------------------|
|                 9021|      124|tour         |t01        |          |TABLE    |IX       |GRANTED    |                  |
|                 9021|      124|tour         |t01        |num       |RECORD   |X,GAP    |GRANTED    |30, 0x000000000503|

说明：

1. 表中不存在21的数据，所以不会产生任何行记录锁，但是还是会存在gap锁。
2. 此时锁定的区间还是为[20,30)，所以其他事务不能向这个区间插入任何数据，但是可以随意删改，没有限制。

验证：验证的sql就不放了，不然篇幅太长，影响阅读。

- 小于

Session A:

```SQL
ROLLBACK;
BEGIN;
SELECT * FROM tour.t01 t WHERE t.num < 18 FOR UPDATE;
```

查询锁

|ENGINE_TRANSACTION_ID|THREAD_ID|OBJECT_SCHEMA|OBJECT_NAME|INDEX_NAME     |LOCK_TYPE|LOCK_MODE    |LOCK_STATUS|LOCK_DATA         |
|---------------------|---------|-------------|-----------|---------------|---------|-------------|-----------|------------------|
|                 9120|      124|tour         |t01        |               |TABLE    |IX           |GRANTED    |                  |
|                 9120|      124|tour         |t01        |num            |RECORD   |X            |GRANTED    |-3, 0x000000000537|
|                 9120|      124|tour         |t01        |num            |RECORD   |X            |GRANTED    |10, 0x000000000538|
|                 9120|      124|tour         |t01        |num            |RECORD   |X            |GRANTED    |15, 0x000000000539|
|                 9120|      124|tour         |t01        |num            |RECORD   |X            |GRANTED    |20, 0x00000000053A|
|                 9120|      124|tour         |t01        |GEN_CLUST_INDEX|RECORD   |X,REC_NOT_GAP|GRANTED    |0x000000000537    |
|                 9120|      124|tour         |t01        |GEN_CLUST_INDEX|RECORD   |X,REC_NOT_GAP|GRANTED    |0x000000000538    |
|                 9120|      124|tour         |t01        |GEN_CLUST_INDEX|RECORD   |X,REC_NOT_GAP|GRANTED    |0x000000000539    |
|                 9120|      124|tour         |t01        |GEN_CLUST_INDEX|RECORD   |X,REC_NOT_GAP|GRANTED    |0x00000000053A    |

说明：

1. 从上可以看有四条X行锁，那么显然-3，10，15，20这四条记录在其他事务里不能进行不兼容X锁的操作。
2. 此时锁定的区间为(-∞,-3],(3,10],(10,15],(15,20]，即其他事务不能向这四个区间里插入数据，但是依然可以进行删除操作。

验证：验证的sql就不放了，不然篇幅太长，影响阅读。

- 大于

Session A:

```SQL
ROLLBACK;
BEGIN;
SELECT * FROM tour.t01 t WHERE t.num > 28 FOR UPDATE;
```

查询锁

|ENGINE_TRANSACTION_ID|THREAD_ID|OBJECT_SCHEMA|OBJECT_NAME|INDEX_NAME     |LOCK_TYPE|LOCK_MODE    |LOCK_STATUS|LOCK_DATA             |
|---------------------|---------|-------------|-----------|---------------|---------|-------------|-----------|----------------------|
|                 9126|      124|tour         |t01        |               |TABLE    |IX           |GRANTED    |                      |
|                 9126|      124|tour         |t01        |num            |RECORD   |X            |GRANTED    |supremum pseudo-record|
|                 9126|      124|tour         |t01        |num            |RECORD   |X            |GRANTED    |30, 0x00000000053B    |
|                 9126|      124|tour         |t01        |num            |RECORD   |X            |GRANTED    |70, 0x00000000053C    |
|                 9126|      124|tour         |t01        |GEN_CLUST_INDEX|RECORD   |X,REC_NOT_GAP|GRANTED    |0x00000000053B        |
|                 9126|      124|tour         |t01        |GEN_CLUST_INDEX|RECORD   |X,REC_NOT_GAP|GRANTED    |0x00000000053C        |

说明：

1. 从上可以看到有两条X行记录锁，所以其他事务不能对30和70进行不兼容X锁的操作。
2. 此时锁定区间为[20,30),[30,70),[70,+∞)，其他事务不能向这个区间插入数据。

验证：验证的sql就不放了，不然篇幅太长，影响阅读。


- between

Session A:

```SQL
ROLLBACK;
BEGIN;
SELECT * FROM tour.t01 t WHERE t.num BETWEEN 13 AND 28 FOR UPDATE;
```

查询锁

|ENGINE_TRANSACTION_ID|THREAD_ID|OBJECT_SCHEMA|OBJECT_NAME|INDEX_NAME     |LOCK_TYPE|LOCK_MODE    |LOCK_STATUS|LOCK_DATA         |
|---------------------|---------|-------------|-----------|---------------|---------|-------------|-----------|------------------|
|                 9137|      124|tour         |t01        |               |TABLE    |IX           |GRANTED    |                  |
|                 9137|      124|tour         |t01        |num            |RECORD   |X            |GRANTED    |15, 0x000000000539|
|                 9137|      124|tour         |t01        |num            |RECORD   |X            |GRANTED    |20, 0x00000000053A|
|                 9137|      124|tour         |t01        |num            |RECORD   |X            |GRANTED    |30, 0x00000000053B|
|                 9137|      124|tour         |t01        |GEN_CLUST_INDEX|RECORD   |X,REC_NOT_GAP|GRANTED    |0x000000000539    |
|                 9137|      124|tour         |t01        |GEN_CLUST_INDEX|RECORD   |X,REC_NOT_GAP|GRANTED    |0x00000000053A    |
|                 9137|      124|tour         |t01        |GEN_CLUST_INDEX|RECORD   |X,REC_NOT_GAP|GRANTED    |0x00000000053B    |

说明：

1. 从上可以看到有15，20，30这三条X行记录锁，所以其他事务不能对这三条记录进行不兼容X锁的操作。
2. 此时锁定区间为[10,15),[15,20),[20,30)，其他事务不能向这个区间插入数据。

验证：验证的sql就不放了，不然篇幅太长，影响阅读。

##### 1.3.2.3 无索引

我们在测试下无索引情况下的间隙锁

Session A:

```SQL
ROLLBACK;
DROP INDEX num ON tour.t01;
BEGIN;
SELECT * FROM tour.t01 t WHERE t.num BETWEEN 13 AND 28 FOR UPDATE;
```

查询锁

|ENGINE_TRANSACTION_ID|THREAD_ID|OBJECT_SCHEMA|OBJECT_NAME|INDEX_NAME     |LOCK_TYPE|LOCK_MODE|LOCK_STATUS|LOCK_DATA             |
|---------------------|---------|-------------|-----------|---------------|---------|---------|-----------|----------------------|
|                 9160|      124|tour         |t01        |               |TABLE    |IX       |GRANTED    |                      |
|                 9160|      124|tour         |t01        |GEN_CLUST_INDEX|RECORD   |X        |GRANTED    |supremum pseudo-record|
|                 9160|      124|tour         |t01        |GEN_CLUST_INDEX|RECORD   |X        |GRANTED    |0x000000000537        |
|                 9160|      124|tour         |t01        |GEN_CLUST_INDEX|RECORD   |X        |GRANTED    |0x000000000538        |
|                 9160|      124|tour         |t01        |GEN_CLUST_INDEX|RECORD   |X        |GRANTED    |0x000000000539        |
|                 9160|      124|tour         |t01        |GEN_CLUST_INDEX|RECORD   |X        |GRANTED    |0x00000000053A        |
|                 9160|      124|tour         |t01        |GEN_CLUST_INDEX|RECORD   |X        |GRANTED    |0x00000000053B        |
|                 9160|      124|tour         |t01        |GEN_CLUST_INDEX|RECORD   |X        |GRANTED    |0x00000000053C        |

无索引的情况就很严重了，直接锁全表，所有增删改都不行，哪怕是share mode read都不行。

##### 1.3.2.4 事务

间隙锁只会在高于READ COMMITTED级别中的事务出现。修改事务级别READ COMMITTED后可以发现没有任何间隙锁。

```SQL
ROLLBACK;
CREATE INDEX num ON tour.t01(num);
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
BEGIN;
SELECT * FROM tour.t01 t WHERE t.num BETWEEN 13 AND 28 FOR UPDATE;
```

查询锁

|ENGINE_TRANSACTION_ID|THREAD_ID|OBJECT_SCHEMA|OBJECT_NAME|INDEX_NAME     |LOCK_TYPE|LOCK_MODE    |LOCK_STATUS|LOCK_DATA         |
|---------------------|---------|-------------|-----------|---------------|---------|-------------|-----------|------------------|
|                 9190|      124|tour         |t01        |               |TABLE    |IX           |GRANTED    |                  |
|                 9190|      124|tour         |t01        |num            |RECORD   |X,REC_NOT_GAP|GRANTED    |15, 0x000000000539|
|                 9190|      124|tour         |t01        |num            |RECORD   |X,REC_NOT_GAP|GRANTED    |20, 0x00000000053A|
|                 9190|      124|tour         |t01        |GEN_CLUST_INDEX|RECORD   |X,REC_NOT_GAP|GRANTED    |0x000000000539    |
|                 9190|      124|tour         |t01        |GEN_CLUST_INDEX|RECORD   |X,REC_NOT_GAP|GRANTED    |0x00000000053A    |

很显然此时没有了间隙锁，隐藏主键和二级索引都是X,REC_NOT_GAP，除了这些符合条件的给定记录不能在其他事务里进行不兼容X的操作，其他的插入都不会被阻塞。

#### 1.3.3 总结

1. 间隙锁只会在高于READ COMMITTED级别中的事务出现。修改事务级别READ COMMITTED后可以禁用间隙锁。
2. 查询主键或唯一键上的一条已有记录不会产生间隙锁，只会产生该条记录的记录锁X,REC_NOT_GAP。
3. 在非唯一二级索引上的更新操作或锁定读的方式会产生间隙锁，这个间隙锁会阻塞其他事务向这个间隙内的数据插入操作，但不会阻塞更新和删除操作。
4. 索引值会组成连续的间隙，在索引记录上查找给定记录时，InnoDB会在第一个不满足查询条件的记录上加gap lock，防止新的满足条件的记录插入。
5. 区间的开闭情况和查询的条件有关，需要实际分析。

### 1.4 自增与锁

很多应用系统经常会使用自增列作为主键，这里涉及到一种自增锁技术。自增的工作模式主要由参数 `innodb_autoinc_lock_mode` 控制，具体细节可参考官网。

### 1.5 外键与锁

#### 1.5.1 本质

外键是保证数据完整性的一种约束，InnoDB在创建外键时会自动对外键列添加索引，但是对于被引用的表的列还是需要手工加上索引的。

向外键所在的表中插入数据时会先检查外表中是否存在该外键值，如果不存在，则直接会报错，如果存在，则会对外表中的引用列加S锁。

```SQL
CREATE TABLE `t_order` (
    `order_no` CHAR(10) NOT NULL COLLATE 'utf8mb4_general_ci',
    PRIMARY KEY (`order_no`)
)
COLLATE='utf8mb4_general_ci'
ENGINE=InnoDB
;

CREATE TABLE `t_order_details` (
    `order_no` CHAR(10) NOT NULL COLLATE 'utf8mb4_general_ci',
    `item_code` CHAR(8) NULL DEFAULT NULL COLLATE 'utf8mb4_general_ci',
    `item_quantity` INT(10) UNSIGNED NULL DEFAULT NULL,
    CONSTRAINT FOREIGN KEY (`order_no`) REFERENCES `t_order` (`order_no`)
)
COLLATE='utf8mb4_general_ci'
ENGINE=InnoDB
;

INSERT INTO t_order(order_no)VALUES('0000000001','0000000002','0000000003');
```

Session A:

```SQL
BEGIN;
INSERT INTO t_order_details(order_no,item_code,item_quantity) VALUES('0000000001','0231A001',100);
```

查询表锁

|ENGINE_TRANSACTION_ID|THREAD_ID|OBJECT_SCHEMA|OBJECT_NAME    |INDEX_NAME|LOCK_TYPE|LOCK_MODE    |LOCK_STATUS|LOCK_DATA   |
|---------------------|---------|-------------|---------------|----------|---------|-------------|-----------|------------|
|                 9766|       50|tour         |t_order_details|          |TABLE    |IX           |GRANTED    |            |
|                 9766|       50|tour         |t_order        |          |TABLE    |IS           |GRANTED    |            |
|                 9766|       50|tour         |t_order        |PRIMARY   |RECORD   |S,REC_NOT_GAP|GRANTED    |'0000000001'|

可以明显看到对外表中的0000000001记录加上了S锁，那么此时是不能对该条记录进行不兼容S锁的操作的，例如删改等操作，这就保证的数据的完整性。

#### 1.5.2 外键工作的几种关联方式

创建外键时可以分别为外表上的delete和update操作指定关联操作：

- RESTRICT: 存在被依赖记录时不允许操作
- CASCADE: 级联操作
- SET NULL: 为依赖记录设置NULL
- NO ACTION: 同RESTRICT
- SET DEFAULT:为依赖记录设置默认值

细节可参考[https://dev.mysql.com/doc/refman/8.0/en/create-table-foreign-keys.html](https://dev.mysql.com/doc/refman/8.0/en/create-table-foreign-keys.html)

### 1.7 锁的转换与升级

InnoDB和Oracle一致，都不存在锁的升级。转换的话可以由next-key转换为record或gap体现。
