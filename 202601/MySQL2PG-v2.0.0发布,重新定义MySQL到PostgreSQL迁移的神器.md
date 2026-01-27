# MySQL2PG v2.0.0 发布，重新定义 MySQL 到 PostgreSQL 迁移的神器

## 说明

在上一篇文档介绍了， MYSQL2PG 的 V1.0 版本的项目咱生的背景已经主要实现的功能。经过这段时间不断的功能迭代和经过大量案例的验证都通过下发布了V2.0的版本，代码适配了MySQL5.7以上的版本，PG12以上的版本，新增了VIEW的自动转换功能，优化了转换DDL时诸多的语法问题，优化了转换FUNCTION诸多的语法问题。

> https://mp.weixin.qq.com/s/KiFUC1cs0_ed8DBbw6XE8Q

## MySQL2PG 能做什么

MySQL2PG 是一款用 Go 语言开发的数据库转换工具，会将 MySQL5.7 +的数据库无缝迁移到 PG10+的数据库。它支持将MySQL的表结构、数据、视图、索引、函数、用户及用户表上的权限全链路的转换到PG数据库中，在转换时支持各种参数的配置，具有高性能、高可靠、高移植性。

## MySQL的优势是什么

| 特性 | MySQL2PG | 其他工具 |
|:----:|:----:|:----:|
| 开发语言 |  Go 语言原生 | JAVA、Python或其他的语言|
|  部署方式 | 单文件，无依赖 | 部署复杂、需要依赖第三方库|
| 跨平台性 | 支持Windows/Linux | 仅支持单平台|
| 执行效率 | go 原生开发效率高 | 效率低下|
| 参数配置 | 简介的config.yml 配置文件 | JSON或其他格式配置复杂|
| 开源性协议 | Apache 2.0 完全开源 | 收费或限制功能 |


## MYSQL2PG 的转换流程

```sql
开始
 │
 ├─▶ [Step 0] test_only 模式？
 │     ├─ 是 → 测试 MySQL & PostgreSQL 连接 → 显示版本 → 退出
 │     └─ 否 → 继续
 │
 ├─▶ [Step 1] 读取 MySQL 表定义
 │     ├─ 若 exclude_use_table_list=true → 从数据库层面过滤 exclude_table_list 中的表
 │     └─ 若 use_table_list=true → 仅获取 table_list 中的表
 │
 ├─▶ [Step 2] 转换表结构 (tableddl: true)
 │     ├─ 字段类型智能映射（如 tinyint(1) → BOOLEAN）
 │     ├─ lowercase_columns/lowercase_tables 控制字段名/表名大小写
 │     └─ 在 PostgreSQL 中创建表（skip_existing_tables 控制是否跳过）
 │
 ├─▶ [Step 3] 转换视图 (views: true)
 │     └─ MySQL 视图定义转换为 PostgreSQL 兼容语法
 │
 ├─▶ [Step 4] 同步数据 (data: true)
 │     ├─ 若 truncate_before_sync=true → 清空目标表
 │     ├─ 分批读取 MySQL 数据（max_rows_per_batch）
 │     ├─ 批量插入 PostgreSQL（batch_insert_size）
 │     ├─ 并发线程数由 concurrency 控制
 │     └─ 自动禁用外键约束和索引提高性能
 │
 ├─▶ [Step 5] 转换索引 (indexes: true)
 │     ├─ 主键、唯一索引、普通索引、全文索引 → 自动重建
 │     └─ 批量处理（max_indexes_per_batch=20）
 │
 ├─▶ [Step 6] 转换函数 (functions: true)
 │     └─ 支持50+函数映射（如 NOW() → CURRENT_TIMESTAMP，IFNULL() → COALESCE()）
 │
 ├─▶ [Step 7] 转换用户 (users: true)
 │     └─ MySQL 用户 → PostgreSQL 角色（保留密码哈希）
 │
 ├─▶ [Step 8] 转换表权限 (table_privileges: true)
 │     └─ GRANT SELECT ON table → GRANT USAGE, SELECT ON table
 │
 └─▶ [Final Step] 数据校验与完成 (validate_data: true)
       ├─ 查询 MySQL 和 PostgreSQL 表行数
       ├─ 启用之前禁用的外键约束和索引
       ├─ 若 truncate_before_sync=false → 记录不一致表，继续执行
       ├─ 输出转换统计报告和性能指标
       └─ 生成不一致表清单（如有）
```

## 如何使用

### 直接下载二进制

在github 以编译好了Windows/Linux平台的二进制，可以直接下载使用。

![二进制文件](https://fastly.jsdelivr.net/gh/bucketio/img6@main/2026/01/18/1768749078872-9ddc61b7-d0b7-44ec-a7f9-36a906c73f05.png)
> 下载地址: https://github.com/xfg0218/MySQL2PG/releases/tag/v2.0.0

### 使用源码编译

需要提前部署好Go 1.24+的环境，按照以下的方式进行编译。

```sql
# 克隆仓库
git clone https://github.com/xfg0218/mysql2pg.git
cd mysql2pg

# 构建项目
make build

# 创建配置文件
cp config.example.yml config.yml

#  修改转换部分的配置选项
conversion:
  # 转换选项，按照以下参数顺序进行
  options:
    tableddl: true    # step1: 转换表DDL
    data: true        # step2: 转换数据（先转DDL后转数据）
    view: true        # step3: 转换表视图
    indexes: true     # step4: 转换索引
    functions: true   # step5: 转换函数
    users: true       # step6: 转换用户
    table_privileges: true # step7: 转换用户在表上的权限
    lowercase_columns: true     # 控制表字段是否需要转小写，true为转小写（默认值）
    skip_existing_tables: true  # 如果表在PostgreSQL中已存在则跳过，否则创建
    use_table_list: false       # 是否使用指定的表列表进行数据同步，其他步骤不生效
    table_list: [table1]  # 指定要同步的表列表，当use_table_list为true时生效，格式为[table1,table2,...]
    exclude_use_table_list: false   # 是否使用跳过表列表，为true时忽略exclude_table_list中的表
    exclude_table_list: [table1]         # 要跳过的表列表，当exclude_use_table_list为true时生效
    validate_data: true         # 同步数据后验证数据一致性
    truncate_before_sync: true  # 同步前是否清空表数据

```

## 问题反馈

欢迎大家进行使用，如果在使用的过程有问题可以及时issue或者联系作者。

