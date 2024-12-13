---
draft: true
date: 2024-11-14
categories:
  - Overlaps
---

# 解决 PostgreSQL 中的重叠查询问题

范围查询是 SQL 中非常常见的任务：选择属于某个指定范围内的日期、数字甚至文本值。例如：3 月份的所有贷款申请，或 50 美元至 500 美元之间的所有销售交易。SQL 开发人员通常擅长编写此类查询并对其进行索引以确保其性能良好。

但当需求稍有变化时——比较两个值的范围以找出它们的重叠之处——情况突然就完全不同了。这些所谓的“重叠查询”比表面上看起来更棘手。

<!-- more -->

## 重叠查询

最常见的重叠查询涉及时间（日期和时间）范围，称为间隔（`intervals`）。例如，想象一下对医院病人的查询。如果您希望查找八月份首次入院的所有患者，这是一个简单的范围查询。但是如果您希望找到住院时间（另一个日期间隔）至少部分位于八月间隔的所有患者，这是一个重叠查询。虽然编写这样的查询并不难，但编写出性能良好的查询却很困难。让我们看看为什么。

### 用例场景

让我们研究一个涉及重叠查询的实际问题。大型博物馆的数据库跟踪所有来访顾客进出建筑物的时间。它还在其视频安全系统中维护所有中断的时间。在一连串轻微盗窃事件之后，安全主管要求提供一份在一次或多次安全中断期间在场的所有顾客的名单。显然，我们有一个日期时间范围，即顾客的停留时间，必须将其与另一个日期时间范围（表示安全中断的间隔）进行比较。目标是找到所有重叠的实例。

### 数据库

顾客表（`patrons`）包含表示访问持续时间的初始和最终时间戳，以及表示关联顾客的 ID。中断表（`outages`）定义了每次中断的开始和结束时间戳。为了简单起见，我们不包括其他列。

``` sql
CREATE TABLE patrons (
      patron_id  INT  generated always as identity NOT NULL,
      enter      Timestamp(0) NOT NULL,
      leave      Timestamp(0) NOT NULL
);
CREATE TABLE outages (
      location_id INT generated always as identity NOT NULL,
      start_time  Timestamp(0) NOT NULL,
      end_time   Timestamp(0) NOT NULL
);
```

为了在大型数据集上测试性能，我们将使用 20 年的随机数据填充这两个表。（填充表格的脚本位于文章末尾结论部分后面的附录部分。）

在 SQL 中，比较间隔的规范方法是使用 `OVERLAPS` 运算符。要查找所有重叠的情况，只需将每个表中的日期分成间隔，然后与此运算符进行比较：

``` sql
SELECT * 
FROM patrons p
       CROSS JOIN outages o
WHERE (p.enter, p.leave) OVERLAPS (o.start_time, o.end_time);
```

此查询可以工作--至少它将返回适当的行--但它非常慢。（在我的测试系统上（Windows 11 和 PostgreSQL 16，测试机器是 AMD 5700x CPU、2GB SSD），至少需要 3 分钟）。首次尝试优化查询可能会导致您在中断开始和结束时间上创建复合索引，并在订阅会员上创建类似的索引：

``` sql
CREATE INDEX ON outages USING BTREE(start_time, end_time);
CREATE INDEX ON patrons USING BTREE(enter, leave);
```

这并没有任何帮助：查询同样缓慢。经过一番研究发现，`OVERLAPS` 运算符不支持索引。因此，我们改为明确指定比较。

第一次尝试这样的查询时，您可能会在一张纸上画线，并看到两个间隔可以重叠的四种不同方式（见下图）。这使得查询相当复杂，但我们可以用一些小技巧来简化它。只有两种方式可以让区间不重叠。如果我们编写不重叠的查询，然后用 `NOT` 进行反转，我们就会得到所需的结果。

![](img/overlaps-interval.png)

例如，用于查找 `interval1` 和 `interval2` 所有不重叠情况的 `WHERE` 子句如下所示：

``` sql
WHERE interval1.start > interval2.end
      OR interval2.start > interval1.end
```

如果我们使用 `NOT` 反转整个查询，我们将得到所有重叠的情况

``` sql
WHERE NOT (date1.start > date2.end 
           OR date2.start > date1.end)
```

这还可简化（使用布尔逻辑规则）为：

``` sql
WHERE date1.start <= date2.end 
      AND date2.start <= date1.end)
```

（注意：编写查询是为了符合时间间隔的 SQL 标准，该标准规定，如果一个间隔的开始时间正好在另一个间隔的结束时间，则会发生重叠。这意味着将包含恰好重叠 0.00 秒的值。如果您希望要求最少的重叠量，请相应地调整您的查询）


> 作者：Lee Asher<br>
> 原文：[https://www.red-gate.com/simple-talk/databases/postgresql/solving-the-overlap-query-problem-in-postgresql/](https://www.red-gate.com/simple-talk/databases/postgresql/solving-the-overlap-query-problem-in-postgresql/)
