---
date: 2025-01-24
slug: anatomy-of-locks-reduce
categories:
  - Locks
---

# 表级锁剖析：减少锁定影响

并非所有操作都需要相同级别的锁定，PostgreSQL 提供了工具和技术来最大限度地减少锁定的影响。

我已经开始在博客上撰写有关 [PostgreSQL 中表级锁的剖析](https://pgroll.com/posts/anatomy-of-table-level-locks-in-postgresql)。在第一篇[文章](anatomy-of-table-level-locks-in-postgresql.md)中，我们讨论了数据库系统使用锁定机制的原因，以及 PostgresQL 如何利用 MVCC 来避免大多数并发问题，从而减少锁定的必要性。然后，我们讨论了 DDL 锁，并解释了 PostgreSQL 锁队列的工作远离。

在这篇后续文章中，我们将讨论锁争用，以探索减少生产系统中锁定影响的方法，以减少与 DDL 更改相关的潜在停机风险。

<!-- more -->

## 锁争用

当多个事务竞争对同一数据库资源（如表或行）的访问权时，就会发生锁争用，并且至少一个事务必须等待，因为它需要的锁与其他事务持有的锁冲突。在 PostgreSQL 中，这种情况通常发生在需要排他锁的模式修改（DDL 操作）期间，或者在同一行上进行大量并发 DML 操作期间。当发生争用时，事务会形成一个队列，等待轮到自己获取所需的锁。较高的锁争用可能会导致吞吐量降低、延迟增加，严重时还会导致应用程序超时或停机。这在高流量生产系统中尤为重要，因为长时间运行的查询或变更模式的时机不当都可能会导致一连串的等待事务，从而有效地阻塞对关键表的访问。

## 减少锁定影响

在 PostgreSQL 中进行模式更改或运行维护任务时，尽量减少锁定以避免阻塞其他查询并降低性能至关重要。并非所有操作都需要相同级别的锁定，PostgreSQL 提供了工具和技术来最大限度地减少锁定的影响。

### 使用 `CONCURRENTLY` 命令

诸如 `CREATE INDEX CONCURRENTLY` 或 `ALTER TABLE DETACH PARTITION CONCURRENCY` 等命令相较于不使用 `CONCURRENTLY` 的相同命令而已，前者获取的锁限制较少，从而允许其他操作继续进行。但是，这些命令有以下限制：

- 需要更长的时间才能完成。
- 是非事务性的（不能处于事务块中，不能回滚）。
- 需要额外小心处理故障，这可能会留下部分变化（有像 `FINALIZE` 这样的命令来清理或完成工作）。

### 拆分复杂操作

减少锁争用的最有效策略之一是将大型 DDL 操作分解为更小、阻塞更少的步骤。

假设您需要添加一个具有默认值的 `NOT NULL` 列：

```sql
ALTER TABLE mytable ADD COLUMN newcol timestamptz NOT NULL DEFAULT clock_timestamp();
```

上述命令需要 `ACCESS EXCLUSIVE` 锁并将重写整个表。对于大表，这可能会导致大量停机时间，因为它：

- **阻止所有并发访问**（甚至是 `SELECT`）
- 在整个表**重写过程中持有锁**
- 对于大表可能需要几分钟甚至几小时

我们可以将上述的单个繁重操作分解为三个较少阻塞的步骤：

``` sql
ALTER TABLE mytable ADD COLUMN newcol timestamptz DEFAULT clock_timestamp();
UPDATE mytable SET newcol = clock_timestamp() WHERE newcol IS NULL;
ALTER TABLE mytable ALTER COLUMN newcol SET NOT NULL;
```

1. 首先，添加具有默认值的可空列：<br>
	``` sql
	ALTER TABLE mytable ADD COLUMN newcol timestamptz DEFAULT clock_timestamp();
    ```
2. 然后，填充任何 `NULL` 值：
	```
	UPDATE mytable SET newcol = clock_timestamp() WHERE newcol IS NULL;
	```
	实际上，您应该批量进行此更新，请记住任何长时间运行的查询都可能导致问题。
3. 最后，添加 `NOT NULL` 约束：<br>
	```sql
	ALTER TABLE mytable ALTER COLUMN newcol SET NOT NULL;
	```
   
这种方法有几个优点：

- 最初的列添加操作非常快，它只需要短暂的 `ACCESS EXCLUSIVE` 锁
- 数据填充可以使用常规 `ROW EXCLUSIVE` 锁进行，从而允许并发操作
- 如果出现问题，每个步骤都可以回滚

这里有几个好的做法需要注意。为了避免事务长时间运行，对大表进行批量更新总是一个好注意。在我们的零停机、多版本模式变更工具 [pgroll](https://github.com/xataio/pgroll) 中，当我们进行回填时，我们会分批进行，以避免对表中的每一行都进行行锁。该工具允许设置自定义回填批量大小（`--backfill-batch-size`）和延迟（`--backfill-batch-delay`）来控制回填的速度。

这种拆分 DDL 操作的模式可以应用于许多其他模式更改。一般原则是找到方法将需要 `ACCESS EXCLUSIVE` 锁的操作分解为更小的步骤，这些步骤可以使用限制较少的锁或持有锁更短的时间。这会减少强锁的持续时间并防止长时间运行的操作阻塞其他操作。

在执行复杂操作时，密切关注锁争用总是一个好主意。有时您甚至可能希望明确锁定某些内容以防止意外的并发访问。并且不要忘记设置适当的 `lock_timeout` 值以确保事务不会永远等待。

### 单独验证约束

正如我们之前所讨论的，通过调整查询，有多种方法可以实现相同的结果。在上一节中，作为最后一步，我们使用以下查询添加了 `NOT NULL` 约束：

``` sql
ALTER TABLE mytable ALTER COLUMN newcol SET NOT NULL;
```

此命令仍然需要很长时间并且会**阻塞写入**，但它是一种改进，因为与原始命令不同，它**不会阻塞读取**，并且由于它只进行一次表扫描，因此它比原始命令花费的时间更短。

我们可以通过利用 `CHECK` 约束进一步优化这一点：

``` sql
ALTER TABLE mytable ADD CONSTRAINT mytable_newcol_not_null CHECK (newcol IS NOT NULL) NOT VALID;
ALTER TABLE mytable VALIDATE CONSTRAINT mytable_newcol_not_null;
ALTER TABLE mytable ALTER COLUMN newcol SET NOT NULL; --optional
ALTER TABLE mytable DROP CONSTRAINT mytable_newcol_not_null; --optional
```

1. 首先，我们可以添加一个 `NOT VALID` 检查约束：
	``` sql
	ALTER TABLE mytable ADD CONSTRAINT mytable_newcol_not_null CHECK (newcol IS NOT NULL) NOT VALID;
	```
2. 然后验证约束：
	```sql
	ALTER TABLE mytable VALIDATE CONSTRAINT mytable_newcol_not_null;
	```
	`VALIDATE CONSTRAINT` 期间的扫描不会阻止写入。

如果您希望在列上设置 `NOT NULL`（而不是 `CHECK` 约束），那么现在设置它只是元数据操作，因为 PostgreSQL 足够智能，可以识别有效的 `CHECK` 约束（但仍然会执行简短的 `ACCESS EXCLUSIVE` 锁）。

拆分操作需要专业知识，但通常需要尽量减少锁定。尝试找到锁定较少的方法。如果没有普通 SQL 中的重度锁定，有些事情是不可能或很难做到的。

## PostgreSQL 不断进步

PostgreSQL 对 DDL 进行了许多优化，并且随着每个版本的发布而不断改进。让我们看一个与上述示例略微不同的示例：

``` sql
ALTER TABLE mytable ADD COLUMN newcol int NOT NULL DEFAULT 1;
```

此命令仍然需要 `ACCESS EXCLUSIVE` 锁并阻止其他操作。然而，在现代 PostgreSQL 版本中，它执行得非常快，因为 PostgreSQL 认识到常量默认值（如 `1`）可以存储为元数据而无需重写表。

相比之下，旧版 PostgreSQL 会触发同一命令的完整表重写，类似于我们之前的 `DEFAULT clock_timestamp()` 示例。特别是对于大表而言，性能差异非常大。

如果可能的话，请始终运行最新版本的 PostgresQL：

- 新版本包括减少所需锁模式的优化。
- 新增了可用于 DDL 操作的 `CONCURRENTLY` 变体
- 锁定机制本身得到改进
- 以前长时间运行的操作可能会变成仅快速更改元数据

运行旧版本的 PostgreSQL 意味着您可能正在处理新版本中已解决的锁定问题。每个主要版本都会对模式修改操作进行改进，通常使它们更快并且对生产工作负载的干扰更小。

## 结论：平衡锁、性能和正确性

PostgreSQL 的锁系统是确保数据一致性和并发性的强大工具，但它也是争用和瓶颈的根源。通过了解锁的类型、它们的影响以及最小化它们的策略，您可以有效地管理锁并保持数据库的性能。

我将于 1 月 29 日在[布拉格 PostgreSQL 开发者日](https://p2d2.cz/2025/program)和 2 月 2 日在 [FOSDEM PostgreSQL Devroom](https://fosdem.org/2025/schedule/track/postgresql/) 上介绍这个主题。如果您在场，过来打个招呼吧！

<hr>

> 作者：Gulcin Yildirim Jelinek<br>
> 原文：[https://xata.io/blog/anatomy-of-locks-reduce#reducing-locking-impact](https://xata.io/blog/anatomy-of-locks-reduce#reducing-locking-impact)
