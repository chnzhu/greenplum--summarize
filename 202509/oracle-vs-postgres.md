# Oracle 与 PostgreSQL:2025 年的完整比较

Oracle 和 PostgreSQL 是两个领先的关系数据库管理系统，具有不同的方法。Oracle 由 Oracle 公司开发，是一个商业企业级数据库，以强大的功能和可靠性而闻名，但需要支付高昂的许可成本。PostgreSQL 是一种功能强大的开源替代方案，提供高级功能、标准合规性和可扩展性，无需许可费。

此比较检查了它们的架构、功能、性能、许可模型和用例，以帮助技术专业人员和业务决策者了解它们的优势和局限性。

# 历史及背景

Oracle 数据库始于 1977 年，当时 Larry Ellison、Bob Miner 和 Ed Oates 创建了第一个基于 SQL 的商业 RDBMS，并于 1979 年作为 Oracle V2 发布。主要发展包括性能改进的 Oracle7 （1992）、支持 Java 的 Oracle 8i （1999）、网格计算的 Oracle10g （2003）、多租户架构的 Oracle12c （2013） 以及当前的长期支持版本 Oracle19c （2019）。Oracle 由 Oracle 公司开发和维护，由商业利益和企业需求驱动。

PostgreSQL 起源于 1980 年代中期加州大学伯克利分校 Michael Stonebraker 教授领导的 POSTGRES 项目。它于 1996 年演变为 PostgreSQL，为其处理复杂数据类型和用户定义函数的能力添加了 SQL 支持。主要版本包括支持 Windows 的 PostgreSQL 8.0 （2005）、内置复制的 9.0 （2010） 以及增强的性能和安全性的 16 （2023）。与 Oracle 不同，PostgreSQL 是由全球社区通过 PostgreSQL 全球开发小组开发的，优先考虑标准合规性和可扩展性，同时保持免费和开源。

# 比较摘要

|   |  Oracle | PostgreSQL |
|:----:|:----:|:----:|
| Architecture | 复杂、以企业为中心 | 更简单、更直接 |
| Licensing | 商业，昂贵（每个内核 47,500+ 美元） | 免费、开源（PostgreSQL 许可证） |
| Community Support | 商业支持 | 活跃的开源社区 |
| Data Types | 标准和一些高级类型 | 广泛，支持更好的 JSON |
| Extensibility | 有限 | 高度可扩展 |
| SQL Compliance | 部分带有专有扩展 | 严格符合标准 |
| Scalability | 与 RAC 一起用于水平扩展 | 良好，依赖第三方解决方案进行集群 |
| High Availability | 内置 RAC 和 Data Guard | 可通过第三方工具获得 |
| Security Features | 全面的企业安全 | 基础安全强，可扩展 |
| Performance (OLTP) | 非常适合非常大的工作负载 | 非常适合大多数常见工作负载 |
| Performance (Analytics) | 具有专业功能的出色 | 很好，使用最新版本进行了改进 |
| Admin (Install) | 复杂且资源密集型 | 简单明了 |
| Admin (Day-to-Day) | 全面的内置工具 | 手动配置和第三方工具 |
| Admin (Monitoring) | 广泛的内置工具 | 带扩展的基本 |
| Cloud Offerings | Oracle 云（价格昂贵但功能丰富） | 适用于所有主要云平台（经济高效）|
| Cost (16 cores) | ~760,000 美元 + 167,200 美元/年支持 | 0 美元（许可） |
| Cloud Cost (2vCPU) | $ 400-500 /月 | $115-150/月 |
| Best For | 任务关键型企业应用程序 | Web 应用程序、初创公司、成本敏感型部署 |

#  详细对比

## 架构 

**Oracle**:具有许多专业组件的企业级：

### Memory

- SGA：共享缓存和 SQL 执行
- PGA：每进程内存
- 包括缓冲区缓存、共享池、大池等。

### Processes

- 服务器进程处理用户查询
- 背景：数据库写入器、日志写入器、检查点、SMON（恢复）、PMON（清理）、归档器

### Storage

- 控制文件、数据文件、重做/存档日志、参数和日志文件

### Logical Structure

- 表空间>段>盘区>块

**PostgreSQL**：更简单的开源架构：

### Processes 

- Postmaster 管理服务器
- 每个连接一个后端
- 后台任务：Writer、Checkpointer、Autovacuum、WAL Writer、Logger

### Memory

- 共享缓冲区、WAL 缓冲区、工作内存、维护内存

### Storage

- 具有`base`、`global`、`pg_wal`的数据目录
- 配置文件：`postgresql.conf`、`pg_hba.conf`

### Logical Structure

- 数据库>模式>表>索引

# 许可和成本结构

## Oracle 提供具有不同定价和许可模式的多个版本：

### 企业版 （EE）

- 完整功能（安全性、性能、HA）
- ~每个处理器内核 47,500 美元或每个指定用户 950 美元（最少 25 个用户）
- 附加组件（例如 RAC、内存中、数据防护）需要额外付费

### 标准版 2 （SE2）

- 对于较小的设置，仅限于 2 个 sockets
- ~每个sockets 17,500 美元或每个指定用户 350 美元（最少 10 个用户）
- 无可选功能

### 快速版 （XE）

- 免费，有限制：2 个 CPU 线程、2GB RAM、12GB 数据
- 对于开发或轻量级应用

### 个人版

- 用于单用户开发
- 价格与 SE2 相似

### 额外费用

- 年度支持（~22% 的许可证成本）
- 管理包、工程系统（例如 Exadata）

## PostgreSQL 使用简单的开源许可证：

- 许可证：PostgreSQL 许可证（MIT/BSD 风格）
- 成本：任何用途均免费 — 无费用、限制或用户限制
- 商业用途：完全允许，包括嵌入专有应用程序

### 潜在成本

- 基础设施（云/本地）
- 可选支持或咨询
- 付费第三方工具

## 示例：16 核部署

| Platform | License Cost | Annual Support | Total (Year 1) |
|:----:|:----:|:----:|:----:|
| Oracle EE | ~$760,000 ~760,000 美元 | 	~$167,200 ~167,200 美元 | ~$927,200 ~927,200 美元 |
| Oracle SE2 | Not suitable | - | - |
| PostgreSQL | $0 | Optional | ~$0 ~0 美元 |

### 托管云定价（2 个 vCPU、8 GB RAM、100 GB 存储）：

- 适用于 Oracle EE 的 AWS RDS：~400–500 USD/月
- AWS RDS for PostgreSQL：~141 USD/月
- Azure Database for PostgreSQL：~141 美元/月
- Azure Database for PostgreSQL：~141 美元/月

## 数据类型和可扩展性

| Feature | Oracle | PostgreSQL |
|:----:|:----:|:----:|
| Standard SQL types | 支持 | 支持 |
| JSON support JSON | 有限（在最新版本中有所改进）| 原生 JSON 和 JSONB |
| XML support XML | XMLType | 原生支持 |
| Spatial types | Oracle 空间 | 内置 PostGIS |
| Geometric types | 不支持 | POINT、LINE、POLYGON |
| Network address types | 不支持 | INET、CIDR |
| Array types | 不支持 | 原生 ARRAY 类型 |
| Range types (e.g., date/number) | 不支持 | 内置支持 |
| Object types / UDTs | 有局限性 | CREATE TYPE |
| Extensible type system | 有限 | 高度可扩展，支持自定义数据类型 |

PostgreSQL 以其丰富且可扩展的类型系统而脱颖而出，为现代格式和复杂的数据建模提供卓越的支持。

## SQL 合规性和扩展

| Feature | Oracle | PostgreSQL |
|:----:|:----:|:----:|
| SQL Standard Compliance | 部分、许多专有扩展 | 强合规性 |
| Procedural Language | PL/SQL | PL/pgSQL |
| Recursive Queries  | CONNECT BY（专有语法） | 具有 WITH RECURSIVE |
| Window Functions | 支持 | 支持 |
| Materialized Views | 支持 | 支持 |
| Full-text Search | 不支持 | 内置支持 |
| Syntax Style | 特定于 Oracle 的 | 符合标准 |

PostgreSQL 更符合 SQL 标准，而 Oracle 提供强大但专有的功能。

## 索引功能

| Feature | Oracle | PostgreSQL |
|:----:|:----:|:----:|
| B-tree indexes | 违约 | 违约 |
| Bitmap indexes | 支持 | 不支持 |
| Function/Expression indexes | 基于功能 | 基于表达式 |
| Partial indexes | 通过基于函数的索引 | 原生支持 |
| Domain indexes | 对于自定义数据类型 | 有限（通过扩展或功能解决方法） |
| Full-text indexes | Oracle 文本 | GIN/GiST 与 tsvector |
| GiST indexes | 不支持 | 广义搜索树 |
| SP-GiST indexes | 不支持 | 空间分区 GiST |
| GIN indexes | 不支持 | 广义倒置指数 |
| BRIN indexes | 不支持 | 块范围索引 |
| Custom index types | 有限 | 完全可扩展 |

PostgreSQL提供了更丰富的索引类型，使其更适合高级和专业的查询场景。

## 并发和事务

| Feature | Oracle | PostgreSQL |
|:----:|:----:|:----:|
| MVCC（多版本并发） | 支持 | 支持 |
| Read consistency level | Statement-level  | Transaction-level  |
| Row-level locking | 支持 | 支持 |
| Isolation levels | 读取已提交，可序列化 | 已提交读取、可重复读取、可序列化 |
| Distributed transactions | 两阶段提交 | 两阶段提交 |
| Autonomous transactions | 支持 | 不支持 |
| Advisory locks | 不支持 | 应用控制锁定 |

两者都支持强大的事务保证，但 PostgreSQL 提供了对隔离和应用程序级锁定的更精细的控制，而 Oracle 则独特地支持自治事务。

## 高可用性和复制

| Feature | Oracle | PostgreSQL |
|:----:|:----:|:----:|
| Clustering | 真实应用集群 （RAC） | 没有内置集群（使用 Patroni、Stolon 等） |
| Physical replication | 数据卫士 | 流式复制 |
| Readable standby | 活动数据防护 | 使用流式复制（热备用） |
| Logical replication | 金门 | 从 v10 开始内置 |
| Synchronous replication | 支持 | 支持 |
| Point-in-time recovery | 闪回数据库 | 内置 PITR 支持 |
| Transparent failover | 应用程序连续性 | 需要外部工具 |
| Connection pooling | 依赖于应用程序 | 通过 pgBouncer 或 Pgpool-II |

Oracle 提供了更集成的企业级 HA 选项，例如 RAC 和应用程序连续性。PostgreSQL 通过灵活性和第三方工具实现了类似的目标。

## 性能特点

| Feature | Oracle | PostgreSQL |
|:----:|:----:|:----:|
| Result cache | 查询 + PL/SQL 函数结果缓存 | 不支持 |
| In-Memory Column Store | 支持 | （可以使用 Citus 或 TimescaleDB 等扩展）|
| Automatic memory management | 支持 | 需要手动调整 |
| Parallel query execution | 支持 | （有限但正在改进） |
| Partitioning | 范围、列表、哈希、复合 | 范围、列表、哈希 |
| Just-in-time (JIT) compilation | 不支持 | 支持 |
| Table/index statistics | 支持 | 支持 |
| SQL optimization / tuning |  自动 SQL 调优、SQL 计划管理 | 基于成本的优化器 |
| Query plan visualization | 支持 | EXPLAIN ANALYZE可视化工具 |
| Workload/resource management | 资源管理器 |  需要手动管理或第三方工具 |
| Connection pooling | 依赖于应用程序 | 通过 pgBouncer， Pgpool-II |
| External data access | Oracle 网关 | 外部数据包装器 （FDW）|

Oracle 为大型工作负载提供了更多内置的企业级性能功能。PostgreSQL 涵盖了大多数基本要素并不断改进，尤其是在最新版本中。

## OLTP 工作负载（性能）

| Feature | Oracle | PostgreSQL |
|:----:|:----:|:----:|
| OLTP optimization | 高吞吐量、低延迟 | 适用于典型工作负载 |
| Concurrency handling | 先进，负载下可预测 | 高效的 MVCC，提高并发性 |
| In-memory data support | 内存中列存储 | （提供扩展） |
| Result caching | 支持 | 不支持 |
| Query optimization | 复杂的优化器 | 基于成本，性能高，适用于简单查询 |
| Performance at scale | 非常好 | 可能需要针对非常高的音量进行调整 |

Oracle 在大容量 OLTP 方面表现出色;PostgreSQL 在调优方面表现良好，适用于大多数事务性工作负载。

# 分析工作负载（性能）

| Feature | Oracle | PostgreSQL |
|:----:|:----:|:----:|
| Parallel query execution | 支持 | 支持 |
| Bitmap indexes | 支持 | 不支持 |
| Star query optimization | 支持 | 不支持 |
| Materialized views | 使用查询重写 | 手动刷新 |
| Partitioning | 成熟，多种策略 | 在最新版本中有所改进 |
| In-memory analytics | 内存中列存储 | 不支持 |
| Semi-structured data support | JSON 支持（有限） | 用于分析的 JSONB |
| External data integration | Oracle 网关 | 外国数据包装器 |
| Time-series data support | 不支持 | 使用 TimescaleDB 扩展 |

Oracle 在大规模分析方面处于领先地位;PostgreSQL 的功能越来越强大，尤其是扩展。

## 基准比较（性能）

### Oracle 数据库一体机 X9-2-HA

- 每秒事务数 （TPS）：超过 35,000 TPS，每个节点 32 个 CPU 内核
- 平均事务响应时间：小于 12.1 毫秒

### PostgreSQL（托管云服务）：

| Platform | Transactions/sec (TPS) | Avg. Latency (ms) |
|:----:|:----:|:----:|
| AWS RDS for PostgreSQL | ~2,700 | ~2.88 |
| Azure Database for PostgreSQL | ~2,400 | ~3.26 |
| Google Cloud SQL for PostgreSQL | ~1,300 | ~5.74 |
| Supabase PostgreSQL | ~1,600 | ~5.10 |

- Oracle 在高吞吐量环境中表现出卓越的性能，尤其是在优化硬件配置的情况下。
- PostgreSQL 为大多数工作负载提供稳定的性能，并具有降低运营成本的额外优势。

## 安全功能

| Feature | Oracle | PostgreSQL |
|:----:|:----:|:----:|
| Row-level security | VPD（虚拟专用数据库） | 本机策略 |
| Multi-level security | 标签安全 | 不支持 |
| Separation of duties | 数据库保管库 | ⚠️ 手动角色管理 |
| Data encryption at rest | 透明数据加密 （TDE） | ⚠️ 文件系统级加密 |
| Column-level data masking | 数据写入 | ⚠️ 需要自定义实现或扩展 |
| Column-level privileges | 支持 | 支持 |
| Role-based access control | 支持 | 支持 |
| SSL/TLS encryption | 支持 | 支持 |
| External authentication | 企业用户安全（LDAP、Kerberos） | LDAP、GSSAPI |
| Audit logging | 内置，全面 | ⚠️ 通过 pgaudit 等扩展 |
| Privilege analysis | 支持 | 不支持 |


Oracle 提供了更多开箱即用的安全工具，适合严格的合规性和企业使用。PostgreSQL 满足大多数核心需求，扩展填补了高级空白。

# 安装和设置（管理）

| Feature | Oracle | PostgreSQL |
|:----:|:----:|:----:|
| Installation complexity | ❌ 复杂的多个组件和配置 | ✅ 提供简单的包管理器（apt、yum 等） |
| Disk space requirements | ❌ 高（最低 ~6.8 GB） | ✅ 低（因平台而异） |
| Pre-installation requirements | ❌ 详细的先决条件（用户、内核参数等） |  ✅ 最低先决条件 |
| Installation tools | ✅ Oracle 通用安装程序 | ✅ 本机安装程序、Docker 容器 |
| Configuration options | ✅ 广泛 | ⚠️ 初始选项更少，可在安装后配置 |

Oracle 的安装更加复杂且资源密集型，而 PostgreSQL 提供了更简单、更直接的设置过程。

# 日常运作（Administration）

| Feature | Oracle | PostgreSQL |
|:----:|:----:|:----:|
| Graphical administration tools | ✅ 企业经理 | ✅ pg管理员 |
| Command-line tools | ✅ SQL*Plus、SQLcl | ✅ psql的 |
| Memory management | ✅ 自动 | ⚠️ 需要手动调整 |
| Storage management | ✅ 自动存储管理 （ASM） | ⚠️ 手动配置 |
| Performance monitoring | ✅ 自动工作负载存储库 （AWR） | ⚠️ pg_stat_statements 等扩展 |
| Workload management | ✅ 数据库资源管理器 | ⚠️ 手动或第三方工具 |
| Backup and recovery | ⚠️ 复杂的程序 | ✅ 简单工具（pg_dump、pg_restore） |

Oracle 提供了全面的内置管理工具，而 PostgreSQL 则更多地依赖手动配置和第三方工具。

# 监察及诊断(Administration)

| Feature | Oracle | PostgreSQL |
|:----:|:----:|:----:|
| Diagnostic repository | ✅ 自动诊断存储库 （ADR） | ❌ 不可用 |
| Performance data collection | ✅ 自动工作负载存储库 （AWR） | ✅ 自动工作负载存储库 （AWR） |
| Session history | ✅ 活动会话历史记录 （ASH） | ❌ 不可用 |
| Query monitoring | ✅ SQL 监控 | ⚠️ 使用 EXPLAIN ANALYZE 进行手动分析 |
| Graphical dashboards | ✅ 企业经理 | ⚠️ 第三方工具（例如 pgAdmin、pganalyze） |
| Log and trace files | ✅ 警报日志、跟踪文件 | ✅ 日志文件 |
| Dynamic performance views | ✅ V$ 次观看 | ✅ pg_stat_* 次观看 |

Oracle 提供了广泛的内置监控和诊断工具，而 PostgreSQL 提供了基本功能，并可选择通过扩展和第三方工具进行增强。

## 云产品比较

### Oracle 自治数据库

- 完全托管（自调整、修补、扩展）
- 支持 OLTP 和分析工作负载
- 估计费用：2 个 vCPU、8GB RAM、100GB 存储 1,500 美元至 2,000 美元/月

### 数据库云服务

- 通过可配置的服务级别进行手动管理
- 估计费用：800 美元至 1,200 美元/月

## PostgreSQL 托管服务

### 适用于 PostgreSQL 的 AWS RDS

- 完全托管，支持多可用区，自动备份
- 性能：~2,700 TPS，2.88 毫秒延迟
- 费用：$ 141.44 /月

### Azure Database for PostgreSQL

- 具有高可用性的灵活服务器
- 性能：~2,400 TPS，3.26 毫秒延迟
- 费用：$ 141.44 /月

### 适用于 PostgreSQL 的 Google Cloud SQL

- 通过自动化维护进行全面管理
- 性能：~1,300 TPS，5.74 毫秒延迟
- 费用： $ 116.70 /月

### 其他提供商

- Supabase、Heroku、DigitalOcean、Aiven、Amazon Aurora（兼容 PostgreSQL）

## 成本比较（2 个 vCPU、8GB RAM、100GB 存储）

| Provider | Monthly Cost | Notes |
|:----:|:----:|:----:|
| Oracle Autonomous Database Oracle | 1,500 美元至 2,000 美元 | 高级功能、自动化 |
| Oracle Database Cloud Service | $800–$1,200 | 手动设置和管理 |
| AWS RDS for PostgreSQL | $141.44 | 高性能，完全托管 |
| Azure Database for PostgreSQL | $141.44 | 配置灵活，高可用 |
| Google Cloud SQL for PostgreSQL | $116.70 | 成本更低，延迟更高 |

注意：价格为近似值，可能因地区和用途而异。
Oracle 以高价提供丰富的企业功能。PostgreSQL 云服务提供了一种经济高效、灵活且性能稳定的替代方案。


# 用例和行业采用

## Oracle 最适合

- 大型企业应用程序（ERP、CRM、财务）
- 关键任务系统（银行、电信、医疗保健）
- 海量数据仓库和实时分析
- 大批量 OLTP（交易、预订、电子商务）

常见用户：财富 500 强公司、主要银行和政府机构、大型医疗保健和电信提供商

## PostgreSQL 最适合：

- Web 和 SaaS 应用程序
- 地理空间应用（使用 PostGIS）
- 开发和 CI/CD 环境
- 精打细算的使用（初创企业、教育机构、非营利组织）
- 混合数据应用（JSON、XML、自定义类型）


## 结论

Oracle 在预算不受限制且需要最大可靠性的企业环境中表现出色。它非常适合任务关键型应用程序、大规模数据仓库以及需要开箱即用的全面企业功能的场景。

PostgreSQL 在现代应用程序、成本敏感型部署以及需要灵活性和可扩展性的场景中大放异彩。它非常适合受益于其现代功能和活跃社区的 Web 应用程序、初创公司和项目。


原文来自于：[oracle-vs-postgres](https://www.bytebase.com/blog/oracle-vs-postgres/)





