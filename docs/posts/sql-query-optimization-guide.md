---
date: 2024-12-23
slug: sql-query-optimization-guide
categories:
  - Performance
---

# SQL 查询优化：全面的开发者指南

面向开发人员的 SQL 优化指南。其中包含最佳实践、警告和专业提示，可加快 SQL 查询优化。

当您有一个数据库时，您将如何优化 SQL 性能？要回答这个问题，您需要投入大量的时间和精力来了解工作负载和性能模式、评估性能下降情况并采取纠正措施。但是，您可以实施标准实践来提高性能。本 SQL 优化指南将展示一些适用于几乎所有数据库的最佳实践，并且可以作为优化数据库工作负载的良好起点。

<!-- more -->

## 如何优化 `SELECT` SQL 查询

### 通过了解查询执行计划来优化 `SELECT` SQL 查询

所有现代数据库（例如 MySQL 和 PostgreSQL®）都会根据所涉及的各个表的基数以及可用的辅助数据结构（例如索引或分区）来定义最佳查询执行计划。[MySQL](https://dev.mysql.com/doc/refman/8.0/en/using-explain.html) 和 [PostgreSQL](https://www.postgresql.org/docs/current/sql-explain.html) 都提供了一个名为 `EXPLAIN` 的命令来显示语句的执行计划。从执行计划中，您可以了解表是如何连接的，是否使用索引，是否修剪分区以及可能影响查询性能其他方面。查询计划提示了每个操作的成本，并且可以标记是否未使用索引。

要获取查询的执行计划，您只需要在查询前加上 `EXPLAIN` 即可，如下所示：

``` sql
EXPLAIN SELECT id
FROM orders
WHERE
order_timestamp between '2024-02-01 00:00:00' and '2024-02-03 00:00:00' 
OR status = 'NEW';
```

在此示例中，数据库返回的执行计划展示了 `idx_order_date` 和 `idx_order_status` 两个索引以及两个结果之间的 `BitmapOr` 的使用情况。

```
                                                                                            QUERY PLAN
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
 Bitmap Heap Scan on orders  (cost=687.35..7149.18 rows=60333 width=4)
   Recheck Cond: (((order_timestamp >= '2024-02-01 00:00:00'::timestamp without time zone) AND (order_timestamp <= '2024-02-03 00:00:00'::timestamp without time zone)) OR (status = 'NEW'::text))
   ->  BitmapOr  (cost=687.35..687.35 rows=60333 width=0)
         ->  Bitmap Index Scan on idx_order_date  (cost=0.00..655.75 rows=60333 width=0)
               Index Cond: ((order_timestamp >= '2024-02-01 00:00:00'::timestamp without time zone) AND (order_timestamp <= '2024-02-03 00:00:00'::timestamp without time zone))
         ->  Bitmap Index Scan on idx_order_status  (cost=0.00..1.43 rows=1 width=0)
               Index Cond: (status = 'NEW'::text)
(7 rows)
```

#### 警告

数据库优化器将根据基数估计生成执行计划。这些估计越接近表中的数据，数据库就越能够定义最佳计划。计划也会随着时间而改变，因此，当查询性能突然下降时，计划的改变可能是原因之一。该计划还将取决于表和数据库中创建的索引等其他辅助结构，在下一节中，我们将分析其他结构如何影响性能。

#### 黄金法则

花时间了解查询执行计划以找到潜在的性能瓶颈。使用 `ANALYZE` 命令自动或手动更新统计数据，为数据库提供有关各个表负载的最新信息。

### 通过使用索引优化 `SELECT` SQL 查询

上面定义的几个选项建议使用索引，但您应该如何使用它们呢？在执行过滤、连接和排序时，索引是关键的性能提升器。因此，了解查询模式并创建涵盖正确子句的适当索引至关重要。[PostgreSQL](https://www.postgresql.org/docs/current/indexes.html) 和 [MySQL](https://dev.mysql.com/doc/refman/8.0/en/mysql-indexes.html) 都提供多种索引类型，每种索引类型都有独特的特性并且非常适合某些用例。

![](imgs/sql-query-optimization-guide/index.avif)

#### 警告

正如在 `DELETE` 和 `INSERT` 部分中提到的，在表上添加的任何索引都会减慢写入操作的速度。此外，索引占用磁盘空间并需要维护。因此，请仔细考虑您想要优化哪些工作负载以及哪组索引可以为您带来最佳结果。

#### 黄金法则

与数据库表不同，索引可以被删除并重新创建而不会丢失数据。因此，定期评估指标集及其状态非常重要。您可以依赖数据库系统表检查索引使用情况，如 [PostgreSQL `pg_stat_user_indexes`](https://www.postgresql.org/docs/current/monitoring-stats.html) 或 [MySQL `table_io_waits_summary_by_index_usage`](https://dev.mysql.com/doc/mysql-perfschema-excerpt/8.3/en/performance-schema-table-io-waits-summary-by-index-usage-table.html)，提供有关影响索引查询的最新统计信息。一旦确定了已使用和未使用的索引，请评估在工作负载发生变化的情况下重组它们的必要性。

#### 专业提示

索引可用于单列、多列甚至函数。如果您希望使用 `upper(name)` 函数等来过滤数据，则可以索引该函数的输出以获得更好的性能（函数索引）。

[Aiven AI 数据库优化器](https://aiven.io/solutions/aiven-ai-database-optimizer)可以根据您的数据库工作负载为您提供索引建议。


### 通过改进连接来优化 `SELECT` SQL 查询

在关系数据库中，经常使用连接来选择来自不同表的数据。了解可用的连接类型及其含义对于实现最佳性能至关重要。以下建议将帮助您确定正确的连接类型。

#### 通过选择内连接来优化 `SELECT` SQL 查询

MySQL 和 PostgreSQL 都提供了多种连接类型，允许您精确定义从连接两侧检索的行集。由于各种原因，它们都是有用的，但并非所有它们都具有相同的性能。`INNER JOIN` 仅检索数据集两侧包含的行通常具有最佳性能。另一方面，与 `INNER JOIN` 相比，`LEFT`、`RIGHT` 和 `OUTER` 连接需要执行一些额外的工作，因此仅在真正必要时才应使用。

![](imgs/sql-query-optimization-guide/join.avif)

##### 警告

仔细检查您的查询，有时如下的查询似乎是合法的：

``` sql
SELECT * 
FROM ORDERS LEFT JOIN USERS ON ORDERS.NAME = USERS.NAME
WHERE USERS.NAME IS NOT NULL 
```

上面使用 `LEFT JOIN` 从 `ORDERS` 中检索所有行，然后筛选出具有 `USERS.NAME IS NOT NULL` 的行。因此相当于 `INNER JOIN`。

##### 黄金法则

评估 `JOIN` 语句的确切要求，并分析现有的 `WHERE` 条件。如果不是绝对必要，最好使用 `INNER JOIN`。

##### 专业提示

还请检查是否可以完全避免连接。例如，如果我们连接数据只是为了验证另一个表中是否存在一行，使用 `EXISTS` 的子查询可能比连接快得多。查看[如何加速 `COUNT(DISTINCT)` 博客中的详细示例](https://www.eversql.com/how-to-speed-up-countdistinct/#Speed_up_COUNTDISTINCT_by_avoiding_explodeimplode_patterns_with_DISTINCTGROUP_BY)。

#### 通过对连接使用相同的列类型来优化 `SELECT` SQL 查询

当连接两个表时，确保连接条件中的列属于同一类型。将一个表中的整数 `Id` 列与另一个表中定义为 `VARCHAR` 的另一个 `customerId` 列连接起来，将强制数据库在比较结果之前将每个 `Id` 转换为字符串，从而降低性能。

![](imgs/sql-query-optimization-guide/type-convert.avif)

##### 警告

您无法在查询时更改源字段类型，但您可以暴露数据类型不一致问题并在数据库表中修复它。在分析 `CustomerId` 字段是否可以从 `VARCHAR` 迁移到 `INT` 时，请检查列中的所有值是否确实都是整数。如果某些值不是整数，则可能存在数据质量问题。

##### 专业提示

如有疑问，请选择更紧凑的连接键表示形式。如果您存储的内容可以明确地定义为数字（例如产品代码 `1234-678-234`），则最好使用数字表示，因为它可以：

- 使用更少的磁盘空间
- 获取速度更快
- 由于整数比较比字符串版本更快，因此连接速度更快

然而，要小心那些看起来像数字但行为却不像数字的东西--例如，电话号码 `015555555` 其中前导零很重要。

#### 通过避免使用连接函数来优化 `SELECT` SQL 查询

与上一节类似，避免在连接中使用不必要的函数。函数可以阻止数据库使用性能优化（例如利用索引）。考虑以下查询：

``` sql
SELECT * 
FROM users 
JOIN orders ON UPPER(users.user_name) = orders.user_name
```

以上代码使用函数将 `user_name` 字段转换为大写。然而，这可能表明订单表中的数据质量较差（缺少外键），需要解决。

##### 警告

类似上述的查询可以展示查询时解决的数据质量问题，这只是一个短期解决方案。在设计数据后端系统时，应优先考虑正确处理数据类型和质量约束。

##### 黄金法则

在关系数据库中，表之间的连接应该可以使用键和外键来完成，而无需任何附加功能。如果您发现自己需要使用某个函数，请修复表中的数据质量问题。在某些边缘情况下，使用函数与索引可以帮助加快复杂或冗长数据类型之间的比较。例如，使用连接函数 `UPPER(SUBSTR(users.user_name, 1, 50))` 和同一函数上的索引，仅比较前 50 个字符，可以加速检查两个长字符串之间的相等性。

#### 通过避免连接来优化 `SELECT` SQL 查询

查询可以由不同的人随着时间的推移而构建，并且具有 CTE（公共表表达式）形式的许多连续步骤。因此，可能很难了解数据输入和输出方面的实际需求。大多数情况下，在编写查询时，您可以在稍后添加一个额外的字段“以防万一”。但是，如果该字段来自需要连接的新表，这可能会对性能产生巨大影响。

始终评估查询的严格数据需求，并且仅包含包含此信息的列和表。

![](imgs/sql-query-optimization-guide/avoid-joins.avif)

#### 警告

仔细检查是否需要连接来过滤两个表中存在的行。在上面的例子中，如果订单表中存在未存储在用户表的 `id` 列中的 `user_id`，我们最终可能会得到不正确的结果。

#### 黄金法则

删除不必要的连接。生成更精简的查询来检索整体数据集，然后仅在必要时查找更多信息，这样性能会更高。

#### 专业提示

上面解释的例子只是 `JOIN` 过度使用的一个例子。另一个例子是，我们连接数据只是为了验证另一个表中某一行的存在。在这种情况下，使用 `EXISTS` 的子查询可能比连接快得多。查看[如何加速 COUNT(DISTINCT)](https://www.eversql.com/how-to-speed-up-countdistinct/#Speed_up_COUNTDISTINCT_by_avoiding_explodeimplode_patterns_with_DISTINCTGROUP_BY) 博客中的详细示例。
	
### 通过改进过滤来优化 `SELECT` SQL 查询

分析完 `JOIN` 连接之后，现在是时候评估和改进查询的 `WHERE` 部分了。与上面的部分一样，对过滤语句的细微改变可能会对查询性能产生巨大影响。

#### 通过避免在过滤中使用函数来优化 `SELECT` SQL 查询

在过滤阶段对列使用函数会降低性能。数据库需要在过滤之前将函数应用于数据集。让我们看一个根据时间戳字段进行过滤的简单示例：

``` sql
SELECT count(*) 
FROM orders
WHERE CAST(order_timestamp AS DATE) > '2024-02-01';
```

上述查询在 100,000,000 行数据集上需要执行 01 分 53 秒，因为它需要在应用过滤器之前将 `order_timestamp` 列的数据类型从时间戳更改为日期。但这不是必需的！正如 [Aiven 的 EverSQL](https://www.eversql.com/?utm_medium=organic&utm_source=ext_blog&utm_content=aivensqloptimization) 所建议的那样，如果您为其提供上述查询和表元数据，则可以将其重写为：

``` sql
SELECT count(*) 
FROM orders
WHERE order_timestamp > '2024-02-01 00:00:00';
```

重写的查询使用本机时间戳字段，无需进行强制转换。这种微小变化的结果是，查询现在只需 20 秒即可运行，比原来快近 6 倍。

![](imgs/sql-query-optimization-guide/filter-cast.avif)

##### 警告

并非所有函数都可以避免，因为有些函数可能需要检索列的部分内容（如上面的 `substring` 示例）或重写它。尽管如此，每次您要在过滤器中添加函数时，请考虑使用本机数据类型运算符的其他方法。

##### 黄金法则

将过滤器应用到列时，请尝试改写过滤器格式而不是列格式。上面是一个完美的例子：将过滤器格式从日期 `2024-02-01` 转换为时间戳 `2024-02-01 00:00:00` 使我们能够使用本机时间戳数据格式和运算符。

##### 专业提示

如果您必须使用函数，请考虑以下两点建议：

- 在表达式上创建索引，[PostgreSQL](https://www.postgresql.org/docs/current/indexes-expressional.html) 和 [MySQL](https://dev.mysql.com/doc/refman/8.0/en/create-index.html#create-index-functional-key-parts) 都支持
- 使用数据库的触发器来生成额外的列，并且列已经完成了数据转换

#### 通过改进子查询来优化 `SELECT` SQL 查询

子查询通常用于过滤器中，以检索要作为过滤器应用的值集。一个常见的例子是检索最近活动的用户列表。

``` sql
SELECT *
FROM users
WHERE id IN (
SELECT DISTINCT user_id 
FROM sessions 
WHERE session_date = '2024-02-01');
```

上述查询从 `SESSIONS` 表中检索不同的用户列表，然后过滤 `USERS` 表上的数据。

但是，有几种更高效的方法可以实现相同的结果。例如使用 `EXISTS`：

``` sql
SELECT * 
FROM users
WHERE EXISTS (
SELECT user_id 
FROM sessions 
WHERE user_id = id and session_date = '2024-02-01'
);
```

`EXISTS` 速度更快，因为它不需要从 `SESSION` 表中检索不同用户的列表，而只是验证表中是否存在特定用户的至少一行。

上述用例仅通过更改子查询部分就从 02 分 08 秒的性能缩短到了 18 秒。

##### 警告

子查询中的细微变化可能会在极端情况下产生不同的结果。有关执行和结果集细微差别的示例，请查看在 [PostgreSQL 中实现 `NOT EXISTS` 的 5 种方法](https://www.eversql.com/5-ways-to-implement-not-exists-in-postgresql)。

##### 黄金法则

当需要使用子查询时，花点时间学习和了解有哪些选项以及它们允许您实现什么。很多时候，选项不止一个，而且有些功能可以提供更好的响应时间。

#### 通过对结果集进行分页优化 `SELECT` SQL 查询

当需要显示一长串行时，对从数据库检索的结果集进行分页很有用。[PostgreSQL](https://www.postgresql.org/docs/current/queries-limit.html) 和 [MySQL](https://dev.mysql.com/doc/refman/8.0/en/select.html) 都提供了 `LIMIT` 限制输出结果集的功能，以仅检索一定数量的行，并且可以通过 `OFFSET` 结合排序获取特定偏移范围的结果集。使用 `LIMIT` 和 `OFFSET` 是将发送给客户端的数据最小化为仅需要显示的数据的好方法。

![](imgs/sql-query-optimization-guide/limit-offset.avif)

##### 警告

使用 `LIMIT` 和 `OFFSET` 的缺点是您需要对每个“页面”进行查询并将其发送至数据库并执行。如果总行数与页面大小相差不大，这可能会带来不便。举例来说，如果您以每页 10 行显示结果，但平均有 15 行需要显示，那么一次性检索整个数据集可能会更好。

##### 黄金法则

如果结果集的大小比页面大小大一个数量级，则使用分页可以有效地确保更好的性能，因为只会从数据库中检索可见的数据集。

##### 专业提示

`LIMIT` 和 `OFFSET` 子句是大多数数据库中的默认分页方法。然而，通过在客户端存储当前页面的起始和结束偏移量并将下一页的过滤推送到 SQL 语句的 `WHERE` 子句中，可以实现更有效的分页实现。在 Markus Winand 的 [PostgreSQL 方式分页演示](https://use-the-index-luke.com/blog/2013-07/pagination-done-the-postgresql-way)中以及 [MySQL 更快分页博客](https://www.eversql.com/faster-pagination-in-mysql-why-order-by-with-limit-and-offset-is-slow/)中提供了此实现的几个示例。

#### 通过将过滤器从 `HAVING` 移至 `WHERE` 子句来优化 `SELECT` SQL 查询

运行查询时，`WHERE` 和 `HAVING` 子句中定义的过滤器会在不同时间应用：

- `WHERE` 过滤器将应用于原始数据集，在 `SELECT` 语句中定义的任何数据转换之前
- `HAVING` 过滤器在聚合后应用，因此是在检索、转换和汇总所有行之后应用的

因此，将过滤器移动到 `WHERE` 中应该是一个优先事项，因为它允许您处理较小的数据集。举个例子，如果我们尝试获取具有跟踪 `user_id` 的会话的日期列表（`user_id` 字段不为空，要了解有关差异的更多信息，请查看 [COUNT DISTINCT 博客文章](https://www.eversql.com/how-to-speed-up-countdistinct/)）：

``` sql
SELECT session_date, count(user_id) nr_sessions
FROM sessions
GROUP BY session_date
HAVING count(user_id) > 0;
```

我们可以通过将过滤器推送到 `WHERE` 子句来重写上述查询：

``` sql
SELECT session_date, count(user_id) nr_sessions
FROM sessions
WHERE user_id is not null
GROUP BY session_date;
```

只需更改过滤语句，100,000,000 行数据集的查询性能就从 21 秒缩短到 18 秒。

##### 警告

仅当我们能够确定适用于行级别而不是适用于聚合级别的类似条件时，才可以将过滤器从 `HAVING` 移至 `WHERE` 子句。

##### 黄金法则

始终尝试在 `WHERE` 子句中进行过滤，因为过滤在数据通过 `SELECT` 语句转换/聚合之前应用。

### 通过定义要检索的列来优化 `SELECT` SQL 查询

查询表时，确定需要检索哪些列是关键。`SELECT * FROM TBL` 通常用作检索所有列的快捷方式，然后定义稍后需要显示哪些列，但是这样做时，需要从数据库中检索并传输更多数据，从而影响性能。

![](imgs/sql-query-optimization-guide/select-all-columns.avif)

#### 警告

使用 `SELECT * FROM TBL` 不仅经常会检索比必要更多的数据，而且还容易出现错误。试想一下，添加一个新列，它会立即显示在应用程序中，或者从数据库中删除一个列，从而破坏前端解析。通过谨慎定义要检索的列的列表，您可以准确地陈述查询的数据需求。

#### 黄金法则

每个 `SELECT` 语句应该仅定义完成作业所需的列的列表。

### 使用聚合函数优化 `SELECT` SQL 查询

有时目标是查询表以获取行的总数或某组列的总值。在这种情况下，从数据库中提取所有数据并在其他地方执行计算是一个坏主意。我们可以利用本机数据库聚合功能在数据所在的位置进行计算，移动更少的数据并实现更好的整体性能。

![](imgs/sql-query-optimization-guide/aggregate-function.avif)

如果您尝试计算列内的不同值，我们有一篇专门介绍[如何提高 `COUNT DISTINCT` 查询速度的文章](https://www.eversql.com/how-to-speed-up-countdistinct/)。

#### 警告

虽然数据库聚合函数满足了广泛的需求（参见 [PostgreSQL](https://www.postgresql.org/docs/current/tutorial-agg.html) 和 [MySQL](https://dev.mysql.com/doc/refman/8.0/en/aggregate-functions.html)），但它们可能不包含您需要执行的精确计算。在这种情况下，您需要了解在该级别提取和聚合所需的最小粒度是什么。

#### 黄金法则

当需要执行聚合时，请检查可用的数据库函数。尽管数据库扩展规则之一是聚合操作推向偏向客户端，但通常值得使用数据库服务器提供的聚合函数来减少传输到客户端的数据量。此外，数据库可以实现先进的技术来优化聚合计算。

### 使用窗口函数优化 `SELECT` SQL 查询

[PostgreSQL](https://www.postgresql.org/docs/current/tutorial-window.html) 和 [MySQL](https://dev.mysql.com/doc/refman/8.0/en/window-functions-usage.html) 中提供的窗口函数提供了一种计算聚合值并将其与当前行值进行比较的方法。例如，当需要计算某次销售对整个国家销售额的总体贡献时，它们真的很有用。使用窗口函数可以简化查询 SQL，并利用数据库优化器来决定检索数据所需的最佳操作集。

![](imgs/sql-query-optimization-guide/window-function.avif)

#### 警告

窗口函数不会改变数据集的基数。例如，如果您在 `ORDER` 级别检索数据，窗口函数仍将为每个订单产生一行。评估输出中所需数据的基数，然后决定是否使用窗口或聚合函数。

#### 黄金法则

与上述类似，采用窗口函数可以成为在将数据发送给客户端之前进行预先计算的有效方法。但是生成它们的代码相当冗长。请在完整查询 SQL 优化和代码可读性和可维护性之间确定优先级。

#### 专业提示

窗口函数不仅可用于在某个级别聚合内容，而且还公开了 `ORDER BY` 语句，允许您执行正在运行的计算。

### 使用分区优化 `SELECT` SQL 查询

提高 `SELECT` 性能的另一种技术是将数据分成多个子表（称为分区）。[PostgreSQL](https://www.postgresql.org/docs/current/ddl-partitioning.html) 和 [MySQL](https://dev.mysql.com/doc/refman/8.0/en/partitioning.html) 中提供分区功能，可根据谓词将数据拆分到多个表中。如果您经常需要使用特定过滤器（例如日期）检索数据，则可能需要将其组织在不同的分区中，每天一个，这样大多数查询只会扫描一个分区而不是整个数据集。

![](imgs/sql-query-optimization-guide/partition-table.avif)

#### 警告

尽管分区对于用户来说是“不可见的”（某个用户可以查询原始表并从所有分区中选择数据），但您需要考虑它的管理。具体来说，对于 LIST 分区，当插入新键时，需要预先创建适当的分区，否则插入将失败。

#### 黄金法则

在使用相同键过滤数据集的一致查询模式的情况下，分区非常有用。在没有明确过滤标准的情况下对表进行分区只会增加数据库的负载，使每个查询变得更慢。

### 使用物化视图优化 `SELECT` SQL 查询

[PostgreSQL](https://www.postgresql.org/docs/current/rules-materializedviews.html) 中提供的物化视图是一种通过预先计算结果来加速 `SELECT` 语句的好方法。物化视图与标准视图不同，因为：

- 它们将查询结果存储在表中
- 它们需要更新以提供最新的结果

不过，如果您读取了写入次数很少的密集环境，那么为可供多次读取使用的物化视图刷新付出代价可能是提高查询性能的最佳方法。

![](imgs/sql-query-optimization-guide/materialized-view.avif)

#### 警告

如上所述，物化视图是静态对象，它们不会在每次插入、更新或删除底层表集时自动更新。因此，务必记住，查询物化视图可能会提供与查询原始表不同的结果。

#### 黄金法则

如果您的主要目标是以不检索最新结果为代价获得更好的性能，那么物化视图是一个完美的解决方案。请记住，物化视图会占用磁盘空间，并且其创建或刷新会消耗时间和性能。

#### 专业提示

当复杂查询扫描过去无法更新的数据时，物化视图非常有用。在这种情况下，为旧数据创建物化视图并实时查询源表以获取新数据可以很好地提供最佳性能而无需报告陈旧数据。

## 如何优化 `INSERT` SQL 查询

### 使用专用功能优化 `INSERT` SQL 查询

在数据库中加载数据的标准做法是使用 `INSERT` 语句。然而，一些数据库有专门的工具可以从各种格式（如二进制或 CSV）的源文件直接插入大块数据。

[PostgreSQL `COPY`]() 或 [MySQL `LOAD DATA`]() 等工具允许您直接指向 CSV 或二进制文件，并针对加载大量数据进行了优化，与一系列 `INSERT` 语句相比，通常可以缩短加载时间。

#### 警告

虽然 [PostgreSQL `COPY`]() 或 [MySQL `LOAD DATA`]() 都比在单独的 INSERT 语句中加载每一行更快，但它们都会在一个大事务中加载数据，从而给数据库带来压力。此外，长事务也将成为复制过程中的瓶颈，导致副本重放滞后。再者，如果在发生大事务时进行备份，则恢复将被迫回滚备份中已存储的所有更改，从而使恢复过程变得更慢。

#### 黄金法则

尝试在提取吞吐量和事务大小之间找到最佳点。将海量数据文件分割成较小的块，然后使用 [PostgreSQL `COPY`]() 或 [MySQL `LOAD DATA`]() 加载它可能是一种完美的中间方案，既能提供所需的速度，又不会给数据库增加太大压力。

#### 专业提示

如果源文件中的数据与目标表的格式不匹配，则使用外部脚本正确地重塑源数据可能比将数据加载到临时表中然后在数据库中重塑数据更快。

### 通过删除不必要的索引来优化 `INSERT` SQL 查询

索引被广泛采用来加速数据库读取模式，但是由于数据库需要同时更新源表和索引，因此它们会对每个写入操作（`INSERT`/`UPDATE`/`DELETE`）造成损失。

![](imgs/sql-query-optimization-guide/insert-index.avif)

因此，为了优化写入工作负载，您可能需要检查表中的所有索引是否都是必要的，并在读取时经常被使用。要检查索引使用情况，您可以依赖数据库系统表，如 [PostgreSQL `pg_stat_user_indexes`](https://www.postgresql.org/docs/current/monitoring-stats.html) 或 [MySQL `table_io_waits_summary_by_index_usage`](https://dev.mysql.com/doc/mysql-perfschema-excerpt/8.3/en/performance-schema-table-io-waits-summary-by-index-usage-table.html)，提供有关影响索引的查询的最新统计信息。PostgreSQL 16 `pg_stat_user_indexes` 输出的示例如下：

```
 relid | indexrelid | schemaname | relname |   indexrelname   | idx_scan |         last_idx_scan         | idx_tup_read | idx_tup_fetch
-------+------------+------------+---------+------------------+----------+-------------------------------+--------------+---------------
 16460 |      16466 | public     | orders  | idx_order_date   |        1 | 2024-01-01 11:15:10.410511+00 |           10 |            10
 16460 |      16465 | public     | orders  | idx_order_status |        4 | 2024-02-08 11:16:00.471185+00 |            0 |             0
(2 rows)
```

我们可以看到 `idx_order_date` 和 `idx_order_status` 索引分别被扫描了一次和四次，其中 `idx_order_date` 最后一次被扫描是在 `2024-01-01 11:15:10.410511+00`。由于该索引自一月份以来就没有被使用过，因此它可能是一个值得删除的索引。

[Aiven 的 EverSQL](https://www.eversql.com/?utm_medium=organic&utm_source=ext_blog&utm_content=aivensqloptimization) 提供了冗余索引检测功能，可以自动发现未使用的索引。

![](imgs/sql-query-optimization-guide/eversql-index.avif)

一旦您确定未使用的索引，删除它们将会在写入性能和减少磁盘空间使用方面带来好处。

#### 警告

索引是加快读取操作的关键，删除它们可能会对所有 `SELECT` 查询产生影响。了解数据库中的读/写模式对于优化整体体验而不是仅仅一个语句至关重要。

#### 黄金法则

分析整体数据库工作负载并定义性能优化优先级。在此基础上，评估可用的整体索引集并确定哪些索引需要保留或可以删除。

#### 专业提示

如果您的数据加载语句发生在读取负责较低的时段，例如在夜间加载数据仓库时，您可以通过以下方式获得更好的提取性能：

- 禁用索引
- 加载数据
- 重新启用索引

### 优化 `INSERT` SQL 查询，就像它是 `SELECT` 语句一样

在许多情况下，`INSERT` 语句包含业务逻辑部分，与 `SELECT` 查询完全一样，这意味着它们可以包含 `WHERE` 子句、子查询等。

提取查询的内部部分并对其进行优化（通过索引、分区、查询重写等），就好像它是一个 `SELECT` 查询一样，可以提高性能。

#### 警告

`INSERT` 是数据库的写入工作负载，因此您需要谨慎地找到可以加速 `INSERT` 语句中的业务逻辑的确切边界，同时又不影响读取工作负载的性能。

#### 黄金法则

如果您的 `INSERT` 语句正在从数据库中的另一个表中选择数据，则可以遵循 `SELECT` 部分中详细说明的规则。如果在 `INSERT` 和 `SELECT` 语句（`WHERE` 条件）中都使用了明确的数据边界，那么对表进行分区可以加快这两个过程。

## 如何优化 `DELETE` SQL 查询

### 通过删除不必要的索引来优化 `DELETE` SQL 查询

同样，插入和删除（和更新）性能都可以从删除不必要的索引中获益。

#### 警告

根据工作负载，删除可能不像插入/更新那样频繁。过度优化删除可能会降低更常见的读/写模式的性能或复杂性。

#### 黄金法则

分析整体数据库工作负载并定义性能优化优先级。在此基础上，评估可用的整体索引集并确定哪些索引需要保留或可以删除。

### 使用 `TRUNCATE` 优化 `DELETE` SQL 查询

如果您的目的是从表中删除所有行，`TRUNCATE` 就是您的好帮手。`TRUNCATE` 在 PostgreSQL 和 MySQL（以及其他数据库）中都可用，它并不执行逐个元组的删除，而只是告诉数据库将所有新表的数据存储到一个新的空位置。因此，与 `DELETE FROM TABLE` 相比，它要快得多。

![](imgs/sql-query-optimization-guide/delete-vs-truncate.avif)

#### 警告

仅当您需要从表中删除所有行时，`TRUNCATE` 才有效。如果您只需要删除一组特定的行，那么这不是一个选项，您可能应该考虑分区。`TRUNCATE` 和 `DELETE` 具有不同的权限、需求和执行，理解它们（在 [MySQL](https://dev.mysql.com/doc/refman/8.0/en/truncate-table.html) 和 [PostgreSQL](https://www.postgresql.org/docs/current/sql-truncate.html) 中）是确保使用正确操作的关键。

#### 黄金法则

在设计数据库表时，还要分析数据保留方面的需求。如果一个表只需要包含“最新”的数据，则在设计时要考虑频繁插入和删除。

#### 专业提示

使用分区或创建时间绑定表（例如 `SALES_01_2024`）可以加快删除速度，因为整个表可以被截断。

### 使用表分区优化 `DELETE` SQL 查询

延续上一节，如果您的目标是根据已知规则定期删除数据（例如根据日期字段），则可以使用分区。PostgreSQL 和 MySQL 中提供分区功能，可根据谓词将数据拆分到多个表中。如果您需要删除分区中包含的整个数据子集，只需截断分区本身即可。

![](imgs/sql-query-optimization-guide/delete-partition-table.avif)

分区谓词可以是：

- **RANGE**: 定义与表中每个分区相关的一组范围
- **LIST**: 定义与表中每个分区关联的固定值列表
- **HASH**: 根据用户定义的函数定义分区
- **KEY**: 在 MySQL 中可用，使用 MySQL 散列函数提供 HASH 分区

#### 警告

尽管分区对于用户来说是“不可见的”（某个用户可以查询原始表并从所有分区中选择数据），但仍需要考虑分区表的管理。具体来说，对于 LIST 分区，当插入新键时，需要预先创建适当的分区。

#### 黄金法则

当数据集在时间或其他列（例如国家）方面存在明确边界时，分区非常有用。在没有明确过滤标准的情况下对表进行分区只会增加数据库的负载，使每个查询变得更慢

### 通过将语句拆分成块来优化 `DELETE` SQL 查询

执行大规模 `DELETE` 操作时，通过应用额外的 `WHERE` 条件将要删除的数据量分成更小的块。通过这种技术，您可以获得一个更容易跟踪的流程、在发生故障时更快地回滚，并且通常总体上具有更好的性能。

#### 警告

将 `DELETE` 操作分成多个较小的语句会使您面临缺乏一致性的风险，并且由于数据子集不重叠而丢失一些行。因此，您需要确保满足一致性要求，并且生成的语句集包含原始查询所针对的整套行。

#### 黄金法则

将 `DELETE` 操作拆分成多个块是提高性能并减少对数据库影响的好方法。如果您有一组特定的数据，并且该数据具有明确的写入和读取操作边界（例如 `COUNTRY` 或 `DATE`），您可以将其用作分割键，并使用此方法。

### 通过减少执行 `DELETE` 的频率来优化 `DELETE` SQL 查询

在某些用例中，您可能希望每天删除数据，例如需要删除超过 30 天的旧数据。但是，此操作需要每天扫描整个表。对于此类用例，您可以利用以下组合逻辑：

- 使用更智能的 `SELECTS` 不显示旧数据集（不检索超过 30 天的数据）
- 仅当空间回收有用时才删除数据（磁盘空间不足时）

此外，在 PostgreSQL 和 MySQL 中，删除表中的一小部分数据集不会节省大量磁盘空间，因为它只会在托管[表数据的页面中产生可用空间](https://www.google.com/url?q=https://jfg-mysql.blogspot.com/2017/07/innodb-compaction.html&sa=D&source=docs&ust=1707753160642232&usg=AOvVaw2nUTuUDmp0082dAi2J4oy6)。

#### 警告

您可能被法律强制按时删除旧数据。在这种情况下，请明智地评估您的数据保留需求并相应地规划数据删除。智能分区设计可以帮助通过截断提供快速删除语句。

#### 黄金法则

明智地设计数据保留策略和相关删除语句。如果您不打算删除大部分数据集，就不要计划节省大量空间。

### 通过不执行 `DELETE` 来优化 `DELETE` SQL 查询

如果要删除大型表中的大部分数据，则以下方法会更快：

- 创建新表
- 复制仍然需要的一小部分
- 删除旧表
- 重命名表

![](imgs/sql-query-optimization-guide/not-executing-delete.avif)

#### 警告

虽然这种方法可能更快，但它涉及数据库中的几个操作，可能会阻止底层表的正常功能。请仔细了解是否可以执行整个过程同时保持正常的数据库功能。此外，请注意，如果您有引用约束（例如外键指向表中某些键），则此解决方法将不起作用。

#### 黄金法则

在数据库负载具有易于预测的使用模式的情况下，上述解决方法是理想的。在流量低峰期间（或根本没有流量时）执行该过程将提供最佳的 `DELETE` 性能，同时最大限度地减少整个数据库功能的中断。

## 总结

优化 SQL 语句可能是一项繁琐的任务：您需要深入了解数据库结构和要优化的工作负载类型才能获得良好的结果。此外，在优化特定工作负载（例如 `INSERT` 语句）时，您可能会影响具有不同访问模式（如 `SELECT` 或 `DELETE`）的其他查询的性能。

要全面了解您的数据资产和支持结构，了解性能变化并获得自动的、AI 辅助的 SQL 优化建议，您可以使用 [Aiven 的 EverSQL](https://www.eversql.com/?utm_medium=organic&utm_source=ext_blog&utm_content=aivensqloptimization)。EverSQL 传感器可安装在任何 PostgreSQL 和 MySQL 数据库上，让您监控缓慢的查询并获得性能见解。人工智能驱动的引擎会分析缓慢的 SQL 语句和现有的支持数据结构，并提供索引建议和 SQL 重写建议，以提高性能。


> 作者：Francesco Tisiot<br>
> 原文：[https://aiven.io/developer/sql-query-optimization-guide](https://aiven.io/developer/sql-query-optimization-guide)

[PostgreSQL `COPY`]: https://www.postgresql.org/docs/current/sql-copy.html
[MySQL `LOAD DATA`]: https://dev.mysql.com/doc/refman/8.0/en/load-data.html


