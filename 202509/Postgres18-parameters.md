# "enable"参数在 Postgres 18 中的工作方式将有所不同

每年我都会在即将推出的PostgreSQL主要版本中浏览对EXPLAIN的更改和添加，以了解我们需要在pgMustard中添加哪些支持。

PostgreSQL 18 中有很多与 EXPLAIN 相关的改进！我要研究的清单上最新的一个是改变了规划者对"enable"参数的响应方式（并传达其效果）。我觉得挺有意思的，所以想分享一下。

## 这些"enable"参数是什么

[Postgres 包含一堆设置](https://www.postgresql.org/docs/current/runtime-config-query.html#RUNTIME-CONFIG-QUERY-ENABLE)，允许您作为用户阻止或阻止规划器使用某些作类型。这些以"enable_"开头，以作名称结尾，例如，可以关闭 `enable_nestloop` 以阻止规划器选择嵌套循环连接（如果可能）。

许多人在希望不鼓励使用顺序扫描时首先遇到这些设置，以检查 [Postgres 是否能够使用索引](https://www.pgmustard.com/blog/why-isnt-postgres-using-my-index)。在这种情况下，他们可以使用以下命令切换 `enable_seqscan`（针对他们的会话）：
```sql
set enable_seqscan = off;
```

## 这些设置的工作原理

其中一些设置总是按照您的想法进行作，因为它们实际上阻止了规划器甚至无法探索具有这些作类型的计划路径。例如，将 `enable_indexonlyscan` 设置为关闭确实会阻止仅索引扫描。

但是，对于可能（在某些情况下）是执行查询的唯一方法的作类型，需要采用不同的方法。因此，将 `enable_seqscan` 设置为关闭只会阻止使用顺序扫描，以便在不存在索引时它仍然可以作为回退选项使用。在这种情况下，当前会在此类作中添加 1^10（100 亿）的巨大"disable_cost"，因此 Postgres 的基于成本的优化器极不可能选择这样的计划，除非它没有其他选择。

这是一个使用 Postgres 17 的简单示例：

```sql
create table t (id bigint generated always as identity);
set enable_seqscan = off;
explain select * from t where id = 1;

                              QUERY PLAN                              
----------------------------------------------------------------------
 Seq Scan on t  (cost=10000000000.00..10000000025.00 rows=6 width=40)
   Filter: (id = 1)
```

如您所见，Postgres 仍然选择顺序扫描，尽管已被要求不要这样做。这是由于我们的表没有索引，因此剩下的唯一选项是顺序扫描。成本统计数据确实显示，巨大的 （10B） "disable_cost"被添加到 `Seq Scan`，但由于没有其他选项，它仍然被选择。

## 那么为什么要改变这一点呢？

虽然 1^10 是一个非常大的数字，但人们在 Postgres 中进行越来越多的分析查询，并且这些查询的成本可能（甚至合理地）变得非常高。因此，即使可以替代计划，有时您最终也会选择包含禁用节点类型的计划！这远非理想，也不是人们想要或期望该功能的工作方式。

[黑客帖子](https://www.postgresql.org/message-id/flat/CAO0i4_SSPV9TVxbbTRVLOnCyewopcc147fBZy%3Df2ABk15eHS%2Bg%40mail.gmail.com)（讨论即将到来的变化）从这样的例子开始，由 Zhenghua Lyu 在开发 [Greenplum](https://en.wikipedia.org/wiki/Greenplum) 时报告。该线程还提到了与"disable_cost"方法的其他一些"不一致"，例如对连接排序的意外影响。

虽然一种选择可能是简单地增加数量，但罗伯特・哈斯（Robert Haas）将此巧妙地描述为"拖延问题"，他着手探索其他替代方案。

## 启用参数在 18 中的工作原理

这个帖子非常长，但如果你想观察一些非常聪明的人讨论一个棘手的问题，并在经过很多来回之后得出一个非常合理和简单的实现，那么非常值得一试。

简而言之，Postgres 将不再向任何禁用的节点添加"disable_cost"，而是为每个路径保留禁用节点的计数。**它将选择禁用节点最少的路径，然后选择成本最低的路径——多么优雅。**

正如罗伯特曾经描述的那样，理想的行为是如果可以选择一个没有任何东西被禁用的计划，那么选择最便宜的计划，如果没有，请选择总体上最优的计划。

在 Postgres 18（beta 2）上测试我们的示例给出了以下查询计划：

```sql
create table t (id bigint generated always as identity);
set enable_seqscan = off;
explain select * from t where id = 1;

                    QUERY PLAN                     
---------------------------------------------------
 Seq Scan on t  (cost=0.00..38.25 rows=11 width=8)
   Disabled: true
   Filter: (id = 1)
```

我们可以看到成本统计信息已恢复正常，并且现在在 `Seq Scan` 节点上显示"Disabled:true"（尽管  `enable_seqscan` 已关闭），但仍被选中）。

有趣的是，此功能的初始提交显示了"已禁用节点"的运行总数，这就是内部工作方式，但幸运的是，作为用户，我们 `David Rowley` 付出了一些努力来反转哪些节点被禁用，以便在阅读/解释这些计划时为我们简化事情。

## 提示:仍使用 SETTINGS 参数

就我个人而言，我仍然建议人们使用与他们的 Postgres 版本支持的尽可能多的 `EXPLAIN` 参数，包括  `SETTINGS` 参数。这将输出任何与计划器相关的非默认设置，包括任何"enable"参数的值。

使用我们的示例：
```sql
explain (settings) select * from t where id = 1; 

                    QUERY PLAN                     
---------------------------------------------------
 Seq Scan on t  (cost=0.00..38.25 rows=11 width=8)
   Disabled: true
   Filter: (id = 1)
 Settings: enable_seqscan = 'off'
```

这显示了从默认值更改的任何参数，即使不存在"disabled"节点。我发现这在单独工作时很有用，但在与他人共享查询计划时更有用。

## 进一步阅读

- [Hackers email thread](https://www.postgresql.org/message-id/flat/CAO0i4_SSPV9TVxbbTRVLOnCyewopcc147fBZy%3Df2ABk15eHS%2Bg%40mail.gmail.com)
- [Commit e22253467](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commitdiff;h=e22253467)
- [Commit c01743aa4](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commitdiff;h=c01743aa4)
- [Commit 161320b4b](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commitdiff;h=161320b4b)
- [Commit 84b8fccbe](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commitdiff;h=84b8fccbe)
- [Draft PostgreSQL 18 release notes ](https://www.postgresql.org/docs/18/release-18.html)

## 总结

总之，Postgres 不再增加任何"enable"参数的成本，而是选择保留每个计划路径的禁用节点计数，并最初根据此计数确定计划选择的优先级，然后仅根据成本确定优先级。这应该确保它确实会尽可能避免使用这些作的计划。

非常感谢所有参与其中的人！