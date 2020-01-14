# 4 外部数据封装器（FDW）和并行查询

本章将介绍两个技术上有趣又实用的功能：外部数据包装器 （FDW） 和并行查询（Parallel Query）。

目前，仅提供 第4.1节FDW；第 4.2节并行查询 尚在构思中。

## 4.1. 外部数据封装器 (FDW)

2003 年，在SQL标准中添加了一个访问远程数据的规范，称为[[外部数据的 SQL 管理(SQL Management of External Data)](https：//wiki.postgresql.org/wiki/SQL/MED)（SQL/MED）。自版本 9.1 以来，PostgreSQL 一直在开发此功能，以实现一项自己的 SQL/MED。

在SQL/MED规范中，远程服务器上的表称作*外部表（foreign table）*。PostgreSQL的**外部数据封装器 (FDW)** 是使用 SQL/MED 来像管理本地表一样来管理类外部表。

**图 4.1. FDW的基础概念**

![Fig. 4.1. Basic concept of FDW.](http://www.interdb.jp/pg/img/fig-4-fdw-1.png)![img]()

安装必要的扩展并进行适当的设置后，可以访问远程服务器上的外表。例如，假设有两个远程服务器，即postgresql 和 mysql，它们分别具有表*foreign_pg_tbl*和 表*foreign_my_tbl *。在此例子中，你可以通过从本地服务器发出 SELECT 查询，来访问外部表，如下所示。

```sql-monosp
localdb=# -- foreign_pg_tbl is on the remote postgresql server. 
localdb-# SELECT count(*) FROM foreign_pg_tbl;
 count 
-------
 20000

localdb=# -- foreign_my_tbl is on the remote mysql server. 
localdb-# SELECT count(*) FROM foreign_my_tbl;
 count 
-------
 10000
```

此外，你可以使用在不同服务器上的表执行join操作，就像使用本地表那样。

```sql-monosp
localdb=# SELECT count(*) FROM foreign_pg_tbl AS p, foreign_my_tbl AS m WHERE p.id = m.id;
 count 
-------
 10000
```

已经有很多 FDW 开发出来并且列在 [Postgres wiki](https://wiki.postgresql.org/wiki/Foreign_data_wrappers)中。但是，几乎所有其他的扩展都没有得到好的维护， 除了[postgres_fdw](https://www.postgresql.org/docs/current/static/postgres-fdw.html)，它是由PostgreSQL全球开发组开发的一个扩展，用于访问远程的PostgreSQL服务器。

PostgreSQL的FDW会在接下来的章节中详细介绍，小节 4.1.1 展示了PostgreSQL中FDW的概览。 小节 4.1.2 介绍了 postgres_fdw 扩展的工作方式。

### 4.1.1. 概览

要使用FDW功能，需要安装恰当的扩展并执行安装命令，比如 [CREATE FOREIGN TABLE](https://www.postgresql.org/docs/current/static/sql-createforeigntable.html), [CREATE SERVER](https://www.postgresql.org/docs/current/static/sql-createserver.html) and [CREATE USER MAPPING](https://www.postgresql.org/docs/current/static/sql-createusermapping.html) (详情请参考 [官方文档](https://www.postgresql.org/docs/9.5/static/postgres-fdw.html#AEN180314)。

提供这些器当的配置后，在查询处理时会调用扩展中定义的函数，以访问外表。

图4.2 简要介绍了PostgreSQL 中 FDW的运作方式。

**图 4.2.  FDW 运作方式**

![Fig. 4.2. How FDWs perform.](http://www.interdb.jp/pg/img/fig-4-fdw-2.png)![img]()

- (1) 分析器创建输入 SQL 的查询树。
- (2) 计划器（或执行器）连接到远程服务器。
- (3) 如果 [use_remote_estimate](https://www.postgresql.org/docs/current/static/postgres-fdw.html#id-1.11.7.43.10.4) 选项设置为 *on* (默认是 *off*)，计划器执行 EXPLAIN 命令来估计每个计划路径的成本。
- (4) 计划器从计划树创建出纯文本SQL语句，这在内部称作*反解析deparesing*。
- (5) 执行器将纯文本 SQL 语句发送到远程服务器，并接收结果。

如有必要，执行器将处理接收的数据。例如，如果执行多表查询，则执行器将执行接收的数据和其他表的join处理。

后面的小节将介绍每个处理的详细信息。

#### 4.1.1.1. 创建一棵查询树

解析器使用外部表的定义，来创建输入SQL到查询树， 外部表的定义存储在 [pg_catalog.pg_class](https://www.postgresql.org/docs/current/static/catalog-pg-class.html) 和 [pg_catalog.pg_foreign_table](https://www.postgresql.org/docs/current/static/catalog-pg-foreign-table.html) 目录中，使用命令 [CREATE FOREIGN TABLE](https://www.postgresql.org/docs/current/static/sql-createforeigntable.html) 或 [IMPORT FOREIGN SCHEMA](https://www.postgresql.org/docs/current/static/sql-importforeignschema.html) 。

#### 4.1.1.2. 连接到远程服务器

要连接到远程服务器，计划器（或执行器）使用特定库连接到远程数据库服务器。例如，要连接到PostgreSQL服务器，postgres_fdw 使用 [libpq](https://www.postgresql.org/docs/current/static/libpq.html)。要连接到mysql服务器， [mysql_fdw](https://github.com/EnterpriseDB/mysql_fdw)（由EnterpriseDB开发的）使用 libmysqlclient。

连接参数，诸如用户名、服务器IP地址以及端口号，存储在 [pg_catalog.pg_user_mapping](https://www.postgresql.org/docs/current/static/catalog-pg-user-mapping.html) 和 [pg_catalog.pg_foreign_server](https://www.postgresql.org/docs/current/static/catalog-pg-foreign-server.html) 目录 ，使用命令 [CREATE USER MAPPING](https://www.postgresql.org/docs/current/static/sql-createusermapping.html) 和 [CREATE SERVER](https://www.postgresql.org/docs/current/static/sql-createserver.html) 。

#### 4.1.1.3. 使用 EXPLAIN 命令创建计划树（可选的）

PostgreSQL的FDW支持获取外部表的统计信息的功能，以估算查询的计划树，一些 FDW 或者会使用这个计划树，比如 postgres_fdw, mysql_fdw, tds_fdw 和 jdbc2_fdw。

如果 *use_remote_estimate*选项被 [ALTER SERVER](https://www.postgresql.org/docs/current/static/sql-alterserver.html)命令设置为*on*，计划器通过执行 EXPLAIN 命令查询远程服务器的计划成本；反之，则默认使用嵌入常量值。

```sql-monosp
localdb=# ALTER SERVER remote_server_name OPTIONS (use_remote_estimate 'on');
```

尽管，一些扩展使用EXPLAIN命令的值，，但只有postgres_fdw才能反映 EXPLAIN 命令的结果，因为 PostgreSQL 的 EXPLAIN 命令返回start-up和total成本。

EXPLAIN 命令的结果不能被其他DBMS fdw扩展用作计划。例如，mysql的EXPLAIN 命令仅返回估计的行数；然而， PostgreSQL的计划器需要多多信息来估算成本（如 [第3章](http://www.interdb.jp/pg/pgsql03.html)所述）。

#### 4.1.1.4. 反解析（Deparesing）

要生成计划树，计划器会从外部表的计划树扫描路径来创建纯文本 SQL 语句。例如，图 4.3 展示了以下 SELECT 语句的计划树。

```sql-monosp
localdb=# SELECT * FROM tbl_a AS a WHERE a.id < 10;
```

图 4.3 显示，从PlannedStmt的计划树连接过来的ForeignScan结点，存储了一个纯SELECT 文本。此处，postgres_fdw 通过解析和分析生成的查询树，来重新创建一个纯SELECT文本，这在 PostgreSQL中称作**反解析（deparsing）** 。

**图 4.3. 扫描一个外部表的计划树的示例**

![Fig. 4.3. Example of the plan tree that scans a foreign table.](http://www.interdb.jp/pg/img/fig-4-fdw-3.png)![img]()

mysql_fdw通过查询树为MySQL 重新创建一个 SELECT 文本。使用 [redis_fdw](https://github.com/pg-redis-fdw/redis_fdw) 或者 [rw_redis_fdw](https://github.com/nahanni/rw_redis_fdw) 创建 [SELECT 命令](https://redis.io/commands/select)。

#### 4.1.1.5. 发送 SQL 语句并接收结果 

反解析后，执行器将反解析的SQL语句饭送到远程服务器并接收结果。

发送SQL语句到远程服务器的方法，取决于每个扩展的开发人员。例如，mysql_fdw不使用事务即可发送 SQL 语句。mysql_fdw中执行 SELECT 查询的 SQL 语句的典型序列如下所示（图 4.4）。

​    (5-1) 将 SQL_MODE 设置为 'ANSI_QUOTES'。

​    (5-2) 向远程服务器发送 SELECT 语句。

​    (5-3) 从远程服务器接收结果。

​    在这里，mysql_fdw将结果转化为PostgreSQL可读的数据。

​    所有的 FDW 扩展都实现了将结果转换为 PostgreSQL可读数据的功能。

**图 4.4. 在mysql_fdw中执行 SELECT 查询的典型 SQL 语句顺序**

![Fig. 4.4. Typical sequence of SQL statements to execute a SELECT query in mysql_fdw](http://www.interdb.jp/pg/img/fig-4-fdw-4.png)![img]()

远程服务器的实际日志可以在下面看到；里面将显示远程服务器收到的语句。

```sql-monosp
mysql> SELECT command_type,argument FROM mysql.general_log;
+--------------+-----------------------------------------------------------------------+
| command_type | argument                                                              |
+--------------+-----------------------------------------------------------------------+
... snip ...

| Query        | SET sql_mode='ANSI_QUOTES'                                            |
| Prepare      | SELECT `id`, `data` FROM `localdb`.`tbl_a` WHERE ((`id` < 10))         |
| Close stmt   |                                                                       |
+--------------+-----------------------------------------------------------------------+
```

在 postgres_fdw中，SQL 语句的顺序是复杂的。postgres_fdw中，执行SELECT查询的典型SQL语句的顺序如下所示 (图 4.5)。

​	(5-1) 启动远程事务

​	 默认的远程事务隔离级别为 REPEATABLE READ；如果本地事务的隔离级别设置为 SERIALIZABLE, 则远程事务也设置为 SERIALIZABLE。

​	(5-2)-(5-4) 声明一个游标

​	SQL 语句本质是以游标方式执行。

​	(5-5) 执行 FETCH 命令来获取结果。

​	默认情况下， FETCH命令会抓取100行。

​	(5-6) 从远程服务器接收结果。

​	(5-7) 关闭游标。

​	(5-8) 提交远程事务。

**图 4.5. postgres_fdw中执行SELECT语句的经典SQL语句顺序**

![Fig. 4.5. Typical sequence of SQL statements to execute a SELECT query in postgres_fdw.](http://www.interdb.jp/pg/img/fig-4-fdw-5.png)![img]()

远程服务器的实际日志可以在下面看到。

```sql-monosp
LOG:  parse : DECLARE c1 CURSOR FOR SELECT id, data FROM public.tbl_a WHERE ((id < 10))
LOG:  bind : DECLARE c1 CURSOR FOR SELECT id, data FROM public.tbl_a WHERE ((id < 10))
LOG:  execute : DECLARE c1 CURSOR FOR SELECT id, data FROM public.tbl_a WHERE ((id < 10))
LOG:  statement: FETCH 100 FROM c1
LOG:  statement: CLOSE c1
LOG:  statement: COMMIT TRANSACTION
```

> **postgres_fdw中的默认远程事务隔离级别**
>
>  [官方文档](https://www.postgresql.org/docs/10/static/postgres-fdw.html#id-1.11.7.43.12)中提供了默认远程事务隔离级别为"REPEATABLE READ "的解释。
>
> > 当本地事务具有 SERIALIDABLE 隔离级别时，远程事务使用 SERIALIZABLE 隔离级别；否则，它使用REPEATABLE READ隔离级别。此选项可确保，如果查询在远程服务器上执行多表扫描，它将获得所有扫描的快照一致性结果。结果是，单个事务中的连续查询将看到来自远程服务器的相同数据，即使由于其他活动在远程服务器上发生并发更新也是如此。

### 4.1.2. Postgres_fdw 扩展的运作方式

postgres_fdw扩展是一个由 PostgreSQL 全球开发组官方维护的特殊模块，其源代码包含在 PostgreSQL 源代码树中。

postgres_fdw正在逐步改善。表4.1 列出了官方文档中有关postgres_fdw的发行说明。

| Version | Description                                                  |
| :------ | :----------------------------------------------------------- |
| 9.3     | postgres_fdw模块发布。                                       |
| 9.6     | 请考虑在远程服务器上执行排序。请考虑在远程服务器上执行join。在可行的情况下，在远程服务器上完全执行UPDATE或DELETE。允许将抓取大小设置为服务器或表的选项。 |
| 10      | 在可能时，将聚合函数推送到远程服务器。                       |

鉴于上一节介绍postgres_fdw如何处理单表查询，下一节将介绍postgres_fdw如何处理多表查询、排序操作和聚合函数。

本子节重点介绍 SELECT 语句；但是，postgres_fdw也可以处理其他 DML（INSERT, UPDATE, 和 DELETE）语句，如下所示。

> **PostgreSQL的FDW 不检测死锁**
>
> postgres_fdw 和 FDW 功能不支持分布式锁管理器和分布式死锁检测功能。 因此，很容易生成死锁。例如，如果 Client_A 更新了一个本地表'tbl_local' 和一个外部表 'tbl_remote' ，同时， Client_B 更新 'tbl_remote' 和 'tbl_local'，则这两个事务处于死锁状态，但 PostgreSQL 无法检测到这个状态。因此，无法提交这些事务。 

```sql
localdb=# -- Client A
localdb=# BEGIN;
BEGIN
localdb=# UPDATE tbl_local SET data = 0 WHERE id = 1;
UPDATE 1
localdb=# UPDATE tbl_remote SET data = 0 WHERE id = 1;
UPDATE 1
localdb=# -- Client B
localdb=# BEGIN;
BEGIN
localdb=# UPDATE tbl_remote SET data = 0 WHERE id = 1;
UPDATE 1
localdb=# UPDATE tbl_local SET data = 0 WHERE id = 1;
UPDATE 1
```

#### 4.1.2.1. 多表查询

要执行多表查询，postgres_fdw使用单表 SELECT 语句获取每个外表，然后在本地服务器上join它们。 

在版本 9.5 或更早版本中，即使多个外部表存储在同一远程服务器中，postgres_fdw也是单独获取它们然后join它们。

在版本 9.6 或更高版本中，postgres_fdw已得到改进，当多个外部表在同一服务器上且 [use_remote_estimate](https://www.postgresql.org/docs/current/static/postgres-fdw.html)选项为on时，可以在远程服务器上执行远程join操作。

执行细节如下所示。

**版本 9.5 或更早版本:**

让我们来探讨 PostgreSQL 如何处理以下连接两个外部表的查询： *tbl_a* and *tbl_b*.

```sql-monosp
localdb=# SELECT * FROM tbl_a AS a, tbl_b AS b WHERE a.id = b.id AND a.id < 200;
```

查询的 EXPLAIN 命令的结果如下所示。

```sql-monosp
localdb=# EXPLAIN SELECT * FROM tbl_a AS a, tbl_b AS b WHERE a.id = b.id AND a.id < 200; 
QUERY PLAN                                  
------------------------------------------------------------------------------
Merge Join  (cost=532.31..700.34 rows=10918 width=16)
Merge Cond: (a.id = b.id)
->  Sort  (cost=200.59..202.72 rows=853 width=8)
       Sort Key: a.id
       ->  Foreign Scan on tbl_a a  (cost=100.00..159.06 rows=853 width=8)
->  Sort  (cost=331.72..338.12 rows=2560 width=8)
       Sort Key: b.id
       ->  Foreign Scan on tbl_b b  (cost=100.00..186.80 rows=2560 width=8)
(8 rows)
```

结果显示，执行器选择merge join，并按照以下步骤进行处理：

**第8行**: 执行器使用外部表扫描获取 tbl_a 。

**第6行**: 执行器对本地服务器上的从tbl_a抓取的行进行排序。

**第11行**: 执行器使用外部表扫描t获取表bl_b。

**第9行**: 执行器对本地服务器上的从tbl_b抓取的行进行排序。

**第4行**: 执行器实现了本地服务器上的merge join操作。

下面描述了执行器如何抓取行（图 4.6）。

​	    (5-1) 开启远程事务

​	    (5-2) 声明游标 c1， SELECT 语句如下所示：

​		`SELECT id,data FROM public.tbl_a WHERE (id < 200)`

​		(5-3) 执行 FETCH 命令来获取游标1的结果。

​		(5-4) 声明游标 c2，SELECT 语句如下所示：

​		`SELECT id,data FROM public.tbl_b`

​		请注意，原始双表查询的 WHERE 子句是 "tbl_a.id = tbl_b.id AND tbl_a.id < 200"；因此，可以逻辑地将 WHERE 子句"tbl_b.id < 200"添加如前所述到 SELECT 语句中。但是，postgres_fdw无法执行此推理；因此，执行器必须执行不包含任何 WHERE 子句的 SELECT 语句，并且必须获取外表 *tbl_b*的所有行。

​		此过程效率低下，因为必须通过网络从远程服务器读取不必要的行。此外，必须对接收的行进行排序才能执行 merge join。

​		(5-5) 执行FETCH命令来获取游标2的结果。

​		(5-6) 关闭游标c1。

​		(5-7) 关闭游标c2。

​		(5-8) 提交事务。

**图 4.6. 在版本9.5或更早版本中，执行多表查询的SQL语句顺序**

![Fig. 4.6. Sequence of SQL statements to execute the Multi-Table Query in version 9.5 or earlier.](http://www.interdb.jp/pg/img/fig-4-fdw-6.png)![img]()

远程服务器的实际日志如下：

```sql-monosp
LOG:  statement: START TRANSACTION ISOLATION LEVEL REPEATABLE READ
LOG:  parse : DECLARE c1 CURSOR FOR
      SELECT id, data FROM public.tbl_a WHERE ((id < 200))
LOG:  bind : DECLARE c1 CURSOR FOR
      SELECT id, data FROM public.tbl_a WHERE ((id < 200))
LOG:  execute : DECLARE c1 CURSOR FOR
      SELECT id, data FROM public.tbl_a WHERE ((id < 200))
LOG:  statement: FETCH 100 FROM c1
LOG:  statement: FETCH 100 FROM c1
LOG:  parse : DECLARE c2 CURSOR FOR
      SELECT id, data FROM public.tbl_b
LOG:  bind : DECLARE c2 CURSOR FOR
      SELECT id, data FROM public.tbl_b
LOG:  execute : DECLARE c2 CURSOR FOR
      SELECT id, data FROM public.tbl_b
LOG:  statement: FETCH 100 FROM c2
LOG:  statement: FETCH 100 FROM c2
LOG:  statement: FETCH 100 FROM c2
LOG:  statement: FETCH 100 FROM c2

... snip

LOG:  statement: FETCH 100 FROM c2
LOG:  statement: FETCH 100 FROM c2
LOG:  statement: FETCH 100 FROM c2
LOG:  statement: FETCH 100 FROM c2
LOG:  statement: CLOSE c2
LOG:  statement: CLOSE c1
LOG:  statement: COMMIT TRANSACTION
```

接收到行后，执行器对tbl_a和tbl_b收到的行来进行排序，然后使用已排序的行执行merge join操作。

**版本 9.6 或更新版本**

如果 use_remote_estimate 选项为 *on* (默认为 off)，postgres_fdw发送多个 EXPLAIN 命令，以获取与外表相关的所有计划的成本。

要发送 EXPLAIN 命令，postgres_fdw发送每个单表查询的 EXPLAIN 命令和 SELECT 语句的 EXPLAIN 命令，以执行远程join操作。在此示例中，以下7个 EXPLAIN 命令发送到远程服务器，以获取每个 SELECT 语句的估算成本；然后，计划器选择成本最小的计划。

```sql-monosp
(1) EXPLAIN SELECT id, data FROM public.tbl_a WHERE ((id < 200))
(2) EXPLAIN SELECT id, data FROM public.tbl_b
(3) EXPLAIN SELECT id, data FROM public.tbl_a WHERE ((id < 200)) ORDER BY id ASC NULLS LAST
(4) EXPLAIN SELECT id, data FROM public.tbl_a WHERE ((((SELECT null::integer)::integer) = id)) AND ((id < 200))
(5) EXPLAIN SELECT id, data FROM public.tbl_b ORDER BY id ASC NULLS LAST
(6) EXPLAIN SELECT id, data FROM public.tbl_b WHERE ((((SELECT null::integer)::integer) = id))
(7) EXPLAIN SELECT r1.id, r1.data, r2.id, r2.data FROM (public.tbl_a r1 INNER JOIN public.tbl_b r2 ON (((r1.id = r2.id)) AND ((r1.id < 200))))
```

让我们在本地服务器上执行EXPLAIN命令，来观察计划器选择的计划。

```sql-monosp
localdb=# EXPLAIN SELECT * FROM tbl_a AS a, tbl_b AS b WHERE a.id = b.id AND a.id < 200;                        QUERY PLAN                         
----------------------------------------------------------- 
Foreign Scan  (cost=134.35..244.45 rows=80 width=16)
   Relations: (public.tbl_a a) INNER JOIN (public.tbl_b b)
(2 rows)
```

结果表明，规划器选择在远程服务器上处理的内部join查询，这样效率很高。

下面介绍了postgres_fdw的处理方式（图 4.7）。

​	   (3-1) 启动远程事务。

​	   (3-2) 执行EXPLAIN 命令来估算每个计划路径的成本 。

​	   在此示例中，执行7个 EXPLAIN 命令。然后，使用执行的 EXPLAIN 命令的结果，计划器选择成本最小的 SELECT 查询。

​	   (5-1) 声明游标 c1，SELECT 语句如下所示：

`SELECT r1.id, r1.data, r2.id, r2.data FROM (public.tbl_a r1 INNER JOIN public.tbl_b r2 ON (((r1.id = r2.id)) AND ((r1.id < 200))))`

​	  (5-2) 从远程服务器接收结果。

​	  (5-3) 关闭游标 c1。

​	  (5-4) 提交事务。

**图 4.7. 版本 9.6 或更新版本中，执行远程join操作的SQL语句序列**

![Fig. 4.7. Sequence of SQL statements to execute the remote-join operation in version 9.6 or later.](http://www.interdb.jp/pg/img/fig-4-fdw-7.png)![img]()

远程服务器的实际日志如下：

```sql-monosp
LOG:  statement: START TRANSACTION ISOLATION LEVEL REPEATABLE READ
LOG:  statement: EXPLAIN SELECT id, data FROM public.tbl_a WHERE ((id < 200))
LOG:  statement: EXPLAIN SELECT id, data FROM public.tbl_b
LOG:  statement: EXPLAIN SELECT id, data FROM public.tbl_a WHERE ((id < 200)) ORDER BY id ASC NULLS LAST
LOG:  statement: EXPLAIN SELECT id, data FROM public.tbl_a WHERE ((((SELECT null::integer)::integer) = id)) AND ((id < 200))
LOG:  statement: EXPLAIN SELECT id, data FROM public.tbl_b ORDER BY id ASC NULLS LAST
LOG:  statement: EXPLAIN SELECT id, data FROM public.tbl_b WHERE ((((SELECT null::integer)::integer) = id))
LOG:  statement: EXPLAIN SELECT r1.id, r1.data, r2.id, r2.data FROM (public.tbl_a r1 INNER JOIN public.tbl_b r2 ON (((r1.id = r2.id)) AND ((r1.id < 200))))
LOG:  parse: DECLARE c1 CURSOR FOR
	   SELECT r1.id, r1.data, r2.id, r2.data FROM (public.tbl_a r1 INNER JOIN public.tbl_b r2 ON (((r1.id = r2.id)) AND ((r1.id < 200))))
LOG:  bind: DECLARE c1 CURSOR FOR
	   SELECT r1.id, r1.data, r2.id, r2.data FROM (public.tbl_a r1 INNER JOIN public.tbl_b r2 ON (((r1.id = r2.id)) AND ((r1.id < 200))))
LOG:  execute: DECLARE c1 CURSOR FOR
	   SELECT r1.id, r1.data, r2.id, r2.data FROM (public.tbl_a r1 INNER JOIN public.tbl_b r2 ON (((r1.id = r2.id)) AND ((r1.id < 200))))
LOG:  statement: FETCH 100 FROM c1
LOG:  statement: FETCH 100 FROM c1
LOG:  statement: CLOSE c1
LOG:  statement: COMMIT TRANSACTION
```

请注意，如果use_remote_estimate 选项为off (默认的)，则很少选择远程join查询，因为需要使用非常大的嵌入值来估计成本。

#### 4.1.2.2. 排序（Sort）操作

在版本 9.5 或更早版本中，排序操作（如 ORDER BY）在本地服务器上处理，即本地服务器在排序操作之前，从远程服务器获取所有目标行。让我们来探讨如何使用 EXPLAIN 命令，来处理包含 ORDER BY 子句的简单查询。

```sql-monosp
localdb=# EXPLAIN SELECT * FROM tbl_a AS a WHERE a.id < 200 ORDER BY a.id;                              QUERY PLAN                               
----------------------------------------------------------------------- 
Sort  (cost=200.59..202.72 rows=853 width=8)
    Sort Key: id
       ->  Foreign Scan on tbl_a a  (cost=100.00..159.06 rows=853 width=8)
(3 rows)
```

**第6行**: 执行器将以下查询发送到远程服务器，然后获取查询结果。

`SELECT id, data FROM public.tbl_a WHERE ((id < 200))`

**第4行**: 执行器对本地服务器上从tbl_a提取的行进行排序。

远程服务器的实际日志如下：

```sql-monosp
LOG:  statement: START TRANSACTION ISOLATION LEVEL REPEATABLE READ
LOG:  parse : DECLARE c1 CURSOR FOR
      SELECT id, data FROM public.tbl_a WHERE ((id < 200))
LOG:  bind : DECLARE c1 CURSOR FOR
      SELECT id, data FROM public.tbl_a WHERE ((id < 200))
LOG:  execute : DECLARE c1 CURSOR FOR
      SELECT id, data FROM public.tbl_a WHERE ((id < 200))
LOG:  statement: FETCH 100 FROM c1
LOG:  statement: FETCH 100 FROM c1
LOG:  statement: CLOSE c1
LOG:  statement: COMMIT TRANSACTION
```

在版本 9.6 或更高版本中，postgres_fdw可以尽可能在远程服务器上使用 ORDER BY 子句执行 SELECT 语句。

```sql-monosp
localdb=# EXPLAIN SELECT * FROM tbl_a AS a WHERE a.id < 200 ORDER BY a.id;                           QUERY PLAN                            
-----------------------------------------------------------------
Foreign Scan on tbl_a a  (cost=100.00..167.46 rows=853 width=8)
(1 row)
```

**第4行**: 执行器将具有 ORDER BY 子句的以下查询发送到远程服务器，然后获取已排序的查询结果。

`SELECT id, data FROM public.tbl_a WHERE ((id < 200)) ORDER BY id ASC NULLS LAST`

远程服务器的实际日志如下所示。此改进减少了本地服务器的工作量。

```sql-monosp
LOG:  statement: START TRANSACTION ISOLATION LEVEL REPEATABLE READ
LOG:  parse : DECLARE c1 CURSOR FOR
	   SELECT id, data FROM public.tbl_a WHERE ((id < 200)) ORDER BY id ASC NULLS LAST
LOG:  bind : DECLARE c1 CURSOR FOR
	   SELECT id, data FROM public.tbl_a WHERE ((id < 200)) ORDER BY id ASC NULLS LAST
LOG:  execute : DECLARE c1 CURSOR FOR
	   SELECT id, data FROM public.tbl_a WHERE ((id < 200)) ORDER BY id ASC NULLS LAST
LOG:  statement: FETCH 100 FROM c1
LOG:  statement: FETCH 100 FROM c1
LOG:  statement: CLOSE c1
LOG:  statement: COMMIT TRANSACTION
```

#### 4.1.2.3. 聚合函数

在版本 9.6 或更早版本中，与上一节中提及的排序操作类似，聚合函数（如AVG() 和 cont()）在本地服务器上的处理步骤，如下所示。

```sql-monosp
localdb=# EXPLAIN SELECT AVG(data) FROM tbl_a AS a WHERE a.id < 200;                              QUERY PLAN                              
-----------------------------------------------------------------------
Aggregate  (cost=168.50..168.51 rows=1 width=4)
   ->  Foreign Scan on tbl_a a  (cost=100.00..166.06 rows=975 width=4)
(2 rows)
```

**第5行**: 执行器将以下查询发送到远程服务器，然后获取查询结果。

`SELECT id, data FROM public.tbl_a WHERE ((id < 200))`

**第4行**: T执行器计算本地服务器上从tbl_a提取的行的平均值。

远服务器的实际日志如下所示。此过程成本高昂，因为发送大量的行会消耗大量网络流量，并且需要花费很长时间。

```sql-monosp
LOG:  statement: START TRANSACTION ISOLATION LEVEL REPEATABLE READ
LOG:  parse : DECLARE c1 CURSOR FOR
      SELECT data FROM public.tbl_a WHERE ((id < 200))
LOG:  bind : DECLARE c1 CURSOR FOR
      SELECT data FROM public.tbl_a WHERE ((id < 200))
LOG:  execute : DECLARE c1 CURSOR FOR
      SELECT data FROM public.tbl_a WHERE ((id < 200))
LOG:  statement: FETCH 100 FROM c1
LOG:  statement: FETCH 100 FROM c1
LOG:  statement: CLOSE c1
LOG:  statement: COMMIT TRANSACTION
```

在版本 10 或更高版本中，postgres_fdw尽可能使用远程服务器上的聚合函数，来执行 SELECT 语句。

```sql-monosp
localdb=# EXPLAIN SELECT AVG(data) FROM tbl_a AS a WHERE a.id < 200;                     QUERY PLAN                      
----------------------------------------------------- 
Foreign Scan  (cost=102.44..149.03 rows=1 width=32) 
    Relations: Aggregate on (public.tbl_a a)
(2 rows)
```

**第4行**: 执行器向远程服务器发送包含AVG() 函数的以下查询，然后获取查询结果。

`SELECT avg(data) FROM public.tbl_a WHERE ((id < 200))`

远程服务器的实际日志如下。此过程显然很有效，因为远程服务器计算平均值，并且只发送一行作为结果。

```sql-monosp
LOG:  statement: START TRANSACTION ISOLATION LEVEL REPEATABLE READ
LOG:  parse : DECLARE c1 CURSOR FOR
	   SELECT avg(data) FROM public.tbl_a WHERE ((id < 200))
LOG:  bind : DECLARE c1 CURSOR FOR
	   SELECT avg(data) FROM public.tbl_a WHERE ((id < 200))
LOG:  execute : DECLARE c1 CURSOR FOR
	   SELECT avg(data) FROM public.tbl_a WHERE ((id < 200))
LOG:  statement: FETCH 100 FROM c1
LOG:  statement: CLOSE c1
LOG:  statement: COMMIT TRANSACTION
```

> **Push-down**
>
> 与给定的示例类似，本地服务器允许远程服务器处理某些操作（如聚合过程），而 **push-down** 是其中的一种。

## 4.2. 并行查询

![img](http://www.interdb.jp/pg/img/udc1.jpg)