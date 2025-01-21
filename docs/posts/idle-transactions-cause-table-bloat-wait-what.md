---
date: 2025-01-17
slug: idle-transactions-cause-table-bloat-wait-what
categories:
  - Idle Transaction
  - Table Bloat
---

# 空闲事务导致表膨胀？等等，这是什么意思？

是的，您没有看错。空闲事务可能会导致严重的表膨胀，而清理进程可能无法解决这一问题。膨胀会导致性能下降，并且可能随着死元组的累积占用越来越多的磁盘空间。

本文深入分析了空闲事务如何导致表膨胀、为什么这会带来问题以及避免这种情况的实用策略。

<!-- more -->

## 什么是表膨胀

当未使用或过时的数据（即死亡元组）在表和索引中积累时，就会发生 PostgreSQL 中的表膨胀。PostgreSQL 使用[多版本并发控制（Multi-Version Concurrency Control, MVCC）](https://stormatics.tech/alis-planet-postgresql/database-concurrency-two-phase-locking-2pl-to-mvcc-part-2)机制来维护数据一致性。每次更新或删除都会创建记录的新版本，同时保留旧版本，直到通过自动清理过程或手动清理将其清理干净。

当这些死亡元组堆积起来并且没有被清理，表和索引的大小增加时，膨胀就会成为问题。表越大，查询速度越慢，导致数据库性能下降和存储成本增加。

## 空闲事务如何导致表膨胀

PostgreSQL 中的空闲事务是连接到数据库但并未主动发出查询的会话。空闲事务有两种主要状态：

1. **Idle**: 连接已经建立，但没有运行事务。
2. **Idle in Transaction**: 事务已经打开（例如，通过 `BEGIN` 语句），但尚未提交或回滚。

!!! note "译者注"

	个人觉得这种说法不太准确，PostgreSQL 中对空闲事务和空闲会话是有明确区分的，对于上面的第一种情况来说不能算是空闲事务。
	
### 1. Autovacuum 阻塞

Autovacuum 是 PostgreSQL 中负责清理死亡元组并回收空间的进程。但是，Autovacuum 无法删除对于打开的事务来说仍然可见的死亡元组。当事务处于 `idle in transaction` 状态时，它保留了数据库的快照，从而防止删除可能仍需要访问的元组。

例如：

- 表更新时产生了死亡元组。
- Autovacuum 被触发但无法删除这些元组，因为空闲事务保留着引用它们的快照。
- 死亡元组仍然存在，导致表膨胀。

### 2. 长期运行的空闲事务

长时间存在的空闲事务易使问题更加严重。这些事务持有锁或快照，阻止 autovacuum 的自动清理流程，甚至阻止 VACUUM 的手动清理流程。这会产生连锁反应：

- 死亡元组不断累积。
- 由于 PostgreSQL 必须扫描膨胀的表，因此查询性能下降。
- 存储使用量不必要的增加。

### 3. 锁争用

空闲事务还可以锁定表，从而阻止其他事务有效地完成插入、更新或删除操作。如果事务被迫重试或者被延迟，这种锁争用可能会导致更多的死亡元组。

### 4. 索引膨胀

死亡元组不仅影响表，还影响索引。当元组被更新或删除时，相应的索引条目被标记为无效，但不会立即被删除。空闲事务可能会延迟这种清理，导致索引膨胀，从而降低查询性能。

## 为什么空闲事务会带来问题

空闲事务除了会导致表膨胀之外，还会导致一系列问题：

1. 资源浪费：
	* 膨胀之后的表会占用更多的磁盘空间和内存，从而增加成本。
	* 较大的表需要更多的 I/O，从而导致查询速度变慢。
2. 性能下降：
	* 随着数据库扫描膨胀的表和索引，查询执行时间会增加。
	* autovacuum 和 analyze 等维护任务需要更长时间才能完成。
3. 增加事务 ID 环绕的风险：
	* 在 PostgreSQL 中，事务 ID（XID）环绕是因为事务 ID 存储为 32 位整数，这意味着它们在大约 20 亿次事务后最终会“回绕”。如果不通过常规 **`VACUUM`** 操作解决此问题，这可能会导致数据损坏，因为旧行会变得不可见或新的事务无法继续。
4. 关键操作受阻：
	* 由于空闲事务持有的锁或快照，手动清理或维护任务可能会失败。

## 如何避免空闲事务并防止表膨胀

幸运的是，通过主动监控、配置和应用程序设计可以避免空闲事务和由此导致的膨胀。以下是一些策略：

### 1. 监控空闲事务

第一步是识别处于 `idle` 和 `idle in transaction` 的会话。使用以下查询来查找它们：

``` sql
SELECT 
    pid, 
    usename AS username, 
    state, 
    state_change, 
    query
FROM 
    pg_stat_activity
WHERE 
    state IN ('idle', 'idle in transaction');
```

- **state**：显示事务是否处于空闲或空闲事务状态。
- **state_change**：指示事务何时进入其当前状态。

使用此信息来识别长时间运行的空闲会话并采取纠正措施。

### 2. 设置空闲事务超时

PostgreSQL 提供了 `idle_in_transaction_session_timeout` 参数来自动终结空闲时间过长的事务。这可以防止长时间运行的空闲事务持有锁和快照。

在 `postgresql.conf` 中全局设置此参数：

```
idle_in_transaction_session_timeout = '5min'
```

或者将其应用于特定角色或数据库：

``` sql
ALTER ROLE my_user SET idle_in_transaction_session_timeout = '5min';
ALTER DATABASE my_database SET idle_in_transaction_session_timeout = '5min';
```

当达到此超时时，PostgreSQL 会自动终止空闲事务并抛出错误。

### 3. 使用连接池

使用 **PgBouncer** 或 **Pgpool-II** 等连接池来管理和限制与数据库的连接数。连接池可确保：

- 连接被有效地重复利用。
- 空闲会话不会不必要地保持打开状态。
- 该应用程序仅在需要时打开事务。

### 4. 优化应用逻辑

大多数空闲事务都是由不良的应用程序设计引起的。确保您的应用程序：

- 完成事务后始终立即发出 `COMMIT` 或 `ROLLBACK`。
- 避免不必要地启动事务（例如，避免在没有立即查询的情况下启动 `BEGIN`）。
- 不使用时关闭数据库连接。

### 5. 优化 Autovacuum

虽然空闲事务会阻止自动清理，但调整自动清理参数可确保更积极地触发清理。请考虑以下调整：

- **autovacuum_vacuum_threshold**：降低该值可以更快地触发清理。
- **autovacuum_vacuum_scale_factor**：减少此值以根据较小百分比的更新行触发清理。这与上述参数一起使用，以确保表格按其大小成比例处理。

例如：

``` sql
ALTER TABLE my_table SET (
    autovacuum_vacuum_threshold = 50,
	autovacuum_vacuum_scale_factor = 0.1
);
```

### 6. 使用监控工具

利用 PostgreSQL 监控视图（如 **`pg_stat_activity`**、**`pg_stat_user_tables`**）或第三方工具（如 **pgAdmin** 或 **pgBadger**）来跟踪空闲事务和表随时间的膨胀。

### 7. 终止有问题的会话

如果需要，您可以手动终止阻止关键操作的空闲事务。使用以下查询终止空闲事务：

``` sql
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'idle in transaction'
AND state_change < NOW() - INTERVAL '10 minutes';
```

## 总结

空闲事务乍一看似乎无害，但它们可能会导致表膨胀、阻塞维护任务并不必要地消耗资源，从而悄悄导致严重的性能问题。通过监控、超时和应用程序优化主动管理空闲事务对于维护高性能 PostgreSQL 数据库至关重要。

通过采用这些最佳实践，您可以防止空闲事务对数据库造成严重破坏，并确保系统干净、高效且可扩展。不要让空闲事务成为 PostgreSQL 性能的无声杀手——立即采取行动，保持数据库的健康和高性能。

<hr>

> 作者：[Umair Shahid](https://stormatics.tech/author/umair)<br>
> 原文：[https://stormatics.tech/blogs/idle-transactions-cause-table-bloat-wait-what](https://stormatics.tech/blogs/idle-transactions-cause-table-bloat-wait-what)

