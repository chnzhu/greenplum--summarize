# 关于如何在PostgreSQL中调整数据库参数和配置的综合指南

PostgreSQL是一个非常通用的数据库系统，能够在低资源环境和与各种其他应用程序共享的环境中高效运行。为了确保它将在许多不同的环境中正常运行，默认配置非常保守，不太适合高性能生产数据库。加上地理空间数据库具有不同使用模式的事实，数据往往由比非地理空间数据库更少、更大的记录组成，您可以看到默认配置不会完全适合我们的目的。

我想我们可以同意，每个人都想要一个快速的数据库。问题是：在哪些方面快速？

在数据库方面，“快”至少有两个不同的方向：

- 每秒的事务数
- 吞吐量或数据处理量

这些是相互关联的，但绝对不一样。两者在I/O方面的要求完全不同。一般来说，您希望不惜一切代价避免I/O。这是因为与访问内存中的数据、不同级别的CPU缓存甚至CPU寄存器相比，I/O总是很慢。根据经验，每一层都会减慢大约1:1000的访问速度。

对于每秒大量事务的高需求系统，您需要尽可能多的并发IO。对于具有高吞吐量的系统，您需要一个每秒可以交付尽可能多字节的IO子系统。

这导致需要在CPU附近拥有尽可能多的数据——例如，在RAM中。至少工作集，即在可接受的时间内给出答案所需的数据集，应该合适。

每个数据库引擎都有一个特定的内存布局，并为不同的目的处理不同的内存区域。

总结一下：我们需要避免IO，我们需要调整内存布局的大小，以便数据库能够有效地工作（并且，我假设所有其他任务都是根据正确的模式设计完成的）。

所有这些配置参数都可以在postgresql. conf数据库配置文件中进行编辑。这是一个常规文本文件，可以使用记事本或任何其他文本编辑器进行编辑。在服务器重新启动之前，更改不会生效。

> 注意：这些值仅为建议值。每个环境都会有所不同，需要进行测试以确定最佳配置。但是这一部分应该会让你有一个好的开始。

# 数据库参数

以下是一些参数，可以根据您的系统和工作负载进行调整以获得最佳性能。

## shared_buffer

PostgreSQL缓冲区称为shared_buffer，是大多数操作系统中最有效的可调参数，该参数设置PostgreSQL将为缓存使用多少专用内存。

shared_buffer的默认值设置得很低，你不会从中得到多少好处。它设置得很低是因为某些机器和操作系统不支持更高的值。但是在大多数现代机器中，你需要增加这个值以获得最佳性能。

如果您有一个1GB或更大内存的专用数据库服务器，shared_buffers的合理起始值是系统内存的25%。您应该尝试一些较低和较高的值，因为在某些情况下，要获得最佳性能，您需要超过25%的设置。Windows系统上shared_buffers的有用范围通常从64MB到512MB。配置取决于您的机器、工作数据集和机器上的工作负载。

在生产环境中，可以观察到shared_buffer的较大值会提供非常好的性能，尽管您应该始终进行基准测试以找到正确的平衡。

```sql
# show shared_buffers;
shared_buffers
----------------
128MB
```

## wal_buffers

PostgreSQL将其WAL（预写日志）记录写入缓冲区，然后这些缓冲区被刷新到磁盘。缓冲区的默认大小（由wal_buffers定义）为16MB，但如果您有多个并发连接，则更高的值可以提供更好的性能。

## effective_cache_size

effective_cache_size参数提供了磁盘缓存可用存储器的估计值。它只是一个指南，而不是确切的分配内存或缓存大小。它不分配实际内存，而是告诉优化器内核中可用的缓存量。这是使用索引成本估计的一个因素；值越高，使用索引扫描的可能性越大，而值越低，使用顺序扫描的可能性越大。如果值设置得太低，查询规划器可以决定不使用某些索引，即使它们会有帮助。还应该考虑不同表上并发查询的预期数量，因为它们必须共享可用空间。因此，设置一个大的值总是有益的。

让我们看看effective_cache_size的一些实际含义。

例子:
```sql
edb=# SET work_mem TO "2MB";
edb=# EXPLAIN SELECT * FROM bar ORDER BY bar.b;

                                    QUERY PLAN                                     

-----------------------------------------------------------------------------------
Gather Merge  (cost=509181.84..1706542.14 rows=10000116 width=24)
   Workers Planned: 4
   ->  Sort  (cost=508181.79..514431.86 rows=2500029 width=24)
         Sort Key: b
         ->  Parallel Seq Scan on bar  (cost=0.00..88695.29 rows=2500029 width=24)
(5 rows)
```
初始查询的排序节点的估计成本为514431.86。成本是任意计算单位。对于上述查询，我们的work_mem只有2MB。出于测试目的，让我们将其增加到256MB，看看是否对成本有任何影响。
```sql
edb=# SET work_mem TO "256MB";
edb=# EXPLAIN SELECT * FROM bar ORDER BY bar.b;

                                    QUERY PLAN                                    

-----------------------------------------------------------------------------------
Gather Merge  (cost=355367.34..1552727.64 rows=10000116 width=24)
   Workers Planned: 4
   ->  Sort  (cost=354367.29..360617.36 rows=2500029 width=24)
         Sort Key: b
         ->  Parallel Seq Scan on bar  (cost=0.00..88695.29 rows=2500029 width=24)
```
查询成本从514431.86降低到360617.36-降低了30%。

## maintenance_work_mem

maintenance_work_mem参数是用于维护任务的内存设置。默认值为64MB。设置大值有助于执行VACUUM、RESTORE、CREATE INDEX、ADD FOREIGN KEY和ALTER TABLE等任务。

```sql
edb=# CREATE INDEX foo_idx ON foo (c);
CREATE INDEX
Time: 170091.371 ms (02:50.091)

set maintenance_work_mem = 256MB

edb=# CHECKPOINT;

edb=# set maintenance_work_mem to '256MB';

edb=# CREATE INDEX foo_idx ON foo (c);
CREATE INDEX

Time: 111274.903 ms (01:51.275)
```

当maintenance_work_mem设置为仅10MB时，索引创建时间为170091.371毫秒，但当我们将maintenance_work_mem设置增加到256MB时，索引创建时间减少到111274.903毫秒。

## synchronous_commit

这用于强制提交将等待WAL写入磁盘，然后再向客户端返回成功状态。这是性能和可靠性之间的权衡。如果您的应用程序被设计为性能比可靠性更重要，您应该关闭synchronous_commit。关闭synchronous_commit，成功状态和保证写入磁盘之间会有时间间隔。在服务器崩溃的情况下，数据可能会丢失，即使客户端在提交时收到成功消息。在这种情况下，事务提交非常快，因为它不会等待WAL文件被刷新，但可靠性受到了损害。

## max_connections

此参数的作用与您想的一样：设置当前连接的最大数量。如果达到限制，您将无法再连接到服务器。每个连接都占用资源，因此数量不应该设置得太高。如果您有长时间运行的会话，您可能需要使用比会话大多是短期的更高的数量。与连池配置保持一致。

## max_prepared_transactions

当您使用预准备事务时，您应该将此参数设置为至少等于max_connections的数量，以便每个连接至少可以有一个预准备事务。您应该查阅首选ORM（Object-Relational-Mapper）的留档，看看是否有特殊提示。

## max_worker_processes

将此设置为您要为PostgreSQL专门共享的CPU数量。这是数据库引擎可以使用的后台进程数。设置此参数将需要重新启动服务器。默认值为8。

运行备用服务器时，您必须将此参数设置为与主服务器相同或更高的值。否则，将不允许在备用服务器上进行查询。

## max_parallel_workers_per_gather

Gather或GatherMerge节点可以使用的最大工作人员。此参数应设置为等于max_worker_processes。当查询执行期间到达Gather节点时，正在实现用户会话的进程将请求与规划器选择的worker数量相等的后台worker进程数量。规划器将考虑使用的后台worker数量限制在max_parallel_workers_per_gather或以下。任何时候可以存在的后台worker总数受到max_worker_processes和max_parallel_workers的限制。因此，并行查询有可能使用比计划更少的worker运行，甚至根本没有worker。最佳计划可能取决于可用worker的数量，因此这可能导致查询性能不佳。

如果这种情况经常发生，考虑增加max_worker_processes和max_parallel_workers以便更多的工人可以同时运行或替代地减少max_parallel_workers_per_gather以便规划器请求更少的worker。

## max_parallel_workers

并行查询的最大并行工作进程数。与max_worker_processes相同。默认值为8。

请注意，此值高于max_worker_processes的设置将不起作用，因为并行worker是从该设置建立的工作进程池中获取的。

## effective_io_concurrency

IO子系统支持的实际并发IO操作数。作为起点：普通HDD尝试设置为2，SSD设置为200，如果您有强大的SAN，则可以从300开始。

## random_page_cost

这个因素基本上告诉PostgreSQL查询规划器访问随机页面比进行顺序访问要贵（或更少）多少。对于SSD或强大的SAN，这似乎没有那么重要，但在传统硬盘驱动器时代是如此。对于SSD或SAN，从1.1开始，对于普通的旧磁盘，将其设置为4。

## min_ and max_wal_size

这些设置在PostgreSQL的事务日志上设置大小边界。基本上，这是在发出检查点之前可以写入的数据量，检查点又将内存数据与磁盘数据同步。

## max_fsm_pages

此选项有助于控制空闲空间映射。当从表中删除某些内容时，它不会立即从磁盘中删除。它只是在空闲空间映射中标记为"空闲"。然后，此空间可以重复用于您在表上执行的任何新INSERT。如果您的设置具有高DELETE和INSERT率，则可能需要增加此值以避免表膨胀。

