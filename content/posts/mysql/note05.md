---
title: "InnoDB Transaction Management"
date: 2019-10-15
type:
- post
- posts
categories:
- mysql
---

# InnoDB 事务管理

事务指的是一组逻辑的不可分割的工作单元。事务的工作与锁机制息息相关，所以想要深刻理解事务必须先要熟悉锁机制。
InnoDB存储引擎的事务符合ACID特性，但是在隔离级别上的实现与ANSI/ISO标准有些区别。

## ACID特性

- A: atomicity，原子性
- C: consistency，一致性
- I: isolation，隔离性
- D: durability，持久性

作为一个标准的关系型事务数据库引擎，ACID特性必不可少。

## 事务隔离级别

### ANSI/ISO 标准

### InnoDB 中的隔离级别

## 事物的实现

## 锁定一致性读

## 非锁定一致性读
