# PostgreSQL中用于处理已删除数据的VACUUM、AUTOVACUUM和ANALYZE进程

# 问题

PostgreSQL 处理删除操作的方式很独特，它会将行标记为过时。它还使用多版本并发控制（MVCC）来管理并发，这与 Oracle 或 SQL Server 的方式不同。为了使这种管理正常工作，它具备独特的功能和流程，例如 VACUUM 和 AUTOVACUUM。

# 解决方式

在这个分为两部分的技巧中，我们将了解 PostgreSQL 中的 VACUUM 和 AUTOVACUUM 进程是什么，如何对它们进行适当的调优，以及 PostgreSQL 如何通过 MVCC 实现并发，包括 PostgreSQL 如何在内存页中存储数据的相关见解。同时，我们还将研究 ANALYZE 进程。

## VACUUM

VACUUM 程序用于回收表中“dead tuples” 所占据的空间。当记录被删除或更新（先删除后插入）时，就会产生死元组。PostgreSQL 不会从表中物理删除旧行，而是为其添加一个 “标记”，使查询不会返回该行。

随着时间的推移，这些行空间会变得过时，并导致碎片化和膨胀。当清理进程运行时，这些失效元组所占据的空间会被标记为可被其他元组重用，从而在需要时有助于缩小数据文件的大小。

VACUUM命令能执行多项操作，不仅仅是回收或重新利用由废弃行占用的磁盘空间。实际上，它还能：

- 使用ANALYZE更新数据统计信息
- 更新可见性映射，这会加快仅索引扫描（稍后会详细介绍）
- 防止因事务ID回卷而丢失极旧的数据（稍后会详细介绍）

VACUUM命令可以带各种选项运行，两种常见模式是VACUUM和VACUUM FULL。

VACUUM仅删除死行，并将空间标记为可供未来重新使用。需要明确的是，此操作不会将空间返还给操作系统；只有当废弃行位于表的末尾时，使用常规VACUUM才能回收空间。

与 VACUUM 相比，VACUUM FULL 采用了一种更激进的算法。类似于 SQL Server 中的 SHRINK，它通过写入一个全新的、没有死空间的表文件来压缩表。显然，这会花费更多时间，并且在操作完成之前，需要额外的磁盘空间来存放表的新副本。此外，还需要对表的独占访问权，这会阻塞对表的 DML 操作。这是通过在 VACUUM FULL 操作期间对表施加显式的独占锁来实现的。

现在，是时候给出手动执行 VACUUM 的实际示例了。首先，我们需要确保数据库中已创建了 PAGEINSPECT 扩展，因为我们需要用它来检查内存页：

```sql
--MSSQLTips.com
 
CREATE EXTENSION pageinspect;
```

接下来，我们可以创建一个新表并插入一些数据：

```sql
--MSSQLTips.com
 
create table test_vacuum_0 (id bigint, descrizione varchar(20));
insert into test_vacuum_0 values (1,'Test Vacuum');
```

让我们来看看它的大小：

```sql
--MSSQLTips.com
 
select pg_size_pretty
(pg_relation_size('test_vacuum_0'));
```

![](https://fastly.jsdelivr.net/gh/bucketio/img3@main/2025/12/03/1764732017765-f2a4724d-3cd8-4c59-92bb-475779bdb9ca.png)

8Kb只是磁盘上的一个内存页。现在，让我们更新这一行以创建一个死元组：

```sql
--MSSQLTips.com
 
update test_vacuum_0
set descrizione='Test'
where id=1;
```

![](https://fastly.jsdelivr.net/gh/bucketio/img16@main/2025/12/03/1764732062819-0e7974fa-9073-4581-aab2-f2beccee8a1f.png)

 然后我们再次检查大小：
 
![](https://fastly.jsdelivr.net/gh/bucketio/img17@main/2025/12/03/1764732082388-9e3cfef4-2866-420e-a853-919acb0f124f.png)

大小与上面相同。让我们在表中使用INSERT和DELETE做更多操作：

```sql
--MSSQLTips.com
 
insert into test_vacuum_0 values (2,'Test Vacuum');
delete from test_vacuum_0
where id=2;
```
让我们直接看看内存页面：

```sql
--MSSQLTips.com
 
SELECT lp as tuple, t_xmin, t_xmax, t_field3 as t_cid, t_ctid 
                FROM heap_page_items(get_raw_page('test_vacuum_0', 0));
```

![](https://fastly.jsdelivr.net/gh/bucketio/img10@main/2025/12/03/1764732170963-039e3bed-5fec-49c6-8177-22eab342eba6.png)

如你所见，我们有每行（元组）的标识符，以及 t_xmin 和 t_xmax，也就是该元组可见的事务 ID 范围值。针对数据库执行的每个事务都会增加一个事务编号 txid。每个事务都有自己的标识符；在这种情况下，元组 3 没有 t_xmax，因此它从事务 ID 1249 开始可见。其他所有元组都有最小值和最大值，因此只有在这两个事务 ID 之间执行 select 语句时，该元组才可见。（注意：简单的 select 语句会增加 txid。）实际上，t_xmax 被设置为删除或更新该元组的事务编号。如果该元组未被删除或更新，t_xmax 则被设为 0。我将在后面的 MVCC 并发部分对此进行更详细的解释。

下面快速了解一下另外两列的含义：

- t_cid 存储着命令 ID（cid），即在此事务中实际执行的 SQL 命令之前已执行的 SQL 命令数量。例如，假设我们在一个事务中执行四个 INSERT 命令：“BEGIN; INSERT; INSERT; INSERT; INSERT; COMMIT;”。如果是第一个命令插入了这个元组，那么 t_cid 就被设置为 0；如果是第二个命令插入了这个元组，t_cid 就被设置为 1，以此类推。
- t_ctid 包含指向自身或新元组的元组标识符（tid）。它用于识别表中的元组。当该元组被更新时，其 t_ctid 指向新元组；否则，t_ctid 指向自身。

这四个数据是元组（行）在表页中存储方式的头部数据的主要部分。要全面了解页面布局，请查阅[官方文档](https://www.postgresql.org/docs/current/storage-page-layout.html)。

让我们回到VACUUM。现在我们可以手动清理这个表：

```sql
--MSSQLTips.com
 
vacuum test_vacuum_0;
```

![](https://fastly.jsdelivr.net/gh/bucketio/img18@main/2025/12/03/1764732350146-c75d3927-05bf-490f-afe4-463438e8e10c.png)

再看一下表格页面：

```sql
--MSSQLTips.com
 
SELECT lp as tuple, t_xmin, t_xmax, t_field3 as t_cid, t_ctid 
                FROM heap_page_items(get_raw_page('test_vacuum_0', 0));
```

![](https://fastly.jsdelivr.net/gh/bucketio/img15@main/2025/12/03/1764732374444-cb1bb11b-12cf-4ea4-b449-32c46844d68b.png)

不出所料，我们只有“dead tuple”元组；所有的失效元组都已从磁盘的内存页中删除。

现在，我们可以用多个表格页面中的更多数据来做一个示例。创建一个新表格并填入更多数据：

```sql
--MSSQLTips.com
 
create table test_vacuum (id bigint, descrizione varchar(20));
insert into test_vacuum values (generate_series(1,100000), 'Test vacuum');
```

![](https://fastly.jsdelivr.net/gh/bucketio/img15@main/2025/12/03/1764732440845-00aac595-670d-40aa-b728-7fc0bdf90234.png)

看看大小：

```sql
--MSSQLTips.com
 
select pg_size_pretty
(pg_relation_size('test_vacuum'));
```

![](https://fastly.jsdelivr.net/gh/bucketio/img0@main/2025/12/03/1764732486563-91d1aa89-2c77-4e19-9e79-c99dd515f65c.png)

有了表和更多数据，我们就可以执行一些更新来创建一些死元组：

```sql
--MSSQLTips.com
 
update test_vacuum
set descrizione='test_vacuum 23'
where (id % 2) = 0;
```

![](https://fastly.jsdelivr.net/gh/bucketio/img19@main/2025/12/03/1764732562642-966020c3-a3e3-4900-8544-f46d85bce1e7.png)

关于代码的一点说明：通过where (id % 2) = 0，我们只筛选出了能被2整除的偶数。我们更新了表中总行数的一半。

再看看大小。现在它应该已经将我们通过更新创建的所有死元组都考虑在内了：

```sql
--MSSQLTips.com
 
select pg_size_pretty
(pg_relation_size('test_vacuum'));
```
![](https://fastly.jsdelivr.net/gh/bucketio/img7@main/2025/12/03/1764732592810-bacf3f7d-1c3c-4708-a020-a2f21ca36922.png)

我们可以看到增长。让我们执行一个简单的VACUUM：

```sql
--MSSQLTips.com
 
vacuum test_vacuum;
```

![](https://fastly.jsdelivr.net/gh/bucketio/img0@main/2025/12/03/1764732620490-d16e9eec-fb41-4c26-8e6c-0c8c57c6b670.png)

再检查一下大小：

![](https://fastly.jsdelivr.net/gh/bucketio/img15@main/2025/12/03/1764732661359-463e7f6f-959e-4e89-aaf0-727c5e336320.png)

由于我们没有使用FULL选项，大小没有变化，该选项也可以回收磁盘空间。让我们试试：

```sql
--MSSQLTips.com
 
vacuum full test_vacuum;
```

磁盘空间减少了吗？

```sql
--MSSQLTips.com
 
select pg_size_pretty
(pg_relation_size('test_vacuum'));
```

![](https://fastly.jsdelivr.net/gh/bucketio/img9@main/2025/12/03/1764732733892-96df58ef-c6fb-49c6-9b3c-287534c646b2.png)


不出所料，大小已恢复到测试开始时的状态。

注意：使用FULL选项时应谨慎。它会在整个VACUUM期间对表施加排他锁（此外还会暂时占用额外空间）。

VACUUM也可用于更新可见性映射。但它是什么呢？

每个关系（表）都有一个可见性映射，用于跟踪哪些页面只包含有效的元组，哪些页面只包含冻结的元组（稍后会详细介绍FREEZE）。它与表的普通数据库页面存储在不同的文件中，但文件名包含对该关系的引用。其形式如下：**relfilenode_vm**。
  
它有两个功能：

1. 它帮助清理操作确定页面是否包含死行。
2. 最重要的是，它可以被仅索引扫描用来更快地回答查询。如果已知页面上的所有元组都是可见的，就可以跳过堆读取，而且由于可见性映射比实际页面小得多，它可以很容易地缓存在内存中。

每次VACUUM操作都会用存活和死亡元组更新这个可见性映射。

VACUUM有两个额外选项：FREEZE和ANALYZE。这让我们了解到一个极其重要的特性：需要使用VACUUM FREEZE或FULL来防止事务ID回卷。

但首先，我们需要了解PostgreSQL是如何使用MVCC实现并发控制（CONCURRENCY）的。

## PostgreSQL中的MVCC

多版本并发控制（MVCC）是 PostgreSQL 用于处理并发并确保数据一致性的一种方法。顾名思义，MVCC 允许多个事务同时访问数据库（例如，进行多个 SELECT 和 UPDATE 操作），无需通过锁来相互阻塞，而是通过访问不同的数据版本来实现。

在上述示例中修改数据时，PostgreSQL 不会覆盖现有数据，而是会创建一个新版本。每个元组 / 行都有元数据，包括事务 ID（t_xmin 和 t_xmax），用于指示创建它的事务以及将其标记为已删除的事务。每个事务都被分配了一个唯一的事务 ID（txid）。通过将事务的 txid 与元组的 t_xmin 和 t_xmax 进行比较，txid 有助于确定一个事务是否能看到该元组。

我们使用事务ID（txid）编号来指向元组的可见性。这个编号会自动分配给数据库上执行的每一笔事务。因此，每一次运行的查询都有一个事务ID，其实际值可以通过一个简单的SELECT语句来查看。注意：即便是这个SELECT语句也会被分配一个事务ID。

```sql
--MSSQLTips.com
 
SELECT txid_current();
```

![](https://fastly.jsdelivr.net/gh/bucketio/img9@main/2025/12/03/1764733443885-47e9f51d-2782-406e-a1dc-976762f53b11.png)

然而，这个数字是有限的。它是一个 32 位无符号整数，因此最大值为 4294967295，约合 42 亿。PostgreSQL 将事务 ID（txid）视为一个循环，就像一个环形时空。前 21 亿个事务 ID 属于 “过去”，接下来的 21 亿个事务 ID 属于 “未来”，因此只有过去的那些是可见的。

现在，让我们假设我们有一个活动元组 Tuple_1，它是通过事务 ID 100 插入的，即 Tuple_1 的 t_xmin 为 100。PostgreSQL 集群已经运行了很长时间，而 Tuple_1 从未被修改过。

当前事务 ID 为 21 亿 + 100，并且执行了一条 SELECT 命令。此时，Tuple_1 是可见的，因为事务 ID 100 处于过去。见图：

![](https://fastly.jsdelivr.net/gh/bucketio/img10@main/2025/12/03/1764733518166-57107ef7-ba1d-4e7a-8016-f28f6671c501.png)

然后，再次执行相同的 SELECT 命令；因此，当前的事务 ID（txid）为 21 亿 + 101。然而，Tuple_1 不再可见，因为事务 ID 100 处于未来！见图：

![](https://fastly.jsdelivr.net/gh/bucketio/img14@main/2025/12/03/1764733544396-9c86d323-a8db-40e8-90e9-f0d0dba1e644.png)

这会导致灾难性的数据丢失。有趣的是，数据仍然存在，但不再可见！这种情况被称为事务 ID（或 txid）环绕失败。

## VACUUM FREEZE

幸运的是，VACUUM FREEZE 可以防止此问题的发生。它是如何工作的呢？

基本上，VACUUM FREEZE 会冻结所有页面的事务 ID，无论这些页面是否被修改，这样所有当前行都能被视为旧行（或属于过去），从而对所有新事务可见。

PostgreSQL 保留了一个特殊的事务 ID，即 FrozenTransactionId，它始终被视为比所有正常的事务 ID 都旧。这个 ID 是在 VACUUM FREEZE 过程中插入到元组头的 t_infomask 列中的。

话不多说，让我们看看实际效果。为了演示这一点，我们需要先创建更多的失效元组，因为之前的那些已经被 VACUUM 和 VACUUM FULL 清理掉了：

```sql
--MSSQLTips.com
 
update test_vacuum
set descrizione='test_vacuum 24'
where (id % 2) = 0;
```

![](https://fastly.jsdelivr.net/gh/bucketio/img19@main/2025/12/03/1764733640178-a8917d38-e1aa-4a2c-91a2-b4da56044b5d.png)

现在，运行带有FREEZE选项的VACUUM：

```sql
--MSSQLTips.com
 
vacuum freeze test_vacuum;
```

看看大小和内存页：

```sql
--MSSQLTips.com
 
select pg_size_pretty
(pg_relation_size('test_vacuum'));
```

![](https://fastly.jsdelivr.net/gh/bucketio/img4@main/2025/12/03/1764733714037-f67ee4a1-6bb2-4644-9c88-f35d486259be.png)

由于我们没有执行 VACUUM FULL 命令，表的大小更大了。磁盘空间没有被回收。只有无效元组被删除了，就像在普通的 VACUUM 操作中一样。

但是，让我们再看一下表页面，这次添加 infomask 列：

```sql
--MSSQLTips.com
 
SELECT lp as tuple, t_xmin, t_xmax, t_field3 as t_cid, t_ctid, t_infomask 
                FROM heap_page_items(get_raw_page('test_vacuum', 0));
```

![](https://fastly.jsdelivr.net/gh/bucketio/img10@main/2025/12/03/1764733776191-e4f6daa7-784e-4891-b19c-a4abca3ae427.png)

现在我们可以看到，有一个单独的标识符表明这些元组（行）已被冻结且可见。

关于FREEZE的几个重要说明：

- 当表被重写时，总会执行强制冻结，因此指定FULL时，此选项是多余的。当我们运行VACUUM FULL时，也会执行FREEZE操作。
- vacuum_freeze_min_age参数控制行何时会被冻结。
- vacuum_freeze_table_age参数控制何时必须扫描整个表。

我们将在本技巧的第二部分更深入地讨论最后这两个参数。

那么，既然我们已经了解到有一种工具可以预防潜在的灾难性问题，我们就需要学习如何使用它！我们是否需要在特定的时间段内手动对数据库中的所有表执行这种VACUUM FREEZE操作呢？

幸运的是，没有，我们有一个名为AUTOVACUUM的自动化工具，它在所有表上都是默认启用的！

但这是本技巧下一章节的内容了！

# 后续计划

在本技巧中，我们了解了 VACUUM 是什么及其最重要的特性和用途，还了解了一些关于 PostgreSQL 默认情况下如何存储数据页和处理并发的理论。

后续文章将讨论什么是 AUTOVACUUM、如何对其进行调优以及 ANALYZE 选项。敬请关注！

一如既往，以下是一些官方文档的链接：

- [VACUUM process](https://www.postgresql.org/docs/current/routine-vacuuming.html#VACUUM-BASICS)
- [Visibility map](https://www.postgresql.org/docs/current/storage-vm.html)
- [MVCC](https://www.postgresql.org/docs/current/mvcc-intro.html)
