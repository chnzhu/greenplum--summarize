# PostgreSQL18-FDW连接的 SCRAM 直通身份验证

PostgreSQL 18 为使用 postgres_fdw 或 dblink_fdw 的人带来了很好的改进：SCRAM 直通身份验证。设置外部服务器连接时，您不再需要在“用户映射”选项中存储纯文本密码。

这是实现它的提交：

```sql
commit 761c79508e7fbc33c1b11754bdde4bd03ce9cbb3
Author: Peter Eisentraut <peter@eisentraut.org>
Date:   Wed Jan 15 17:55:18 2025 +0100

   postgres_fdw: SCRAM authentication pass-through

   This enables SCRAM authentication for postgres_fdw when connecting to
   a foreign server without having to store a plain-text password on user
   mapping options.

   This is done by saving the SCRAM ClientKey and ServeryKey from the
   client authentication and using those instead of the plain-text
   password for the server-side SCRAM exchange.  The new foreign-server
   or user-mapping option "use_scram_passthrough" enables this.

   Co-authored-by: Matheus Alcantara <mths.dev@pm.me>
   Co-authored-by: Peter Eisentraut <peter@eisentraut.org>
   Discussion: https://www.postgresql.org/message-id/flat/27b29a35-9b96-46a9-bc1a-914140869dac@gmail.com
```

正如提交消息本身所说，当PostgreSQL服务器连接到FOREIGN SERVER时，如果设置了use_scram_passthrough，它将使用原始客户端连接中的SCRAM密钥，而不需要纯文本密码。它更安全，避免了混乱的凭据重复。

要使用此功能，请确保：

- 外部服务器需要 scram-sha-256 身份验证（否则它只会失败）。
- 只有“客户端”（您使用 postgres_fdw 或 dblink_fdw）需要是 PostgreSQL 18+。
- 两台服务器必须为用户设置相同的 SCRAM 密钥。这意味着哈希值、盐值和迭代次数确实完全相同。
- 从客户端到主服务器的初始连接也必须使用 SCRAM（因此"直通"：必须使用 SCRAM 进出）。

## 如何使用postgres_fdw进行设置

我们将使用两个 Postgres 服务器：一个充当"传入"（fdw 客户端），一个充当"外部"服务器。

请注意，对于这些示例，我将使用 psql Postgres 客户端。

1. 在两台服务器上创建相同的用户

```sql
CREATE USER example;
```

在外部服务器上，创建一个示例表以供以后查询：

```sql
CREATE TABLE fdw_table AS SELECT g as a, b+2 as b FROM generate_series(1,100) g(g);
```

退出 psql 并使用新创建的用户重新登录，然后在两台服务器上设置密码

```sql
\password
```

2. 更新 pg_hba.conf 以需要 SCRAM

必须将两台服务器配置为强制执行 scram-sha-256：

```sql
local   all             all                                     scram-sha-256
host    all             all             127.0.0.1/32            scram-sha-256
host    all             all             ::1/128                 scram-sha-256
```

您可以使用以下方法找到 pg_hba.conf 的路径：

```sql
SHOW hba_file;
```

3. 同步加密密码（SCRAM 密钥）

从传入服务器获取加密密码：
```sql
SELECT rolpassword FROM pg_authid WHERE rolname = 'example';
```

现在在外部服务器上设置完全相同的密码（SCRAM 哈希）：
```sql
ALTER ROLE example PASSWORD 'scram-sha-256$...'; -- paste the whole thing
```
这一步至关重要——机密必须完全匹配。

4. 设置postgres_fdw

在传入服务器上：

```sql
CREATE EXTENSION IF NOT EXISTS postgres_fdw;

CREATE SERVER foreign_fdw
 FOREIGN DATA WRAPPER postgres_fdw
 OPTIONS (host 'localhost', dbname 'postgres', use_scram_passthrough 'true');

CREATE USER MAPPING FOR example
 SERVER foreign_fdw
 OPTIONS (user 'example');
```
> 注意：无需在映射中设置密码！

5. 导入国外表
```sql
IMPORT FOREIGN SCHEMA public LIMIT TO (fdw_table)
 FROM SERVER foreign_fdw INTO public;
```

现在只需运行：
```sql
SELECT * FROM fdw_table;
```
繁荣 💥 — 我们正在使用 SCRAM 直通跨服务器进行查询。

## dblink_fdw呢？

所有设置步骤都是相同的，但你不会导入表，而是直接调用 dblink_fdw（） ：

```sql
SELECT * FROM dblink('foreign_fdw', 'SELECT * FROM fdw_table')
 AS fdw_table(a int, b int);
```

## 最后的思考

SCRAM 直通是 PostgreSQL 服务器之间安全、无凭据连接的一项重要功能。它在您联合访问多个数据库并且不想在用户映射中处理密码的设置中特别有用。

更少的样板，更多的安全。这是一个胜利。
