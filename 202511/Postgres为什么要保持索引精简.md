# Postgres为什么要保持索引精简

您的 API 速度变慢。您检查数据库并在`users`表上找到 42 个索引。哪些可以安全掉落？它们会花费您多少性能？让我们看看当索引过多时，Postgres 中实际会发生什么。

如果您是后端或全栈工程师，您可能不想成为索引专家 - 您只希望您的 API 快速稳定，而无需照顾`pg_stat_user_indexes`。

索引维护包括多项活动：删除未使用的索引、删除冗余索引以及定期重建索引以摆脱索引膨胀（当然，还要保持自动清理的良好调整）。

我们需要保持索引集精简的原因有很多，其中一些原因很棘手。

# 为什么要删除未使用的冗余索引

多年来，我一直在收集这些想法。这是当前列表（更多即将推出）：

1. 额外的索引会减慢写入速度--臭名昭著的"索引写入放大"
2. 额外的索引会减慢 SELECT 的速度，有时甚至会从根本上减慢（令人惊讶但确实如此）
3. 额外的索引会浪费磁盘空间
4. 额外的索引会污染缓冲池和作系统页面缓存
5. 额外的索引增加了自动真空工作
6. 额外的索引会生成更多的 WAL，从而影响复制和备份

至于指数膨胀，原因 3-6 也适用于指数膨胀。此外，如果索引非常臃肿（例如，90%+ 或最佳大小的 >10 倍），索引扫描延迟就会受到影响。Postgres B 树实现缺乏合并作——一旦页面拆分，这些页面即使在删除后也永远不会合并回一起。随着时间的推移，这会导致碎片化和膨胀的增加。PG13+ 中的重复数据删除有助于压缩重复键，而 PG14+ 中的自下而上的删除则通过在插入过程中更积极地删除死元组来减少膨胀。但是，这些功能无法解决页面拆分造成的结构退化问题。定期监控和重建臃肿的索引仍然是重要的维护工作。

让我们检查列表中的每个项目，研究 Postgres 源代码（这里是 Postgres 18）

# 写放大

每个 `INSERT` 或非 HOT `UPDATE` 都必须修改所有索引。

```sql
/*
 * for each index, form and insert the index tuple
 */
for (i = 0; i < numIndices; i++)
{
    Relation indexRelation = relationDescs[i];
    // ...
    index_insert(indexRelation, values, isnull, tupleid,
                 heapRelation, checkUnique, indexUnchanged, indexInfo);
}
```

循环显式迭代所有索引 （`numIndices`） 并为每个索引调用 `index_insert()`

HOT 更新可以提供帮助——但前提是新元组适合同一页面并且没有更改索引列。来自 `heapam.c`：

```sql
if (newbuf == buffer)
{
    /*
     * Since the new tuple is going into the same page, we might be able
     * to do a HOT update. Check if any of the index columns have been
     * changed.
     */
    if (!bms_overlap(modified_attrs, hot_attrs))
        use_hot_update = true;
}
```

否则，必须更新所有索引。

HOT 更新可以帮助：当表在设计时考虑到它们时，它们非常有效——使用适当的`fillfactor`设置并仔细考虑哪些列需要索引。虽然它们需要同页元组放置并且没有索引列更改，但正确的模式设计可以最大限度地提高 HOT 更新的适用性。有关详细信息，请参阅[文档](https://www.postgresql.org/docs/current/storage-hot.html)。

相关文章：

- [Percona 基准测试测](https://www.percona.com/blog/benchmarking-postgresql-the-hidden-cost-of-over-indexing/)得 39 个指数与 7 个指数的吞吐量损失高达 58%
- 生产案例研究：Adyen 通过`fillfactor`调整，在其 50TB+ 数据库上实现了 10% 的 WAL 降低

# 额外的索引会减慢 SELECT 的速度

规划器必须检查所有索引以找到最佳查询计划。

我们最近讨论过：

- [PostgresMarathon 2-004：索引过多会损害 SELECT 查询性能](https://postgres.ai/blog/20251008-postgres-marathon-2-004)
- [PostgresMarathon 2-005：更多 Postgres 18 的 LWLock：LockManager 基准测试](https://postgres.ai/blog/20251009-postgres-marathon-2-005)

查看代码，`indxpath.c`：

```sql
/*
 * Examine each index of the table, and see if it is useful for this query.
 */
foreach(lc, rel->indexlist)
{
    IndexOptInfo *index = (IndexOptInfo *) lfirst(lc);
    
    /* Identify the restriction clauses that can match the index. */
    match_restriction_clauses_to_index(root, index, &rclauseset);
    
    /* Build index paths from the restriction clauses. */
    get_index_paths(root, rel, index, &rclauseset, bitindexpaths);
}
```

每个索引路径都会触发 `costsize.c` 中的昂贵成本计算，包括I/O成本建模、选择性计算和页面相关性分析等复杂计算。

用于评估单个索引的规划开销是 O（N），但当规划器考虑在位图扫描中组合多个索引时，可以接近 O（N²）。来自 `indxpath.c` L528-531 的源代码注释：

```sql
/*
 * Note: check_index_only() might do a fair amount of computation,
 * but it's not too bad compared to the planner's startup overhead,
 * especially when the expressions are complicated.
 */
```

相关新闻： [Percona 基准测试](https://www.percona.com/blog/postgresql-indexes-can-hurt-you-negative-effects-and-the-costs-involved/)测量了 O（N） 到 O（N²） 复杂性的规划开销，即使缓存命中率为 99.7%，也会影响高频查询。

从哪里开始手动检查它 – 以下是快速找到索引最重的表的方法：

```sql
select schemaname, tablename, count(*) as index_count
from pg_indexes
group by 1, 2
having count(*) > 10
order by 3 desc;
```

# 磁盘空间浪费

这一点是显而易见的每个索引都存储为一个单独的关系文件。在某些情况下，表索引占用的磁盘空间明显超过表数据本身的空间这可以用作过度索引的弱信号（需要优化的信号）。

相关博客文章：[Haki Benita](https://hakibenita.com/postgresql-unused-index-size) 通过删除未使用的索引释放了 20 GiB，一个部分索引将存储从 769 MiB 减少到 5 MiB – 节省了 99%（但是，考虑到部分索引，请记住，[迁移到部分索引可能会使某些 HOT 更新成为非 HOT](https://postgres.ai/blog/20211029-how-partial-and-covering-indexes-affect-update-performance-in-postgresql)）。

快速检查 – 查找从未与此非常基本的查询一起使用过的索引：

```sql
select schemaname, relname, indexrelname, pg_size_pretty(pg_relation_size(indexrelid))
from pg_stat_user_indexes
where idx_scan = 0
order by pg_relation_size(indexrelid) desc;
```

此分析中还有重要的其他细微差别，例如：

- 显然，我们需要将唯一索引排除在考虑之外
- 统计数据必须足够老
- 还需要分析备用

这些方面我们下次再详细讨论。

在 PostgresAI，我们将这些检查转化为自动化工作流程：我们持续监控数据库运行状况和工作负载模式，提出安全的丢弃/重新索引缓解措施，只需工程师付出最少的努力，无需手动调整即可保持高性能。

# 缓存污染

更多索引 = 更多索引页 = 更多缓冲池压力 = 更低的缓存命中率。

从[缓冲区管理器自述文件](https://github.com/postgres/postgres/blob/fc295beb7b74d1f216a53cab61d22027121a763e/src/backend/storage/buffer/README)

> PostgreSQL使用共享缓冲池来缓存磁盘页面。所有后端共享一个公共缓冲池...当请求的页面不在缓冲池中时，缓冲区管理器必须逐出页面以腾出空间。

索引页与堆页竞争 Postgres 缓冲池和作系统页缓存中的有限缓存空间。棘手的部分：主动写入表上未使用的索引仍然会消耗缓存，因为每个 `INSERT` 和非 HOT `UPDATE` 都必须修改所有索引，从而强制索引页进入内存。

这会显着影响 Postgres 缓冲池和作系统页面缓存的缓存效率（命中/读取比）。

相关新闻： 即使缓存命中率为 99.7%，[Percona 基准测试](https://www.percona.com/blog/benchmarking-postgresql-the-hidden-cost-of-over-indexing/)也显示，由于过多的索引争夺缓存空间，吞吐量损失高达 58%。

# Autovacuum 开销

Vacuum 在批量删除阶段处理所有索引，通常在清理阶段再次处理（如果索引指示不需要通过 `amvacuumcleanup` 结果进行清理，则可以跳过该阶段）。

来自 `vacuumlazy.c`：

```sql
for (int idx = 0; idx < vacrel->nindexes; idx++)
{
    Relation indrel = vacrel->indrels[idx];
    IndexBulkDeleteResult *istat = vacrel->indstats[idx];
    
    vacrel->indstats[idx] = lazy_vacuum_one_index(indrel, istat,
                                                   old_live_tuples, vacrel);
}
```

然后再次在 `lazy_cleanup_all_indexes()` 中进行清理阶段。

更多的索引 = 更慢的真空 = 更高的表膨胀（这里有可能出现一些正反馈循环）。

# WAL 生成

每个索引更改作都会生成WAL记录。

来自 `nbtinsert.c`

```sql
recptr = XLogInsert(RM_BTREE_ID, xlinfo);
```

B 树作有 [15 种不同的 WAL 记录类型](https://github.com/postgres/postgres/blob/fc295beb7b74d1f216a53cab61d22027121a763e/src/include/access/nbtxlog.h#L27-L45)：插入、拆分、删除、清理、重复数据删除等。

更多索引 = 更多 WAL = 复制、备份和恢复过程的更大压力。

在负载系统中，生成过多的 WAL 可能会导致作困难，甚至某些关键的性能悬崖。在 PostgresAI，我们从 WAL 和性能模式自动检测并经常预测此类悬崖，然后在它们变成事件之前发现具体、安全的作（例如删除冗余索引或调整自动清理）。

# 总结

每增加一个指数，您就会付出以下代价：

- Writes：`INSERT` `UPDATE` 循环遍历所有索引
- Reads：规划器检查所有索引（O（N） 到 O（N²））
- Memory：未使用的索引在写入时仍会消耗缓存
- Vacuum：处理所有索引两次
- WAL：更多的索引 = 复制和备份的压力更大

未使用的冗余索引和索引膨胀不是免费的——它们在每次写入时都会被修改。

删除未使用的索引。删除冗余索引。重新索引降级（臃肿）的索引。并且不要忘记所有这些作的 `CONCURRENTLY`。

手动完成所有这些是可能的，但当您每周发布功能时，它不会扩展。

保持索引集精简。