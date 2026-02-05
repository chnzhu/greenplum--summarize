#  深度解密MySQL2PG工具MySQL至PostgreSQL语法全景拆解过程

# 前言

在第一篇中介绍了现有转换工具的痛点，以及MySQL2PG该工具产生的背景，避免了手动转换SQL的效率低下、容易出错、存在兼容性差等困境。在MySQL2PG V2.0的版本中介绍了完善的功能，该工具的核心的优势以及标准化的转换流程，让大家对该工具有了全面的认识。接下来分享该工具在解析 MySQL语法转换到PG 时的核心特点以及主要做的针对性的测试案例，从而确保该工具的稳定、可靠。

# 目录结构说明

程序的主要的结构如下，其中在 `scripts/mysql` 内创建了`create_table.sql` 脚本改脚本内共有100同步的表，每个表都按照真实的客户场景进行模拟，涵盖几乎所有的DDL常见语法。 `create_index.sql` 对`create_table.sql`  你的每一张表都创建了索引，使索引全覆盖。在`create_view.sql` 按照真实的业务场景创建了30多个view，并且每个view都包含多表的JOIN操作，语法的转换。在`create_user.sql`内创建了两个用户，并为该用户赋予了不同的权限。在`create_function.sql` 中公创建了100个模拟的操作，每个案例都有不同的表的操作，都涉及到复杂的语法等，

```sql
mysql2pg/
├── cmd/                          # 应用程序入口
├── internal/                     # 转换代码
│   ├── config/                   # 配置管理
│   ├── converter/                # 数据转换核心
│   │   ├── postgres/             # PostgreSQL 连接
│   │   └── greenplum/            # Greenplum 连接
│   ├── mysql/                    # mysql 连接
│   ├── postgres/                 # PostgreSQL 连接
├── scripts/                      # 脚本文件
│   ├── integration/              # 集成测试
│   ├── mysql/                    # MySQL 测试脚本
│   ├── postgres/                 # PostgreSQL 验证脚本
├── test/                         # 测试相关
```

# CREATE_TABLE 语法兼容

在`scripts/mysql`目录下，创建了 `create_table.sql` 脚本，该脚本共创建了100张表，这100张表几乎覆盖了MySQL常见的DDL语法，大约有35+的数据类型、10+以上的类型修饰符，8+以上的不同的表类型、6+以上的表约束类型等。充分模拟了客户的真是的场景，该100张表覆盖了主键表、字段表、复合主键、外键关联、生成列、临时表、带有COMMIT的表等冲锋考虑了表设计的多样性和复杂性，具体的数据类型统计如下：

```sql
1、包含 tinyint、smallint、mediumint、int、bigint、char、varchar、text、blob 、json、geometry、enum、set、bit、year 等35+以上的数据类型。
2、包含 unsigned、zerofill、character set、collate、auto_increment 等10+以上的类型修饰符
3、包含 engine、default charset、collate、row_format、stats_persistent 等8+中表的选项
4、包含 primary key、unique key、key、index、foreign key、check 等6+余种的约束类型
5、包含 range、list、hash、linear hash、subpartition 等5+以上的分区类型
6、包含 生成列、临时表、反引号标识符等
```

# CREATE_INDEX

在`scripts/mysql`目录下，创建了 `create_index.sql` 脚本，该脚本是按照`create_table.sql` 中的表来来创建的对应的索引，覆盖了普通索引、唯一索引、全文索引、空间索引等4+以上的索引类型，覆盖了BLOB&TEXT的大字段的前缀索引的创建案例，包含了多列的符合索引覆盖了不同的索引创建案例。


# CREATE_VIEW

在 scripts/mysql 目录下，创建了 create_view.sql 脚本，该脚本是从`create_table.sql` 中获取的表，进行模拟业务的真是的查询场景，包含了30+的业务视图，这些VIEW 的特点为包含mysql的整数类型表的简单查询、布尔类型、浮点数类型、字符类型、日期时间类型、子查询、聚合函数、条件函数、窗口函数、JSON操作、CASE表达式、CTE嵌套、复杂连接和子查询等复杂的SQL语法，而这些语法能够完全无缝转到到PG中，具体的语法转换如下:

```sql
1、包含 if、ifnull、case、json_extract、json_unquote、year、month、day、date_format、round 50+余种的函数。
2、包含 left join、right join、inner join 3 余种的连接类型。
3、包含 多层嵌套的子查询
4、包含 count、sum、avg、max、min、group_concat 8 余种的聚合函数
5、包含 json_extract、json_unquote、json_keys、json_length、->> 10 余种 的json 操作
6、包含 rlike、instr、replace 5 余种正则表达式
```

# CREATE_FUNCTION


在 scripts/mysql 目录下，创建了 `create_function.sql` 脚本，该脚本中共创建了 100 种的FUNCTION的类型，这100种的存储函数覆盖了 MySQL几乎所有的常见的存储过程及函数的相关语法，语法中包含了游标的全生命周期操作、异常处理机制、6+种的 流程控制语法及字符串操作函数、单表数据查询、字段计算函数，10+种的多表关联、游标遍历、异常捕获、循环逻辑、嵌套判断、批量数据操作等复杂特性的函数结构，具体的语法如下：

```sql
1、支持完整的 mysql 存储函数的定义语法
2、支持多种类型的变量声明与赋值
3、支持完整的declare、open、fetch、close游标生命周期管理
4、支持 declare continue handler 语法的异常处理
5、支持 if、elseif、else、case、loop、leave 流程控制语法
6、支持 concat、substring、length 字符串操作的语法
7、支持10+表关联的复杂多表连接查询的连接语法
```

通过正则匹配和语法解析数的形式，适配了`MySQL`的独有的语法，实现了从MySQL 的5.x 到8.x的语法全覆盖。解决了从MySQL到PG语法不兼容的问题，通过内置类型映射表实现精准转换，避免了数据丢失或存储异常的问题，进一步提升了迁移过程的准确性、迁移的效率、建好了手动处理SQL的复杂性。

# 使用及反馈

### 直接下载二进制

在github 以编译好了Windows/Linux平台的二进制，可以直接下载使用。

> 下载地址：https://github.com/xfg0218/MySQL2PG/releases

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
