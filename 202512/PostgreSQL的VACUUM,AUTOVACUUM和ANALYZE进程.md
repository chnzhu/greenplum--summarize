# PostgreSQL的VACUUM,AUTOVACUUM和ANALYZE进程

# 问题
在本[系列的第一部分](https://www.mssqltips.com/sqlservertip/8103/postgresql-vacuum-autovacuum-analyze-deleted-data/)中，我们探讨了PostgreSQL VACUUM执行的各种功能、手动运行的功能和选项，以及其背后的理论，以及PostgreSQL如何管理并发。第二部分则是一个更实用的建议，重点讲解它是什么以及如何调优，而第一部分则更多讲述了VACUUM的工作原理以及MVCC在PostgreSQL中实现的原理。

# 解决方案

这个建议将详细讨论AUTOVACUUM和ANALYZE以及如何调校它们。还将分享如何使用脚本监控数据库中的表和索引膨胀、包裹性问题及其他与VACUUM相关的问题。

### PostgreSQL AUTOVACUUM

我们在第一部分结尾引入了AUTOVACUUM，这是一个自动检查需要vacuumed或analyzed的表，并防止txid环绕的过程。该过程默认在 PostgreSQL 集群中所有数据库的表和索引上启用，因此在所有关系上也都启用。我们可以选择性地在表上禁用AUTOVACUM，并用各种参数控制单个表或整个集群的进程行为。

具体来说，我们有几个参数告诉AUTOVACUUM何时需要VACUUM，还有一些参数控制它能使用多少资源的速度。让我们来看看其中一些。

AUTOVACUUM 由启动器和多个工作进程组成。最多允许autovacuum_max_workers个工作进程。该参数控制能够同时执行AUTOVACUUM作业的工人（线程）数量。

启动器每 autovacuum_naptime 秒启动一名工作者，随后工作者检查每个表中的插入、更新和删除，并根据插入的阈值和相应参数执行 VACUUM 和/或分析。

以下阈值可以在表格基础上或全局修改：

- `autovacuum_vacuum_threshold`：默认值为50。
- `autovacuum_vacuum_scale_factor`：默认值为0.2。

控制表是否应被AUTOVACUUM的vacuumed的公式为：pg_stat_user_tables.n_dead_tup > (pg_class.reltuples x autovacuum_vacuum_scale_factor) + autovacuum_vacuum_threshold

例如，在一个有10,000行的表中，dead rows数必须超过2,050（（10,000 x 0.2）+ 50），AUTOVACUUM才会启动。

让我们看看实际作。使用第一部分中使用的表格，test_vacuum，我们首先模拟大量更新，保持标准参数。我们可以通过 pgAdmin 图形界面轻松查看表格的所有参数，点击"表格属性"，查看参数标签：

![](https://fastly.jsdelivr.net/gh/bucketio/img12@main/2025/11/29/1764428919079-ad483d44-a477-463e-9661-132750ef3d0e.png)

通过在目录视图pg_stat_all_tables查询，我们可以查出该表上次被vacuumed或autovacuumed（以及分析）的时间：

```sql
--MSSQLTips.com
 
select schemaname, relname,last_vacuum,last_autovacuum, last_analyze,last_autoanalyze 
from pg_stat_all_tables
where relname='test_vacuum';
```

![](https://fastly.jsdelivr.net/gh/bucketio/img18@main/2025/11/29/1764428983326-fb1afe71-12d2-475c-b8d0-e10587dd772d.png)

我们还需要检查参数autovacuum_naptime，以了解AUTOVACUUM过程启动的频率：

```sql
--MSSQLTips.com
 
show autovacuum_naptime;
```

![](https://fastly.jsdelivr.net/gh/bucketio/img8@main/2025/11/29/1764429774417-be842556-8ced-410c-983b-27292a4f52ac.png)

正如预期的，默认是：1分钟。

既然我们已经确认情况干净，接下来做几次表更新，以确定有若干dead tuples：

```sql
--MSSQLTips.com
 
update test_vacuum
set descrizione='test_vacuum 25'
where (id % 10) = 0;
```

![](https://fastly.jsdelivr.net/gh/bucketio/img14@main/2025/11/29/1764429822670-e089d588-9e22-40d3-a8fe-4b712580c350.png)

我们等了1分钟，再次向pg_stat_all_tables发出查询：

正如预期的那样，我们没有看到任何变化：

![](https://fastly.jsdelivr.net/gh/bucketio/img6@main/2025/11/29/1764429842183-48d1ea23-f338-40c4-a4ad-1905128cd05d.png)

事实上，我们只看到10,000个死元组，这些元组会在上一个查询中添加一列：

```sql
--MSSQLTips.com
 
select schemaname, relname,last_vacuum,last_autovacuum, last_analyze,last_autoanalyze, n_dead_tup 
from pg_stat_all_tables
where relname='test_vacuum';
```

![](https://fastly.jsdelivr.net/gh/bucketio/img3@main/2025/11/29/1764429870067-0583f652-bbba-46c6-9fb8-9455be9a6eee.png)

为了按照上述公式启动自动真空，我们至少需要 （（100,000 x 0.2） + 50） = 20,050 行。所以，我们需要添加更多死元组来看看会发生什么：

```sql
--MSSQLTips.com
 
update test_vacuum
set descrizione='test_vacuum 27'
where (id % 3) = 0;
```

![](https://fastly.jsdelivr.net/gh/bucketio/img18@main/2025/11/29/1764429898268-5097099d-8d9d-4841-9922-658b3a107b62.png)

我们再在1分钟后再提一次之前的问题：

![](https://fastly.jsdelivr.net/gh/bucketio/img7@main/2025/11/29/1764429915026-fb743249-7a98-4d65-9ee0-eb130321a914.png)

我们终于看到AUTOVACUUM被启动，清理了所有死元组，并且用AUTOANALYSIS分析了表，最终更新了表的统计数据（稍后会详细说明ANALYZE）。

现在我们想节奏，让它在这张特定桌子上表现得更频繁。为此，我们通过发布 ALTER 表来修改该表的参数：

```sql
--MSSQLTips.com
 
ALTER TABLE public.test_vacuum SET (
    autovacuum_vacuum_scale_factor = 0.1,
   autovacuum_vacuum_threshold=0);
```

![](https://fastly.jsdelivr.net/gh/bucketio/img1@main/2025/11/29/1764429975014-40941ecb-56ad-448d-841e-090f770dbc99.png)

再次尝试更新更多10,000行，以获得超过10,000个死元组（新的阈值）：

```sql
--MSSQLTips.com
 
update test_vacuum
set descrizione='test_vacuum test'
where (id % 9) = 0;
```

![](https://fastly.jsdelivr.net/gh/bucketio/img6@main/2025/11/29/1764429998749-0eb275be-957b-4639-8373-ef96d4dd5da2.png)

下图显示，1分钟后，在该表上执行AUTOVACUUM，所有死元组被删除：

![](https://fastly.jsdelivr.net/gh/bucketio/img4@main/2025/11/29/1764430015443-13343ade-1f02-4ac5-ad0d-69ff23fa9420.png)

# 在PostgreSQL中调优AUTOVACUUM

到目前为止，我们研究了如何为单个表格调校AUTOVACUUM，使其运行频率高于默认设置。关于AUTOVACUUM的一个重要点是，即使它只对表做真空，而不是FULL/FREEZE（取决于FREEZE的阈值是否达到），它依然是一个资源密集型过程。所以，在拥有大量大表和高负载的大型数据库中，完成可能需要一些时间和资源。

这是有意为之——避免使用过多的I/O资源。在此类 OLTP 数据库中（UPDATE 和 DELETE 进程数量众多），我们可能需要调整进程以加快运行速度，避免表中出现过多臃肿。

我们有几个可以利用的参数：

- `autovacuum_max_workers`：如上所述，有一个参数autovacuum_max_workers，默认值为3，控制AUTOVACUUM使用的总工人数量。例如，我们可以增加这个数字，让更多并行工作者参与。当处理大型分区表时，这一作非常高效，因为工作可以在多个分区上更好地并行化。这也是增加autovacuum workers数量的唯一原因，正如我们稍后会看到的。
- `autovacuum_vacuum_cost_limit`：默认值为200，是AUTOVATUMM可达到的总成本上限。这个限制可以提高。
- `autovacuum_vacuum_cost_delay`：默认值为2毫秒。这个数值可以被减少，甚至设置为0。在这种情况下，它会让自动吸尘器达到和手动吸尘器一样快，也就是尽可能快。
- `vacuum_cost_page_hit`：默认值为1。
- `vacuum_cost_page_miss`：默认值为10。
- `vacuum_cost_page_dirty`：默认值为20。

`autovacuum_vacuum_cost_limit`参数基于最后三个成本的组合。请注意，这些分额在所有运行autovacuum_max_workers中均分：

```sql
individual thread's cost_limit = autovacuum_vacuum_cost_limit / autovacuum_max_workers
```

因此，增加autovacuum_max_workers可能会反直觉地延迟当前运行的AUTOVACUUM工作者的AUTOVACUUM执行，因为工作线程数量增加会降低每个线程的成本限制。每个线程随后被分配较低的成本限制，因此当成本阈值容易达到时，它会更频繁地进入睡眠，导致整个AUTOVACUUM进程运行较慢。


相反，增加 autovacuum_vacuum_cost_limit 参数可能会导致 IO 瓶颈，因为所有 VACUUM 进程都是 IO 密集型的。对于那些有大量 DML 操作且需要更密集 VACUUM 的特定大型表，最好是微调这些参数，而不是在 postgresql.conf 文件中全局更改这些参数。

让我们试试一个例子，调整我们的工作台，让AUTOVACUUM运行得更快——不仅更频繁，而且更快！

为了监控 AUTOVACUUM 的持续时间，我们需要修改日志参数 log_autovacuum_min_duration。这样，我们会记录所有持续时间超过该参数值（以毫秒为单位）的 AUTOVACUUM。此参数可以在 postgresql.conf 中修改，适用于整个 PostgreSQL 集群，或者仅通过执行 ALTER TABLE 来针对某个表进行修改：

```sql
--MSSQLTips.com
 
alter table public.test_vacuum set (log_autovacuum_min_duration=1);
```

![](https://fastly.jsdelivr.net/gh/bucketio/img15@main/2025/11/30/1764515094068-01b292fa-a6bc-4bdc-8bf7-23d2d0359abd.png)

这样一来，该表上所有持续时间超过 1 毫秒的自动清理（AUTOVACUUM）操作都会被记录下来。现在我们可以向该表中添加更多行，以获得更具代表性且规模更大的数据集，这样我们就能切实看到自动清理速度的差异了。要记住，这个插入操作本身会促使自动清理在该表上启动：

```sql
--MSSQLTips.com
 
insert into test_vacuum values (generate_series(1000000,11000000), 'Test vacuum new');
```

![](https://fastly.jsdelivr.net/gh/bucketio/img5@main/2025/11/30/1764515143567-bd1564f3-4ed0-4ee7-b86b-d3cea19b687e.png)

查看日志：

![](https://fastly.jsdelivr.net/gh/bucketio/img3@main/2025/11/30/1764515163590-6c1a9e53-d967-44ff-a78d-c1b7b251b6e1.png)

我们这里有大量信息。我们现在感兴趣的是经过时间13.06秒。我们发送一个更新，生成一些死元组并触发另一个AUTOVACUUM：

```sql
--MSSQLTips.com
 
update test_vacuum
set descrizione='test_vacuum test'
where (id % 9) = 0;
```

![](https://fastly.jsdelivr.net/gh/bucketio/img11@main/2025/11/30/1764515272878-3bdbcbc7-494b-4153-8dc2-5147b6b144d3.png)

再检查一次日志：

![](https://fastly.jsdelivr.net/gh/bucketio/img8@main/2025/11/30/1764515306236-7233c5d1-214f-4d0b-9054-7ed00b7617bc.png)

我们可以看到，AUTOVACUUM 清理所有死元组耗时 21.94 秒。让我们修改参数，为 test_vacuum 表的 AUTOVACUUM 分配更多资源 —— 提高成本限制并降低延迟：

```sql
--MSSQLTips.com
 
ALTER TABLE public.test_vacuum SET (autovacuum_vacuum_cost_limit = 500, autovacuum_vacuum_cost_delay = 1);
```

是时候再次发布更新，看看新的AUTOVACUUM功能：

```sql
--MSSQLTips.com
 
update test_vacuum
set descrizione='test_vacuum test 2'
where (id % 9) = 0;
```

![](https://fastly.jsdelivr.net/gh/bucketio/img2@main/2025/11/30/1764515382106-289038c1-d6da-47a8-864f-536ee7943f67.png)

![](https://fastly.jsdelivr.net/gh/bucketio/img14@main/2025/11/30/1764515397676-ebb5e75d-d917-4144-88f3-caeb5843ec5b.png)

正如我们所见，在死元组数量相同的情况下，耗时降至 4.17 秒，这是一个巨大的提升。即便冻结元组的数量更少，这个 AUTOVACUUM（自动清理）无论如何都会更快。实际上，平均读写速率分别为 121.771 MB/s 和 123.940 MB/s，而之前的速率分别是 22.598 MB/s 和 27.872 MB/s！


### 监控 PostgreSQL AUTOVACUUM 进程、表膨胀和环绕

有一些有用的查询可用于监控自动清理（AUTOVACUUM）的执行情况以及对其（或清理（VACUUM））的需求。其中一些已经介绍过了。另一个非常重要的查询是在自动清理运行时专门使用的 pg_stat_progress_vacuum，即通过该视图来实现。

让我们在测试表上再触发一次AUTOVACUUM，并更新一下，看看它的实际作：

```sql
--MSSQLTips.com
 
select pid,
  now() - xact_start AS duration,
  vacprog.datname AS database,
  relid::regclass AS table,
  phase, index_vacuum_count,
  num_dead_tuples, max_dead_tuples
from pg_stat_progress_vacuum as vacprog
JOIN pg_stat_activity as active using (pid);
```

![](https://fastly.jsdelivr.net/gh/bucketio/img7@main/2025/11/30/1764515585233-b95b1b33-a4cb-42cb-bbaa-195a011ff1cf.png)

我们可以看到我们捕捉到了这个过程的最开始阶段：仍处于扫描阶段，正在统计死元组，持续时间还不到一秒。这是一个查看大型数据库 / 表上的 AUTOVACUUM 在运行时的进度的好方法。请注意，它还会指示是否对索引执行该过程，因为索引与表的处理方式相同。

另一个用于识别交易ID环绕可能性的有用查询是：

```sql
--MSSQLTips.com
 
select datname, age(datfrozenxid) as frozen_xid_age,
round(100*(age(datfrozenxid)/2146483647.0::float)) as consumed_txid_pct,
current_setting('autovacuum_freeze_max_age')::int - age(datfrozenxid) as remaining_aggressive_vacuum
from pg_database;
```

![](https://fastly.jsdelivr.net/gh/bucketio/img16@main/2025/11/30/1764515620949-b64ce15c-ac91-4ec5-ae44-6f4001096954.png)

这个查询能让我们立即了解测试数据库中 PostgreSQL 集群的状况。我们有充足的 txid，并且远未达到任何环绕情况。

我上面已经介绍了一个用于监控死元组的查询，以及自动清理（AUTOVACUUM）/ 清理（VACUUM）和自动分析（AUTOANALYZE）/ 分析（ANALYZE）的最近执行情况。但为了更清楚地了解表膨胀问题，需要安装一个特定的扩展 ——pgstattuple：

```sql
--MSSQLTips.com
 
CREATE EXTENSION pgstattuple;
```

![](https://fastly.jsdelivr.net/gh/bucketio/img0@main/2025/11/30/1764515676033-f1258dbd-4128-48e8-86be-96e106b95c01.png)

此时，我们可以使用扩展来检查测试表:

```sql
--MSSQLTips.com
 
select pg_size_pretty(table_len) as table_size, tuple_count, pg_size_pretty(tuple_len) as live_tuple_size, 
tuple_percent as pct_live_tuples,dead_tuple_count, pg_size_pretty(dead_tuple_len) as dead_tuple_size,
dead_tuple_percent, pg_size_pretty(free_space) as free_space_size, free_percent
from pgstattuple('test_vacuum');
```

![](https://fastly.jsdelivr.net/gh/bucketio/img9@main/2025/11/30/1764515702985-85d11d2a-23fc-40b8-9d86-fa21ca241853.png)

如你所见，我们拥有关于表的大小、死元组和活元组的数量以及百分比的信息。最重要的数据是死元组的百分比和大小，以及表的空闲大小。这些数据让我们了解表的膨胀程度，也就是说，让我们知道是否需要执行 VACUUM 或 VACUUM FULL 操作。例如，我们可以运行一个简单的查询来获取数据库中膨胀最严重的表的列表，然后使用下面的查询对它们进行更深入的分析：

```sql
--MSSQLTips.com
 
select relname,(pgstattuple(oid)).dead_tuple_percent 
from pg_class 
where relkind = 'r' 
order by dead_tuple_percent desc;
```

![](https://fastly.jsdelivr.net/gh/bucketio/img5@main/2025/11/30/1764515742539-add3e068-e1a4-4215-9eb9-c43a1edf515b.png)

这组结果让我们了解需要进一步研究的表，或许还需要对 AUTOVACUUM 参数进行微调。

### PostgreSQL 分析与自动分析进程

我们刚才针对 AUTOVACUUM 提出的相同考量和建议也适用于 AUTOANALYZE 进程。但首先，我们需要更详细地解释 ANALYZE 和 AUTOANALYZE 的作用！

如前所述，ANALYZE 操作会检查所有表的内容，并收集每个表中各列的值分布统计信息。PostgreSQL 查询引擎利用这些统计信息来找到最佳的查询计划。当数据库中的行被插入、删除和更新时，列的统计信息也会发生变化。在 AUTOVACUUM 之后自动运行的 ANALYZE 确保了统计信息会得到更新。注意：如果我们改为手动运行 ANALYZE，就会重新生成统计信息，这会给 PostgreSQL 集群的 I/O 资源带来更大压力。这也是不运行手动 VACUUM ANALYZE 的一个原因。

控制AUTOANALYSIS的参数与上文AUTOVACUUM的参数相呼应：

- autovacuum_analyze_threshold
- autovacuum_analyze_scale_factor

与autovacuum_vacuum参数含义相同，我们可以在测试表上更改它们。请记住，AUTOVACUUM之后总是会跟随AUTOANALYSIS，以保持统计数据的更新。

```sql
--MSSQLTips.com
 
ALTER TABLE public.test_vacuum SET (
    autovacuum_analyze_scale_factor = 0.1,
    autovacuum_analyze_threshold=1);
```

![](https://fastly.jsdelivr.net/gh/bucketio/img4@main/2025/11/30/1764515863935-9099004b-c5f0-417a-aad3-ebd05fb58192.png)

正如你从下面的这个简单查询中看到的，我们也修改了AUTOANALYSIS的参数，即使它是在AUTOVACUUM后自动触发的：

```sql
--MSSQLTips.com
 
select schemaname, relname,last_vacuum,
last_autovacuum, last_analyze,last_autoanalyze, 
n_dead_tup 
from pg_stat_all_tables
where relname='test_vacuum';
```

![](https://fastly.jsdelivr.net/gh/bucketio/img1@main/2025/11/30/1764515887341-ffb4ed6d-6a51-410c-be99-bec5d27ee469.png)

last_autoanalyze 是在 last_autovacuum 完成后立即执行的。

### 关于AUTOVACUUM/VACUUM和AUTOANALYZE/ANALYZE流程需要注意的事项

正如我们所指出的，这些流程是 I/O 资源密集型的。因此，除非处于特定场景，否则最好不要执行手动 VACUUM，而是将其交由 AUTOVACUUM 进程来处理。正如我们所了解到的，如果某些表有特定需求，建议对这些表的 AUTOVACUUM 进程进行微调。

一种特定的情况是，在对表执行大量 INSERT 或 DELETE 操作，导致表及其索引的大小发生显著变化后，对该表运行 VACUUM ANALYZE 是有意义的。由于这是一个单一操作，而且我们不想更改该特定表的 AUTOVACUUM 参数，也不想等待它启动，所以手动运行它是合理的。始终要记住，对于大型表来说，这会给资源带来一定压力！

一种特定的情况是，在对表执行大量 INSERT 或 DELETE 操作，导致表及其索引的大小发生显著变化后，对该表运行 VACUUM ANALYZE 是有意义的。由于这是一个单一操作，而且我们不想更改该特定表的 AUTOVACUUM 参数，也不想等待它启动，所以手动运行它是合理的。始终要记住，对于大型表来说，这会给资源带来一定压力！

### 下一步

在本提示中，我们回顾了PostgreSQL VACUUM最重要的流程之一，分享了有关如何优化自动AUTOVACUUM和AUTOANALYZE的建议，以及许多用于监控这些进程和表/索引膨胀情况的脚本。

一如既往，以下是官方文档的链接：

- [AUTOVACUUM daemon](https://www.postgresql.org/docs/current/routine-vacuuming.html#AUTOVACUUM)
- [pg_stat_progress_vacuum 以及所有进度目录视图](https://www.postgresql.org/docs/current/progress-reporting.html)
- [pgstattuple](https://www.postgresql.org/docs/current/pgstattuple.html)

