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

事务指的是一组逻辑的不可分割的工作单元。  
事务的工作与锁机制息息相关，要深刻理解事务必须先要熟悉锁机制，参考[上篇文章](https://wengxk.netlify.com/2019/note03/)。  
InnoDB存储引擎的事务符合ACID特性，但是在隔离级别上的实现与ANSI/ISO标准有些区别。

## ACID特性

- A: atomicity，原子性
- C: consistency，一致性
- I: isolation，隔离性
- D: durability，持久性

作为一个标准的关系型事务数据库引擎，ACID特性必不可少。

## 事务隔离级别

### Oracle 中的隔离级别

|Isolation Level| Dirty Read| Nonrepeatable Read| Phantom Read|
|:---|:---|:---|:---|
|Read uncommitted| Possible| Possible| Possible|
|Read committed| Not possible| Possible| Possible|
|Repeatable read| Not possible| Not possible| Possible|
|Serializable| Not possible| Not possible| Not possible|

### InnoDB 中的隔离级别

|Isolation Level| Dirty Read| Nonrepeatable Read| Phantom Read|
|:---|:---|:---|:---|
|Read uncommitted| Possible| Possible| Possible|
|Read committed| Not possible| Possible| Possible|
|Repeatable read| Not possible| Not possible| Not Possible|
|Serializable| Not possible| Not possible| Not possible|

### 以上两者的区别

InnoDB存储引擎中和Oracle数据库中的事务有着三个基本的区别：

1. InnoDB存储引擎事务默认隔离级别为Repeatable read，由参数 `transaction_isolation` 控制；Oracle数据库事务默认隔离级别为Read committed。
2. Oracle和InnoDB中事务级别对读的影响区别在于InnoDB存储引擎中的Repeatable read不会出现幻读，这是因为next-key locking机制。
3. Oracle中的事务级别不仅会影响读，还会影响写，DML语句中的where条件子查询会受事务级别的影响；
   而InnoDB存储引擎事务隔离级别只会影响读，而不会影响写，其DML语句中的where条件子查询不会受事务级别的影响。
   参考 [https://dev.mysql.com/doc/refman/8.0/en/innodb-deadlocks.html](https://dev.mysql.com/doc/refman/8.0/en/innodb-deadlocks.html)
   中的一段描述：The possibility of deadlocks is not affected by the isolation level, because the isolation level changes the behavior of read operations, while deadlocks occur because of write operations.

验证第3点：

**Oracle**

-----------------------------------------

```SQL
CREATE TABLE employee
(
id NUMBER PRIMARY KEY,
NAME VARCHAR2(100)
);
```

Session A:

```SQL
INSERT INTO employee(id,NAME) VALUES (1,'Tom');
```

Session B:

```SQL
UPDATE employee t SET t.NAME = t.NAME || '_01' WHERE t.id = 1;
```

Session B 直接返回结果：0 rows affected

**InnoDB**

-----------------------------------------

```SQL
CREATE TABLE employee
(
id INT PRIMARY KEY,
NAME VARCHAR(100)
);
```

Session A:

```SQL
BEGIN;
INSERT INTO employee(id,NAME) VALUES (1,'Tom');
```

Session B:

```SQL
BEGIN;
UPDATE employee t SET t.NAME = CONCAT(t.NAME,'_01') WHERE t.id = 1;
```

上面的语句执行时会被阻塞，取消执行，修改事务级别。

```SQL
ROLLBACK;
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
BEGIN;
UPDATE employee t SET t.NAME = CONCAT(t.NAME,'_01') WHERE t.id = 1;
```

执行时同样会被阻塞。

以上两个示例说明了InnoDB存储引擎事务隔离级别不会影响写，其DML语句中的where条件子查询不会受事务级别的影响，
或者说是DML语句中的where条件子查询都是Read uncommitted级别的。

## 事务的实现

## 锁定一致性读

## 非锁定一致性读
