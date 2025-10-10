# 了解PostgreSQL内存对齐

质疑和实验是对的！这就是我们真正学习的方式。内存对齐对数据库性能非常重要，不仅在PostgreSQL中，在许多其他系统中也是如此。当数据对齐不正确时，计算机的处理器可能需要做额外的工作来读取它，这可能会减慢速度。

将计算机的内存想象成一系列邮箱。每个邮箱都有一个地址。当您的计算机想要读取数据时，它更喜欢以整齐地适合这些邮箱的"chunks"读取数据。如果一段数据在邮箱中间开始或结束，计算机可能需要读取两个邮箱，然后将数据拼凑在一起，这需要更多时间。

内存对齐确保数据从其大小的倍数（或系统"对齐边界"的倍数）开始。例如，如果您的系统在8字节边界上对齐数据，理想情况下，8字节整数应该从0、8、16等地址开始。

这是因为当你进入像PostgreSQL这样的特定系统的本质时，通常会发现对齐的一般解释似乎有点简化。

## 系统特定优化

PostgreSQL和许多复杂的系统一样，有自己处理内存和数据结构的内部方法。即使您以特定顺序定义列，它也可能应用填充、重新排序列或使用其他技术来优化对齐。

## 编译器/架构差异

实际的对齐行为还可能取决于用于构建PostgreSQL的特定编译器和底层硬件架构（例如，32位与64位系统）。

## 数据类型大小

不同的数据类型有不同的大小和对齐要求。例如，`BIGINT`（8字节）通常比 `CHAR(1)`（1字节）具有更严格的对齐要求。

对齐理论在PostgreSQL中发挥作用的最经典示例之一是表中列的顺序。PostgreSQL通常会尝试有效地打包数据。如果您定义一个包含不同大小的列的表，顺序会影响需要多少"padding"。

## 场景1

### 效率较低的列排序

想象一下，我们有一个包含不同大小列的表，例如 `BIGINT`（8字节）、`BOOLEAN`（1字节）和 `INTEGER`（4字节）。

```sql
CREATE TABLE bad_alignment_example (
    id BIGINT,
    is_active BOOLEAN,
    some_value INTEGER
);
```

让我们跟踪它是如何存储的。在具有8字节对齐的64位系统上

id（BIGINT）
需要8个字节，自然对齐。

is_active（BOOLEAN）
需要1个字节。它紧跟在id之后。

some_value (INTEGER)
需要4个字节。这就是它变得有趣的地方。要在4字节边界上对齐 `some_value`（如果系统愿意，可以对齐8字节），PostgreSQL可能会在 `is_active` 之后插入3个字节的“padd”，以使 `some_value` 从对齐的地址开始。

所以，这个结构可能看起来像

```sql
[BIGINT (8 bytes)] [BOOLEAN (1 byte)] [PADDING (3 bytes)] [INTEGER (4 bytes)]
```

Total:
```sql
8 + 1 + 3 + 4 = 16 bytes. 8+1+3+4=16字节。
```

### 场景2

更高效的列排序

现在，让我们重新排序列，将较小的数据类型放在一起。

```sql
CREATE TABLE good_alignment_example (
    id BIGINT,
    some_value INTEGER,
    is_active BOOLEAN
);
```

这是它的储存方式

id (BIGINT) 
需要8个字节，自然对齐。（还是不错的！）

some_value (INTEGER) 
需要4个字节。紧跟在id之后。

is_active (BOOLEAN) 
需要1个字节。紧跟在some_value之后。

结构可能看起来像
```sql
[BIGINT (8 bytes)] [INTEGER (4 bytes)] [BOOLEAN (1 byte)] [PADDING (3 bytes)]
```

Total:
```sql
8 + 4 + 1 + 3 = 16 bytes. 8+4+1+3=16字节。
```

等等，在这个特定的例子中，总大小是一样的！这就是“不完全准确”部分的用武之地。虽然打包较小类型的理论好处可能意味着更少的填充，但PostgreSQL的内部规则仍然可以在元组的末尾添加填充以进行整体元组对齐，或者如果它需要确保后续元组也从对齐的边界开始。

关键是PostgreSQL通常旨在有效地对齐数据。它可能会在内部为其存储重新排序列，即使您以"suboptimal"的方式定义它们。然而，从最大到最小显式定义列是一种很好的做法，因为它有助于减少填充，尤其是在您有许多小列的情况下。

检查pg_column_size
要真正了解一行的存储大小，可以使用 `pg_column_size()` 函数。这将显示一个值占用的实际磁盘空间。

```sql
INSERT INTO bad_alignment_example VALUES (1, TRUE, 100);
INSERT INTO good_alignment_example VALUES (1, 100, TRUE);

SELECT pg_column_size(t.*) FROM bad_alignment_example t;
SELECT pg_column_size(t.*) FROM good_alignment_example t;
```

有时您可能会惊讶地看到类似的结果，这表明PostgreSQL的内部优化正在发挥作用。

### VACUUM FULL和CLUSTER

虽然不直接涉及单个数据类型的对齐，但 `VACUUM FULL` 和 `CLUSTER` 可以帮助解决整体表碎片并改善物理存储顺序，这通过提高顺序读取的效率间接影响性能。

使用 `CREATE TABLE ... AS ...` 复制数据

如果您从列顺序不太理想的表中复制数据，则使用所需的列顺序创建一个新表，然后 `INSERT INTO ... SELECT ...` 有时会导致更有效地存储表。

### EXPLAIN ANALYZE

当您尝试优化性能时，`EXPLAIN ANALYZE` 是您最好的朋友。它可以帮助您了解PostgreSQL是如何执行查询的，这可以间接揭示数据访问模式是否因物理存储或对齐问题而效率低下。

### 数据类型选择

有时，最好的优化不是对齐，而是简单地选择正确的数据类型。例如，如果 `SMALLINT`（2个字节）就足够了，请不要使用 `INTEGER`（4个字节）。较小的数据意味着更少的内存、更少的I/O，并且通常具有更好的性能。

文档来自于: [Understanding PostgreSQL Memory Alignment: Beyond the Basics](https://iifx.dev/en/articles/457640000)