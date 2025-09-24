# Postgres18引入BUFFERS为EXPLAIN ANALYZE 中的默认选项

从 Postgres 18 开始，BUFFERS 选项成为 EXPLAIN ANALYZE 中的默认选项，这是个好消息。

在分析 Postgres 查询执行计划时，我始终建议使用 BUFFERS 选项：

```sql
explain (analyze, buffers) <query>;
```

# Example

```sql
test=# explain (analyze, buffers) select * from t1 where num > 10000 order by num limit 1000;
QUERY PLAN
----------------------------------------------------------
Limit  (cost=312472.59..312589.27 rows=1000 width=16) (actual time=314.798..316.400 rows=1000 loops=1)
Buffers: shared hit=54173
...
Rows Removed by Filter: 333161
Buffers: shared hit=54055
Planning Time: 0.212 ms
Execution Time: 316.461 ms
(18 rows)

```

如果在没有 `BUFFERS` 的情况下使用 `EXPLAIN ANALYZE`，那么分析将缺少有关缓冲池 IO 的信息。

首选使用 `EXPLAIN (ANALYZE, BUFFERS)` 而不是仅使用 `EXPLAIN ANALYZE` 的原因:

1. `IO` 作，缓冲池可用于计划中的每个节点。
2. 它提供了对所涉及的数据量的了解（提供的缓冲区命中数可能涉及多次 "hitting" 同一缓冲区）。
3. 如果分析侧重于 `IO`  数字，则可以使用较弱的硬件（更少的 RAM、较慢的磁盘），但仍具有可靠的数据用于查询优化。

为了更好地理解，建议将缓冲区编号转换为字节。在大多数系统上，1 个缓冲区是 8 KiB。因此，10 个缓冲区读取是 80 KiB。

但是，请注意可能的混淆：值得记住的是，`EXPLAIN (ANALYZE, BUFFERS)` 提供的数字不是数据量，而是 IO 数字，已完成的 `IO` 工作量。例如，对于内存中的单个缓冲区，可能有 10 次命中，在这种情况下，我们的缓冲池中不存在 80 KiB，我们只是处理了 80 KiB，处理了 10 次相同的缓冲区。实际上，这是一个不完美的命名：它表示为 `Buffers: shared hit=5` ，但这个数字是 `buffer hits` 而不是 `buffers hit` 作的数量，而不是数据的大小。

# 总结

始终使用 `EXPLAIN (ANALYZE, BUFFERS)` ，而不仅仅是 `EXPLAIN ANALYZE` 这样你就可以看到 Postgres 在执行查询时完成的实际 IO 工作。

这样可以更好地了解所涉及的数据量。如果您开始将缓冲区编号转换为字节，那就更好了--只需将它们乘以块大小（在大多数情况下为 8 KiB）。

当您处于优化过程中时，不要考虑时间数字，这可能会让人感觉违反直觉，但这就是让您忘记环境差异的原因。这就是允许使用瘦克隆的原因，看看 `Database Lab Engine` 以及其他公司如何处理它。

最后，在优化查询时，如果您设法减少了 `BUFFERS` 数量，这意味着要执行此查询，`Postgres` 将需要更少的缓冲区在所涉及的缓冲池中，从而减少 `IO` ，最大限度地降低争用风险，并在缓冲池中留出更多空间用于其他内容。遵循此方法最终可能会为数据库的总体性能提供全局积极影响。

来自：[postgres-ai](https://gitlab.com/postgres-ai/docs/-/blob/master/docs/postgres-howtos/performance-optimization/query-tuning/explain-analyze-buffers.md)
