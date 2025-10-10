# how-to-find-query

# The problem of jumping from pgss to EXPLAIN

一旦确定了有问题的 pgss 记录，首先要做的是了解优化的方向。

需要优化的 pgss 记录最常见的两种基本情况：

1. 如果calls高（大量 QPS，每秒查询），则优化的主要方法是减少此数字——这将在客户端（应用程序）端完成。
2. 如果 mean_exec_time + mean_plan_time 很高（对于 OLTP 上下文 - Web 和移动应用程序 - 100ms应被视为慢），这是我们需要应用查询微优化的时候 - 使用 EXPLAIN

当然，将这两种基本情况结合在一起的情况也并不少见——查询非常频繁，延迟很差。

在本文中，我们不会讨论如何使用EXPLAIN和EXPLAIN (ANALYZE, BUFFERS)。相反，我们将重点为EXPLAIN寻找合适的素材，那些需要研究和改进的特定查询示例。

值得记住的是，单个 pgss 记录可以与以不同方式执行的单个查询相关联，使用不同的计划。一个基本示例来说明它：

```sql
nik=# create table t1 as select 1::int8 as c1;
SELECT 1
nik=# insert into t1 select 2
from generate_series(1, 1000000);
INSERT 0 1000000
nik=# create index on t1(c1);
CREATE INDEX
nik=# vacuum analyze t1;
VACUUM
nik=# explain select from t1 where c1 = 1;
                               QUERY PLAN
-------------------------------------------------------------------------
 Index Only Scan using t1_c1_idx on t1  (cost=0.42..4.44 rows=1 width=0)
   Index Cond: (c1 = 1)
(2 rows)

nik=# explain select from t1 where c1 = 2;
                         QUERY PLAN
------------------------------------------------------------
 Seq Scan on t1  (cost=0.00..16981.01 rows=1000001 width=0)
   Filter: (c1 = 2)
(2 rows)

```

- 这里的两个查询都会在 pgss中被注册为select * from t1 where c1 = select * from t1 where c1 = $1。但执行计划不同，因为对于c1 = 1，我们有很高的选择性，而对于c1 = 2，选择性则非常差（目标是表中除 1 行之外的所有行）。

这意味着仅查看显示查询延迟较差的 pgss 记录，我们无法快速跳转到使用 EXPLAIN – 我们需要找到特定的查询样本来处理。

下面，我们讨论解决此问题的选项。

# 选项 1 猜测

在某些情况下，猜测可能没问题。但是，当我浪费了很多时间猜测错误时，我遇到过非常糟糕的情况。例如，在处理布尔列时，我决定使用具有非常差选择性的值，并花费了大量时间优化这种情况，然后才意识到应用程序代码永远不会使用它。

使用pg_statistic来改进猜测可能很诱人。但不幸的是，在一般情况下，这并不有效，因为缺乏多列统计数据（除非它是显式创建的）, 没有它，我们将在很多情况下出现不切实际的参数变体。

所以这种方法是有限的，只能用于简单的情况。

# 选项 2 从 Postgres 日志中获取示例

可以在 Postgres 日志中找到示例，当然，如果它们被记录下来（通常通过 log_min_duration_statement 参数或 auto_explain 扩展）。要查找给定 pgss 记录的示例，我们需要能够找到记录的查询和 pgss 记录的关联。两个选项：

1. 对于 PG14+，选项 compute_query_id 可以向日志条目提供与 pg_stat_statements 中使用的相同 queryid 值。
2. 或者，我们可以使用一个优秀的库 [libpg_query]（https://github.com/pganalyze/libpg_query；Ruby、Go、Python 和其他选项也可用）。它可以应用于规范化（pgss记录）和单个查询，产生所谓的痕迹，然后可以用来找到我们需要的关系。

一般来说，使用 Postgres 日志查找查询示例是一种很好的方法，但对于无法记录所有查询的重负载系统，它只会为我们提供非常慢的示例，那些超过 log_min_duration_statement 的示例（通常设置为一些相当高的值，例如 500ms）。

这种情况可以通过采样和降低慢查询的阈值甚至完全摆脱它来改善。它的参数：

- log_min_duration_sample (PG13+)
- log_statement_sample_rate （PG13+）
- log_transaction_sample_rate （PG12+）

或者，可以使用auto_explain，它还支持采样（auto_explain.sample_rate，所有当前支持的版本），而且它看起来非常有吸引力，因为它带来了生产时的计划。auto_explain的安装应仔细测试和分析开销。

# 选项 3 来自pg_stat_activity的示例查询

这种方法可能很有吸引力，因为它不需要我们打开太多查询的昂贵日志记录。即使是低延迟查询，如果它们足够频繁，如果我们观察 pg_stat_activity.query 足够长的时间，最终也会被捕获。

但是，这里有两个重要的限制。

首先,列 pg_stat_statements.query_id 是最近在 PG14 中添加的，可用于将 pg_stat_activity （pgsa） 中的样本与 pgss 记录连接起来。对于旧版本，我们最终会使用一些正则表达式（实现可能很麻烦/脆弱）的libpg_query指纹（这意味着我们需要对所有 pgsa 记录进行采样，然后对其进行后处理）。所以这个方法最好在PG14+中使用。

默认情况下,pg_stat_activity.query 被截断为 1024 个字符，这是由 track_activity_query_size 定义的，默认为 1024 个字符。建议大幅增加它，例如，增加到 10k，以允许对更大的查询进行采样和分析。不幸的是，更改此设置需要重新启动。

# 选项 4 eBPF

这个选项尚未完全开发，但有一种观点认为它在未来可能是一个非常好的替代方案：使用 eBPF 进行采样查询（pgsa），甚至用于采样查询 + 规范化它们（作为 pgsa 和 pgss 的替代）。

所以,待定。同时，查看这些有趣的资源：

- [pgtracer](https://github.com/Aiven-Open/pgtracer)
- [PostgresTV 上的演示](https://youtube.com/watch?v=tvJgMV-8nfU)

使用 perf 和 eBPF 分析 Postgres 性能问题

- [Andres Freund 描述如何结合 pg_stat_statements + BPF](https://youtu.be/HghP4D72Noc?si=tFuQuDWKrScJ8w2i&t=1389 )
- [pg-bpftrace](https://github.com/anarazel/pg-bpftrace)

# 选项 5 通用计划

待定：

- pg16 的 EXPLAIN 中的新功能
- 版本 <16 的技巧

# 总结

- 在 PG14+ 中，使用 compute_query_id 在 Postgres 日志和 pg_stat_activity 中具有query_id值
- 增加track_activity_query_size（需要重启）以便能够跟踪pg_stat_activity中的较大查询
- 组织工作流以组合pg_stat_statements中的记录以及日志和pg_stat_activity中的查询示例，以便在查询优化方面，您可以准备好与 EXPLAIN (ANALYZE, BUFFERS)一起使用的良好示例。
