---
date: 2024-10-06
categories:
  - Log
  - Settings
---

# PostgreSQL 日志：您需要知道的一切

PostgreSQL 日志是解决问题、跟踪性能和审计数据库活动的宝贵资源。在将应用程序部署到生产环境之前，有必要对日志配置进行微调，以确保记录足够多的信息来诊断问题，同时不会减慢基本数据库操作的速度。

<!-- more -->

为了实现理想的平衡，深入了解各种 PostgreSQL 日志参数是至关重要的。有了这些知识，再加上对日志使用方式的理解，您就可以很好地定制日志设置，以精确满足您的[监控](https://betterstack.com/community/comparisons/postgresql-monitoring-tools/)和故障分析需求。

当您的应用程序已经出现问题时，再来调整日志参数是不正确的做法。主动关注数据库日志是充分利用它们的最佳方法。

本文讨论了不同类型的 PostgreSQL 日志、如何配置 PostgreSQL 提供的许多日志参数以及如何解释日志消息。本文还提供了一些有关如何使用 PostgreSQL 日志来解决问题和提高数据库性能的提示。

## 前提

您需要在计算机上[安装](https://www.postgresql.org/download/)并运行 PostgreSQL。确保您拥有 [PostgreSQL 15](https://www.postgresql.org/docs/release/15.0/) 或更高版本，因为它引入了以 [JSON 格式输出结构化日志](https://betterstack.com/community/guides/logging/structured-logging/)的功能。

要检查客户端版本，请使用以下命令：

``` bash
$ psql --version # client version
```

``` title="Output"
psql (PostgreSQL) 15.3 (Ubuntu 15.3-1.pgdg22.04+1)
```

可以通过执行以下命令查看服务器版本：

``` bash
sudo -u postgres psql -c 'SHOW server_version;'
```
``` title="Output"
          server_version
----------------------------------
 15.3 (Ubuntu 15.3-1.pgdg22.04+1)
```

## 搭建示例数据库

为了遵循本文中的示例，您可以设置流行的 [Chinook 示例数据库](https://github.com/morenoh149/postgresDBSamples/blob/master/chinook-1.4/) ，它代表一个数字媒体商店，其中包括艺术家、专辑、媒体曲目、发票和客户的表。

首先将[数据库文件](https://gist.githubusercontent.com/ayoisaiah/ebf628402706de0b6053699f9e0776ed/raw/43a4ce34a3c7ef397d2cf235f20f9217985f18e1/chinook.sql)下载到您的计算机：

``` bash
$ curl -LO https://gist.github.com/ayoisaiah/ebf628402706de0b6053699f9e0776ed/raw/43a4ce34a3c7ef397d2cf235f20f9217985f18e1/chinook.sql
```

接下来，使用默认的 `postgres` 用户启动 `psql`：

``` bash
$ sudo -u postgres psql
```

如下所示，创建 `chinook` 数据库：

``` sql
CREATE DATABASE chinook OWNER postgres;
```

``` title="Output"
CREATE DATABASE
```

连接刚刚创建的 `chinook` 数据库：

``` psql
\c chinook
```

最后，导入 `chinook.sql` 文件来创建表并填充数据：

``` psql
\i chinook.sql
```

上述命令执行完成后，您可以运行以下查询来确认数据是否导入：

``` sql
TABLE album LIMIT 10;
```

``` title="Output"
 albumid |                 title                 | artistid
---------+---------------------------------------+----------
       1 | For Those About To Rock We Salute You |        1
       2 | Balls to the Wall                     |        2
       3 | Restless and Wild                     |        2
       4 | Let There Be Rock                     |        1
       5 | Big Ones                              |        3
       6 | Jagged Little Pill                    |        4
       7 | Facelift                              |        5
       8 | Warner 25 Anos                        |        6
       9 | Plays Metallica By Four Cellos        |        7
      10 | Audioslave                            |        8
(10 rows)
```

您现在可以退出 `psql`：

``` psql
\q
```

现在您已经有了一些可以使用的数据，让我们继续下一部分，您将了解在哪里可以找到 PostgreSQL 服务器生成的日志。

??? tip "提示"

    为了测试目的，您可以在 `psql` 中输入 `SET log_statement = 'all'`，以便所有数据库查询都记录在服务器日志中。

## PostgreSQL 日志存储在哪里？

PostgreSQL 服务器默认将其日志输出到标准错误流。您可以通过在 `psql` 中执行以下查询来确认这一点：

``` sql
SHOW log_destination;
```

``` title="Output"
 log_destination
-----------------
 stderr
(1 row)
```

它还提供了一个日志收集器进程，负责捕获发送到标准错误的日志并将其路由到日志文件。此日志收集器的行为由 `logging_collector` 参数控制。

让我们继续通过 psql 查看 `logging_collector` 参数的当前值：

```sql
SHOW logging_collector;
```

``` title="Output"
 logging_collector
-------------------
 off
(1 row)
```

默认情况下，`logging_collector` 是关闭的，因此 PostgreSQL 日志（发送到服务器的标准错误流）不会由日志收集器进程处理。这意味着日志的目的地取决于服务器的标准错误输出流的指向位置。

在 Ubuntu 上，您可以使用 `pg_lsclusters` 命令来查找计算机上运行的所有 PostgreSQL 集群的日志文件：

```bash
$ pg_lsclusters
```

**日志文件**列指示文件的位置：

``` title="Output"
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```

如果您的机器上已启用 `logging_collector`，您可以在 `psql` 中执行以下查询来找出文件的位置：

``` sql
SELECT pg_current_logfile();
```

``` title="Output"
   pg_current_logfile
------------------------
 log/postgresql-Thu.log
(1 row)
```

指定的文件（在本例中为 `log/postgresql-Thu.log`）与 PostgreSQL 数据目录相关，您可以通过输入以下查询来找到该目录：

``` sql
SHOW data_directory;
```

``` title="Output"
   data_directory
---------------------
 /var/lib/pgsql/data
(1 row)
```

本例中日志文件的完整路径是：

```
/var/lib/pgsql/data/log/postgresql-Thu.log
```

当您定位为到了 PostgreSQL 日志文件后，便可继续下一部分，我们将在浏览文件内容时可能遇到的各种类型的日志。

## PostgreSQL 日志的类型

基于您的配置，您可能会在日志文件中遇到各种类型的 PostgreSQL 日志。本部分将介绍其中最重要的内容以及它们的外观。

请注意，为了简洁起见，下面的示例有意省略了日志行前缀（以下 `<log_message>` 之前的所有内容）。仅显示 `<log_message>` 部分。

```
2023-07-30 08:31:50.628 UTC [2176] postgres@chinook <log_message>
```

### 1. 启动和关闭日志
 
这些日志描述了 PostgreSQL 服务器的启动和关闭过程。启动日志包括服务器版本、IP 地址、端口和 UNIX 套接字。它们还报告服务器上次关闭的时间并表明其已准备好接受新的连接：

```
LOG:  starting PostgreSQL 15.3 (Ubuntu 15.3-0ubuntu0.22.04.1) on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 11.3.0-1ubuntu1~22.04.1) 11.3.0, 64-bit
LOG:  listening on IPv4 address "127.0.0.1", port 5432
LOG:  listening on Unix socket "/var/run/postgresql/.s.PGSQL.5432"
LOG:  database system was shut down at 2023-07-27 17:45:08 UTC
LOG:  database system is ready to accept connections
```

另一方面，关闭日志是描述服务器关闭的原因和方式，这在调查意外的数据库故障时非常有用。下面的日志描述了当 `SIGINT` 信号发送到服务器时的快速关闭过程：

```
LOG:  received fast shutdown request
LOG:  aborting any active transactions
LOG:  background worker "logical replication launcher" (PID 82894) exited with exit code 1
LOG:  shutting down
LOG:  database system is shut down
```

### 2. 查询日志

这些日志代表针对 PostgreSQL 数据库执行的各种 SQL 查询，包括 `SELECT`、`INSERT`、`UPDATE`、`DELETE` 等。

```
STATEMENT:  select c.firstname, c.lastname, i.invoiceid, i.invoicedate, i.billingcountry
        from customer as c, invoice as i
        where c.country = 'Brazil' and
        select distinct billingcountry from invoice;
LOG:  statement: select distinct billingcountry from invoice;
LOG:  statement: select albumid, title from album where artistid = 2;
```

### 3. 查询持续时间日志

与查询日志密切相关的是持续时间日志，它跟踪查询完成所需的时间：

```
LOG:  duration: 13.020 ms
LOG:  duration: 13.504 ms  statement: UPDATE album SET title = 'Audioslave' WHERE albumid = 10;
```

### 4. 错误日志

错误日志可帮助您识别导致服务器出现错误情况的查询。这些情况包括查询特定的错误（例如约束违规或数据类型不匹配）以及更严重的问题（例如死锁和资源耗尽）。以下是此类日志的示例：

```
ERROR:  relation "custome" does not exist at character 15
STATEMENT:  select * from custome where country = 'Brazil';
ERROR:  duplicate key value violates unique constraint "pk_album"
DETAIL:  Key (albumid)=(10) already exists.
STATEMENT:  UPDATE album SET albumid = 10 WHERE albumid = 2;
```

### 5. 连接和断开连接日志

有关客户端连接和断开数据库的信息也可以记录在日志中。这些记录包括源 IP 地址、用户名、数据库名称和连接状态。如果发生断开连接，它们还包括会话总时长。

```
LOG:  connection received: host=[local]
LOG:  connection authenticated: identity="postgres" method=peer (/etc/postgresql/15/main/pg_hba.conf:90)
LOG:  connection authorized: user=postgres database=chinook application_name=psql
LOG:  disconnection: session time: 0:00:08.602 user=postgres database=chinook host=[local]
```

### 6. 检查点和预写日志 (WAL)

检查点表示预写日志 (WAL) 将修改后的数据从内存（共享缓冲区）刷新到磁盘的时刻。这些日志如下所示：

```
LOG:  checkpoint starting: time
LOG:  checkpoint complete: wrote 3 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.005 s, sync=0.002 s, total=0.025 s; sync files=2, longest=0.001 s, average=0.001 s; distance=0 kB, estimate=0 kB
```

## 开启日志收集器

如前所述，日志收集器是一个后台进程，它捕获发送到标准错误的日志消息并将其重定向到文件中。当您启用收集器时，PostgreSQL 日志将不再重定向到 `/var/log/postgresql/postgresql-15-main.log` 文件（在 Ubuntu 上），而是存储在单独的目录中。

在开始修改 PostgreSQL 配置之前，您需要使用以下查询在您的机器上找到它的位置：

``` sql
SHOW config_file;
```

``` title="Output"
               config_file
-----------------------------------------
 /etc/postgresql/15/main/postgresql.conf
```

使用 root 权限在您最喜欢的文本编辑器中打开文件路径：

```bash
$ sudo nano /etc/postgresql/15/main/postgresql.conf
```
该文件包含几个可配置的参数，注释以 `#` 字符开头。

现在，找到 `logging_collector` 选项，取消注释该行并将其打开：

``` title="/etc/postgresql/15/main/postgresql.conf"
logging_collector = on
```

保存文件，然后在终端中重新启动 PostgreSQL 服务器（确保在修改此文件中的任何设置后执行此操作）：

``` bash
$ sudo systemctl restart postgresql
```

之后，查看 `/var/log/postgresql/postgresql-15-main.log` 文件中的最后几行：

``` bash
$ tail /var/log/postgresql/postgresql-15-main.log
```

您应该看到两行内容，确认日志收集器已打开，并且将来的日志输出现在被放置在 `log` 目录中：

```
. . .
LOG:  redirecting log output to logging collector process
HINT:  Future log output will appear in directory "log".
```

该 `log` 由 `log_directory` 参数控制：


``` title="/etc/postgresql/15/main/postgresql.conf"
log_directory = 'log'
```

从上面的注释中可以看出，`log_directory` 控制着 PostgreSQL 日志的写入位置。当提供相对路径时，它相对于 `data_directory` 参数的值：

``` title="/etc/postgresql/15/main/postgresql.conf"
data_directory = '/var/lib/postgresql/15/main'
```

因此，我们可以推断日志文件应该位于 `/var/lib/postgresql/15/main/log/` 目录中。使用 root 权限执行以下命令来查找：

``` bash
$ sudo ls /var/lib/postgresql/15/main/log/
```

您应该观察到目录中至少有一个日志文件，表明日志收集器正常工作：

``` title="Output"
postgresql-2023-07-30_172237.log
```

在下一节中，您将学习如何自定义日志文件名和相关选项。

## 自定义日志文件

PostgreSQL 允许通过 `log_filename` 参数自定义日志文件名，默认的文件名如下：

``` title="/etc/postgresql/15/main/postgresql.conf"
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
```

此配置生成包含文件创建日期和时间的文件名。[Open Group 的 `strftime` 规范](https://pubs.opengroup.org/onlinepubs/009695399/functions/strftime.html)中列出了支持的 `%` 转义。

为了使本教程简单易懂，您可以将日志文件名更改为以下内容：

``` title="/etc/postgresql/15/main/postgresql.conf"
log_filename = 'postgresql-%a.log'
```

这将生成具有当天缩写名称的文件，如 `postgresql-Thu.log` 等。请注意，虽然文件名以 `.log` 结尾，但如果启用了 JSON 或 CSV 格式，则生成的文件将分别以 `.json` 和 `.csv` 结尾（例如 `postgresql-Thu.json`）。

如果您想控制每个日志文件的创建方式，您可以更改 `log_file_mode` 参数：

``` title="/etc/postgresql/15/main/postgresql.conf"
log_file_mode = 0600 # the default
```

`0600` 权限通常用于只有文件所有者才能访问和修改的敏感文件。通过此设置，您只能通过切换到 `postgres` 用户或使用 root 权限来访问、查看或修改此文件。这是一个很好的安全措施，因为 PostgreSQL 日志通常包含敏感信息。

## 设置日志文件轮换

您可能已经知道，如果没有适当的[日志轮换策略](https://betterstack.com/community/guides/logging/how-to-manage-log-files-with-logrotate-on-ubuntu-20-04/)，日志文件可能会很快耗尽所有磁盘空间，对于[打开查询日志记录](https://betterstack.com/community/guides/logging/how-to-start-logging-with-postgresql/#how-to-log-postgresql-queries)的 PostgreSQL 日志尤其如此。

幸运的是，PostgreSQL 提供了一些配置选项来控制其日志文件的轮换方式：

- `log_rotation_age`：这指定了日志文件在轮换之前的最大使用期限。默认值为 24 小时 (`1d`)。它可以接受带有单位的整数值，例如 `m` 表示分钟、`d` 表示天、`w` 表示周等。如果未提供单位，则假定为分钟。您还可以通过将其设置为 `0` 来禁用基于时间的日志轮换。
- `log_rotation_size`：此值决定了日志文件在轮换为新日志文件之前的最大大小。默认值为 10 兆字节 (`10MB`)。如果要禁用基于大小的日志轮换，请将其设置为 `0`。
- `log_truncate_on_rotation`：仅当对日志文件采用基于时间的轮换时此参数才适用。它用于覆盖任何现有的同名日志文件，而不是附加到文件。当您的 `log_filename` 参数类似于 `postgresql-%a.log` 时，这很有用，它会生成像 `postgresql-Mon.log`、`postgresql-Tue.log` 等文件。启用此选项（与将 `log_rotation_age` 设置为 `7d` 一起）可确保每周的日志覆盖前一周的日志，但文件名保持不变。

如您所见，上述选项提供了基本的日志文件轮换功能。 如果您需要更多自定义选项，例如自动压缩轮换文件的功能，请关闭 PostgreSQL 的日志文件轮换选项，然后使用 [`logrotate`](https://betterstack.com/community/guides/logging/how-to-manage-log-files-with-logrotate-on-ubuntu-20-04/) 程序。

## 日志格式

PostgreSQL 通过 `log_destination` 参数控制日志格式，默认值为 `stderr`。您还可以使用 `jsonlog` 和 `csvlog` 选项分别以 JSON 和 CSV 格式输出日志。稍后将提供更多信息。

``` title="/etc/postgresql/15/main/postgresql.conf"
log_destination = 'stderr'
```
当 `stderr` 启用时，您可以通过 `log_line_prefix` 参数修改其输出，该参数包含 `printf` 样式的字符串，用于确定日志信息头需要包含的内容。在 `psql` 中输入以下内容来找出其当前值：

``` sql
SHOW log_line_prefix;
```
``` title="Output"
 log_line_prefix
------------------
 %m [%p] %q%u@%d
(1 row)
```

该值确保发送到 `stderr` 的每个日志记录都包含以下详细信息：

- `%m`：事件的时间（以毫秒为单位）。
- `%p`：创建日志的特定 PostgreSQL 实例的进程 ID。
- `%q`：此标记不产生任何输出，但会通知后台进程（background processes）忽略此后的所有内容。后续的 `％u@％d` 仅在会话进程（backend processes）中可用。
- `%u`：触发事件的连接用户。
- `%d`：用户连接到的数据库。

使用上述格式生成的日志条目如下所示：

```
2023-07-30 08:31:50.628 UTC [2176] postgres@chinook LOG:  statement: select albumid, title from album where artistid = 2;
```

注意，`postgres@chinook` 之后的所有内容不受 `log_line_prefix` 控制，并且无法自定义。它由日志级别和日志消息组成，在本例中是 `SELECT` 语句。

### 自定义日志格式

修改 `stderr` 日志格式的唯一方法是通过 `log_line_prefix` 设置。请参阅[可用的转义序列的完整列表](https://www.postgresql.org/docs/current/runtime-config-logging.html#GUC-LOG-LINE-PREFIX)。我们建议包括以下变量：

- `%a`：应用程序名称（使用 PostgreSQL 连接字符串中的 `application_name` 参数）。
- `%p`：PostgreSQL 实例的进程 ID。
- `%u`：已连接的用户名。
- `%d`：数据库名。
- `%m`：事件发生的时间，以毫秒为单位（如果过您更喜欢 UNIX 纪元，则使用 `%n`）。
- `%q`：将仅会话变量与在所有上下文中有效的变量分开。
- `%i`：标识已执行的 SQL 查询的命令标签。
- `%e`：相关的 [PostgreSQL 错误代码](https://www.postgresql.org/docs/current/errcodes-appendix.html)。
- `%c`：当前会话 ID。

为了使日志更易于阅读和解析，您可以在每个变量前加上一个键，如下所示：

``` title="/etc/postgresql/15/main/postgresql.conf"
log_line_prefix = 'time=%m pid=%p error=%e sess_id=%c %qtag=%i usr=%u db=%d app=%a '
```

建议在结束引号前使用空格（或其他字符）将日志前缀与实际日志消息分开。完成此配置后，重新启动 PostgreSQL 服务器后您将看到以下格式的日志：

```
time=2023-07-30 22:02:27.929 UTC pid=17678 error=00000 sess_id=64c6ddf3.450e LOG:  database system is ready to accept connections
time=2023-07-30 22:03:33.613 UTC pid=17769 error=00000 sess_id=64c6de20.4569 tag=idle usr=postgres db=chinook app=psql LOG:  statement: UPDATE album SET title = 'Audioslave' WHERE albumid = 10;
```

上面的第一条记录是由非会话进程（又称后台进程）创建的。这些进程执行维护活动、后台任务和其他内部功能来支持数据库的运行。`%q` 转义的包含会排除其后的所有内容，因为这样的序列在非会话进程中无效。如不包含 `%q`，则 `tag`、`usr`、`db` 和 `app` 将出现在日志中但为空。

另一方面，会话进程（又名用户后端进程）是直接与特定用户和数据库绑定的进程，它们处理客户端应用程序发出的实际查询和命令。在这种情况下，所有转义序列都是有效的，因此 `%q` 将不起作用。

### 自定义时区

`log_timezone` 参数控制用于记录时间戳信息的时区。Ubuntu 中的默认值是 Etc/UTC，即 [UTC 时间](https://en.wikipedia.org/wiki/Coordinated_Universal_Time)。如果您希望时间与服务器时间相对应，请相应地更改此值：

``` title="/etc/postgresql/15/main/postgresql.conf"
log_timezone = 'Europe/Helsinki'
```

但是，[我建议坚持使用 UTC 时间](https://betterstack.com/community/guides/logging/log-formatting/#always-include-the-timestamp)，以便于日志关联和规范化。

## JSON 结构化日志

自 2008 年 2 月发布 [v8.3](https://www.postgresql.org/docs/release/8.3.0/) 以来，PostgreSQL 就一直支持 CSV 格式的日志记录，但直到最近在 [v15](https://www.postgresql.org/about/news/postgresql-15-released-2526/) 版本中才支持 JSON。您现在可以通过将 `jsonlog` 添加到 `log_destination` 参数中来将 PostgreSQL 日志格式化为 JSON 格式，如下所示：

``` title="/etc/postgresql/15/main/postgresql.conf"
log_destination = 'stderr,jsonlog'
```

当您重新启动服务器并查看日志目录时，您将观察到两个具有相同名称但不同扩展名（`.log` 和 `.json`）的新日志文件。这些文件包含相同的日志，但格式不同。`.log` 文件与以前一样遵循 `stderr` 格式，而 `.json` 文件可预见地包含 JSON 日志。

以下是具有相同日志的两个文件的比较：

``` title="/var/lib/postgresql/15/main/log/postgresql-Sun.log"
time=2023-07-30 22:48:01.817 UTC pid=18254 error=00000 sess_id=64c6e836.474e tag=idle usr=postgres db=chinook app=psql LOG:  statement: SHOW data_directory;
time=2023-07-30 22:49:21.808 UTC pid=18254 error=00000 sess_id=64c6e836.474e tag=idle usr=postgres db=chinook app=psql LOG:  statement: TABLE albu limit 10;
time=2023-07-30 22:49:21.808 UTC pid=18254 error=42P01 sess_id=64c6e836.474e tag=SELECT usr=postgres db=chinook app=psql ERROR:  relation "albu" does not exist at character 7
time=2023-07-30 22:49:21.808 UTC pid=18254 error=42P01 sess_id=64c6e836.474e tag=SELECT usr=postgres db=chinook app=psql STATEMENT:  TABLE albu limit 10;
```

这些日志根据前面演示的 `log_line_prefix` 参数进行格式化。另一方面，JSON 日志包含除空值字段之外的所有[可用字段](https://www.postgresql.org/docs/current/runtime-config-logging.html#RUNTIME-CONFIG-LOGGING-JSONLOG)：

``` title="/var/lib/postgresql/15/main/log/postgresql-Sun.json"
{"timestamp":"2023-07-30 22:48:01.817 UTC","user":"postgres","dbname":"chinook","pid":18254,"remote_host":"[local]","session_id":"64c6e836.474e","line_num":1,"ps":"idle","session_start":"2023-07-30 22:46:14 UTC","vxid":"4/3","txid":0,"error_severity":"LOG","message":"statement: SHOW data_directory;","application_name":"psql","backend_type":"client backend","query_id":0}
{"timestamp":"2023-07-30 22:49:21.808 UTC","user":"postgres","dbname":"chinook","pid":18254,"remote_host":"[local]","session_id":"64c6e836.474e","line_num":2,"ps":"idle","session_start":"2023-07-30 22:46:14 UTC","vxid":"4/4","txid":0,"error_severity":"LOG","message":"statement: TABLE albu limit 10;","application_name":"psql","backend_type":"client backend","query_id":0}
{"timestamp":"2023-07-30 22:49:21.808 UTC","user":"postgres","dbname":"chinook","pid":18254,"remote_host":"[local]","session_id":"64c6e836.474e","line_num":3,"ps":"SELECT","session_start":"2023-07-30 22:46:14 UTC","vxid":"4/4","txid":0,"error_severity":"ERROR","state_code":"42P01","message":"relation \"albu\" does not exist","statement":"TABLE albu limit 10;","cursor_position":7,"application_name":"psql","backend_type":"client backend","query_id":0}
```

敏锐的观察者会注意到 `error=42P01` 事件在 `stderr` 格式下包含两个日志条目。但是，同一事件以 JSON 格式则只出现在单个日志中。

以 [JSON 格式记录日志](https://betterstack.com/community/guides/logging/log-formatting/#prefer-structured-formats-over-plaintext)的明显优势是日志可以被日志管理工具自动解析和分析。但是，字段无法以任何方式进行自定义。您无法排除字段或自定义其名称，这一点与 `stderr` 格式不同，后者在这方面更加灵活。如果您确实需要这样的功能，您可以采用[日志发送器来转换日志](https://betterstack.com/community/guides/logging/log-shippers-explained/)，然后再将其发送到最终目的地。

其余示例将继续以 `stderr` 格式显示，但为了简洁起见，不显示 `log_line_prefix`。

## 如何记录 PostgreSQL 查询

`log_statement` 参数控制在服务器日志文件中记录哪些 SQL 查询。其有效选项如下：

- `none (default)`：不记录任何 SQL 查询。
- `ddl`：仅记录数据定义语言（DDL）语句，例如修改数据库架构的 `CREATE`，`ALTER`，和 `DROP` 语句。
- `mod`：除了 DDL 语句之外，此值还记录数据修改语言 (DML) 语句，例如 `INSERT`、`UPDATE` 和 `DELETE`。
- `all`：记录对服务器发出的所有 SQL 查询，除了由于解析错误而在执行阶段之前失败的查询（参见 `log_min_error_statement`）。

``` title="/etc/postgresql/15/main/postgresql.conf"
log_statement = mod
```

在生产中使用 `all` 设置将在繁忙的系统上生成大量数据，这可能会因将每个查询写入磁盘的开销而降低数据库的速度。如果您记录所有查询的动机是为了审计，请考虑使用 [pgAudit](https://github.com/pgaudit/pgaudit)，因为它为配置审计日志提供了更好的控制。

### 记录查询持续时间

记录查询日志的主要用例之一是识别需要优化的慢查询。但是，`log_statement` 参数不会在其输出中跟踪查询持续时间。您需要使用 `log_duration` 参数来实现此目的。它是一个布尔值，表示它可以是关闭（`0`）或打开（`1`）：

``` title="/etc/postgresql/15/main/postgresql.conf"
log_duration = 1
```

重启服务器并再次查看日志（译者注：这里可以直接重新加载配置文件即可生效）。您应该开始观察包含以毫秒为单位的持续时间值的条目：

``` hl_lines="2"
LOG:  statement: UPDATE album SET title = 'Audioslave' WHERE albumid = 10;
LOG:  duration: 18.498 ms
```

当打开 `log_duration` 时，服务器执行的所有语句都会生成持续时间日志，无一例外（即使是简单的 `SELECT 1`）。这将产生大量日志，可能损害性能并不必要地消耗磁盘空间。

`log_duration` 的另一个问题是它取决于 `log_statement` 参数，您可能不知道哪个查询产生了持续时间日志。为了测试这一点，请将 `log_statement` 设置为 `ddl`，并保持 `log_duration` 处于启用状态。重新启动服务器并在 `psql` 中输入以下查询：

``` sql
UPDATE album SET title = 'Audioslave' WHERE albumid = 10;
```

查询成功完成后，您将看到以下日志：

```
LOG:  duration: 13.492 ms
```

至关重要的是，产生该日志的语句根本没有被记录下来，这使得它实际上毫无用处。如果必须使用 `log_duration`，请确保 `log_statement` 也设置为 `all`，以便每个持续时间日志都可以与生成它的语句相关联。

另一种方法是关闭 `log_duration` 并启用 `log_min_duration_statement`：

``` title="/etc/postgresql/15/main/postgresql.conf"
log_duration = 0
log_min_duration_statement = 0
```

`log_min_duration_statement` 参数不是布尔值。它接受以毫秒为单位的整数值，并为执行时间超过指定值的查询生成日志。当它被设置为 `0`（如上所述）时，所有已完成语句的持续时间都会被记录。

应用上面更新的配置并重新启动 PostgreSQL 服务器，保持 `log_statement` 处于 `ddl` 模式。之后，重复 `UPDATE` 查询。您应该观察到如下所示的持续时间日志：

```
LOG:  duration: 13.504 ms  statement: UPDATE album SET title = 'Audioslave' WHERE albumid = 10;
```

这是对 `log_duration` 输出的改进，因为产生持续时间日志的查询是可见的。

但仍然存在一个弱点，那就是您无法在查询完成之前在日志文件中看到该查询。如果服务器在查询完成之前崩溃，您将不知道崩溃之前正在运行哪些查询，因为没有记录。

但是，如果查询生成自己的日志，日志记录行为就会发生变化。要观察这一点，请将 `log_statement` 更改为 `all`，然后重新启动服务器。在 `psql` 中运行 `SELECT pg_sleep(5)` 并观察日志。

您应该立即看到以下行：

```
LOG:  statement: SELECT pg_sleep(5);
```

五秒钟后，会出现下面一行：

```
LOG:  duration: 5005.975 ms
```

通过这种配置，如果在五秒钟过去之前发生问题，您至少会知道哪些查询可能在事件中发挥了作用。

[PostgreSQL 文档](https://www.postgresql.org/docs/current/runtime-config-logging.html#GUC-LOG-MIN-DURATION-STATEMENT)建议您在 `log_line_prefix` 中包含进程和会话 ID，以便您在读取和分析日志时可以轻松关联两个日志。


### 记录慢查询

记录服务器上所有查询的持续时间必然会使您的日志文件充斥着大量没有多大价值的琐碎日志。仅记录运行缓慢的查询更有用，以便可以相应地调查和优化它们。由于记录的日志量将大幅减少，因此它还可以减轻 PostgreSQL 服务器的压力。

慢查询的产生显然是取决于应用程序的类型和具体的工作负载。确保使用一个可以捕获可能导致性能瓶颈的资源密集型查询的值。连接大型表、执行复杂计算或涉及递归的查询是需要监控的主要候选对象。

对于交互式应用程序来说，几百毫秒的阈值可能是一个很好的起点，然后您可以随时进行调整：

``` title="/etc/postgresql/15/main/postgresql.conf"
log_min_duration_statement = 250
```

使用此配置，只有执行时间超过指定值（250ms）的查询才会被记录。例如：

``` sql
SELECT pg_sleep(1);
```

``` title="Output"
LOG:  statement: SELECT pg_sleep(1);
LOG:  duration: 1013.675 ms
```

## PostgreSQL 中的消息严重性级别

PostgreSQL 使用以下日志级别来表明其生成的每个日志记录的严重性：

- `DEBUG5`，`DEBUG4`，`DEBUG3`，`DEBUG2`，`DEBUG1`：这些级别用于记录对故障排除有用的详细信息，其中 `DEBUG5` 是最详细的，而 `DEBUG1` 是最不详细的。
- `INFO`：报告用户隐式请求的信息。
- `NOTICE`：报告可能对用户有帮助的详细信息。
- `WARNING`：警告潜在问题，例如在事务块之外使用 `COMMIT`。
- `ERROR`：报告中止当前命令的错误。
- `LOG`：报告一般数据库活动。
- `FATAL`：报告中止当前会话但其他会话保持活动状态的错误。
- `PANIC`：报告中止所有现有数据库会话的严重错误。

以下是标有各个严重性级别的消息的示例：

```
DEBUG:  server process (PID 13557) exited with exit code 0
INFO:  vacuuming "chinook.public.album"
NOTICE:  identifier "very_very_very_very_very_very_very_very_long_table_name_with_more_than_63_characters" will be truncated to "very_very_very_very_very_very_very_very_long_table_name_with_mo"
WARNING:  SET LOCAL can only be used in transaction blocks
LOG:  statement: UPDATE album SET title = 'Audioslave' WHERE albumid = 10;
ERROR:  relation "albu" does not exist at character 7
FATAL:  role "root" does not exist
PANIC: database system shutdown requested
```

默认日志级别通过 `log_min_messages` 设置设置为 `WARNING`，我建议保持这种状态。

``` title="/etc/postgresql/15/main/postgresql.conf"
log_min_messages = warning
```

## 减少日志详细程度

PostgreSQL 允许您通过 `log_error_verbosity` 设置来控制事件日志的详细程度，与其名称所暗示的不同，该设置适用于每条已记录的消息。可能的值包括 `terse`、`default` 和 `verbose`。

使用 `default` 设置，您将看到某些查询带有额外的 `DETAIL`、`HINT`、`QUERY` 或 `CONTEXT` 信息。

```
LOG:  statement: DROP table album;
ERROR:  cannot drop table album because other objects depend on it
DETAIL:  constraint fk_trackalbumid on table track depends on table album
HINT:  Use DROP ... CASCADE to drop the dependent objects too.
STATEMENT:  DROP table album;
```

此处所有五个条目均由同一查询产生。如果 `log_error_verbosity` 设置为 `terse`，则会删除 `DETAIL` 和 `HINT` 行。选项 `verbose` 则将包括 `default` 的所有内容以及有关错误的更低级详细信息，例如其源代码文件名、函数名称和行号：

```
LOCATION:  reportDependentObjects, dependency.c:1189
```

请注意，如果您以 JSON 格式记录日志，则 `DETAIL`、`HINT` 和 `STATEMENT` 将包含在同一个日志条目中，并且 `LOCATION` 将分为 `func_name`、`file_name` 和 `file_line_num` 属性：

``` json
{"timestamp":"2023-08-03 15:56:26.988 UTC","user":"postgres","dbname":"chinook","pid":14155,"remote_host":"[local]","session_id":"64cbce27.374b","line_num":5,"ps":"DROP TABLE","session_start":"2023-08-03 15:56:23 UTC","vxid":"3/8","txid":16414,"error_severity":"ERROR","state_code":"2BP01","message":"cannot drop table album because other objects depend on it","detail":"constraint fk_trackalbumid on table track depends on table album","hint":"Use DROP ... CASCADE to drop the dependent objects too.","statement":"DROP table album;","func_name":"reportDependentObjects","file_name":"dependency.c","file_line_num":1189,"application_name":"psql","backend_type":"client backend","query_id":0}
```

## 记录连接和断开连接

以下设置控制 PostgreSQL 如何记录客户端连接和断开连接。默认情况下，这两项均处于关闭状态：

``` title="/etc/postgresql/15/main/postgresql.conf"
log_connections = off
log_disconnections = off
```

开启 `log_connections (1)` 将记录每次尝试连接到服务器以及客户端身份验证和授权的完成情况：

``` title="/etc/postgresql/15/main/postgresql.conf"
log_connections = 1
```

```
LOG:  connection received: host=[local]
LOG:  connection authenticated: identity="postgres" method=peer (/etc/postgresql/15/main/pg_hba.conf:90)
LOG:  connection authorized: user=postgres database=chinook application_name=psql
```
从安全角度来看，这些信息很有用，因为您可以看到谁在连接到您的系统以及从哪里连接。如果与服务器有多个短暂的连接并且未使用连接池，则会导致大量日志。

`log_disconnections` 参数的工作原理类似，它包括会话持续时间：

``` title="/etc/postgresql/15/main/postgresql.conf"
log_disconnections = 1
```

```
LOG:  disconnection: session time: 0:00:02.824 user=postgres database=chinook host=[local]
```

## 创建自定义日志消息

到目前为止，我们已经讨论了 PostgreSQL 服务器生成的各种类型的日志以及如何定制它们。如果您想在 SQL 查询中或在使用 [PL/pgSQL](https://www.postgresql.org/docs/current/plpgsql-overview.html) 编写触发函数时创建自己的日志消息，您可以按如下方式使用 [RAISE](https://www.postgresql.org/docs/current/plpgsql-errors-and-messages.html) 语句：

``` plpgsql
CREATE OR REPLACE FUNCTION custom_logs_func()
RETURNS TRIGGER LANGUAGE PLPGSQL AS $$
BEGIN
    RAISE LOG 'This is an informational message';
    RAISE WARNING 'Something unexpected happened';
    RAISE EXCEPTION 'An unexpected error';
END;
$$;

CREATE OR REPLACE TRIGGER UPDATE_LAST_EDITED_TIME BEFORE UPDATE ON ALBUM
FOR EACH ROW EXECUTE FUNCTION custom_logs_func();
```

根据您的 `log_min_messages` 设置，在执行 `custom_logs_func()` 时，您将在日志文件中观察到以下内容：

```
LOG:  This is an informational message
WARNING:  Something unexpected happened
ERROR:  An unexpected error
```

## 使用 PgBadger 分析日志文件

PgBadger 是 PostgreSQL 日志文件的命令行日志分析工具。它旨在解析和分析 PostgreSQL 日志文件，提取有价值的见解并生成有关数据库活动、性能和使用模式的详细报告。它可以生成包含大量统计数据的综合 HTML 报告，例如：

- 解析和处理的日志条目总数。
- 最慢的查询及其执行时间。
- 随时间变化的连接数。
- 最常见的错误消息及其发生的次数。
- 事务的提交和回滚统计信息。
- 临时文件使用率最高的表。
- 查询和会话持续时间的直方图。
- 涉及热门查询的用户和应用程序。
- 最耗时的 `prepare`/`bind` 查询。
- 自动清理进程的数量及其执行时间。
- 以及更多！

安装 `pgbadger` 后，请查看其[推荐的日志配置设置](https://github.com/darold/pgbadger#postgresql-configuration)，并在 PostgreSQL 配置文件中进行相应的配置。请注意，这些设置是为了产生尽可能多的信息，以便 pgBadger 可以有效地分析数据库活动。

该工具提供了许多选项，但生成 HTML 报告的最基本设置是提供配置的日志行前缀、日志文件以及报告的所需名称/位置。在撰写本文时，它仅适用于 `stderr` 和 CSV 格式，但不适用于 JSON。

``` bash
$ sudo pgbadger --prefix 'time=%m pid=%p error=%e sess_id=%c %qtag=%i usr=%u db=%d app=%a ' /var/lib/postgresql/15/main/log/postgresql-Thu.log -o postgresql.html
```

``` title="Output"
[========================>] Parsed 289019 bytes of 289019 (100.00%), queries: 102, events: 42
LOG: Ok, generating html report...
```

当您在浏览器中打开 HTML 文件时，您应该会看到如下报告：

![](imgs/pgbadger.avif)

请参阅 [PgBadger 网站上的示例](https://pgbadger.darold.net/examples/report/index.html)及其[文档](https://pgbadger.darold.net/documentation.html)以了解更多详细信息。

## 总结

如果配置正确，PostgreSQL 日志是深入了解数据库活动和性能的有效方法。这也许不是一件光鲜的事情，但坚持下去肯定会获得回报。正确设置可能需要一些反复试验，但我希望本文能为您指明正确的方向。

> 作者：Ayooluwa Isaiah<br>
> 原文：https://betterstack.com/community/guides/logging/how-to-start-logging-with-postgresql/


