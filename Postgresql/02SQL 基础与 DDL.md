# 第 2 章 SQL 基础与 DDL

> 本章目标:能熟练使用 `CREATE / ALTER / DROP` 管理数据库、Schema、表、序列;理解表空间与模板数据库;掌握标识符大小写规则;彻底搞懂 `NULL` 与三值逻辑;会用 `COMMENT ON` 与 `\copy`。

---

## 2.1 SQL 语言分类与基本语法

| 类别 | 全称 | 典型语句 |
|---|---|---|
| DDL | 数据定义语言 | `CREATE` `ALTER` `DROP` `TRUNCATE` `COMMENT` |
| DML | 数据操作语言 | `INSERT` `UPDATE` `DELETE` `MERGE` `COPY` |
| DQL | 数据查询语言 | `SELECT` |
| DCL | 数据控制语言 | `GRANT` `REVOKE` |
| TCL | 事务控制语言 | `BEGIN` `COMMIT` `ROLLBACK` `SAVEPOINT` |

基本语法规则:
- 语句以 `;` 结束(`psql` 中必须加)。
- 关键字不区分大小写,`select` 与 `SELECT` 等价。
- 字符串常量用**单引号**:`'hello'`;其中的单引号写两个:`'it''s'`。
- **双引号**用于标识符(表名、列名),不是字符串!
- 注释:`-- 单行注释`、`/* 多行注释 */`。

```sql
-- 这是单行注释
SELECT 'it''s PG' AS msg;      /* 多行
                                  注释 */
```

---

## 2.2 DDL:数据库(DATABASE)

### 2.2.1 创建

```sql
CREATE DATABASE shop
    OWNER = alice
    ENCODING = 'UTF8'
    LOCALE_PROVIDER = 'libc'          -- PG15+ 可选 libc / icu
    LC_COLLATE = 'en_US.UTF-8'
    LC_CTYPE = 'en_US.UTF-8'
    TEMPLATE = template0              -- 要指定与模板不同的编码/locale 时必须用 template0
    CONNECTION LIMIT = 100;
```

常用选项:

| 选项 | 说明 |
|---|---|
| `OWNER` | 所有者 |
| `ENCODING` | 字符编码(**建库后不能改**) |
| `LC_COLLATE` / `LC_CTYPE` | 排序规则 / 字符分类(**建库后不能改**) |
| `TEMPLATE` | 以哪个模板库为蓝本 |
| `TABLESPACE` | 默认表空间 |
| `CONNECTION LIMIT` | 最大连接数限制,`-1` 不限 |

> **踩坑**:中文环境下,若使用 `en_US.UTF-8` 或 `C` 排序,中文按"拼音"排序需要特殊 collation(见第 3 章)。`C` / `C.UTF-8` 排序最快,但对语言不敏感。

### 2.2.2 修改与删除

```sql
ALTER DATABASE shop RENAME TO shop_v2;                 -- 不能改当前连接的库
ALTER DATABASE shop_v2 OWNER TO bob;
ALTER DATABASE shop_v2 SET timezone = 'Asia/Shanghai'; -- 库级默认参数
ALTER DATABASE shop_v2 CONNECTION LIMIT 50;

DROP DATABASE IF EXISTS shop_v2;
-- 有连接时会失败;PG13+ 可强制:
DROP DATABASE shop_v2 WITH (FORCE);
```

> 注意:`DROP DATABASE` **不能在事务块中执行**,也不能删除当前连接的数据库。

### 2.2.3 查看

```sql
\l                                   -- psql 元命令
SELECT datname, pg_encoding_to_char(encoding) AS enc, datcollate
FROM pg_database;
SELECT pg_size_pretty(pg_database_size('shop'));
```

---

## 2.3 模板数据库

`CREATE DATABASE` 的本质是**复制一个模板库**。集群中有两个模板:

| 模板 | 作用 |
|---|---|
| `template1` | **默认模板**。你在里面创建的对象、扩展,会出现在之后所有新库中 |
| `template0` | **干净的原始模板**,不允许连接、不要修改;用于指定不同编码/locale 建库、`pg_dump` 恢复 |

```sql
-- 例:在 template1 里装好扩展,以后新库都自带
\c template1
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- 把一个已建好的库当模板,快速克隆(要求源库此时无其他连接)
CREATE DATABASE shop_test TEMPLATE shop;

-- 设置某个库为模板
UPDATE pg_database SET datistemplate = true WHERE datname = 'shop';
```

> 快速克隆测试库是日常常用招数,但源库不能有活动连接。

---

## 2.4 DDL:模式(SCHEMA)

Schema 是数据库内的"命名空间",用于:分类管理对象、多租户隔离、权限分组、避免命名冲突。

```sql
CREATE SCHEMA IF NOT EXISTS sales AUTHORIZATION alice;

-- 建 schema 的同时建表
CREATE SCHEMA hr
    CREATE TABLE emp (id int PRIMARY KEY, name text)
    CREATE VIEW emp_names AS SELECT name FROM emp;

ALTER SCHEMA sales RENAME TO sales_v2;
ALTER SCHEMA sales_v2 OWNER TO bob;

DROP SCHEMA IF EXISTS sales_v2;            -- 非空会失败
DROP SCHEMA IF EXISTS sales_v2 CASCADE;    -- 连同里面所有对象一起删(危险!)
```

**多租户示例**:每个租户一个 schema,结构相同。

```sql
CREATE SCHEMA tenant_a;
CREATE SCHEMA tenant_b;
CREATE TABLE tenant_a.orders (id serial PRIMARY KEY, amount numeric(10,2));
CREATE TABLE tenant_b.orders (LIKE tenant_a.orders INCLUDING ALL);

-- 应用连接后按租户切换
SET search_path TO tenant_a;
```

---

## 2.5 DDL:表(TABLE)

### 2.5.1 创建表

```sql
CREATE TABLE IF NOT EXISTS public.customers (
    id          bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name        varchar(50)  NOT NULL,
    email       text         UNIQUE,
    age         int          CHECK (age >= 0),
    status      text         NOT NULL DEFAULT 'active',
    created_at  timestamptz  NOT NULL DEFAULT now()
);
```

> 约束、`GENERATED` 列、范式设计等细节将在第 4 章深入;这里先掌握基本结构。

### 2.5.2 常用建表方式

```sql
-- 1) 复制结构(含约束/索引/默认值)
CREATE TABLE customers_bak (LIKE customers INCLUDING ALL);

-- 2) 查询结果建表(只复制结构+数据,不含约束索引)
CREATE TABLE vip_customers AS
SELECT * FROM customers WHERE status = 'vip';

-- 3) 只要结构,不要数据
CREATE TABLE customers_empty AS TABLE customers WITH NO DATA;

-- 4) 临时表(会话结束自动删除)
CREATE TEMP TABLE tmp_calc (id int, v numeric);
```

### 2.5.3 修改表(`ALTER TABLE`)

```sql
-- 增删列
ALTER TABLE customers ADD COLUMN phone text;
ALTER TABLE customers ADD COLUMN level int NOT NULL DEFAULT 1;
ALTER TABLE customers DROP COLUMN phone;
ALTER TABLE customers DROP COLUMN IF EXISTS phone CASCADE;

-- 改列
ALTER TABLE customers ALTER COLUMN name TYPE varchar(100);
ALTER TABLE customers ALTER COLUMN age SET DEFAULT 18;
ALTER TABLE customers ALTER COLUMN age DROP DEFAULT;
ALTER TABLE customers ALTER COLUMN email SET NOT NULL;
ALTER TABLE customers ALTER COLUMN email DROP NOT NULL;
ALTER TABLE customers RENAME COLUMN name TO full_name;

-- 改表
ALTER TABLE customers RENAME TO clients;
ALTER TABLE clients SET SCHEMA archive;
ALTER TABLE archive.clients OWNER TO bob;

-- 约束
ALTER TABLE clients ADD CONSTRAINT uk_email UNIQUE (email);
ALTER TABLE clients DROP CONSTRAINT uk_email;
```

> **运维提醒**:`ALTER TABLE` 会持有 `ACCESS EXCLUSIVE` 锁,会阻塞读写;`ALTER COLUMN TYPE` 多数情况会**重写整张表**。PG 11+ 中 `ADD COLUMN ... DEFAULT <常量>` 不再重写表。在线变更策略见第 4、16 章。

### 2.5.4 删除与清空

```sql
DROP TABLE IF EXISTS clients;
DROP TABLE a, b, c;
DROP TABLE a CASCADE;            -- 同时删除依赖它的视图/外键约束

TRUNCATE TABLE customers;                         -- 快速清空,不触发行级 DELETE 触发器
TRUNCATE TABLE customers RESTART IDENTITY CASCADE; -- 重置自增序列并级联清空外键引用表
```

`DELETE` 与 `TRUNCATE` 的区别:`DELETE` 逐行删除、可带 `WHERE`、会产生死元组;`TRUNCATE` 直接换新文件,瞬时完成,但要 `ACCESS EXCLUSIVE` 锁。**两者在 PG 里都可以回滚。**

### 2.5.5 DDL 也可以回滚

这是 PG 的招牌特性:

```sql
BEGIN;
CREATE TABLE t1 (id int);
ALTER TABLE t1 ADD COLUMN name text;
DROP TABLE customers;
ROLLBACK;                 -- 以上所有 DDL 全部撤销
\dt                       -- t1 不存在,customers 还在
```

这让"发布脚本"可以放进一个事务里,出错整体回滚——非常适合做数据库迁移。

---

## 2.6 DDL:序列(SEQUENCE)

序列是生成递增数字的数据库对象,是 `serial` 与 `IDENTITY` 的底层。

### 2.6.1 手工创建与使用

```sql
CREATE SEQUENCE order_no_seq
    START WITH 1000
    INCREMENT BY 1
    MINVALUE 1000
    MAXVALUE 9999999
    CACHE 20
    NO CYCLE;

SELECT nextval('order_no_seq');   -- 1000
SELECT nextval('order_no_seq');   -- 1001
SELECT currval('order_no_seq');   -- 当前会话最近一次 nextval 的值
SELECT lastval();                 -- 当前会话最近一次任意序列的值
SELECT setval('order_no_seq', 5000);   -- 手工设置,下次 nextval 得到 5001

CREATE TABLE orders (
    order_no bigint PRIMARY KEY DEFAULT nextval('order_no_seq'),
    amount   numeric(10,2)
);
```

### 2.6.2 `serial` 与 `IDENTITY`

```sql
-- 老写法:serial(语法糖,自动建序列并设置 DEFAULT)
CREATE TABLE t_old (id serial PRIMARY KEY);

-- 推荐写法(SQL 标准,PG10+):IDENTITY
CREATE TABLE t_new (
    id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY
);
-- ALWAYS:不允许手工指定 id(除非 OVERRIDING SYSTEM VALUE)
-- BY DEFAULT:允许手工指定
```

| | `serial` | `IDENTITY` |
|---|---|---|
| 标准性 | PG 专有 | SQL 标准 |
| 序列归属 | 依赖关系较松散 | 与列紧密绑定,权限/复制更省心 |
| 防止误插入 | 不能 | `ALWAYS` 可以 |

### 2.6.3 序列的重要特性(面试高频)

```sql
CREATE TABLE seq_demo (id bigint GENERATED ALWAYS AS IDENTITY, v text);

BEGIN;
INSERT INTO seq_demo(v) VALUES ('a');   -- id = 1
ROLLBACK;
INSERT INTO seq_demo(v) VALUES ('b');   -- id = 2,而不是 1!
```

- 序列**不受事务回滚影响**,所以**会出现空洞**,不要把它当"连续编号"。
- 序列取值是并发安全的,多会话不会拿到相同值。
- `CACHE` 大于 1 时,崩溃/断连可能让一批号码被跳过。
- 若要"严格连续的发票号",需要用单独的计数表 + 行锁,代价是串行化。

### 2.6.4 修改与重置

```sql
ALTER SEQUENCE order_no_seq RESTART WITH 1;
ALTER SEQUENCE order_no_seq INCREMENT BY 2;
ALTER SEQUENCE order_no_seq OWNED BY orders.order_no;   -- 绑定到列,列删则序列删
DROP SEQUENCE IF EXISTS order_no_seq;

-- 导入数据后,把序列对齐到表中最大值
SELECT setval(pg_get_serial_sequence('t_old', 'id'),
              COALESCE((SELECT max(id) FROM t_old), 1));
```

---

## 2.7 表空间(TABLESPACE)

表空间让你把表/索引的物理文件放到**文件系统的其他目录**(例如不同的磁盘:SSD 放热点索引,HDD 放归档)。

```bash
# 先在操作系统创建目录,并交给 postgres 用户
sudo mkdir -p /ssd/pgdata && sudo chown postgres:postgres /ssd/pgdata
```

```sql
CREATE TABLESPACE fast_ssd LOCATION '/ssd/pgdata';

CREATE TABLE hot_orders (id int) TABLESPACE fast_ssd;
CREATE INDEX idx_hot ON hot_orders(id) TABLESPACE fast_ssd;

ALTER TABLE hot_orders SET TABLESPACE pg_default;     -- 迁移(会锁表并复制文件)
ALTER DATABASE shop SET TABLESPACE fast_ssd;          -- 需无其他连接

SET default_tablespace = fast_ssd;                    -- 会话默认

\db+                                                  -- 查看表空间
```

内置表空间:
- `pg_default`:放在 `PGDATA/base`
- `pg_global`:放共享系统表

> 现实建议:使用云盘/LVM/RAID 的场景下,表空间的必要性降低;但它在**分区表冷热分层、容量扩展**时仍有价值。表空间目录**是数据库集群的一部分**,备份/迁移时不能漏。

---

## 2.8 标识符、大小写与命名规则

### 2.8.1 标识符规则

- 标识符:库、Schema、表、列、索引、函数等对象的名称。
- 默认长度上限 **63 字节**(超出会被**静默截断**)。
- 可由字母(含非英文字母)、数字、下划线、`$` 组成,且**不能以数字开头**。

### 2.8.2 大小写规则(必考)

**不加引号的标识符会被自动折叠为小写;加双引号则严格保留大小写。**

```sql
CREATE TABLE UserInfo (ID int, UserName text);
-- 实际创建的是 userinfo(id, username)

SELECT * FROM UserInfo;        -- OK
SELECT * FROM userinfo;        -- OK,等价
SELECT * FROM "UserInfo";      -- ERROR: relation "UserInfo" does not exist

CREATE TABLE "UserInfo2" ("ID" int);
SELECT * FROM UserInfo2;       -- ERROR!(被折成小写 userinfo2)
SELECT * FROM "UserInfo2";     -- OK
SELECT "ID" FROM "UserInfo2";  -- OK
```

**强烈建议的命名规范**:
- 全部使用**小写字母 + 下划线**(snake_case),**不要**用双引号。
- 不使用关键字作为名称(如 `user`、`order`、`group`、`table`)。想用就改成 `users`、`orders`。
- 表名用复数或统一风格,主键 `id`,外键 `<表单数>_id`,时间列 `created_at`、`updated_at`。
- 索引、约束名统一前缀:`pk_`、`uk_`、`fk_`、`idx_`、`ck_`。

### 2.8.3 区分字符串与标识符

```sql
SELECT 'name';      -- 字符串常量,结果就是 name 这个字
SELECT "name";      -- 标识符,取名为 name 的那一列
SELECT 'It''s';     -- 字符串里含单引号,写两个

-- 美元引用:写函数体/含大量引号的字符串时非常方便
SELECT $$It's "fine"$$;
SELECT $tag$ 任意内容 $tag$;
```

---

## 2.9 NULL 与三值逻辑

`NULL` 表示"**未知/不存在**",不是 0,不是空字符串 `''`,也不等于任何值(包括它自己)。

### 2.9.1 比较与判断

```sql
SELECT NULL = NULL;          -- NULL(未知,不是 true)
SELECT NULL <> 1;            -- NULL
SELECT NULL IS NULL;         -- true
SELECT 1 IS NOT NULL;        -- true
SELECT '' IS NULL;           -- false(空串不是 NULL)
SELECT 0 IS NULL;            -- false

-- NULL 安全的比较
SELECT NULL IS NOT DISTINCT FROM NULL;   -- true
SELECT 1    IS DISTINCT FROM NULL;       -- true
```

### 2.9.2 三值逻辑真值表

SQL 的布尔有三种值:`TRUE`、`FALSE`、`UNKNOWN`(即 `NULL`)。

| A | B | A AND B | A OR B |
|---|---|---|---|
| T | NULL | NULL | **T** |
| F | NULL | **F** | NULL |
| NULL | NULL | NULL | NULL |

`NOT NULL` 结果仍为 `NULL`。

`WHERE` 只保留条件结果为 **TRUE** 的行,`FALSE` 和 `NULL` 都会被过滤掉。

```sql
CREATE TABLE emp (id int, name text, bonus int);
INSERT INTO emp VALUES (1,'A',100), (2,'B',0), (3,'C',NULL);

SELECT * FROM emp WHERE bonus = 0;        -- 只有 B
SELECT * FROM emp WHERE bonus <> 0;       -- 只有 A(C 被丢掉了!)
SELECT * FROM emp WHERE bonus IS NULL;    -- C
-- 想要"非 0 或为空":
SELECT * FROM emp WHERE bonus IS DISTINCT FROM 0;   -- A 和 C
```

### 2.9.3 常见陷阱

**陷阱 1:`NOT IN` 遇到 `NULL`**

```sql
SELECT 1 WHERE 1 NOT IN (2, 3);        -- 有结果
SELECT 1 WHERE 1 NOT IN (2, 3, NULL);  -- 无结果!(1<>NULL 为 NULL,整体为 NULL)
```
→ 子查询可能返回 `NULL` 时,请改用 `NOT EXISTS`。

**陷阱 2:聚合函数忽略 NULL**

```sql
SELECT count(*), count(bonus), sum(bonus), avg(bonus) FROM emp;
-- 3, 2, 100, 50   (avg 是 (100+0)/2,不是 /3)
```

**陷阱 3:运算遇到 NULL 结果就是 NULL**

```sql
SELECT 100 + NULL;                  -- NULL
SELECT 'a' || NULL;                 -- NULL
SELECT concat('a', NULL, 'b');      -- 'ab'(concat 函数会忽略 NULL)
```

**陷阱 4:排序中的 NULL**

```sql
SELECT * FROM emp ORDER BY bonus;                 -- 升序时 NULL 排最后
SELECT * FROM emp ORDER BY bonus DESC;            -- 降序时 NULL 排最前
SELECT * FROM emp ORDER BY bonus DESC NULLS LAST; -- 手动指定
```

**陷阱 5:唯一约束允许多个 NULL**

```sql
CREATE TABLE u (email text UNIQUE);
INSERT INTO u VALUES (NULL), (NULL);   -- 都成功!
-- PG15+ 如需"NULL 也视为相同":
-- email text UNIQUE NULLS NOT DISTINCT
```

### 2.9.4 NULL 处理函数

```sql
SELECT COALESCE(bonus, 0) FROM emp;        -- 取第一个非 NULL
SELECT NULLIF(bonus, 0) FROM emp;          -- 两值相等则返回 NULL(常用于防除零)
SELECT 10 / NULLIF(0, 0);                  -- NULL,而不是报 division by zero
```

---

## 2.10 注释:`COMMENT ON`

给对象写"元数据注释",保存在系统目录中,`psql` 的 `\d+` 能直接看到。

```sql
COMMENT ON DATABASE shop IS '电商主库';
COMMENT ON SCHEMA sales IS '销售域';
COMMENT ON TABLE customers IS '客户主表';
COMMENT ON COLUMN customers.status IS '状态:active/vip/blocked';
COMMENT ON COLUMN customers.status IS NULL;     -- 删除注释

\d+ customers

-- 用 SQL 查看列注释
SELECT a.attname, col_description(a.attrelid, a.attnum) AS comment
FROM pg_attribute a
WHERE a.attrelid = 'customers'::regclass AND a.attnum > 0 AND NOT a.attisdropped;
```

> 好习惯:**建表时就写注释**。团队协作、生成文档、数据字典都靠它。

---

## 2.11 `COPY` 与 `\copy` 入门

用于**批量导入导出**,比逐条 `INSERT` 快一到两个数量级。

| | `COPY` | `\copy` |
|---|---|---|
| 执行位置 | **服务器**端 | **客户端**(`psql`)端 |
| 文件路径 | 服务器上的文件 | 本机文件 |
| 权限 | 需超级用户或 `pg_read_server_files` / `pg_write_server_files` 角色 | 普通权限即可,读本机文件 |

```sql
-- 准备
CREATE TABLE stu (id int, name text, score numeric);

-- 客户端导入 CSV(最常用,推荐)
\copy stu FROM './stu.csv' WITH (FORMAT csv, HEADER true, ENCODING 'UTF8')

-- 客户端导出
\copy (SELECT * FROM stu WHERE score >= 60) TO './pass.csv' WITH (FORMAT csv, HEADER true)

-- 服务器端导入(文件在数据库服务器上)
COPY stu FROM '/tmp/stu.csv' WITH (FORMAT csv, HEADER true);

-- 通过标准输入输出,适合脚本管道
-- cat stu.csv | psql -c "COPY stu FROM STDIN WITH (FORMAT csv, HEADER true)"
```

`stu.csv` 示例:

```csv
id,name,score
1,张三,90
2,李四,
3,"王,五",55.5
```

> 注意 `2,李四,` 末尾为空,默认导入为 `NULL`;可用 `NULL ''` 选项自定义 NULL 的表示。更多导入导出细节见第 9 章。

---

## 2.12 本章综合示例:搭建一个小型图书馆库

```sql
-- 1. 创建数据库(在 postgres 库中执行)
CREATE DATABASE library ENCODING 'UTF8' TEMPLATE template0
    LC_COLLATE 'C.UTF-8' LC_CTYPE 'C.UTF-8';
\c library

-- 2. 创建 schema 并设置搜索路径
CREATE SCHEMA lib;
SET search_path TO lib, public;

-- 3. 序列:借阅单号
CREATE SEQUENCE borrow_no_seq START 100000;

-- 4. 表
CREATE TABLE books (
    id         bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    isbn       varchar(20) NOT NULL UNIQUE,
    title      text        NOT NULL,
    author     text,
    price      numeric(8,2) CHECK (price >= 0),
    created_at timestamptz  NOT NULL DEFAULT now()
);

CREATE TABLE borrows (
    borrow_no   bigint PRIMARY KEY DEFAULT nextval('borrow_no_seq'),
    book_id     bigint NOT NULL REFERENCES books(id),
    reader      text   NOT NULL,
    borrowed_at date   NOT NULL DEFAULT current_date,
    returned_at date
);

-- 5. 注释
COMMENT ON TABLE books IS '图书表';
COMMENT ON COLUMN borrows.returned_at IS '归还日期,NULL 表示尚未归还';

-- 6. 数据
INSERT INTO books (isbn, title, author, price) VALUES
 ('978-7-111', 'PostgreSQL 实战', '张三', 89.00),
 ('978-7-222', '数据库原理',     NULL,   59.50);
INSERT INTO borrows (book_id, reader) VALUES (1, '小明'), (2, '小红');
UPDATE borrows SET returned_at = current_date WHERE reader = '小明';

-- 7. 查询:尚未归还的借阅(体会 IS NULL)
SELECT b.title, r.reader, r.borrowed_at
FROM borrows r JOIN books b ON b.id = r.book_id
WHERE r.returned_at IS NULL;

-- 8. 体会 NULL:作者缺失时显示默认值
SELECT title, COALESCE(author, '佚名') AS author FROM books;

-- 9. 事务性 DDL:试验性变更后回滚
BEGIN;
ALTER TABLE books ADD COLUMN publisher text;
\d books
ROLLBACK;
\d books      -- publisher 列消失
```

---

## 2.13 本章练习

1. 创建数据库 `school`(UTF8),并让它以后每次连接默认时区为 `Asia/Shanghai`。
2. 创建 schema `edu`,在其中创建表 `students(id identity 主键, name 非空, birthday date, score int 检查 0~100)`。
3. 创建序列 `edu.stu_no_seq` 从 20260001 开始,并新增列 `stu_no bigint DEFAULT nextval(...)`。
4. 做一个实验:在事务里 `INSERT` 一行后 `ROLLBACK`,再插入一行,观察 identity 值是否连续,并解释原因。
5. 创建表 `"Student"` 与 `student`,观察它们是否能共存,并说明大小写折叠规则。
6. 构造表 `t(a int)`,插入 `1, 2, NULL`,分别写出下列语句结果并上机验证:
   - `SELECT * FROM t WHERE a <> 1;`
   - `SELECT * FROM t WHERE a NOT IN (SELECT a FROM t WHERE a = 2 OR a IS NULL);`
   - `SELECT count(*), count(a), avg(a) FROM t;`
7. 使用 `COALESCE` 与 `NULLIF` 写一个"安全除法"表达式:分母为 0 或 NULL 时返回 0。
8. 导出 `students` 表为 CSV(含表头),清空后再用 `\copy` 导回,核对行数。
9. 为表和每一列写 `COMMENT ON`,并用 `\d+` 验证。
10. 在一个事务里执行 `CREATE TABLE`、`ALTER TABLE`、`DROP TABLE`,然后回滚,验证"事务性 DDL"。

**参考答案提示**

```sql
-- 1
CREATE DATABASE school ENCODING 'UTF8';
ALTER DATABASE school SET timezone = 'Asia/Shanghai';

-- 7:分母为 0 或 NULL 都返回 0
SELECT COALESCE(10 / NULLIF(den, 0), 0) FROM (VALUES (0), (NULL), (5)) v(den);
```

第 6 题结果提示:第 1 条返回 `2`(NULL 被过滤);第 2 条返回**空集**(`NOT IN` 子查询含 NULL);第 3 条为 `3, 2, 1.5`。

---

## 2.14 本章小结

- SQL 分为 DDL / DML / DQL / DCL / TCL;字符串用单引号,标识符用双引号。
- `CREATE DATABASE` 本质是**克隆模板库**;`template1` 是默认模板,`template0` 是干净模板;编码与 locale 建库后不可改。
- Schema 是命名空间,配合 `search_path` 使用;`DROP ... CASCADE` 要格外谨慎。
- 表的创建方式:直接建、`LIKE`、`CREATE TABLE AS`、临时表;`ALTER TABLE` 会加重锁,部分操作会重写表。
- **PG 的 DDL 是事务性的**,可以回滚。
- 序列不随事务回滚、**会有空洞**;新项目优先用 `GENERATED ... AS IDENTITY`。
- 表空间用于把对象放到其他存储目录,是集群级对象。
- 标识符:不加引号 → 折叠为小写;推荐 snake_case,不用双引号、不用关键字。
- `NULL` 是"未知":`= NULL` 永远不对,要用 `IS NULL` / `IS DISTINCT FROM`;警惕 `NOT IN` 与聚合中的 NULL 行为。
- `COMMENT ON` 写好数据字典;`\copy` 是客户端侧批量导入导出的首选。

**下一章预告**:数据类型——数值、文本、日期时间、`jsonb`、数组、范围等 PG 特色类型,以及类型转换规则。
