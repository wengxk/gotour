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

留后介绍，日后再说

## 内存结构



## 后台线程
