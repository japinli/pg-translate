---
date: 2025-01-24
slug: group-by-fixing-optimizer-estimates
categories:
  - Aggregation
---

# GROUP BY: 修复优化器估计值

如果您使用 PostgreSQL 进行分析或大规模聚合，您可能会注意到规划器偶尔会对行数做出错误的估计。虽然这对于小规模的聚合来说不是问题，但对于具有许多不同维度的大规模聚合来说，这确实是个问题。

简而言之：`GROUP BY` 语句包含的列越多，优化器高估行数的可能性就越大。

本文解释了如何在 PostgreSQL 中处理这个问题。

<!-- more -->

## 在 PostgreSQL 中填充示例数据

对于这个例子，我创建了一个包含几个整数值的小表：

``` sql
CREATE TABLE t_sample (
	a 	int, 
	b 	int, 
	c 	int, 
	d 	int, 
	e 	int, 
	f 	int 	DEFAULT random()*100000
);
```

正如我们所看到的，我们有 5 个整数列（`a-e`）以及一个包含默认值的列。向该表添加 1000 万行数据的最简单方法是使用 `generate_series` 函数：

``` sql
INSERT INTO t_sample 
    SELECT x % 10, x % 11, x % 12, x % 13, x % 14 
    FROM generate_series(1, 10000000) AS x;
```

这里的关键点是我们的数据具有一些有趣的属性。每列只能包含一组固定的值（第一列有 10 个不同的值，第二列有 11 个不同的值，依此类推）。

一旦数据加载完成，我将使用 `ANALYZE` 命令来创建新的优化器统计数据：

``` sql
ANALYZE;
```

## 优化器统计是什么意思？

PostgreSQL 优化器如何工作？最根本的思想是列统计。对于每一列，PostgreSQL 都保存重要信息，例如不同值的数量、最常见的值、直方图等等：

``` sql
SELECT
    tablename, attname, 
    n_distinct, most_common_vals 
FROM
    pg_stats 
WHERE
    tablename = 't_sample'
    AND attname IN ('a', 'b');
```
``` title="输出"
 tablename | attname | n_distinct |     most_common_vals     
-----------+---------+------------+--------------------------
 t_sample  | b       |         11 | {9,6,7,5,0,8,10,1,4,3,2}
 t_sample  | a       |         10 | {9,0,4,5,8,1,2,3,6,7}
(2 rows)
```

请注意，此信息针对每一列进行存储。在正常情况下，autovacuum 将确保统计信息保持最新。

## 当列相互依赖时

以下查询非常常见：我们想要确定每个组中有多少个条目。关键细节是 `GROUP BY` 子句包含五个维度。

为了使示例更容易理解，我们首先禁用 PostgreSQL 中的并行查询。这不是针对 PostgreSQL 的一般性能建议，而仅仅是为了简化执行计划：

``` sql
SET max_parallel_workers_per_gather TO 0;
explain analyze
    SELECT
	    a, b, c, d, e, count(*) 
    FROM
        t_sample 
    GROUP BY
	    1, 2, 3, 4, 5;
```
``` title="输出"
                               QUERY PLAN
--------------------------------------------------------------------------
 HashAggregate  (cost=982455.57..1102046.81 rows=240240 width=28) 
    (actual time=1638.327..1784.077 rows=60060 loops=1)
   Group Key: a, b, c, d, e
   Planned Partitions: 4  Batches: 5  
    Memory Usage: 8241kB  Disk Usage: 53072kB
   ->  Seq Scan on t_sample  (cost=0.00..163696.15 rows=10000115 width=20) 
    (actual time=0.051..335.137 rows=10000000 loops=1)
 Planning Time: 0.167 ms
 Execution Time: 1787.892 ms
(6 rows)
```

这里最引人注目的是什么？归结为两个数字：`240240` 和 `60060`。第一个数字表示优化器的估计——它假设将返回超过 200,000 行。但实际结果只有 60060 行。为什么会这样？原因是 `a`、`b`、`c`、`d`、`e` 的某些组合根本不存在。然而，PostgreSQL 并不知道这一点。请记住，优化器维护有关每列的统计信息 - 它并不知道所有这些实际排列。

PostgreSQL 如何得出 `240240`？考虑以下计算：

``` sql
SELECT 10 * 11 * 12 * 13 * 14;
```
``` title="输出"
 ?column? 
----------
   240240
(1 row)
```

PostgreSQL 只是简单地将每个维度中的元素数量相乘，这会导致高估（这通常比低估的问题更小）。

## 创建统计数据：修正估计值

解决这些估计问题的方法是使用扩展统计信息。以下命令可用于解决该问题：

``` psql
blog=# \h CREATE STATISTICS
```
```title="输出"
Command:     CREATE STATISTICS
Description: define extended statistics
Syntax:
CREATE STATISTICS [ [ IF NOT EXISTS ] statistics_name ]
    ON ( expression )
    FROM table_name
 
CREATE STATISTICS [ [ IF NOT EXISTS ] statistics_name ]
    [ ( statistics_kind [, ... ] ) ]
    ON { column_name | ( expression ) }, { column_name | ( expression ) } [, ...]
    FROM table_name
 
URL: https://www.postgresql.org/docs/17/sql-createstatistics.html
```

在我们的例子中，我们需要 `CREATE STATISTICS` 的 `ndistinct` 选项：

``` sql
CREATE STATISTICS mystats (ndistinct)
ON a, b, c, d, e
FROM t_sample;
ANALYZE;
```

这将创建排列的统计信息，而不仅仅是单个值和列的统计信息。当我们再次运行查询时，我们将看到估计值有了显着改善：

``` sql
explain analyze
    SELECT
        a, b, c, d, e, count(*) 
    FROM
        t_sample 
    GROUP BY
        1, 2, 3, 4, 5;
```
``` title="输出"
                                QUERY PLAN
----------------------------------------------------------------------------
 HashAggregate  (cost=313697.88..314288.34 rows=59046 width=28) 
    (actual time=1587.681..1733.595 rows=60060 loops=1)
   Group Key: a, b, c, d, e
   Batches: 5  Memory Usage: 8241kB  Disk Usage: 53072kB
   ->  Seq Scan on t_sample  (cost=0.00..163696.15 rows=10000115 width=20) 
    (actual time=0.060..309.344 rows=10000000 loops=1)
 Planning Time: 0.623 ms
 Execution Time: 1737.495 ms
(6 rows)
```

哇，这太接近了：`59046` 与 `60060` —— PostgreSQL 是正确的。在这种情况下，它不会影响性能。然而，在现实世界中，修复统计数据可能会对速度和整体数据库效率产生巨大影响。

问题可能自然而然地出现：优化器使用什么信息来得出如此准确的估计？答案如下：

``` pgsql
blog=# \x
Expanded display is on.
blog=# SELECT * FROM pg_stats_ext;
-[ RECORD 1 ]----------+----------------------------------------------------------------
schemaname             | public
tablename              | t_sample
statistics_schemaname  | public
statistics_name        | mystats
statistics_owner       | hs
attnames               | {a,b,c,d,e}
exprs                  | 
kinds                  | {d}
inherited              | f
n_distinct             | {"1, 2": 110, "1, 3": 60, "1, 4": 130, "1, 5": 70, 
              "2, 3": 132, "2, 4": 143, "2, 5": 154, "3, 4": 156, 
              "3, 5": 84, "4, 5": 182, "1, 2, 3": 660, "1, 2, 4": 1430, 
              "1, 2, 5": 770, "1, 3, 4": 780, "1, 3, 5": 420, 
              "1, 4, 5": 910, "2, 3, 4": 1716, "2, 3, 5": 924, 
              "2, 4, 5": 2002, "3, 4, 5": 1092, "1, 2, 3, 4": 8576, 
              "1, 2, 3, 5": 4621, "1, 2, 4, 5": 10026, "1, 3, 4, 5": 5457,  
              "2, 3, 4, 5": 11949, "1, 2, 3, 4, 5": 59046}
dependencies           | 
most_common_vals       | 
most_common_val_nulls  | 
most_common_freqs      | 
most_common_base_freqs | 
```

`pg_stats_ext` 视图告诉我们 PostgreSQL 在这种情况下如何处理 `n_distinct` 列。

## 最后 ...

优化查询和优化器通常都是值得探索的有趣话题。如果您想了解更多并深入挖掘，请考虑阅读我关于 [PostgreSQL 查询优化器工作原理](https://www.cybertec-postgresql.com/en/how-the-postgresql-query-optimizer-works/)的文章。

<hr>

> 作者：Hans-Jürgen Schönig<br>
> 原文：[https://www.cybertec-postgresql.com/en/group-by-fixing-optimizer-estimates/](https://www.cybertec-postgresql.com/en/group-by-fixing-optimizer-estimates/)


