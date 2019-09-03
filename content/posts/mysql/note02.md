---
title: "InnoDB Storage Engine"
date: 2019-08-29
type:
- post
- posts
categories:
- mysql
---

# InnoDB存储引擎

InnoDB整体上采用了类似于Oracle数据库的架构，完整支持事务ACID安全模型，正是得益于InnoDB的众多优秀特性，MySQL被越来越广泛的使用于各种OLTP应用中。

## 介绍

InnoDB引擎架构整体分为内存结构和磁盘结构。

![InnoDB Architecture](/images/mysql/innodb_arch.png)

1. 磁盘结构以一定的逻辑结构和物理结构维护着各种数据，主要包括Undo Log files、Redo Log files、表和索引。
   这是数据库的基本功能，能够持久化数据。

2. 内存结构维护了一块很大的内存池，其中包含不同用途的高速缓冲区：Buffer Pool、Change Buffer、Log Buffer。
   这些高速缓冲区能够有效提高数据库的读写性能，减少与磁盘的IO交互。

3. 不同的工作者以不同的数据承担着不同的职责，区分为不同用途的线程。

## 磁盘结构

TODO

## 内存结构

### Buffer Pool

在内存结构中，Buffer Pool是最主要的一块区域，在一些特定的服务器中，其所占内存空间甚至可以占到全部物理内存的80%。
Buffer Pool最主要的意义就是缓存近期访问过的磁盘数据，主要是索引页，数据页，undo页，自适应哈希索引（如果启用了该功能）、锁和系统元数据等。
因为磁盘IO速率相对于内存IO速率来说很低的，所以InnoDB特地分配了Buffer Pool这样一块大内存，减少重复数据的磁盘IO，提高对热点数据的访问效率。

Buffer Pool可以分成多个实例，由参数 `innodb_buffer_pool_instances` 控制，默认为1，该参数不可动态设置。
使用 `SHOW ENGINE INNODB STATUS;` 可以查看每个Buffer Pool实例的具体状态。
每个Buffer Pool的实例内部都是以访问过的整页数据的方式缓存数据，默认页的大小为16K。

#### Buffer Pool中的LRU算法

 Buffer Pool是链表的结构，采用LRU的算法来维护热点数据，驱逐陈旧数据。Buffer Pool中的LRU在常规LRU算法中加入了midpoint，由参数 `innodb_old_blocks_pct` 控制，默认值为37（差不多3/8），
 即表示37%的Buffer Pool的内存区域保存旧数据（Old  Page List），另外63%（差不多5/8）的区域保存着较新数据（Young Page List）。

![LRU List](/images/mysql/innodb_buffer_pool_lru.png)

 当发出的sql从磁盘中查询出数据，需要放在Buffer Pool中，InnoDB首先会检查有没有空闲页（Free Page List），如果有，选取空闲页
 加入到LRU List的MID位置；如果没有空闲页，InnoDB就会将LRU List中的陈旧数据（Old  Page List）淘汰驱逐，以获得新的空闲页来保存数据。
 当发出的sql可以从Old  Page List获得数据时，这时访问到的Old  Page就会加入到Young Page List中，这个过程称为page made young。

#### Buffer Pool相关的参数配置

[see](https://dev.mysql.com/doc/refman/8.0/en/innodb-buffer-pool.html)

## 后台线程
