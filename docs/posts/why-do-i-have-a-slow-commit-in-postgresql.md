---
date: 2024-10-18
categories:
  - Commit
---

# 为什么我在 PostgreSQL 中的 COMMIT 很慢？

有时，我们的某个客户查看数据库中最耗时的语句（使用 `pg_stat_statements` 或 `pgBadger`），并发现 `COMMIT` 排名靠前。通常，`COMMIT` 是 PostgreSQL 中非常快的语句，因此值得研究。在本文中，我将探讨 `COMMIT` 缓慢的可能原因并讨论您可以采取的措施。

<!-- more -->

## PostgreSQL 中的基本 `COMMIT` 活动

缓慢的 `COMMIT` 是一个令人惊讶的观察，因为在 PostgreSQL 中提交事务是一个非常简单的活动。大多数情况下，`COMMIT` 要做的就是：

- 将提交日志中事务的两位设置为 `TRANSACTION_STATUS_COMMITTED (0b01)`（保存在 `pg_xact` 中）。
- 如果 `track_commit_timestamp` 设置为 `on`，则记录提交时间戳（保存在 `pg_commit_ts` 中）。
- 将预写日志 (WAL)（保存在 `pg_wal` 中）刷新到磁盘，除非 `synchronous_commit` 设置为 `off`。

请注意，由于 PostgreSQL 的多版本架构，`COMMIT` 和 `ROLLBACK` 通常都是非常快的操作：它们都不需要触碰表，它们只需要在提交日志中注册事务的状态。

## `COMMIT` 缓慢的最常见原因：磁盘问题

从上面可以清楚地看出，导致速度缓慢的一个潜在原因是磁盘 I/O。毕竟，将 WAL 刷新到磁盘会导致 I/O 请求。因此，您应该首先检查磁盘是否有问题或负载过大：

- 在 Linux 上，您可以使用 `vmstat 1` 或 `sar -p 1` 等命令来测量等待 I/O 的 CPU 时间百分比（`vmstat` 中的 `wa` 和 `sar` 中的 `%iowait`）。
- 使用 NAS 时，您应该检查 TCP 网络是否过载。
- 如果存储是共享 SAN 或 NAS，则磁盘可能会与其他机器共享，您应该检查存储系统上是否存在争用。
- 磁盘故障、其他硬件问题或操作系统问题可能会导致间歇性性能问题。检查内核日志以获取消息。

如果我们可以排除磁盘问题是导致 `COMMIT` 缓慢的原因，我们就必须进行进一步调查。

## 阅读源码以了解更多信息

如果我们想了解事务 `COMMIT` 期间发生了什么，最简单的方法就是阅读源代码。许多人没有意识到开源的真正力量：您不必猜测或购买软件供应商的支持，而是可以亲自见证。PostgreSQL 源代码写得很好并且有充足的文档，许多部分不需要专家 C 技能就可以理解。相关代码在 [`src/backend/access/transam/xact.c`](https://git.postgresql.org/gitweb/?p=postgresql.git;a=blob;f=src/backend/access/transam/xact.c) 文件中的 `CommitTransaction()` 函数中。

在特殊情况下，`COMMIT` 速度慢的原因有多种，比如许多并发 `NOTIFY` 语句的争用（您可以从 [`pg_locks`](https://www.postgresql.org/docs/current/view-pg-locks.html) 中数据库 `0` 上的锁来诊断这种情况）。然而，罪魁祸首通常是下面描述的三种情况之一。

## 延迟约束和触发器导致 `COMMIT` 缓慢

通常，PostgreSQL 将检查约束作为修改受约束表的语句的一部分。通过延迟约束 `deferred constraints`，PostgreSQL 会在事务结束时执行约束检查。一个用例是将数据插入具有循环外键约束的表中：

``` sql
CREATE TABLE department (
  department_id bigint PRIMARY KEY,
  name text NOT NULL,
  manager bigint NOT NULL
);
 
CREATE TABLE employee (
  employee_id bigint PRIMARY KEY,
  name text NOT NULL,
  department_id bigint
    REFERENCES department NOT NULL
);
 
-- deferred foreign key
ALTER TABLE department
  ADD FOREIGN KEY (manager) REFERENCES employee
    DEFERRABLE INITIALLY DEFERRED;
```

延迟外键可以轻松创建新部门：

``` sql
BEGIN;

-- won't raise a foreign key violation yet
INSERT INTO department (department_id, name, manager)
VALUES (12, 'Flower Picking', 123);

INSERT INTO employee (employee_id, name, department_id)
VALUES (123, 'John Wurzelrupfer', 12);

-- deferred constraint is valid now
COMMIT;
```

由于 PostgreSQL 在提交时检查延迟约束，它们可能会减慢 `COMMIT` 处理速度。通常，检查约束非常快——它是一个索引查找。然而，许多这样的检查会在较大的事务中累积，并且集体执行时间会大大减慢 `COMMIT` 的速度。

PostgreSQL 还具有可延迟的约束触发器 `constraint triggers`，并且可以像延迟约束一样减慢 `COMMIT` 的速度。有关约束触发器的用例的进一步讨论，请参阅[本文](https://www.cybertec-postgresql.com/en/triggers-to-enforce-constraints/)。

如果您能确定延迟约束或触发器是导致 `COMMIT` 缓慢的原因，那么可能就没问题，无需担心。毕竟，您必须在某个时候检查这些约束。

## 游标保持导致 `COMMIT` 缓慢

游标 `cursor` 允许客户端分块获取查询结果集，这可以简化处理并避免客户端内存不足。但是，常规游标只能存在于数据库事务的上下文中。因此，当涉及用户交互时，您不能使用游标：游标持有的快照会阻碍 `VACUUM` 的进度并导致表膨胀和更严重的问题。此外，在事务持续期间持有的 `ACCESS SHARE` 锁将导致并发 `ALTER TABLE` 或 `TRUNCATE` 出现问题。

为了避免受限于事务，可以使用 `WITH HOLD` 游标。这样的游标可以比创建它的事务存活更长时间，例如，对于[实现分页很有用](https://www.cybertec-postgresql.com/en/pagination-problem-total-result-count/#paginate-cursor)。PostgreSQL 通过在**提交时具体化结果集**来实现 `WITH HOLD` 游标。如果游标后面的查询很昂贵，那么 `COMMIT` 可能会变得非常慢。另外，完成后一定不要忘记关闭这些游标，否则物化结果集将占用服务器资源，直到数据库会话结束。

如果您可以将 `WITH HOLD` 游标确定为 `COMMIT` 缓慢的原因，那么您可以通过调整游标定义中的查询来提高运行速度，从而改善这种情况。由于 PostgreSQL 没有优化游标中的查询来加快计算完整结果集的速度，因此有时将 [`cursor_tuple_fraction`](https://www.postgresql.org/docs/current/runtime-config-query.html#GUC-CURSOR-TUPLE-FRACTION) 设置为 `1.0` 可以帮助加快 `COMMIT` 的处理。

## 同步复制导致 `COMMIT` 缓慢

[物理流复制](https://www.postgresql.org/docs/current/warm-standby.html#STREAMING-REPLICATION)和[逻辑复制](https://www.postgresql.org/docs/current/logical-replication.html)默认都是异步的。如果您使用复制来实现高可用性，并且不想冒丢失已提交事务的风险，则可以使用同步复制。使用同步复制，`COMMIT` 时的操作顺序为：

- 将 `COMMIT` 记录写入 WAL 并刷新（这是实际的持久提交）。
- 等待同步备用数据库报告已获取所有 WAL 信息。
- 使事务在主服务器上可见。
- 向客户报告成功。

如果主服务器和同步备服务器之间的网络延迟较高，则 `COMMIT` 将需要很长时间。您可以通过检查 [`pg_stat_activity`](https://www.postgresql.org/docs/current/monitoring-stats.html#MONITORING-PG-STAT-ACTIVITY-VIEW) 来诊断是否存在频繁或长时间的 `SyncRep` 等待事件。通常，您只想在网络延迟较低的机器（即物理上靠近的机器）之间使用同步复制。

## 第三方扩展导致 `COMMIT` 缓慢

PostgreSQL 最出色的特性之一是它的可扩展性。它允许您编写与 PostgreSQL 核心交互的扩展，而无需修改服务器代码。第三方代码可以“挂载”到 PostgreSQL 中来修改其行为。除了许多其他选项之外，您还可以使用 C 函数 `RegisterXactCallback()` 来注册 PostgreSQL 在 `COMMIT` 时执行的回调。因此，如果您寻找 `COMMIT` 缓慢的原因，您还应该查看数据库中安装的扩展。例如，在远程数据源实现事务处理的外部数据包装器可能希望在 PostgreSQL 提交本地事务时提交远程事务。然后，远程端的高网络延迟或缓慢的事务处理将会减慢 PostgreSQL 的 `COMMIT` 速度。

## 总结

通过查看源代码，我们可以很容易地找到导致 `COMMIT` 缓慢的可能原因：除了磁盘问题的明显原因之外，您还应该考虑延迟约束、`WITH HOLD` 游标和同步复制等可能的原因。

> 作者：Laurenz Albe<br>
> 原文：https://www.cybertec-postgresql.com/en/why-do-i-have-a-slow-commit-in-postgresql/
