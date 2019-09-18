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

## 锁

### 锁机制的整体行为表现

InnoDB存储引擎的锁机制实现类似于Oracle，整体都是基于行锁的实现，其整体行为表现大同小异。
在Oracle官方文档对于锁机制的一些讲解我认为写的非常好，而且对于我的工作也起到了很大的帮助，我认为这些基本原则同样也适用于InnoDB存储引擎。

1. row is locked only when modified by a writer.
2. writer of a row blocks a concurrent writer of the same row.
3. reader never blocks a writer.
4. writer never blocks a reader.

### 锁的分类

本文使用MySQL 8.0.16版本，`information_schema.INNODB_LOCKS` 和 `information_schema.INNODB_LOCK_WAITS` 已经被废弃，
由 `performance_schema.data_locks` 和 `performance_schema.data_lock_waits`替代。

#### 共享锁和排他锁

InnoDB存储引擎实现了两种标准的行级锁：

1. 共享锁：S Lock，允许事务读取一行数据
2. 排他锁：X Lock，允许事务删除和修改一行数据

关于两种锁的兼容性见意向锁总结列表。

#### 意向锁

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

#### 示例

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

`mysql> SELECT * FROM performance_schema.data_locks\G;`

*************************** 1. row ***************************  
               ENGINE: INNODB  
       ENGINE_LOCK_ID: 2578029025072:1107:2577999219240  
ENGINE_TRANSACTION_ID: 7818  
            THREAD_ID: 50  
             EVENT_ID: 419  
        OBJECT_SCHEMA: tour  
          OBJECT_NAME: students  
       PARTITION_NAME: NULL  
    SUBPARTITION_NAME: NULL  
           INDEX_NAME: NULL  
OBJECT_INSTANCE_BEGIN: 2577999219240  
            LOCK_TYPE: TABLE  
            LOCK_MODE: IX  
          LOCK_STATUS: GRANTED  
            LOCK_DATA: NULL  
*************************** 2. row ***************************  
               ENGINE: INNODB  
       ENGINE_LOCK_ID: 2578029025072:50:4:2:2577999216456  
ENGINE_TRANSACTION_ID: 7818  
            THREAD_ID: 50  
             EVENT_ID: 419  
        OBJECT_SCHEMA: tour  
          OBJECT_NAME: students  
       PARTITION_NAME: NULL  
    SUBPARTITION_NAME: NULL  
           INDEX_NAME: PRIMARY  
OBJECT_INSTANCE_BEGIN: 2577999216456  
            LOCK_TYPE: RECORD  
            LOCK_MODE: X,REC_NOT_GAP  
          LOCK_STATUS: GRANTED  
            LOCK_DATA: 1  
2 rows in set (0.00 sec)

`mysql> SELECT * FROM performance_schema.data_lock_waits\G;`

Empty set (0.00 sec)

Session B:

`mysql> SELECT * FROM tour.students t WHERE t.id = 1 LOCK IN SHARE MODE;`

50s会后出现如下错误，因为等待锁定超时，超时时间由参数 `innodb_lock_wait_timeout` 配置。

ERROR 1205 (HY000): Lock wait timeout exceeded; try restarting transaction

`mysql> SELECT * FROM performance_schema.data_locks\G;`

*************************** 1. row ***************************  
               ENGINE: INNODB  
       ENGINE_LOCK_ID: 2578029025072:1107:2577999219240  
ENGINE_TRANSACTION_ID: 7818  
            THREAD_ID: 50  
             EVENT_ID: 419  
        OBJECT_SCHEMA: tour  
          OBJECT_NAME: students  
       PARTITION_NAME: NULL  
    SUBPARTITION_NAME: NULL  
           INDEX_NAME: NULL  
OBJECT_INSTANCE_BEGIN: 2577999219240  
            LOCK_TYPE: TABLE  
            LOCK_MODE: IX  
          LOCK_STATUS: GRANTED  
            LOCK_DATA: NULL  
*************************** 2. row ***************************  
               ENGINE: INNODB  
       ENGINE_LOCK_ID: 2578029025072:50:4:2:2577999216456  
ENGINE_TRANSACTION_ID: 7818  
            THREAD_ID: 50  
             EVENT_ID: 419  
        OBJECT_SCHEMA: tour  
          OBJECT_NAME: students  
       PARTITION_NAME: NULL  
    SUBPARTITION_NAME: NULL  
           INDEX_NAME: PRIMARY  
OBJECT_INSTANCE_BEGIN: 2577999216456  
            LOCK_TYPE: RECORD  
            LOCK_MODE: X,REC_NOT_GAP  
          LOCK_STATUS: GRANTED  
            LOCK_DATA: 1  
*************************** 3. row ***************************  
               ENGINE: INNODB  
       ENGINE_LOCK_ID: 2578029025944:1107:2577999224216  
ENGINE_TRANSACTION_ID: 284053005736600  
            THREAD_ID: 59  
             EVENT_ID: 142  
        OBJECT_SCHEMA: tour  
          OBJECT_NAME: students  
       PARTITION_NAME: NULL  
    SUBPARTITION_NAME: NULL  
           INDEX_NAME: NULL  
OBJECT_INSTANCE_BEGIN: 2577999224216  
            LOCK_TYPE: TABLE  
            LOCK_MODE: IS  
          LOCK_STATUS: GRANTED  
            LOCK_DATA: NULL  
*************************** 4. row ***************************  
               ENGINE: INNODB  
       ENGINE_LOCK_ID: 2578029025944:50:4:2:2577999221432  
ENGINE_TRANSACTION_ID: 284053005736600  
            THREAD_ID: 59  
             EVENT_ID: 142  
        OBJECT_SCHEMA: tour  
          OBJECT_NAME: students  
       PARTITION_NAME: NULL  
    SUBPARTITION_NAME: NULL  
           INDEX_NAME: PRIMARY  
OBJECT_INSTANCE_BEGIN: 2577999221432  
            LOCK_TYPE: RECORD  
            LOCK_MODE: S,REC_NOT_GAP  
          LOCK_STATUS: WAITING  
            LOCK_DATA: 1  
4 rows in set (0.00 sec)

`mysql> SELECT * FROM performance_schema.data_lock_waits\G;`

*************************** 1. row ***************************  
                          ENGINE: INNODB  
       REQUESTING_ENGINE_LOCK_ID: 2578029025944:50:4:2:2577999221432  
REQUESTING_ENGINE_TRANSACTION_ID: 284053005736600  
            REQUESTING_THREAD_ID: 59  
             REQUESTING_EVENT_ID: 142  
REQUESTING_OBJECT_INSTANCE_BEGIN: 2577999221432  
         BLOCKING_ENGINE_LOCK_ID: 2578029025072:50:4:2:2577999216456  
  BLOCKING_ENGINE_TRANSACTION_ID: 7818  
              BLOCKING_THREAD_ID: 50  
               BLOCKING_EVENT_ID: 419  
  BLOCKING_OBJECT_INSTANCE_BEGIN: 2577999216456  
1 row in set (0.00 sec)

Session A:

`mysql> rollback;`

`mysql> SELECT * FROM performance_schema.data_locks\G;`

Empty set (0.00 sec)

`mysql> SELECT * FROM performance_schema.data_lock_waits\G;`

Empty set (0.00 sec)

分析：

在会话A中显示开启一个事务，然后更新表tour.students主键id为1的数据，根据查询结果可以看到
此过程给表tour.students加了IX锁，且给主键id为1的记录加了X锁，此时没有任何资源等待。

另起会话B，以锁定方式读取表tour.students主键id为1的数据，此时可以看到表 `performance_schema.data_locks`多了两条记录，
分别为表tour.students上的IS锁和主键id为1记录上的S锁，但该S锁是 `WAITING` 状态，因为S与X锁不兼容。
表 `performance_schema.data_lock_waits` 有一条数据，BLOCKING_THREAD_ID: 50，REQUESTING_THREAD_ID: 59，
即上面会话A中事务对应的线程ID：50 阻塞了会话B中事务对应的线程ID：59。

若会话A在参数 `innodb_lock_wait_timeout` 设定的时间内既没提交也没回滚，则会话B会报错，等待资源锁超时。

### 行锁的三种分类

- Record Lock：单行记录锁

- Gap Lock：间隙锁，锁定一个区间，但不包括记录本身

- Next-Key Lock：Record Lock + Gap Lock，锁定一个区间，并锁定记录本身



### 非锁定一致性读

### 锁定一致性读

### 自增与锁

### 外键与锁

### 死锁

### 锁的转换与升级

## 事务
