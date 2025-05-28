---
date: 2025-05-27
slug: archive-postgres-partitions-to-iceberg
tags:
  - PostgreSQL
  - Iceberg
---

# 将 PostgreSQL 分区归档到 Iceberg

PostgreSQL [内置了分区功能](https://www.crunchydata.com/blog/native-partitioning-with-postgres)，您还可以借助 `pg_partman` 来进一步协助维护分区。如果您的主要工作负载是查询以时间序列为中心的小型数据子集，它可以很好地对数据进行分区，从而轻松保留有限的数据集并提高性能。通常，在实施分区时，您只保留一部分数据，然后随着成本管理的需要而删除旧数据。

但是如果我们可以将旧分区无缝移动到 Iceberg 并永久保留所有数据，同时只在 PostgreSQL 中维护最近的分区，那会怎样？我们是否可以拥有一个完美的世界：Iceberg 中有完整的长期副本，可以从数据仓库轻松查询，而 PostgreSQL 仍作为具有最近 30 天数据的操作数据库？

借助 [Crunchy 数据仓库的最新复制](https://www.crunchydata.com/blog/logical-replication-from-postgres-to-iceberg)支持，这可以无缝实现。让我们深入了解一下。

<!-- more -->

## 分区创建

首先，我们需要创建分区表。如果您想在家中跟着操作，这里有一些代码可以设置一个模仿网络分析数据集的分区数据示例。

``` sql linenums="1"
CREATE TABLE page_hits (
    id SERIAL,
    site_id INT NOT NULL,
    ingest_time TIMESTAMPTZ NOT NULL,
    url TEXT NOT NULL,
    request_country TEXT,
    ip_address INET,
    status_code INT,
    response_time_msec INT,
    PRIMARY KEY (id, ingest_time)
) PARTITION BY RANGE (ingest_time);
```

下面的代码将为我们创建一组过去 30 天的分区。

``` sql linenums="1"
DO $$
DECLARE
  d DATE;
BEGIN
  FOR d IN SELECT generate_series(DATE '2025-04-20', DATE '2025-05-19', INTERVAL '1 day') LOOP
    EXECUTE format($f$
      CREATE TABLE IF NOT EXISTS page_hits_%s PARTITION OF page_hits
      FOR VALUES FROM ('%s') TO ('%s');
    $f$, to_char(d, 'YYYY_MM_DD'), d, d + INTERVAL '1 day');
  END LOOP;
END $$;
```

您的数据库看起来应该是这样的：

```
List of relations
 Schema |          Name           |       Type        |       Owner
--------+-------------------------+-------------------+-------------------
 public | page_hits               | partitioned table | postgres
 public | page_hits_2025_04_20    | table             | postgres
 public | page_hits_2025_04_21    | table             | postgres
 ...
 public | page_hits_2025_05_18    | table             | postgres
 public | page_hits_2025_05_19    | table             | postgres
 public | page_hits_id_seq        | sequence          | postgres
```

现在我们可以生成一些示例数据。在本例中，我们将为每个表每天生成 1000 行数据：

``` sql linenums="1"
DO $$
DECLARE
  d DATE;
BEGIN
  FOR d IN
    SELECT generate_series(DATE '2025-04-20', DATE '2025-05-19', '1 day'::INTERVAL)
  LOOP
    INSERT INTO page_hits (site_id, ingest_time, url, request_country, ip_address, status_code, response_time_msec)
    SELECT
        (RANDOM() * 30)::INT,
        d + (i || ' seconds')::INTERVAL,
        'http://example.com/' || substr(md5(random()::text), 1, 12),
        (ARRAY['China', 'India', 'Indonesia', 'USA', 'Brazil'])[1 + (random() * 4)::INT],
        inet '10.0.0.0' + (random() * 1000000)::INT,
        (ARRAY[200, 200, 200, 404, 500])[1 + (random() * 4)::INT],
        (random() * 300)::INT
    FROM generate_series(1, 1000) AS s(i);
  END LOOP;
END $$;
```

现在我们的 PostgreSQL 设置中已经有了一些数据，让我们将它们连接到 Crunchy 数据仓库并进行复制。

## Iceberg 复制

在设置中，您需要指定通过根分区发布 - `root=true`。这将保留 PostgreSQL 中的分区，但不会对 Iceberg 进行分区，因为它有自己的数据文件组织结构。

``` sql linenums="1"
CREATE PUBLICATION hits_to_iceberg
FOR TABLE page_hits
WITH (publish_via_partition_root = true);
```

设置复制用户：

``` sql linenums="1"
-- create a new user
CREATE USER replication_user WITH REPLICATION PASSWORD '****';

-- grant appropriate permissions
GRANT SELECT ON ALL TABLES IN SCHEMA public TO replication_user;
```

在数据仓库端，订阅原始数据。由于我们指定了 `create_tables_using Iceberg`，这些数据将存储在 Iceberg 中。

``` sql linenums="1"
CREATE SUBSCRIPTION http_to_iceberg
CONNECTION 'postgres://replication_user:****@p.qzyqhjdg3fhejocnta3zvleomq.db.postgresbridge.com:5432/postgres?sslmode=require'
PUBLICATION hits_to_iceberg
WITH (create_tables_using = 'iceberg', streaming, binary, failover);
```

这是 Iceberg 表。

```
                          List of relations
 Schema |          Name           |     Type      |       Owner
--------+-------------------------+---------------+-------------------
 public | page_hits               | foreign table | postgres

```

## 从 PostgreSQL 查询存储在 Iceberg 中的数据

在这里，我们可以看到每个国家/地区的每日流量洞察，细分点击次数、成功率、平均响应时间和主要错误代码：

``` sql linenums="1"
SELECT
  date_trunc('day', ingest_time) AS day,
  request_country,
  COUNT(*) AS total_hits,
  ROUND(100.0 * SUM(CASE WHEN status_code = 200 THEN 1 ELSE 0 END) / COUNT(*), 2) AS success_rate_percent,
  ROUND(AVG(response_time_msec), 2) AS avg_response_time_msec,
  MODE() WITHIN GROUP (ORDER BY status_code) AS most_common_status
FROM
  page_hits
GROUP BY
  day, request_country
ORDER BY
  day, request_country
;
```
``` title="Output"
          day           | request_country | total_hits | success_rate_percent | avg_response_time_msec | most_common_status
------------------------+-----------------+------------+----------------------+------------------------+--------------------
 2025-04-20 00:00:00+00 | Brazil          |        128 |                68.75 |                 146.83 |                200
 2025-04-20 00:00:00+00 | China           |        138 |                65.94 |                 145.67 |                200
 2025-04-20 00:00:00+00 | India           |        245 |    64.90000000000001 |                  153.8 |                200
 2025-04-20 00:00:00+00 | Indonesia       |        230 |    64.34999999999999 |                 151.43 |                200

```

## 删除旧的 PostgreSQL 分区

由于数据已被复制并且副本位于 Iceberg 中，我们可以在特定时间删除分区以释放主操作 PostgreSQL 数据库上的存储和内存。

``` sql linenums="1"
--drop partition
DROP TABLE page_hits_2025_04_20;
```

```
-- show missing partition in the table list
                            List of relations
 Schema |          Name           |       Type        |       Owner
--------+-------------------------+-------------------+-------------------
 public | page_hits               | partitioned table | postgres
 public | page_hits_2025_04_21    | table             | postgres
 public | page_hits_2025_04_22    | table             | postgres
```

```
-- query iceberg, data is still there
          day           | request_country | total_hits | success_rate_percent | avg_response_time_msec | most_common_status
------------------------+-----------------+------------+----------------------+------------------------+--------------------
 2025-04-20 00:00:00+00 | Brazil          |        128 |                68.75 |                 146.83 |                200
 2025-04-20 00:00:00+00 | China           |        138 |                65.94 |                 145.67 |                200
 2025-04-20 00:00:00+00 | India           |        245 |    64.90000000000001 |                  153.8 |                200
 2025-04-20 00:00:00+00 | Indonesia       |        230 |    64.34999999999999 |                 151.43 |                200

```

## 总结

![Logical Replication from Partition to Crunchy Data Warehouse](imgs/logical-replication-partitons-to-iceberg.avif)
/// Caption
从分区到 Crunchy 数据仓库的逻辑复制
///

以下是简单的 PostgreSQL 归档方法，可实现长期经济高效的数据保留：

- 对高吞吐量数据进行分区 - 这对于性能和管理来说都是理想的选择。
- 将数据复制到 Iceberg，以便于报告和长期归档。
- 以理想的间隔删除分区。
- 持续从 PostgreSQL 查询归档数据。

> 作者：Craig Kerstiens<br>
> 原文：https://www.crunchydata.com/blog/archive-postgres-partitions-to-iceberg#now-drop-the-older-postgres-partition
