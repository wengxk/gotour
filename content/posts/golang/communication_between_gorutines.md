---
date: 2019-08-01
title: Communication between Gorutines
type:
- post 
- posts
categories:
- golang
---

# GO协程同步通信机制

## 同步原语与锁机制

### 介绍

在GO中，锁机制是最基础的并发控制手段，其他上层的并发安全对象，诸如Channel和Context等内部也都是利用锁来实现。

GO中锁基本原语由 `sync` 包提供，主要包括 `Mutex`、 `RWMutex`、 `Map`、 `Pool`、 `Cond`、 `Once`、 `WaitGroup`。
