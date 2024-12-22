---
date: 2024-12-22
slug: some-of-my-favorite-things-postgres-queries
categories:
  - Performance
---

# 我最喜欢的一些东西 - Postgres 查询

本着节日的气氛，我想写一篇简短的文章，介绍我在日常使用 Postgres 时最喜欢的一些查询。其中一些查询是我自己开发的，其他一些则是在互联网上找到的（向之前发布过帖子的人致敬），并得到了进一步完善。

在我的 github 网站上还可以找到更多：

[https://github.com/shane-borden/sqlScripts/tree/master/postgres](https://github.com/shane-borden/sqlScripts/tree/master/postgres)

希望这些查询也能帮助您在日常工作中让 Postgres 运行得更好！

<!-- more -->

前三个查询根据执行次数、`mean_exec_time` 和 `total_exec_time` 对 `pg_stat_statements` 中的 TOP SQL 进行排名。我喜欢使用这些查询来快速了解我应该重点调整的内容。鉴于 `pg_stat_statements` 跟踪很多内容，您可以根据需要过滤掉某些“查询文本”，这样它们就不会影响排名。

## 按平均执行时间排序的 TOP SQL

``` sql
WITH
hist AS (
SELECT queryid::text,
       SUBSTRING(query from 1 for 1000) query,
       ROW_NUMBER () OVER (ORDER BY mean_exec_time::numeric DESC) rn,
       SUM(mean_exec_time::numeric) mean_exec_time
  FROM pg_stat_statements
 WHERE queryid IS NOT NULL
        AND query::text not like '%pg_%'
        AND query::text not like '%g_%'
        /* Add more filters here */
 GROUP BY
       queryid,
       SUBSTRING(query from 1 for 1000),
       mean_exec_time::numeric
),
total AS (
SELECT SUM(mean_exec_time::numeric) mean_exec_time FROM hist
)
SELECT DISTINCT
       h.queryid::text,
       ROUND(h.mean_exec_time::numeric,3) mean_exec_time,
       ROUND(100 * h.mean_exec_time / t.mean_exec_time, 1) percent,
       h.query
  FROM hist h,
       total t
 WHERE h.mean_exec_time >= t.mean_exec_time / 1000 AND rn <= 14
 UNION ALL
SELECT 'Others',
       ROUND(COALESCE(SUM(h.mean_exec_time), 0), 3) mean_exec_time,
       COALESCE(ROUND(100 * SUM(h.mean_exec_time) / AVG(t.mean_exec_time), 1), 0) percent,
       NULL sql_text
  FROM hist h,
       total t
 WHERE h.mean_exec_time < t.mean_exec_time / 1000 OR rn > 14
 ORDER BY 3 DESC NULLS LAST;
```

## 按总执行时间排序的 TOP SQL

``` sql
WITH
hist AS (
SELECT queryid::text,
       SUBSTRING(query from 1 for 100) query,
       ROW_NUMBER () OVER (ORDER BY total_exec_time::numeric DESC) rn,
       SUM(total_exec_time::numeric) total_exec_time
  FROM pg_stat_statements
 WHERE queryid IS NOT NULL
        AND query::text not like '%pg_%'
        AND query::text not like '%g_%'
        /* Add more filters here */
 GROUP BY
       queryid,
       SUBSTRING(query from 1 for 100),
       total_exec_time::numeric
),
total AS (
SELECT SUM(total_exec_time::numeric) total_exec_time FROM hist
)
SELECT DISTINCT
       h.queryid::text,
       ROUND(h.total_exec_time::numeric,3) total_exec_time,
       ROUND(100 * h.total_exec_time / t.total_exec_time, 1) percent,
       h.query
  FROM hist h,
       total t
 WHERE h.total_exec_time >= t.total_exec_time / 1000 AND rn <= 14
 UNION ALL
SELECT 'Others',
       ROUND(COALESCE(SUM(h.total_exec_time::numeric), 0), 3) total_exec_time,
       COALESCE(ROUND(100 * SUM(h.total_exec_time) / AVG(t.total_exec_time), 1), 0) percent,
       NULL sql_text
  FROM hist h,
       total t
 WHERE h.total_exec_time < t.total_exec_time / 1000 OR rn > 14
 ORDER BY 3 DESC NULLS LAST;
```

## 按执行次数排名的 TOP SQL

``` sql
WITH
hist AS (
SELECT queryid::text,
       SUBSTRING(query from 1 for 100) query,
       ROW_NUMBER () OVER (ORDER BY calls DESC) rn,
       calls
  FROM pg_stat_statements 
 WHERE queryid IS NOT NULL
        AND query::text not like '%pg_%'
        AND query::text not like '%g_%'
        /* Add more filters here */
 GROUP BY
       queryid,
       SUBSTRING(query from 1 for 100),
       calls
),
total AS (
SELECT SUM(calls) calls FROM hist
)
SELECT DISTINCT
       h.queryid::text,
       h.calls,
       ROUND(100 * h.calls / t.calls, 1) percent,
       h.query
  FROM hist h,
       total t
 WHERE h.calls >= t.calls / 1000 AND rn <= 14
 UNION ALL
SELECT 'Others',
       COALESCE(SUM(h.calls), 0) calls,
       COALESCE(ROUND(100 * SUM(h.calls) / AVG(t.calls), 1), 0) percent,
       NULL sql_text
  FROM hist h,
       total t
 WHERE h.calls < t.calls / 1000 OR rn > 14
 ORDER BY 2 DESC NULLS LAST;
```

## 包含 TOAST 的对象大小

显示表的总大小，包括其索引和 toast 大小。

``` sql
SELECT
  *,
  pg_size_pretty(table_bytes) AS table,
  pg_size_pretty(toast_bytes) AS toast,
  pg_size_pretty(index_bytes) AS index,
  pg_size_pretty(total_bytes) AS total
FROM (
  SELECT
    *, total_bytes - index_bytes - COALESCE(toast_bytes, 0) AS table_bytes
  FROM (
    SELECT
      c.oid,
      nspname AS table_schema,
      relname AS table_name,
      c.reltuples AS row_estimate,
      pg_total_relation_size(c.oid) AS total_bytes,
      pg_indexes_size(c.oid) AS index_bytes,
      pg_total_relation_size(reltoastrelid) AS toast_bytes
    FROM
      pg_class c
      LEFT JOIN pg_namespace n ON n.oid = c.relnamespace
    WHERE relkind = 'r'
  ) a
) a
WHERE table_schema like '%'
AND table_name like '%'
AND total_bytes > 0
ORDER BY total_bytes DESC;
```

## 使用 CPU 的 SQL 语句

使用 `pg_stat_statements`，此查询将把总时间分配为 CPU 时间。

``` sql
SELECT
    pss.userid,
    pss.dbid,
    pd.datname AS db_name,
    pss.queryid,
    round((pss.total_exec_time + pss.total_plan_time)::numeric, 2) AS total_time,
    pss.calls,
    round((pss.mean_exec_time + pss.mean_plan_time)::numeric, 2) AS mean,
    round((100 * (pss.total_exec_time + pss.total_plan_time) / sum((pss.total_exec_time + pss.total_plan_time)::numeric) OVER ())::numeric, 2) AS cpu_portion_pctg,
    substr(pss.query, 1, 200) short_query
FROM
    pg_stat_statements pss,
    pg_database pd
WHERE
    pd.oid = pss.dbid
    AND query::text NOT LIKE '%FOR UPDATE%'
    /* Add more filters here */
ORDER BY
    (pss.total_exec_time + pss.total_plan_time) DESC
LIMIT 30;
```

## 统计/回收脚本

此脚本查看为 vacuum 和 analyze 设置的数据库和表选项，以提供计划运行 vacuum/analyze 的时间以及上次运行时间的报告。此脚本将让您很好地了解 vacuum 和 analyze 的运行情况：

``` sql
WITH tbl_reloptions AS (
SELECT
    oid,
    oid::regclass table_name,
    substr(unnest(reloptions), 1,  strpos(unnest(reloptions), '=') -1) option,
    substr(unnest(reloptions), 1 + strpos(unnest(reloptions), '=')) value
FROM
    pg_class c
WHERE reloptions is NOT null)
SELECT
    s.schemaname ||'.'|| s.relname as relname,
    n_live_tup live_tup,
    n_dead_tup dead_dup,
    n_tup_hot_upd hot_upd,
    n_mod_since_analyze mod_since_stats,
    n_ins_since_vacuum ins_since_vac,
    case
      when avacinsscalefactor.value is not null and avacinsthresh.value is not null
        then ROUND(((n_live_tup * avacinsscalefactor.value::numeric) + avacinsthresh.value::numeric),0)
      when avacinsscalefactor.value is null and avacinsthresh.value is not null
        then ROUND(((n_live_tup * (select setting::numeric from pg_settings where name = 'autovacuum_vacuum_insert_scale_factor')) + avacinsthresh.value::numeric),0)
      when avacinsscalefactor.value is not null and avacinsthresh.value is null
        then ROUND(((n_live_tup * avacinsscalefactor.value::numeric) + (select setting::numeric from pg_settings where name = 'autovacuum_vacuum_insert_threshold')),0)
      else ROUND(((n_live_tup * (select setting::numeric from pg_settings where name = 'autovacuum_vacuum_insert_scale_factor')) + (select setting::numeric from pg_settings where name = 'autovacuum_vacuum_insert_threshold')),0) 
    end as ins_for_vac,
    case
      when avacscalefactor.value is not null and avacthresh.value is not null
        then ROUND(((n_live_tup * avacscalefactor.value::numeric) + avacthresh.value::numeric),0)
      when avacscalefactor.value is null and avacthresh.value is not null
        then ROUND(((n_live_tup * (select setting::numeric from pg_settings where name = 'autovacuum_vacuum_scale_factor')) + avacthresh.value::numeric),0)
      when avacscalefactor.value is not null and avacthresh.value is null
        then ROUND(((n_live_tup * avacscalefactor.value::numeric) + (select setting::numeric from pg_settings where name = 'autovacuum_vacuum_threshold')),0)
      else ROUND(((n_live_tup * (select setting::numeric from pg_settings where name = 'autovacuum_vacuum_scale_factor')) + (select setting::numeric from pg_settings where name = 'autovacuum_vacuum_threshold')),0) 
    end as mods_for_vac,
    case
      when avacanalyzescalefactor.value is not null and avacanalyzethresh.value is not null
        then ROUND(((n_live_tup * avacanalyzescalefactor.value::numeric) + avacanalyzethresh.value::numeric),0)
      when avacanalyzescalefactor.value is null and avacanalyzethresh.value is not null
        then ROUND(((n_live_tup * (select setting::numeric from pg_settings where name = 'autovacuum_analyze_scale_factor')) + avacanalyzethresh.value::numeric),0)
      when avacanalyzescalefactor.value is not null and avacanalyzethresh.value is null
        then ROUND(((n_live_tup * avacanalyzescalefactor.value::numeric) + (select setting::numeric from pg_settings where name = 'autovacuum_analyze_threshold')),0)
      else ROUND(((n_live_tup * (select setting::numeric from pg_settings where name = 'autovacuum_analyze_scale_factor')) + (select setting::numeric from pg_settings where name = 'autovacuum_analyze_threshold')),0) 
    end as mods_for_stats,
    case
      when avacfreezeage is not null
        then ROUND((greatest(age(c.relfrozenxid),age(t.relfrozenxid))::numeric / avacfreezeage.value::numeric * 100),2) 
      else ROUND((greatest(age(c.relfrozenxid),age(t.relfrozenxid))::numeric / (select setting::numeric from pg_settings where name = 'autovacuum_freeze_max_age') * 100),2) 
      end as avac_pct_frz,
    greatest(age(c.relfrozenxid),age(t.relfrozenxid)) max_txid_age,
    to_char(last_vacuum, 'YYYY-MM-DD HH24:MI') last_vac,
    to_char(last_analyze, 'YYYY-MM-DD HH24:MI') last_stats,
    to_char(last_autovacuum, 'YYYY-MM-DD HH24:MI') last_avac,
    to_char(last_autoanalyze, 'YYYY-MM-DD HH24:MI') last_astats,
    vacuum_count vac_cnt,
    analyze_count stats_cnt,
    autovacuum_count avac_cnt,
    autoanalyze_count astats_cnt,
    c.reloptions,
    case
      when avacenabled.value is not null
        then avacenabled.value::text
      when (select setting::text from pg_settings where name = 'autovacuum') = 'on'
        then 'true'
      else 'false'
    end as autovac_enabled
FROM
    pg_stat_all_tables s
JOIN pg_class c ON (s.relid = c.oid)
LEFT JOIN pg_class t ON c.reltoastrelid = t.oid
LEFT JOIN tbl_reloptions avacinsscalefactor on (s.relid = avacinsscalefactor.oid and avacinsscalefactor.option = 'autovacuum_vacuum_insert_scale_factor')
LEFT JOIN tbl_reloptions avacinsthresh on (s.relid = avacinsthresh.oid and avacinsthresh.option = 'autovacuum_vacuum_insert_threshold')
LEFT JOIN tbl_reloptions avacscalefactor on (s.relid = avacscalefactor.oid and avacscalefactor.option = 'autovacuum_vacuum_scale_factor')
LEFT JOIN tbl_reloptions avacthresh on (s.relid = avacthresh.oid and avacthresh.option = 'autovacuum_vacuum_threshold')
LEFT JOIN tbl_reloptions avacanalyzescalefactor on (s.relid = avacanalyzescalefactor.oid and avacanalyzescalefactor.option = 'autovacuum_analyze_scale_factor')
LEFT JOIN tbl_reloptions avacanalyzethresh on (s.relid = avacanalyzethresh.oid and avacanalyzethresh.option = 'autovacuum_analyze_threshold')
LEFT JOIN tbl_reloptions avacfreezeage on (s.relid = avacfreezeage.oid and avacfreezeage.option = 'autovacuum_freeze_max_age')
LEFT JOIN tbl_reloptions avacenabled on (s.relid = avacenabled.oid and avacenabled.option = 'autovacuum_enabled')
WHERE
    s.relname IN (
        SELECT
            t.table_name
        FROM
            information_schema.tables t
            JOIN pg_catalog.pg_class c ON (t.table_name = c.relname)
            LEFT JOIN pg_catalog.pg_user u ON (c.relowner = u.usesysid)
        WHERE
            t.table_schema like '%'
            AND (u.usename like '%' OR u.usename is null)
            AND t.table_name like '%'
            AND t.table_schema not in ('information_schema','pg_catalog')
            AND t.table_type not in ('VIEW')
            AND t.table_catalog = current_database())
    AND n_dead_tup >= 0
    AND n_live_tup > 0
ORDER BY 3;
```

## 未使用/很少使用的索引

为了保持系统运行良好，维护尽可能少的索引非常重要。这将显示哪些索引最近没有被使用过。

``` sql
WITH table_scans AS (
    SELECT
        relid,
        tables.idx_scan + tables.seq_scan AS all_scans,
        (tables.n_tup_ins + tables.n_tup_upd + tables.n_tup_del) AS writes,
        pg_relation_size(relid) AS table_size
    FROM
        pg_stat_all_tables AS tables
    WHERE
        schemaname NOT IN ('pg_toast', 'pg_catalog', 'partman')
),
all_writes AS (
    SELECT
        sum(writes) AS total_writes
    FROM
        table_scans
),
indexes AS (
    SELECT
        idx_stat.relid,
        idx_stat.indexrelid,
        idx_stat.schemaname,
        idx_stat.relname AS tablename,
        idx_stat.indexrelname AS indexname,
        idx_stat.idx_scan,
        pg_relation_size(idx_stat.indexrelid) AS index_bytes,
        indexdef ~* 'USING btree' AS idx_is_btree
    FROM
        pg_stat_user_indexes AS idx_stat
        JOIN pg_index USING (indexrelid)
        JOIN pg_indexes AS indexes ON idx_stat.schemaname = indexes.schemaname
            AND idx_stat.relname = indexes.tablename
            AND idx_stat.indexrelname = indexes.indexname
    WHERE
        pg_index.indisunique = FALSE
),
index_ratios AS (
    SELECT
        schemaname,
        tablename,
        indexname,
        idx_scan,
        all_scans,
        round((
            CASE WHEN all_scans = 0 THEN
                0.0::numeric
            ELSE
                idx_scan::numeric / all_scans * 100
            END), 2) AS index_scan_pct,
        writes,
        round((
            CASE WHEN writes = 0 THEN
                idx_scan::numeric
            ELSE
                idx_scan::numeric / writes
            END), 2) AS scans_per_write,
        pg_size_pretty(index_bytes) AS index_size,
        pg_size_pretty(table_size) AS table_size,
        idx_is_btree,
        index_bytes
    FROM
        indexes
        JOIN table_scans USING (relid)
),
index_groups AS (
    SELECT
        'Never Used Indexes' AS reason,
        *,
        1 AS grp
    FROM
        index_ratios
    WHERE
        idx_scan = 0
        AND idx_is_btree
    UNION ALL
    SELECT
        'Low Scans, High Writes' AS reason,
        *,
        2 AS grp
    FROM
        index_ratios
    WHERE
        scans_per_write <= 1
        AND index_scan_pct < 10
        AND idx_scan > 0
        AND writes > 100
        AND idx_is_btree
    UNION ALL
    SELECT
        'Seldom Used Large Indexes' AS reason,
        *,
        3 AS grp
    FROM
        index_ratios
    WHERE
        index_scan_pct < 5
        AND scans_per_write > 1
        AND idx_scan > 0
        AND idx_is_btree
        AND index_bytes > 100000000
    UNION ALL
    SELECT
        'High-Write Large Non-Btree' AS reason,
        index_ratios.*,
        4 AS grp
    FROM
        index_ratios,
        all_writes
    WHERE (writes::numeric / (total_writes + 1)) > 0.02
    AND NOT idx_is_btree
    AND index_bytes > 100000000
ORDER BY
    grp,
    index_bytes DESC
)
SELECT
    reason,
    schemaname,
    tablename,
    indexname,
    index_scan_pct,
    scans_per_write,
    index_size,
    table_size
FROM
    index_groups
WHERE
    tablename LIKE '%';
```

## 排名等待事件

这是一项很棒的查询，可以对 `pg_stat_activity` 中观察到的等待事件从最频繁到最不频繁进行排序。这不提供历史参考，而是查看当前时刻：

``` sql
WITH waits AS (
    SELECT
        wait_event,
        rank() OVER (ORDER BY count(wait_event) DESC) rn
    FROM pg_stat_activity
    WHERE wait_event IS NOT NULL
GROUP BY wait_event ORDER BY count(wait_event) ASC
),
total AS (
    SELECT
        SUM(rn) total_waits
    FROM
        waits
)
SELECT DISTINCT
    h.wait_event,
    h.rn,
    ROUND(100 * h.rn / t.total_waits, 1) percent
FROM
    waits h,
    total t
WHERE
    h.rn >= t.total_waits / 1000
    AND rn <= 14
UNION ALL
SELECT
    'Others',
    COALESCE(SUM(h.rn), 0) rn,
    COALESCE(ROUND(100 * SUM(h.rn) / AVG(t.total_waits), 1), 0) percent
FROM
    waits h,
    total t
WHERE
    h.rn < t.total_waits / 1000
    OR rn > 14
ORDER BY
    2 DESC NULLS LAST;
```

## 缺少外键索引的表

通常，当外键没有索引支持时，查询的运行时间会很差。以下查询可以显示缺少的索引：

```sql
WITH y AS (
    SELECT
        pg_catalog.format('%I.%I', n1.nspname, c1.relname) AS referencing_tbl,
        pg_catalog.quote_ident(a1.attname) AS referencing_column,
        t.conname AS existing_fk_on_referencing_tbl,
        pg_catalog.format('%I.%I', n2.nspname, c2.relname) AS referenced_tbl,
        pg_catalog.quote_ident(a2.attname) AS referenced_column,
        pg_relation_size(pg_catalog.format('%I.%I', n1.nspname, c1.relname)) AS referencing_tbl_bytes,
        pg_relation_size(pg_catalog.format('%I.%I', n2.nspname, c2.relname)) AS referenced_tbl_bytes,
        pg_catalog.format($$CREATE INDEX ON %I.%I(%I);$$, n1.nspname, c1.relname, a1.attname) AS suggestion
    FROM
        pg_catalog.pg_constraint t
        JOIN pg_catalog.pg_attribute a1 ON a1.attrelid = t.conrelid
            AND a1.attnum = t.conkey[1]
        JOIN pg_catalog.pg_class c1 ON c1.oid = t.conrelid
        JOIN pg_catalog.pg_namespace n1 ON n1.oid = c1.relnamespace
        JOIN pg_catalog.pg_class c2 ON c2.oid = t.confrelid
        JOIN pg_catalog.pg_namespace n2 ON n2.oid = c2.relnamespace
        JOIN pg_catalog.pg_attribute a2 ON a2.attrelid = t.confrelid
            AND a2.attnum = t.confkey[1]
    WHERE
        t.contype = 'f'
        AND NOT EXISTS (
            SELECT
                1
            FROM
                pg_catalog.pg_index i
            WHERE
                i.indrelid = t.conrelid
                AND i.indkey[0] = t.conkey[1]))
SELECT
    referencing_tbl,
    referencing_column,
    existing_fk_on_referencing_tbl,
    referenced_tbl,
    referenced_column,
    pg_size_pretty(referencing_tbl_bytes) AS referencing_tbl_size,
    pg_size_pretty(referenced_tbl_bytes) AS referenced_tbl_size,
    suggestion
FROM
    y
ORDER BY
    referencing_tbl_bytes DESC,
    referenced_tbl_bytes DESC,
    referencing_tbl,
    referenced_tbl,
    referencing_column,
    referenced_column;
```

## 阻塞锁树

[postgres.ai](https://postgres.ai/blog/20211018-postgresql-lock-trees) 是获取一些可观察性查询的好地方，这是我最喜欢的查询之一：

``` sql
with recursive activity as (
  select
    pg_blocking_pids(pid) blocked_by,
    *,
    age(clock_timestamp(), xact_start)::interval(0) as tx_age,
    -- "pg_locks.waitstart" – PG14+ only; for older versions:  age(clock_timestamp(), state_change) as wait_age,
    age(clock_timestamp(), (select max(l.waitstart) from pg_locks l where a.pid = l.pid))::interval(0) as wait_age
  from pg_stat_activity a
  where state is distinct from 'idle'
), blockers as (
  select
    array_agg(distinct c order by c) as pids
  from (
    select unnest(blocked_by)
    from activity
  ) as dt(c)
), tree as (
  select
    activity.*,
    1 as level,
    activity.pid as top_blocker_pid,
    array[activity.pid] as path,
    array[activity.pid]::int[] as all_blockers_above
  from activity, blockers
  where
    array[pid] <@ blockers.pids
    and blocked_by = '{}'::int[]
  union all
  select
    activity.*,
    tree.level + 1 as level,
    tree.top_blocker_pid,
    path || array[activity.pid] as path,
    tree.all_blockers_above || array_agg(activity.pid) over () as all_blockers_above
  from activity, tree
  where
    not array[activity.pid] <@ tree.all_blockers_above
    and activity.blocked_by <> '{}'::int[]
    and activity.blocked_by <@ tree.all_blockers_above
)
select
  pid,
  blocked_by,
  case when wait_event_type <> 'Lock' then replace(state, 'idle in transaction', 'idletx') else 'waiting' end as state,
  wait_event_type || ':' || wait_event as wait,
  wait_age,
  tx_age,
  to_char(age(backend_xid), 'FM999,999,999,990') as xid_age,
  to_char(2147483647 - age(backend_xmin), 'FM999,999,999,990') as xmin_ttf,
  datname,
  usename,
  (select count(distinct t1.pid) from tree t1 where array[tree.pid] <@ t1.path and t1.pid <> tree.pid) as blkd,
  format(
    '%s %s%s',
    lpad('[' || pid::text || ']', 9, ' '),
    repeat('.', level - 1) || case when level > 1 then ' ' end,
    left(query, 1000)
  ) as query
from tree
order by top_blocker_pid, level, pid;
```

> 作者：Shane Borden<br>
> 原文：[https://stborden.wordpress.com/2024/12/17/some-of-my-favorite-things-postgres-queries/](https://stborden.wordpress.com/2024/12/17/some-of-my-favorite-things-postgres-queries/)
