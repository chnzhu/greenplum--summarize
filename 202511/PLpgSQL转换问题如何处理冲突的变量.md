# PL/pgSQL转换问题:如何处理冲突的变量。

Pl\pgSQL 的一个有趣的事实是，它是 PostgreSQL 中的一个扩展，每当我们创建任何新数据库时，它都会作为默认创建。

转换数据库，主要是像 Oracle 这样的数据库，涉及使用 PL/pgSQL 将用 PL/SQL 编写的多个函数、过程或包转换为 PostgreSQL。在转换阶段，我们利用转换工具来自动化初始迁移的很大一部分。但是，请务必注意，自动转换的代码可能并不总是遵循 PostgreSQL 开发的最佳实践。

将代码审查作为迁移的一部分非常重要。每当机会出现时，我们都应该采用最佳实践。分享一个这样的转换学习经验帮助我了解了一些隐藏的 PL/pgSQL 复杂性。

让我们考虑下面的示例 PL\pgSQL 代码片段以供参考。

```sql
create table testproc(seqno integer, commit bool, comments1 text , comments2 text);
insert into testproc values(1,true, 'sample1', null);
 
create or replace function func1(seqno bigint, comments1 text ) 
returns setof testproc language plpgsql as
$$ 
begin
    return query update testproc 
                 set comments2=comments1 
                 where seqno=seqno
    RETURNING *;
end;
$$;
```

请重新查看示例程序，以下是一些关键亮点。

- 函数中使用的表、列名和变量通常具有相同的名称。
- 存在不一致的变量命名约定。

让我们运行示例代码，看看默认输出是什么。

```sql
=> select func1(2,'sample3'); 
 
ERROR:  column reference "seqno" is ambiguous
LINE 1: update testproc set comments2=comments1 where seqno=seqno
                                                      ^
DETAIL:  It could refer to either a PL/pgSQL variable or a table column.
QUERY:  update testproc set comments2=comments1 where seqno=seqno
    RETURNING *
CONTEXT:  PL/pgSQL function func1(bigint,text) line 3 at RETURN QUERY
```

过程代码中的模棱两可的引用可能是一个挑战。让我们来了解一些我们可以采用的解决方案来有效解决这些问题。

# 使用 variable_conflict use_variable

PL/pgSQL提供了多种选项来解决过程块中的变量冲突。默认情况下，它设置为"error"，这就是它在解决冲突时在上一次执行中失败的原因。

```sql
=> \dconfig plpgsql.variable_conflict
 List of configuration parameters
         Parameter         | Value 
---------------------------+-------
 plpgsql.variable_conflict | error
(1 row)
```

我们有不同的选项，例如use_variable和use_column，我们可以为特定功能启用，如下所示。

```sql
create or replace function func1(seqno bigint, comments1 text ) 
returns setof testproc language plpgsql as
$$ 
#variable_conflict use_variable
begin
    return query update testproc 
                 set comments2=comments1 
                 where seqno=seqno
    RETURNING *;
end;
$$;
```

让我们使用相同的输入参数重新运行函数，请注意传递的 seqno 不存在。

```sql
=> select func1(2,'sample3'); 
         func1         
-----------------------
 (1,t,sample1,sample3)
(1 row)
```

仅在 UPDATE 语句的 SET 组件中解决对不明确名称的冲突引用。但是，WHERE 子句仍然可以引用这些变量，这可能会导致不正确的更新。为避免这种情况，我们需要为模棱两可的名称添加其他别名。使用别名将帮助我们进一步解决它。

# 使用 Blockname 或 Function name 作为别名。

函数或过程中的所有过程块都具有与函数或过程本身相同的块

使用块名称作为引用有助于我们区分函数或过程变量，但始终建议对列引用进行适当的别名。

```sql
create or replace function func1(seqno bigint, comments1 text ) 
returns setof testproc language plpgsql as
$$ 
begin
    return query update testproc 
    set comments2=func1.comments1 
   where testproc.seqno=func1.seqno
    RETURNING *;
end;
$$;
 
dcg=> select func1(2,'sample3');                                                                                                        func1 
-------
(0 rows)
```

# 结论

无论是在转换还是新开发过程中，都必须避免变量名称，因为变量名称可能会在函数或过程中造成歧义。使用默认块或表级别使用适当的别名有助于克服这些冲突。在某些情况下，您还可以在功能级别利用plpgsql.variable_conflict配置。

原文：[PL/pgSQL Conversion Gotchas: How to Handle Conflicting Variables.](https://databaserookies.wordpress.com/2024/04/10/pl-pgsql-conversion-gotchas-how-to-handle-conflicting-variables/)