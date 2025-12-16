# PostgreSQL16中的双流复制

## 前言

在PG16之前的版本仅支持单项流的复制，而在PG16之后支持了双向流的复制功能，有了双向流的复制，便可以实现两套集群的数据实时同步，从而实现集群的高可用性、灾难恢复和读写分离等目标。

## 环境

1. 使用单机环境
2. 使用PG16.1初始化两个数据库:
    • db1 的 port=5432，wal_level=logical，listen_addresses='*'
    • db2 的 port=5433，wal_level=logical，listen_addresses='*'
3. 配置连接方式为免密
```sql
host    all     all             0.0.0.0/32                 trust
```

## 步骤

### db1
```sql
postgres=# create database clone;
CREATE DATABASE
postgres=# \c clone
You are now connected to database "clone" as user "postgres".
```

### db2
```sql
postgres=# create database clone;
CREATE DATABASE
postgres=# \c clone
You are now connected to database "clone" as user "postgres".
```

### db1
```sql
clone=# CREATE TABLE t1 (id int);
CREATE TABLE
clone=# CREATE TABLE t2 (id int);
CREATE TABLE
clone=# CREATE TABLE t3 (id int);
CREATE TABLE
```

### db2
```sql
clone=# CREATE TABLE t1 (id int);
CREATE TABLE
clone=# CREATE TABLE t2 (id int);
CREATE TABLE
clone=# CREATE TABLE t3 (id int);
CREATE TABLE
```


## 创建发布者

在以下的数据库中创建发布所有的表

### db1
```sql
clone=# create publication pub0 forall tables;
CREATE PUBLICATION
clone=# select*from pg_publication;
  oid  | pubname | pubowner | puballtables | pubinsert | pubupdate | pubdele
te | pubtruncate | pubviaroot
-------+---------+----------+--------------+-----------+-----------+--------
---+-------------+------------
16413| pub0    |       10| t            | t         | t         | t
   | t           | f
(1row)

clone=# select*from pg_publication_tables ;
 pubname | schemaname | tablename | attnames | rowfilter
---------+------------+-----------+----------+-----------
 pub0    | public     | t1        | {id}     |
 pub0    | public     | t2        | {id}     |
 pub0    | public     | t3        | {id}     |
(3rows)
```

### db2
```sql
clone=# create publication pub00 forall tables;
CREATE PUBLICATION
clone=# select*from pg_publication;
  oid  | pubname | pubowner | puballtables | pubinsert | pubupdate | pubdelet
e | pubtruncate | pubviaroot
-------+---------+----------+--------------+-----------+-----------+---------
--+-------------+------------
16410| pub00   |       10| t            | t         | t         | t
| t           | f
(1row)

clone=# select*from pg_publication_tables ;
 pubname | schemaname | tablename | attnames | rowfilter
---------+------------+-----------+----------+-----------
 pub00   | public     | t1        | {id}     |
 pub00   | public     | t2        | {id}     |
 pub00   | public     | t3        | {id}     |
(3rows)
```

### 创建订阅者

在以下的数据库中创建订阅对应的表。

### db1
```sql
clone=# create subscription sub0 connection 'host=127.0.0.1 dbname=clone port=5433' publication pub00 with (copy_data =false, origin =none);
NOTICE:  created replication slot "sub0" on publisher
CREATE SUBSCRIPTION
clone=# select*from pg_subscription;
  oid  | subdbid | subskiplsn | subname | subowner | subenabled | subbinary
| substream | subtwophasestate | subdisableonerr | subpasswordrequired | sub
runasowner |              subconninfo              | subslotname | subsyncco
mmit | subpublications | suborigin
-------+---------+------------+---------+----------+------------+-----------
+-----------+------------------+-----------------+---------------------+----
-----------+---------------------------------------+-------------+----------
-----+-----------------+-----------
16414|   16401|0/0        | sub0    |       10| t          | f
| f         | d                | f               | t                   | f
           | host=127.0.0.1 dbname=clone port=5433| sub0        | off
     | {pub00}         |none
(1row)
```

### db2
```sql
clone=# create subscription sub00 connection 'host=127.0.0.1 dbname=clone port=5432' publication pub0 with (copy_data =false, origin =none);
NOTICE:  created replication slot "sub00" on publisher
CREATE SUBSCRIPTION
clone=# select*from pg_subscription;
  oid  | subdbid | subskiplsn | subname | subowner | subenabled | subbinary |
 substream | subtwophasestate | subdisableonerr | subpasswordrequired | subru
nasowner |              subconninfo              | subslotname | subsynccommi
t | subpublications | suborigin
-------+---------+------------+---------+----------+------------+-----------+
-----------+------------------+-----------------+---------------------+------
---------+---------------------------------------+-------------+-------------
--+-----------------+-----------
16411|   16400|0/0        | sub00   |       10| t          | f         |
 f         | d                | f               | t                   | f
         | host=127.0.0.1 dbname=clone port=5432| sub00       | off
| {pub0}          |none
(1row)
```

### 插入数据进行测试

### db1
```sql
clone=# insert into t1 values (1);
INSERT 0 1
```

### db2
```sql
clone=# select * from t1 ;
 id
----
  1
(1 row)
```

### db2 插入数据
```sql
clone=# insert into t1 values (2);
INSERT 0 1
```

### db1 插入数据
```sql
clone=# insert into t1 values (2);
INSERT 0 1
```


