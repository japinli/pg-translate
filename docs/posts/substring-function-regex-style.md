---
date: 2024-12-13
categories:
  - Function
  - Regexp
---

# `substring` 函数正则表达式

当我在查看一些旧代码时，我偶然发现了一些我完全忘记了，或者别人早就知道的东西——可以使用 `substring` 函数进行正则表达式工作。

<!-- more -->

PostgreSQL 中大多数正则表达式函数通常以 `regexp` 开头，或者使用像 `~` 这样的运算符。但是我完全忘记了 `substring` 函数，它也可以理解正则表达式。当我必须使用正则表达式解析出一些小段文本时，我通常会使用 `regexp_*` 函数之一，如 `regexp_replace`、`regexp_match` 或 `regexp_matches`，并且仅当我想通过索引号从字符串中提取一系列字符时才使用 `substring`。但那里的代码告诉我，也许我或其他人当时要聪明得多。所有这些函数都总结在[Documentation: PostgreSQL string functions](https://www.postgresql.org/docs/current/functions-string.html)。

大多数人使用的常见 `substring` 函数时：

```sql
substring(string, start_position, length)
```

也可以写成：

```sql
substring(string FROM start_position FOR length)
```

`length` 和 `FOR length` 子句是可选的，如果省略，则返回从 `start_position` 位置到字符串末尾的字符串。

如下所示：

```sql
SELECT substring('Jill lugs 5 pails', 11, 7);
```

返回：`5 pails`

这些普通形式的 `substring` 甚至存在于完全不擅长执行正则表达式的数据库中。我不会说出这些数据库的名字。只要知道他们存在。PostgreSQL 还有另一种形式，我发现有时比上述 `regexp_*` 函数做同样的事情更容易理解和编写。

让我们重复上面的例子，但要明白我们需要挑选出 Jill 到底拖着什么以及她拖了多少。

```sql
SELECT substring('Jill lugs 5 pails', '[0-9]+\s[A-Za-z]+');
```

它返回相同的内容，但无需知道位置，而只需关心措辞。

现在可以肯定的是，在许多情况下，您最好使用更强大的 `regexp_match` 或 `regexp_matches`，就像当您试图一次性分离语句的各个部分时一样。就像也许我希望将桶的数量与物品桶的数量分开，那么我当然会这样做。

```sql
SELECT r[1] AS who, r[2] AS action, r[3] AS how_many, r[4] AS what
FROM regexp_match('Jill lugs 5 pails',
                  '([A-Za-z]+)\s+([A-Za-z]+)\s+([0-9]+)\s+([A-Za-z]+)') AS r;
 who  | action | how_many | what
------+--------+----------+-------
 Jill | lugs   | 5        | pails
(1 row)
```
而当我需要有点贪心并返回多条记录时，因为我要在一堆词汇中抓取每个句子中符合我的措辞的部分，我会使用 `regexp_matches`。

```sql
SELECT r[1] AS who, r[2] AS action, r[3] AS how_many, r[4] AS what
FROM regexp_matches('Jill lugs 5 pails. Jack lugs 10 pails',
                    '([A-Za-z]+)\s+([A-Za-z]+)\s+([0-9]+)\s+([A-Za-z]+)',
                    'g') AS r;
 who  | action | how_many | what
------+--------+----------+-------
 Jill | lugs   | 5        | pails
 Jack | lugs   | 10       | pails
(2 rows)
```

虽然这两者都很棒，但它们总是返回数组或数组集。

`regexp_matches` 好的一面是返回结果集，因此如果您需要多个与您的模式匹配的答案，它可以解决问题，但是当没有任何匹配项时，您的查询就会失败，不会返回任何记录。这使得在 `FROM` 子句中使用有点危险，除非您知道这一点并且不在乎或已经想出了避免的方法，例如使用 `LEFT JOIN`。

`regexp_match` 总是返回，因为它不返回一个集合，但很烦人，因为您总是返回一个数组，您必须挑选值。因此，原始问题变成了额外的 `()[]` 噪音，如下所示：

```sql
SELECT (regexp_match('Jill lugs 5 pails', '[0-9]+\s[A-Za-z]+'))[1];
```

随着年龄的增长，那些额外的字符让我很烦恼，因为它需要更多键盘敲击和更多无用的字符来查看。虽然我还没有测试过，但我怀疑它也慢了。

> 作者：Leo Hsu, Regina Obe<br>
> 原文：[https://www.postgresonline.com/journal/index.php?/archives/415-Substring-function-Regex-style.html](https://www.postgresonline.com/journal/index.php?/archives/415-Substring-function-Regex-style.html)

