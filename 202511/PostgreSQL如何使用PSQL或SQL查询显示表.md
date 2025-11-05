# PostgreSQL:如何使用PSQL或SQL查询显示表

如果你来自MySQL，可能会本能地在PostgreSQL中输入SHOW TABLES;，结果却得到一个错误。PostgreSQL不包含该命令，但有简单的替代方法。

在这篇文章中，我们将介绍两种列出表的方法：使用psql的内置命令和来自系统目录的SQL查询。你还会看到一些优化结果的额外技巧。

# 方法1：使用PSQL Shell

从终端连接到数据库：

```sql
\c postgres
```

然后列出当前模式中的所有表：

```sql
\dt
```

想要查看所有模式中的所有表吗？

```sql
\dt *.*
```

你也可以查看特定表的详细信息：

```sql
\d table_name
```

**额外提示**：仅显示用户拥有的表：

```sql
\dt *.* | grep alice
```

# 方法2：使用SQL查询

对PostgreSQL的目录或 `information_schema` 执行查询：

```sql
SELECT table_name
FROM information_schema.tables
WHERE table_schema = 'public'
  AND table_type = 'BASE TABLE';
```

或者按模式分组显示所有表：

```sql
SELECT table_schema, table_name
FROM information_schema.tables
ORDER BY table_schema, table_name;
```

你甚至可以使用内部的 `pg_catalog.pg_tables` 视图：

```sql
SELECT schemaname, tablename, tableowner
FROM pg_catalog.pg_tables
ORDER BY schemaname;
```

额外视图：

```sql
SELECT table_name, table_type
FROM information_schema.tables
WHERE table_schema = 'public';
```

# 额外示例：使用数据库客户端

如果你更喜欢可视化工具，像 **DbVisualizer** 这样的客户端能让你在左侧的 schema 浏览器中直接查看所有表格，并交互式地运行相同的查询。

```sql
-- Example: run in DbVisualizer SQL Commander
SELECT * FROM information_schema.tables WHERE table_schema = 'public';
```

# FAQ

### PostgreSQL有SHOW TABLES吗？

不，不是的。请改用\dt或系统目录查询。

### 我能否只获取一个模式中的表 ?

是的，添加一个 `WHERE table_schema = 'schema_name'` 条件。

### 我可以包含视图或外部表吗？

移除 `AND table_type = 'BASE TABLE'` 以列出所有对象。

### 我能查看更多信息吗，比如所有者和类型？

是的，使用 `pg_catalog.pg_tables` 并包含tableowner。

# 结论

PostgreSQL没有直接的 `SHOW TABLES` 命令，但替代方法同样有效。通过shell中的`\dt`、系统目录查询以及DbVisualizer等可视化客户端，你可以快速查看数据库结构和模式。

要深入了解完整的查询和截图，请查看DbVisualizer的TheTable博客上关于在PostgreSQL中显示表的 [原始文章](https://www.dbvis.com/thetable/show-tables-postgresql-guide-two-different-approaches/)。

原文：[PostgreSQL: How to Show Tables Using PSQL or SQL Queries](https://dev.to/dbvismarketing/postgresql-how-to-show-tables-using-psql-or-sql-queries-2mni)