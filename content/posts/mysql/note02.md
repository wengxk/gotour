---
title: "InnoDB Storage Engine"
date: 2019-08-29
type:
- post
- posts
categories:
- mysql
---

# InnoDB 存储引擎

InnoDB整体上采用了类似于Oracle数据库的架构，完整支持事务ACID安全模型，正是得益于InnoDB的众多优秀特性，MySQL被越来越广泛的使用于各种OLTP应用中。

## 1. 介绍

InnoDB引擎架构整体分为内存结构和磁盘结构。

![InnoDB Architecture](/images/mysql/innodb_arch.png)

1. 磁盘结构以一定的逻辑结构和物理结构维护着各种数据，主要包括Undo Log files、Redo Log files、表和索引。
   这是数据库的基本功能，能够持久化数据。

2. 内存结构维护了一块很大的内存池，其中包含不同用途的高速缓冲区：Buffer Pool、Change Buffer、Log Buffer。
   这些高速缓冲区能够有效提高数据库的读写性能，减少与磁盘的I/O交互。

3. 不同的工作者以不同的数据承担着不同的职责，区分为不同用途的线程。

## 2. 磁盘结构

[see InnoDB On-Disk Structures](https://wengxk.netlify.com/2019/note03/)

## 3. 内存结构

### 3.1 Buffer Pool

在内存结构中，Buffer Pool是最主要的一块区域，在一些特定的服务器中，其所占内存空间甚至可以占到全部物理内存的80%。
Buffer Pool最主要的意义就是缓存近期访问过的磁盘数据，主要是索引页，数据页，undo页，自适应哈希索引（如果启用了该功能）、锁和系统元数据等。
因为磁盘I/O速率相对于内存I/O速率来说很低的，所以InnoDB特地分配了Buffer Pool这样一块大内存，减少重复数据的磁盘I/O，提高对热点数据的访问效率。

Buffer Pool可以分成多个实例，由参数 `innodb_buffer_pool_instances` 控制，默认为1，该参数不可动态设置。
使用 `SHOW ENGINE INNODB STATUS;` 可以查看每个Buffer Pool实例的具体状态。
每个Buffer Pool的实例内部都是以访问过的整页数据的方式缓存数据，默认页的大小为16K。

#### 3.1.1 Buffer Pool 中的LRU算法

 Buffer Pool是链表的结构，采用LRU的算法来维护热点数据，驱逐陈旧数据。Buffer Pool中的LRU在常规LRU算法中加入了midpoint，由参数 `innodb_old_blocks_pct` 控制，默认值为37（差不多3/8），
 即表示37%的Buffer Pool的内存区域保存旧数据（Old  Page List），另外63%（差不多5/8）的区域保存着较新数据（Young Page List）。

![LRU List](/images/mysql/innodb_buffer_pool_lru.png)

 当发出的sql从磁盘中查询出数据，需要放在Buffer Pool中，InnoDB首先会检查有没有空闲页（Free Page List），如果有，选取空闲页
 加入到LRU List的MID位置；如果没有空闲页，InnoDB就会将LRU List中的陈旧数据（Old  Page List）淘汰驱逐，以获得新的空闲页来保存数据。
 当发出的sql可以从Old  Page List获得数据时，这时访问到的Old  Page就会加入到Young Page List中，这个过程称为page made young。

#### 3.1.2 Buffer Pool 相关的参数配置

[see](https://dev.mysql.com/doc/refman/8.0/en/innodb-buffer-pool.html)

### 3.2 Change Buffer

Change Buffer是专门用来缓存DML操作对非唯一二级索引的修改的内存结构。当我们发出DML操作时，可能会对二级索引产生相应的数据修改，
并且当该二级索引是非唯一时，InnoDB就会将修改缓存到Change Buffer中，在一段时间之后会随读操作将修改合并到Buffer Pool中，最后再合并到磁盘中。

参数 `innodb_change_buffering` 可设置具体的缓冲操作类型

|值|数值|说明|
|:---|:---|:---|
|none|0|不缓冲任何操作|
|inserts|1|缓冲insert操作|
|deletes|2|缓冲delete-marking操作|
|changes|3|缓冲insert和delete-marking操作|
|purges|4|缓冲真实的物理删除操作|
|all|5|默认，缓冲上述所有操作|

参数 `innodb_change_buffer_max_size` 可配置Change Buffer占Buffer Pool的比例大小，默认是25，最大值50。

### 3.3 Log Buffer

重做日志缓冲，用来缓冲在事物期间所产生的重做信息，参数 `innodb_log_buffer_size` 可配置重做日志的大小，在一定的机制下才会刷到磁盘中。

在通常情况下，8MB的重做日志缓冲池足以满足绝大部分的应用，因为重做日志在下列三种情况下会将重做日志缓冲中的内容刷新到外部磁盘的重做日志文件中。

- Master Thread每一秒将重做日志缓冲刷新到重做日志文件；
- 每个事务提交时会将重做日志缓冲刷新到重做日志文件；
- 当重做日志缓冲池剩余空间小于1/2时，重做日志缓冲刷新到重做日志文件

## 4. 后台线程

### 4.1 Master Thread

主要的后台线程，承担多种任务，绝大多数的都是I/O相关的操作，例如将缓冲池中的数据异步刷新到磁盘，保证数据的一致性，
包括脏页的刷新、合并插入缓冲、undo页的回收等。Master Thread的工作方式是尽量保证对当前数据库不造成过多负面影响，
例如，当前磁盘每秒读写次数很高，超过一定阈值，则Master Thread会判定当前磁盘I/O压力很高，则不会急着再进行大量I/O操作。

参数 `innodb_io_capacity` 指示了磁盘对InnoDB可提供的最大每秒I/O操作次数。比如以前磁盘的I/O性能很低，则该参数默认值只有100，
而现在很多服务器都是固态硬盘配置，则可以适当增大该参数值，直到或接近实际最佳的值。
与磁盘I/O配置这块还有很多其他重要的参数，可以查找相关资料学习。

### 4.2 I/O Thread

负责响应各种I/O请求，比如DML或DDL的操作，与之相关的主要参数是 `innodb_read_io_threads` 和 `innodb_write_io_threads` ，
分别控制读和写的后台服务线程数，默认值都是4，可接受的区间为1-64，每个线程最多可以处理256个待处理的I/O请求。

### 4.3 Purge Thread

用户在进行DML操作时会产生相应的Undo Log，但是这些Undo Log并不需要一直保存。
在当前事物提交之后，并且这些Undo Log同时也没有任何其他事务使用时，它们就可以被清除掉。
Purge Thread就是负责回收这些无用的Undo Log，参数 `innodb_purge_threads` 可以配置Purge Thread的数量。
在InnoDB 1.1版本之前，purge操作仅能在Master Thread中完成。
而从InnoDB 1.1版本开始，purge操作可以独立到单独的线程中进行，以此来减轻Master Thread的工作，从而提高CPU的使用率以及提升存储引擎的性能。
