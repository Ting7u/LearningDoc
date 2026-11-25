# 第 1 章 PostgreSQL 概述与安装

> 本章目标:知道 PostgreSQL 是什么、怎么装、装完后磁盘上有什么、怎么用 `psql` 连上去,并理解"集群 → 数据库 → Schema → 表"这套对象层级。

---

## 1.1 PostgreSQL 是什么

PostgreSQL(简称 PG,读作 "post-gres-Q-L")是一个开源的**对象-关系型数据库管理系统(ORDBMS)**,以稳定、标准兼容度高、扩展性强著称。

### 1.1.1 发展简史

| 时间 | 事件 |
|---|---|
| 1986 | 加州大学伯克利分校,Michael Stonebraker 教授启动 **POSTGRES** 项目(Ingres 的后继,"Post-Ingres") |
| 1995 | Andrew Yu、Jolly Chen 为其加入 SQL 解释器,发布 **Postgres95** |
| 1996 | 项目更名 **PostgreSQL**,版本号从 6.0 起步,社区接手开发 |
| 2005 | 8.0 发布,原生支持 Windows、PITR、表空间 |
| 2010 | 9.0 发布,内置流复制与 Hot Standby |
| 2017 | 10 发布,版本号规则变化;引入声明式分区、逻辑复制 |
| 2018 之后 | 每年一个大版本:并行查询、JIT、`MERGE`(15)、`JSON_TABLE`(17)等陆续加入 |

许可证为 **PostgreSQL License**(类 BSD/MIT),可免费商用、修改、闭源再分发。没有单一商业公司"拥有"它,由全球社区(PGDG)维护。

### 1.1.2 核心特性

- **ACID 事务**与基于 **MVCC** 的并发控制(读不阻塞写,写不阻塞读)
- **事务性 DDL**:`CREATE/ALTER/DROP TABLE` 也可以回滚(MySQL、Oracle 不行)
- 丰富的数据类型:`jsonb`、数组、范围、几何、网络地址、全文检索……
- 多种索引:B-tree、Hash、GiST、SP-GiST、GIN、BRIN
- 强大的 SQL:CTE、递归查询、窗口函数、`LATERAL`、`MERGE`
- 可扩展:自定义类型、函数、运算符、索引方法、过程语言、扩展(PostGIS、pgvector、TimescaleDB 等)
- 复制与高可用:流复制、逻辑复制
- 分区表、外部数据包装器(FDW)、行级安全(RLS)

### 1.1.3 与 MySQL / Oracle 对比

| 维度 | PostgreSQL | MySQL(InnoDB) | Oracle |
|---|---|---|---|
| 许可 | BSD 类,完全开源 | GPL / 商业双许可(Oracle 持有) | 商业闭源 |
| 标准兼容 | 高 | 中 | 高(有较多私有语法) |
| 复杂查询/分析 | 强(窗口函数、CTE、多种 Join) | 近年来增强 | 很强 |
| 数据类型 | 极丰富 | 相对少 | 丰富 |
| JSON | `jsonb` + GIN,功能强 | `JSON` 类型 | 有 JSON 支持 |
| 事务性 DDL | 支持 | 不支持(隐式提交) | 不支持 |
| 扩展能力 | 极强(扩展生态) | 插件式存储引擎 | 有限 |
| 并发模型 | 多进程,MVCC(旧版本存表内,需 VACUUM) | 多线程,undo log 实现 MVCC | 多进程/线程,undo 段 |
| 运维门槛 | 中(要理解 VACUUM、WAL) | 低 | 高 |

> 选型直觉:**复杂业务、数据分析、地理/向量/时序扩展、对标准与数据正确性要求高** → PG;**简单高并发读写、团队生态偏 MySQL** → MySQL;**已有 Oracle 体系** → 评估迁移到 PG 的成本。

### 1.1.4 版本策略

- 自 PG 10 起版本号为 `主版本.小版本`,如 `16.3`:**16** 是大版本(Major),**3** 是小版本(Minor)。PG 10 之前是三段式,如 `9.6.24`,大版本是前两段 `9.6`。
- **大版本**:每年一次(通常 9~10 月),带新功能,**数据目录格式不兼容**,升级需 `pg_upgrade` 或逻辑复制。
- **小版本**:约每季度一次(2、5、8、11 月),只含 bug 与安全修复,**同大版本内数据目录兼容**,停机替换二进制并重启即可。
- **支持周期**:每个大版本获得约 **5 年**社区支持,到期即 EOL。
- 生产建议:选用受支持的最新大版本中**较稳的版本**,并始终保持最新小版本。具体支持表以官网 [versioning policy](https://www.postgresql.org/support/versioning/) 为准。

查看当前版本:

```sql
SELECT version();
SHOW server_version;
SHOW server_version_num;   -- 例如 170002,便于程序判断
```

---

## 1.2 安装方式

### 1.2.1 Linux 包管理器(推荐用于生产)

使用官方 PGDG 仓库,可获得最新版本。

**Ubuntu / Debian(apt)**

```bash
# 官方一键脚本添加 PGDG 仓库
sudo apt install -y postgresql-common
sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh

sudo apt install -y postgresql-17
# Debian 系安装后会自动 initdb 并启动集群 main
pg_lsclusters
```

**RHEL / Rocky / CentOS(dnf)**

```bash
# 先按官网 https://www.postgresql.org/download/linux/redhat/ 安装 pgdg 仓库 rpm
sudo dnf -qy module disable postgresql
sudo dnf install -y postgresql17-server postgresql17-contrib

# RHEL 系需要手动初始化
sudo /usr/pgsql-17/bin/postgresql-17-setup initdb
sudo systemctl enable --now postgresql-17
```

### 1.2.2 Docker(学习首选,最快)

```bash
docker run -d --name pg17 \
  -e POSTGRES_PASSWORD=secret \
  -e POSTGRES_DB=demo \
  -p 5432:5432 \
  -v pgdata:/var/lib/postgresql/data \
  postgres:17

# 进入 psql
docker exec -it pg17 psql -U postgres -d demo
```

> 注意:不同大版本的官方镜像对数据卷挂载路径的约定可能不同,换版本时请阅读镜像文档。

### 1.2.3 macOS / Windows

```bash
# macOS(Homebrew)
brew install postgresql@17
brew services start postgresql@17
```

Windows:从官网下载 EDB 图形安装包,一路向导即可(会设置超级用户 `postgres` 的密码、端口 5432,并安装 pgAdmin)。

### 1.2.4 源码编译(了解)

适合需要定制编译选项、做内核开发的场景。

```bash
tar xf postgresql-17.x.tar.bz2 && cd postgresql-17.x
./configure --prefix=/opt/pg17 --with-openssl --with-icu
make -j8 world
sudo make install-world
```

---

## 1.3 初始化与启停

### 1.3.1 `initdb`:创建"数据库集群"

`initdb` 会创建一个数据目录(`PGDATA`),并生成 `template0`、`template1`、`postgres` 三个初始数据库。

```bash
# 以 postgres 操作系统用户执行
initdb -D /data/pg17 \
       -E UTF8 \
       --locale=C.UTF-8 \
       --data-checksums \
       -U postgres --pwprompt
```

| 选项 | 含义 |
|---|---|
| `-D` | 数据目录位置 |
| `-E UTF8` | 默认编码 |
| `--locale` | 排序/字符分类规则(**一旦建好,影响索引排序,轻易不改**) |
| `--data-checksums` | 开启页校验和,能发现磁盘静默损坏(**建议初始化时就开**) |
| `-U / --pwprompt` | 超级用户名与密码 |

> `PGDATA` 目录权限必须是 `0700`,且属于运行 PG 的用户,否则启动会报错。

### 1.3.2 `pg_ctl`:启停管理

```bash
pg_ctl -D /data/pg17 -l /data/pg17/logfile start
pg_ctl -D /data/pg17 status
pg_ctl -D /data/pg17 reload            # 重新加载配置(不中断连接)
pg_ctl -D /data/pg17 restart -m fast
pg_ctl -D /data/pg17 stop -m fast
```

三种停止模式:

| 模式 | 行为 |
|---|---|
| `smart` | 等所有客户端自己断开后再关闭 |
| `fast`(默认) | 回滚活动事务、断开客户端、正常关闭(有检查点) |
| `immediate` | 直接中止进程,不做检查点,下次启动需崩溃恢复 |

### 1.3.3 systemd 管理

```bash
sudo systemctl start postgresql-17
sudo systemctl status postgresql-17
sudo systemctl reload postgresql-17
sudo systemctl enable postgresql-17     # 开机自启
```

### 1.3.4 首次连接与基础配置

默认只监听 `localhost`,要允许远程访问需要两步:

```ini
# postgresql.conf
listen_addresses = '*'
port = 5432
```

```conf
# pg_hba.conf —— 允许 192.168.1.0/24 网段用密码访问所有库
# TYPE  DATABASE  USER  ADDRESS           METHOD
host    all       all   192.168.1.0/24    scram-sha-256
```

```bash
# 修改 listen_addresses 需要 restart;修改 pg_hba.conf 只需 reload
sudo systemctl restart postgresql-17
```

---

## 1.4 目录结构

数据目录 `PGDATA`(Debian 系默认 `/var/lib/postgresql/17/main`)的主要内容:

```text
PGDATA/
├── PG_VERSION            # 大版本号文件
├── postgresql.conf       # 主配置文件
├── postgresql.auto.conf  # ALTER SYSTEM 写入的配置(优先级更高)
├── pg_hba.conf           # 客户端认证规则
├── pg_ident.conf         # 操作系统用户 ↔ 数据库用户映射
├── postmaster.pid        # 主进程 PID 与锁文件
├── base/                 # 各数据库的数据文件(每个库一个以 OID 命名的子目录)
├── global/               # 集群共享的系统表(如 pg_database、pg_authid)和 pg_control
├── pg_wal/               # 预写日志 WAL 文件
├── pg_xact/              # 事务提交状态(旧称 pg_clog)
├── pg_multixact/         # 多事务(行锁共享)状态
├── pg_stat/ pg_stat_tmp/ # 统计信息
├── pg_tblspc/            # 表空间的符号链接
├── pg_replslot/          # 复制槽信息
├── pg_logical/           # 逻辑复制/解码相关
└── pg_commit_ts/ pg_subtrans/ pg_snapshots/ pg_twophase/ ...
```

**把它们和实际对象对应起来(动手试)**

```sql
-- 数据目录位置
SHOW data_directory;

-- 某个库的 OID(对应 base/ 下的目录名)
SELECT oid, datname FROM pg_database;

-- 某张表对应的磁盘文件路径(相对 PGDATA)
CREATE TABLE t_demo (id int);
SELECT pg_relation_filepath('t_demo');
-- 结果类似: base/16384/16397
```

要点:
- **一张表(或索引)在磁盘上至少对应一个文件**,单文件超过 1GB 会切成 `.1`、`.2` ……
- 同目录下还有 `_fsm`(空闲空间图)、`_vm`(可见性图)后缀的附属文件(第 21 章详述)。
- **`pg_wal` 与 `base` 绝不能手工删除文件**;磁盘满了请先查复制槽/归档,而不是 `rm`。

---

## 1.5 客户端工具

### 1.5.1 psql(必须熟练)

```bash
psql -h 127.0.0.1 -p 5432 -U postgres -d demo
# 或使用连接串
psql "postgresql://postgres:secret@127.0.0.1:5432/demo?sslmode=prefer"
```

常用环境变量:`PGHOST`、`PGPORT`、`PGUSER`、`PGDATABASE`、`PGPASSWORD`;密码也可放在 `~/.pgpass`(权限 `0600`)。

**常用元命令(以反斜杠开头,是 psql 自带命令,不是 SQL)**

| 命令 | 作用 |
|---|---|
| `\?` / `\h CREATE TABLE` | 元命令帮助 / SQL 语法帮助 |
| `\l` | 列出数据库 |
| `\c dbname` | 切换数据库 |
| `\dn` | 列出 Schema |
| `\dt` / `\dt+` | 列出表(`+` 显示大小等详情) |
| `\d tablename` | 查看表结构、索引、约束 |
| `\di` `\dv` `\ds` `\df` | 索引 / 视图 / 序列 / 函数 |
| `\du` | 角色(用户) |
| `\dx` | 已安装扩展 |
| `\db` | 表空间 |
| `\x` | 切换"纵向显示"(宽表很好用),`\x auto` 自动 |
| `\timing` | 显示每条语句耗时 |
| `\e` | 用外部编辑器编辑上一条 SQL |
| `\i file.sql` | 执行 SQL 文件 |
| `\o file` | 输出结果重定向到文件 |
| `\copy` | 客户端侧的 COPY(第 2、9 章) |
| `\set` / `\echo` | 变量与输出 |
| `\watch 2` | 每 2 秒重复执行上一条语句 |
| `\conninfo` | 当前连接信息 |
| `\q` | 退出 |

**一段完整的体验流程**

```sql
postgres=# \l
postgres=# CREATE DATABASE demo;
postgres=# \c demo
demo=# CREATE TABLE users (id serial PRIMARY KEY, name text, age int);
demo=# INSERT INTO users(name, age) VALUES ('Alice', 30), ('Bob', 25);
demo=# \d users
demo=# \x
demo=# SELECT * FROM users;
demo=# \timing on
demo=# SELECT count(*) FROM users;
```

小技巧:`~/.psqlrc` 可放默认设置,例如

```text
\set QUIET 1
\timing on
\x auto
\pset null '(null)'
\set HISTSIZE 5000
\set QUIET 0
```

`\pset null '(null)'` 能让 `NULL` 与空字符串在显示上区分开,学习阶段非常有用。

### 1.5.2 图形工具

- **pgAdmin**:官方图形管理工具,功能全面
- **DBeaver**:通用数据库客户端,开发人员常用
- **DataGrip / Navicat** 等:商业工具

---

## 1.6 逻辑对象层级

```text
Cluster(集群 = 一个 PGDATA = 一个 postmaster 实例,监听一个端口)
 └── Database(数据库)           ← 连接时必须指定其一,不能跨库查询
      └── Schema(模式 / 命名空间)
           ├── Table / View / Materialized View
           ├── Index / Sequence
           └── Function / Type / Domain / ...
```

要点:
1. 一个**集群**里有多个**数据库**,它们**相互隔离**:一个连接只能访问一个库,不能直接 `SELECT * FROM otherdb.public.t`(需要 FDW 或 dblink)。
2. 一个数据库里有多个 **Schema**,新建库默认带 `public`。
3. **角色(用户)和表空间是集群级**对象,所有数据库共享。
4. 完整表名:`database.schema.table`,但 SQL 中通常只写 `schema.table`。

### 1.6.1 `search_path`

当你写 `SELECT * FROM users` 而不带 schema 时,PG 按 `search_path` 顺序依次查找。

```sql
SHOW search_path;
-- "$user", public

CREATE SCHEMA app;
CREATE TABLE app.orders (id int);

SELECT * FROM orders;           -- ERROR: relation "orders" does not exist
SET search_path TO app, public;
SELECT * FROM orders;           -- OK,在 app 里找到

-- 持久化设置
ALTER DATABASE demo SET search_path = app, public;
ALTER ROLE alice SET search_path = app, public;
```

> 安全提示:不要把不可信用户可写的 schema 放在 `search_path` 前面,否则存在"同名对象劫持"风险。PG 15 起,普通用户默认无权在 `public` 中创建对象。

---

## 1.7 系统目录

系统目录(system catalogs)是 PG 存放**元数据**的普通表,学会查它们就能"自省"整个数据库。

### 1.7.1 `pg_catalog`(PG 专有,最全)

| 系统表/视图 | 内容 |
|---|---|
| `pg_database` | 数据库列表 |
| `pg_namespace` | Schema 列表 |
| `pg_class` | 表、索引、视图、序列等(统称 relation) |
| `pg_attribute` | 列信息 |
| `pg_index` | 索引信息 |
| `pg_constraint` | 约束 |
| `pg_type` | 数据类型 |
| `pg_proc` | 函数、存储过程 |
| `pg_roles` / `pg_authid` | 角色 |
| `pg_settings` | 所有配置参数当前值 |
| `pg_stat_activity` | 当前会话(监控常用) |

```sql
-- 列出 public 下所有普通表
SELECT c.relname, c.relkind, pg_size_pretty(pg_total_relation_size(c.oid)) AS size
FROM pg_class c
JOIN pg_namespace n ON n.oid = c.relnamespace
WHERE n.nspname = 'public' AND c.relkind = 'r'
ORDER BY pg_total_relation_size(c.oid) DESC;

-- 查某个参数的值、来源、是否需重启
SELECT name, setting, unit, context, source
FROM pg_settings
WHERE name IN ('shared_buffers', 'work_mem', 'max_connections');
```

`relkind` 常见取值:`r` 表、`i` 索引、`v` 视图、`m` 物化视图、`S` 序列、`p` 分区表、`f` 外部表。

### 1.7.2 `information_schema`(SQL 标准)

可移植性好,但信息比 `pg_catalog` 少:

```sql
SELECT table_schema, table_name
FROM information_schema.tables
WHERE table_schema NOT IN ('pg_catalog', 'information_schema');

SELECT column_name, data_type, is_nullable
FROM information_schema.columns
WHERE table_name = 'users';
```

> 经验:写跨数据库兼容的工具用 `information_schema`;做 PG 专属的运维、巡检用 `pg_catalog`。

---

## 1.8 本章实践练习

1. 用 Docker 启动 PostgreSQL 17,并用 `psql` 连接。
2. 执行 `SELECT version();`,记录大版本和小版本。
3. 创建数据库 `shop`,切换进去,创建表 `customers(id serial primary key, name text)` 并插入 3 行。
4. 用 `pg_relation_filepath('customers')` 找到表文件,进入容器 `ls -l` 查看其大小;再多插入 1 万行(`generate_series`)后观察文件变化。
5. 创建 schema `sales`,在其中建表 `orders`,通过修改 `search_path` 做到不带前缀也能查询。
6. 查询 `pg_settings` 中 `shared_buffers`、`max_connections` 的值,并说明修改它们是否需要重启(看 `context` 列)。
7. 在 `~/.psqlrc` 中加入 `\pset null '(null)'` 与 `\timing on`,重新进入 psql 验证效果。

**参考答案提示(第 4 题)**

```sql
INSERT INTO customers(name) SELECT 'user_' || g FROM generate_series(1, 10000) g;
SELECT pg_relation_filepath('customers'), pg_size_pretty(pg_relation_size('customers'));
```

---

## 1.9 本章小结

- PG 是开源、标准兼容度高、扩展性强的对象-关系型数据库,支持**事务性 DDL**。
- 版本:`大版本.小版本`;大版本年更、5 年支持;小版本只修 bug,务必保持最新。
- 安装:学习用 Docker,生产用 PGDG 包;`initdb` 创建集群,建议开启 `--data-checksums`。
- 启停:`pg_ctl` / systemd;改配置分 `reload` 与 `restart` 两类。
- 目录:`base` 放数据,`pg_wal` 放日志,`global` 放共享系统表;不要手工动文件。
- 层级:**集群 → 数据库 → Schema → 对象**;角色是集群级;`search_path` 决定无前缀名称的解析。
- 元数据靠 `pg_catalog` 与 `information_schema`;熟练使用 `psql` 元命令能大幅提升效率。

**下一章预告**:SQL 基础与 DDL——动手创建数据库、Schema、表与序列,并彻底搞懂 `NULL`。
