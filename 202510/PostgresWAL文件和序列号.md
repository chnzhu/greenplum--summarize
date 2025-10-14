# PostgresWAL文件和序列号

Postgres 预写日志 （WAL） 是数据库的功能组件。WAL 使许多关键功能成为可能，例如时间点恢复备份、从事件中恢复、流复制等等。有时，数据库深处的人需要直接使用 WAL 文件来诊断或恢复。

最近在与 Crunchy Data 的一位客户合作时，我遇到了一种情况，即理解名称和序列号很重要。在与我的几位致力于 Postgres 项目的同事合作时，我收集了有关 WAL 内部一些细节的笔记。今天的目标是查看 WAL 的 LSN 和命名约定，以帮助用户更好地理解 WAL 文件。

# 日志序列号

PostgreSQL中的事务创建WAL记录，这些记录最终附加到WAL日志（文件）中。插入发生的位置称为日志序列号 （LSN）。可以比较LSN（类型 `pg_lsn` ）的值，以确定两个不同偏移量（以字节为单位）之间生成的WAL量。以这种方式使用时，重要的是要知道如果使用多个WAL日志，则计算假设使用了完整的WAL段（16MB）。与此处使用的计算类似的计算通常用于确定副本的延迟。

LSN 是一个 64 位整数，表示预写日志流中的位置。这个 64 位整数分为两段（高 32 位和低 32 位）。它打印为两个十六进制数字，用斜杠分隔 （XXXXXXXX/YYZZZZZZ）。"X" 代表 LSN 的高 32 位，"Y" 是低 32 位部分的高 8 位。"Z" 表示文件中的偏移位置。每个元素都是一个十六进制数。“X”和“Y”值用于默认PostgreSQL部署的WAL文件的第二部分。

# WAL 文件

WAL文件名的格式为`TTTTTTTTXXXXXXXXYYYYYYYYY`。这里“T”是时间线，"X" 是 LSN 的高 32 位，"Y" 是 LSN 的低 32 位。

![images](https://fastly.jsdelivr.net/gh/bucketio/img5@main/2025/10/10/1760088612263-4d5cb824-ed6a-465e-b0ad-7d169457ae86.png)

首先查看当前的 WAL LSN 并插入 LSN。`pg_current_wal_lsn` 是上次写入的位置。`pg_current_wal_insert_lsn` 是逻辑位置，反映缓冲区中尚未写入磁盘的数据。还有一个 flush 值，用于显示已写入持久存储的内容。

```sql
[postgres] # select pg_current_wal_lsn(), pg_current_wal_insert_lsn();
 pg_current_wal_lsn | pg_current_wal_insert_lsn
--------------------+---------------------------
 76/7D000000        | 76/7D000028
(1 row)
```

虽然你可以根据上面的输出猜出WAL文件的名称，但最好使用 `pg_walfile_name` 函数。

```sql
[postgres] # select pg_walfile_name('76/7D000028');
     pg_walfile_name
--------------------------
 00000001000000760000007D
(1 row)
```

查看文件系统，我们发现 Segment 00000001000000760000007D 确实是最新修改的文件。请注意，如果数据库处于空闲状态，则00000001000000760000007D可能是最旧的文件（基于 O/S 上次修改日期）。可能的原因是 PostgreSQL 重用了较旧的 WAL 段。在 `pg_switch_wal` 期间，旧文件被重命名。修改的 O/S 日期/时间仅在写入文件时更改。在 ls 命令中使用 -i 显示文件的索引节点（第一列）。重复使用文件时，此数字不会更改，因为文件只是重命名。

```sql
$ ls -larti 00*
169034323 -rw-------   1 bpace  staff       376 Jan 30 09:13 0000000100000075000000A0.00000060.backup
169564733 -rw-------   1 bpace  staff  16777216 Feb 13 15:49 00000001000000760000007E
167120667 -rw-------   1 bpace  staff  16777216 Feb 13 15:50 00000001000000760000007F
167120673 -rw-------   1 bpace  staff  16777216 Feb 13 16:00 00000001000000760000007B
167120686 -rw-------   1 bpace  staff  16777216 Feb 13 16:18 00000001000000760000007C
169564722 -rw-------   1 bpace  staff  16777216 Feb 13 16:18 00000001000000760000007D
```

让我们创建一个小表并执行 WAL 切换。

```sql
[postgres] # create table test (a char(1));
CREATE TABLE
Time: 23.770 ms

[postgres] # select pg_switch_wal();

 pg_switch_wal
--------------
76/7D018FD8
(1 row)
```

再次查看文件，我们现在看到00000001000000760000007D文件已更新（从作系统角度更改日期/时间）。由于其他一些项目在后台发生，00000001000000760000007E下一个段也在 `pg_switch_wal` 后收到了一些写入。

```sql
$ ls -larti 00*
169034323 -rw-------  1 bpace  staff       376 Jan 30 09:13 0000000100000075000000A0.00000060.backup
167120667 -rw-------  1 bpace  staff  16777216 Feb 13 15:50 00000001000000760000007F
167120673 -rw-------  1 bpace  staff  16777216 Feb 13 16:00 000000010000007600000080
167120686 -rw-------  1 bpace  staff  16777216 Feb 13 16:18 000000010000007600000081
169564722 -rw-------  1 bpace  staff  16777216 Feb 13 16:24 00000001000000760000007D
169564733 -rw-------  1 bpace  staff  16777216 Feb 13 16:24 00000001000000760000007E
```

在新创建的表中，插入一条记录。确保数据库是安静的，并在更改之前和之后获取当前的 WAL LSN。请注意，我们将 1 个字节 （'a'） 插入到具有单列的单个表中。

```sql
[postgres] # select pg_current_wal_lsn(), pg_current_wal_insert_lsn();
 pg_current_wal_lsn | pg_current_wal_insert_lsn
--------------------+---------------------------
 76/7E000060        | 76/7E000060
(1 row)

[postgres] # insert into test (a) values ('a');
INSERT 0 1

[postgres] # select pg_current_wal_lsn(), pg_current_wal_insert_lsn();
 pg_current_wal_lsn | pg_current_wal_insert_lsn
--------------------+---------------------------
 76/7E000108        | 76/7E000108
(1 row)
```

使用两个 LSN 位置，我们可以计算 `INSERT` 生成的 WAL 量（在本例中为 168 字节）。

```sql
[postgres] # select '76/7E000108'::pg_lsn - '76/7E000060'::pg_lsn size_bytes;
 size_bytes
------------
        168
(1 row)
```

抓取当前的 WAL LSN 并 WAL 插入 LSN 并再执行一次切换。然后列出文件。

```sql
[postgres] # select pg_switch_wal();
 pg_switch_wal
---------------
 76/7E0001D0
(1 row)


$ ls -larti 00*
169034323 -rw-------  1 bpace  staff       376 Jan 30 09:13 0000000100000075000000A0.00000060.backup
167120673 -rw-------  1 bpace  staff  16777216 Feb 13 16:00 000000010000007600000080
167120686 -rw-------  1 bpace  staff  16777216 Feb 13 16:18 000000010000007600000081
169564722 -rw-------  1 bpace  staff  16777216 Feb 13 16:24 000000010000007600000082
169564733 -rw-------  1 bpace  staff  16777216 Feb 13 16:26 00000001000000760000007E
167120667 -rw-------  1 bpace  staff  16777216 Feb 13 16:26 00000001000000760000007F
```

根据我们在前面步骤中捕获的信息，使用 `pg_waldump` 获取 WAL 段内容的人类可读摘要。在以下命令中，指定了起始位置（-s）和结束位置（-e）以及WAL文件名（00000001000000760000007E）。开始位置`INSERT`的`current_wal_lsn`，结束位置是插入之后的`current_wal_lsn`。以前，仅使用 LSN，我们确定了从事务写入 WAL 的 168 字节。查看 waldump 会发现 `INSERT` 占用了 103 个字节（`INSERT` 为 57 个字节，`COMMIT` 为 46 个字节）。

```sql
$ pg_waldump -s 76/7E000060 -e 76/7E000108 00000001000000760000007E
rmgr: Heap        len (rec/tot):     57/    57, tx:   59555584, lsn: 76/7E000060, prev 76/7E000028, desc: INSERT+INIT off 1 flags 0x08, blkref #0: rel 1663/5/53434 blk 0
rmgr: Transaction len (rec/tot):     46/    46, tx:   59555584, lsn: 76/7E0000A0, prev 76/7E000060, desc: COMMIT 2023-02-13 16:25:19.441483 EST
rmgr: Standby     len (rec/tot):     50/    50, tx:          0, lsn: 76/7E0000D0, prev 76/7E0000A0, desc: RUNNING_XACTS nextXid 59555585 latestCompletedXid 59555584 oldestRunningXid 59555585
```

从我们的 `INSERT` 之前的点查看整个 WAL 文件，可以看到 `INSERT` 本身，然后是 `COMMIT`最后，记录检查点并从执行的 `pg_switch_wal()` 切换。如果 `wal_level` 设置为副本或更高版本，则会添加`RUNNING_XACTS` 条目。`RUNNING_XACTS` 条目捕获当前快照（活动事务）的详细信息。最后一个条目 `SWITCH` 是执行的 `pg_switch_wal` 。

```sql
$ pg_waldump 00000001000000760000007E
rmgr: Standby     len (rec/tot):     50/    50, tx:          0, lsn: 76/7E000028, prev 76/7D018FC0, desc: RUNNING_XACTS nextXid 59555584 latestCompletedXid 59555583 oldestRunningXid 59555584
rmgr: Heap        len (rec/tot):     57/    57, tx:   59555584, lsn: 76/7E000060, prev 76/7E000028, desc: INSERT+INIT off 1 flags 0x08, blkref #0: rel 1663/5/53434 blk 0
rmgr: Transaction len (rec/tot):     46/    46, tx:   59555584, lsn: 76/7E0000A0, prev 76/7E000060, desc: COMMIT 2023-02-13 16:25:19.441483 EST
rmgr: Standby     len (rec/tot):     50/    50, tx:          0, lsn: 76/7E0000D0, prev 76/7E0000A0, desc: RUNNING_XACTS nextXid 59555585 latestCompletedXid 59555584 oldestRunningXid 59555585
rmgr: Standby     len (rec/tot):     50/    50, tx:          0, lsn: 76/7E000108, prev 76/7E0000D0, desc: RUNNING_XACTS nextXid 59555585 latestCompletedXid 59555584 oldestRunningXid 59555585
rmgr: XLOG        len (rec/tot):    114/   114, tx:          0, lsn: 76/7E000140, prev 76/7E000108, desc: CHECKPOINT_ONLINE redo 76/7E000108; tli 1; prev tli 1; fpw true; xid 0:59555585; oid 61620; multi 799; offset 1657; oldest xid 716 in DB 1; oldest multi 1 in DB 1; oldest/newest commit timestamp xid: 0/0; oldest running xid 59555585; online
rmgr: XLOG        len (rec/tot):     24/    24, tx:          0, lsn: 76/7E0001B8, prev 76/7E000140, desc: SWITCH
```

# 结束语

我希望你不需要经常深入研究 WAL，愿你的 `pg_switch_wal` 事件很少。提醒一下，各种 initdb 选项和编译时选项可能会改变计算和假设的结果。WAL 是一个复杂的话题，任何与 WAL 相关的元素都应非常小心地对待。

文档来自于: [Postgres WAL Files and Sequence Numbers](https://www.crunchydata.com/blog/postgres-wal-files-and-sequuence-numbers#log-sequence-number)