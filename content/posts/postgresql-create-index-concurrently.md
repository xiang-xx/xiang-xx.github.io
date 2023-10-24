---
date: 2023-10-24T21:31:55+08:00
title: "PostgreSQL 在线创建索引卡住的问题"
description: ""
tags: ["PostgreSQL"]
series: []
---

摘要：本文通过一个 PostgreSQL 在线创建索引卡住的问题，介绍 PG 在线创建索引的步骤，以及创建索引卡住时的解决办法。

# 问题分析及解决方案

近期观察数据库性能时，发现有个新业务逻辑使用了未带索引的查询，而且查询频率较高，造成了 CPU、IO 使用率升高。查询条件里使用了一个 update_time 字段，理论上只需要给此字段加上索引，就能大大增加查询效率。

由于是生产环境在使用的核心表，而且数据量已经有百万级别，所以计划采用 PostgreSQL 的在线创建索引方案。`CREATE INDEX CONCURRENTLY "索引名称" on "public"."表名" ("update_time" ASC);` ，此语句一执行便一直阻塞了，期间没有管它，等了好久也没有反应。按理说才百万级别的数据不可能这么慢。

查看数据库负载情况，发现新增了 Lock:virtualxid 锁的负载开销。virtualxid 锁很特殊，它是每个事务始终持有的、自己的虚拟事务 ID 上的独占锁。当事务运行时，任何其他事务都无法获取它。这样做的目的是允许一个事务等待，直到另一个事务使用 PostgreSQL 的锁定机制提交或回滚，并且在内部使用。

PG 并发创建索引的逻辑[^1]
> 在并发索引构建中，索引实际上在一个事务中作为“无效”索引输入到系统目录中，然后在另外两个事务中发生两次表扫描。在每次表扫描之前，索引构建必须等待已修改表的现有事务终止。在第二次扫描之后，索引构建必须等待任何在第二次扫描之前具有快照的事务终止，包括在其他表上构建并发索引的任何阶段使用的事务（如果涉及的索引是部分索引或具有不是简单列引用的列。然后最后索引可以被标记为“有效”并准备好使用，并且CREATE INDEX命令终止。然而，即便如此，索引也可能无法立即用于查询：在最坏的情况下，只要在索引构建开始之前存在事务，就无法使用索引。

基本可以判断为：创建索引的事务被 virtualxid 锁阻塞了。按照这篇文章[^2]的方案，查询阻塞的事务并停止它们。

```sql
-- 找到被锁的创建索引事务
select virtualxid,pid,mode,granted from pg_locks where granted is not true; 

-- 根据 virtualxid 查询此锁对应的事务列表，这种场景下会有两条，一条使被卡住的创建索引事务，另一条是 virtualxid 对应的事务
select virtualxid,pid,mode,granted from pg_locks where virtualxid='33/500';

-- 查询 virtualxid 对应的事务的详情
select * from pg_stat_activity where pid=11111;  -- eg. pid=11111 

-- 如果此事务可以被安全关闭，则执行
select pg_cancel_backend(11111);

-- 有时候 pg_cancel_backend 不一定能关掉，可以使用 pg_terminate_backend
select pg_terminate_backend(11111);
```

可能你关掉一个 pid 后，发现事务还是被锁住，只是换了个 virtualxid，那么你可以按照 sql 语句从 pg_stat_activity 里一次性全部查出来，批量关掉。

# 此案例问题根因

正常来说，事务肯定会执行完，所以按照 PG 并发创建索引的逻辑，不应该存在事务阻塞这么久。但凡出现长时间阻塞，那可能是业务程序未正确关闭事务引起的 bug。

在我这个场景里，阻塞的事务语句是一条普通的查询，session wait 信息是 `ClientRead`，即等待事务的下一条语句，`state` 是 `idle in transaction`，这种场景通常是因为没有正确关闭事务，导致事务一直存在。

有两种解决办法[^3]：
1. 找出系统中未关闭事务的程序，修掉 bug
2. 通过设置 `idle_in_transaction_session_timeout`，让长时间未关闭的事务自动回滚；这可以解决事务长期持有锁的问题，但可能造成应用程序的 bug


[^1]: [SQL-CREATEINDEX-CONCURRENTLY](https://www.postgresql.org/docs/devel/sql-createindex.html#SQL-CREATEINDEX-CONCURRENTLY)

[^2]: [PostgreSQL CREATE INDEX CONCURRENTLY 的原理以及哪些操作可能堵塞索引的创建](https://developer.aliyun.com/article/590359)

[^3]: [Query hanging in ClientRead and blocking all others](https://dba.stackexchange.com/questions/251937/query-hanging-in-clientread-and-blocking-all-others)
