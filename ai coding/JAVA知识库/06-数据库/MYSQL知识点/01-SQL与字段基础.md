# 阶段一：SQL 与字段基础

## 目录

- [1. 阶段目标](#1-阶段目标)
- [2. SQL 逻辑执行顺序](#2-sql-逻辑执行顺序)
- [3. WHERE 和 HAVING](#3-where-和-having)
  - [3.1 核心区别](#31-核心区别)
  - [3.2 性能与规范](#32-性能与规范)
  - [3.3 面试回答](#33-面试回答)
- [4. DELETE、TRUNCATE 和 DROP](#4-deletetruncate-和-drop)
  - [4.1 DELETE](#41-delete)
  - [4.2 TRUNCATE](#42-truncate)
  - [4.3 DROP](#43-drop)
  - [4.4 面试回答](#44-面试回答)
- [5. UNION 和 UNION ALL](#5-union-和-union-all)
- [6. INNER JOIN 和 LEFT JOIN](#6-inner-join-和-left-join)
  - [6.1 INNER JOIN](#61-inner-join)
  - [6.2 LEFT JOIN](#62-left-join)
  - [6.3 ON 和 WHERE 的经典陷阱](#63-on-和-where-的经典陷阱)
  - [6.4 查找没有订单的用户](#64-查找没有订单的用户)
- [7. COUNT(*)、COUNT(1) 和 COUNT(字段)](#7-countcount1-和-count字段)
  - [7.1 结果差异](#71-结果差异)
  - [7.2 COUNT(DISTINCT)](#72-countdistinct)
  - [7.3 InnoDB 为什么不能直接保存精确总行数？](#73-innodb-为什么不能直接保存精确总行数)
- [8. CHAR 和 VARCHAR](#8-char-和-varchar)
- [9. NULL 的常见陷阱](#9-null-的常见陷阱)
  - [9.1 判断 NULL](#91-判断-null)
  - [9.2 NULL 参与计算](#92-null-参与计算)
  - [9.3 聚合函数与 NULL](#93-聚合函数与-null)
  - [9.4 NOT IN 与 NULL](#94-not-in-与-null)
  - [9.5 唯一索引与 NULL](#95-唯一索引与-null)
  - [9.6 空值安全比较](#96-空值安全比较)
- [10. 主键、唯一索引和普通索引](#10-主键唯一索引和普通索引)
- [11. 为什么不建议 SELECT *](#11-为什么不建议-select)
- [12. INT(11) 的含义](#12-int11-的含义)
- [13. DATETIME 和 TIMESTAMP](#13-datetime-和-timestamp)
- [14. 高频组合题](#14-高频组合题)
  - [14.1 查询订单数超过 10 的用户](#141-查询订单数超过-10-的用户)
  - [14.2 查询没有支付记录的订单](#142-查询没有支付记录的订单)
  - [14.3 对比已支付订单数和支付时间非空数](#143-对比已支付订单数和支付时间非空数)
- [15. 阶段自测](#15-阶段自测)

## 1. 阶段目标

这一阶段要求做到：

```text
理解 SQL 的逻辑执行顺序
  -> 能正确使用过滤、连接和聚合
  -> 能处理 NULL 和重复数据
  -> 能解释常用字段类型与索引约束
```

面试回答应先给结论，再讲原因，最后补一个 SQL 示例或边界条件。

## 2. SQL 逻辑执行顺序

一个查询的书写顺序通常是：

```sql
SELECT ...
FROM ...
JOIN ... ON ...
WHERE ...
GROUP BY ...
HAVING ...
ORDER BY ...
LIMIT ...;
```

但逻辑执行顺序可以简化为：

```text
FROM / JOIN
  -> ON
  -> WHERE
  -> GROUP BY
  -> 聚合计算
  -> HAVING
  -> SELECT
  -> DISTINCT
  -> ORDER BY
  -> LIMIT
```

这个顺序可以解释很多面试问题，例如为什么 `WHERE` 不能直接使用聚合函数，以及为什么 `LEFT JOIN` 右表条件的位置会改变结果。

## 3. WHERE 和 HAVING

### 3.1 核心区别

- `WHERE` 在分组和聚合前过滤原始行。
- `HAVING` 在分组和聚合后过滤分组结果。

查询已支付订单总额超过 1000 元的用户：

```sql
SELECT user_id, SUM(amount) AS total_amount
FROM orders
WHERE status = 'PAID'
GROUP BY user_id
HAVING SUM(amount) > 1000;
```

其中：

- `WHERE status = 'PAID'` 先排除未支付订单。
- `HAVING SUM(amount) > 1000` 再过滤聚合后的用户分组。

### 3.2 性能与规范

普通字段条件尽量放在 `WHERE` 中，让数据库在分组前减少数据量，也更容易利用索引。聚合结果条件放在 `HAVING` 中。

不能简单说 `HAVING` 完全不能使用普通字段。MySQL 在某些查询中允许这样写，但语义、可移植性和优化机会通常都不如把行级条件放入 `WHERE`。

### 3.3 面试回答

> `WHERE` 用于分组和聚合前过滤原始行，`HAVING` 用于分组和聚合后过滤结果。普通字段条件优先放在 `WHERE`，聚合函数条件放在 `HAVING`，这样语义清晰，也能尽早减少参与分组的数据量。

## 4. DELETE、TRUNCATE 和 DROP

| 操作 | 作用 | 保留表结构 | 支持 `WHERE` | 事务与典型行为 |
|---|---|---:|---:|---|
| `DELETE` | 删除数据行 | 是 | 是 | DML；InnoDB 中提交前通常可以回滚 |
| `TRUNCATE` | 清空整张表 | 是 | 否 | DDL；隐式提交，通常重置自增值 |
| `DROP` | 删除表对象 | 否 | 否 | DDL；数据、结构和索引一起删除 |

### 4.1 DELETE

```sql
DELETE FROM orders
WHERE create_time < '2025-01-01';
```

特点：

- 可以删除部分数据，也可以不带条件删除全部数据。
- InnoDB 会记录事务相关日志，未提交前通常能够回滚。
- 通常不会自动重置 `AUTO_INCREMENT` 计数。
- 大批量删除会产生较多日志、锁和主从复制压力，生产中常分批执行。

### 4.2 TRUNCATE

```sql
TRUNCATE TABLE orders;
```

特点：

- 只能清空整张表，不能带 `WHERE`。
- 通常比无条件 `DELETE` 更适合快速清空表。
- 属于 DDL，会隐式提交，不能按普通 DML 事务方式回滚。
- 通常会重置自增值。
- 不会逐行触发 `DELETE` 触发器。
- 表被外键引用时可能无法直接执行。

### 4.3 DROP

```sql
DROP TABLE orders;
```

执行后表、数据和索引都不存在。要继续使用只能重新建表或恢复备份。

### 4.4 面试回答

> `DELETE` 可以按条件删除行，InnoDB 中事务提交前通常能够回滚；`TRUNCATE` 是清空整张表的 DDL，不能带条件，通常更快并重置自增值；`DROP` 会将表结构、索引和数据一起删除。

## 5. UNION 和 UNION ALL

两者都用于纵向合并多个查询结果。

```sql
SELECT user_id FROM orders_2025
UNION
SELECT user_id FROM orders_2026;
```

`UNION` 会对最终结果去重。

```sql
SELECT user_id FROM orders_2025
UNION ALL
SELECT user_id FROM orders_2026;
```

`UNION ALL` 保留重复行，避免额外去重工作，通常性能更好。没有去重需求时优先选择它。

使用要求：

- 每个查询返回的列数必须相同。
- 对应位置的数据类型应兼容。
- 最终列名通常由第一个查询决定。
- 对合并后的结果排序，应在最后写全局 `ORDER BY`。

```sql
SELECT id, create_time FROM orders_2025
UNION ALL
SELECT id, create_time FROM orders_2026
ORDER BY create_time DESC;
```

面试回答：

> `UNION` 会去重，`UNION ALL` 保留重复数据。去重需要额外处理，因此确定不需要去重时应优先使用 `UNION ALL`。

## 6. INNER JOIN 和 LEFT JOIN

### 6.1 INNER JOIN

只返回左右两张表匹配成功的数据：

```sql
SELECT u.id, u.name, o.id AS order_id
FROM users u
INNER JOIN orders o ON o.user_id = u.id;
```

没有订单的用户不会出现在结果中。

### 6.2 LEFT JOIN

保留左表全部数据：

```sql
SELECT u.id, u.name, o.id AS order_id
FROM users u
LEFT JOIN orders o ON o.user_id = u.id;
```

没有订单的用户仍然返回，右表字段为 `NULL`。

### 6.3 ON 和 WHERE 的经典陷阱

保留所有用户，同时只连接已支付订单：

```sql
SELECT u.id, o.id
FROM users u
LEFT JOIN orders o
    ON o.user_id = u.id
   AND o.status = 'PAID';
```

如果把右表条件写入 `WHERE`：

```sql
SELECT u.id, o.id
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE o.status = 'PAID';
```

未匹配行的 `o.status` 为 `NULL`，会被 `WHERE` 过滤，效果接近内连接。

可以记住：

```text
ON 决定怎样匹配
WHERE 对连接后的结果再次过滤
```

### 6.4 查找没有订单的用户

```sql
SELECT u.id, u.name
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE o.id IS NULL;
```

面试回答：

> `INNER JOIN` 只返回两边匹配的行；`LEFT JOIN` 会保留左表全部数据，右表没有匹配时字段为 `NULL`。要特别注意把右表过滤条件放入 `WHERE` 可能过滤掉未匹配行。

## 7. COUNT(*)、COUNT(1) 和 COUNT(字段)

### 7.1 结果差异

- `COUNT(*)`：统计结果集行数。
- `COUNT(1)`：`1` 是非空常量，也统计结果集行数。
- `COUNT(column)`：只统计该字段不为 `NULL` 的行。

如果 `payment_time` 的值是：

```text
2026-01-01
NULL
2026-01-03
```

那么：

```text
COUNT(*)            = 3
COUNT(1)            = 3
COUNT(payment_time) = 2
```

统计总行数时建议直接使用语义清晰的 `COUNT(*)`。在现代 MySQL/InnoDB 中，没有必要为了所谓性能把它机械替换为 `COUNT(1)`；实际代价与执行计划、索引和数据分布有关。

### 7.2 COUNT(DISTINCT)

```sql
SELECT COUNT(DISTINCT user_id)
FROM orders;
```

统计不同的非空用户数量。

### 7.3 InnoDB 为什么不能直接保存精确总行数？

InnoDB 支持事务和 MVCC。不同事务根据各自的可见性视图，可能看到不同数量的行，因此不能用一个全局固定数字准确回答所有事务的 `COUNT(*)`。

面试回答：

> `COUNT(*)` 和 `COUNT(1)` 都统计结果行数，通常不需要为了性能互换；`COUNT(字段)` 会忽略字段值为 `NULL` 的行。统计总行数时建议使用 `COUNT(*)`。

## 8. CHAR 和 VARCHAR

| 对比项 | `CHAR(n)` | `VARCHAR(n)` |
|---|---|---|
| 长度 | 定长 | 变长 |
| 空间 | 可能补齐并浪费空间 | 按实际内容使用空间，另有长度信息 |
| 适用数据 | 固定长度编码 | 姓名、地址、标题等变长内容 |
| `n` 的含义 | 最大字符数 | 最大字符数 |

`VARCHAR(100)` 的 `100` 表示最多 100 个字符，不是 100 字节。使用 `utf8mb4` 时，一个字符最多可能占 4 字节，所以还要考虑行大小和索引长度。

`CHAR` 不一定天然比 `VARCHAR` 快。定长访问简单，但可能浪费空间；`VARCHAR` 更紧凑，有时反而能减少数据页和 I/O。应根据数据语义和长度分布选择。

典型设计：

```sql
country_code CHAR(2),
user_name    VARCHAR(100)
```

面试回答：

> `CHAR` 是定长类型，适合固定长度数据；`VARCHAR` 是变长类型，适合长度变化较大的数据。`n` 表示字符数而不是字节数，实际空间还受字符集影响。

## 9. NULL 的常见陷阱

`NULL` 表示未知、缺失或不适用，不等于空字符串，也不等于数字零。

### 9.1 判断 NULL

错误：

```sql
WHERE payment_time = NULL;
```

正确：

```sql
WHERE payment_time IS NULL;
WHERE payment_time IS NOT NULL;
```

### 9.2 NULL 参与计算

```sql
SELECT 100 + NULL;
```

结果为 `NULL`。需要默认值时可以使用：

```sql
SELECT 100 + COALESCE(discount, 0)
FROM orders;
```

MySQL 也提供 `IFNULL(discount, 0)`。`COALESCE` 是标准 SQL，可以接收多个参数并返回第一个非空值。

### 9.3 聚合函数与 NULL

`COUNT(column)`、`SUM`、`AVG` 等聚合计算通常忽略 `NULL`，而 `COUNT(*)` 统计结果行数。

### 9.4 NOT IN 与 NULL

```sql
SELECT *
FROM users
WHERE id NOT IN (1, 2, NULL);
```

由于与 `NULL` 的普通比较结果是未知，查询可能得不到预期数据。应确保子查询不返回 `NULL`，或者使用 `NOT EXISTS`：

```sql
SELECT *
FROM users u
WHERE NOT EXISTS (
    SELECT 1
    FROM blacklist b
    WHERE b.user_id = u.id
);
```

### 9.5 唯一索引与 NULL

MySQL 唯一索引通常允许多个 `NULL`。如果业务要求字段必须有值且唯一，应同时使用 `NOT NULL` 和唯一索引。

```sql
email VARCHAR(100) NOT NULL,
UNIQUE KEY uk_email (email)
```

### 9.6 空值安全比较

MySQL 提供空值安全等号 `<=>`：

```sql
SELECT NULL <=> NULL; -- 1
```

面试回答：

> `NULL` 代表未知值，应该用 `IS NULL` 判断。它参与普通计算通常仍得到 `NULL`，`COUNT(字段)` 会忽略它，`NOT IN` 中出现 `NULL` 也可能导致意外结果。此外，MySQL 唯一索引通常允许多个 `NULL`。

## 10. 主键、唯一索引和普通索引

| 类型 | 唯一 | 允许 NULL | 数量 | 核心用途 |
|---|---:|---:|---:|---|
| 主键 | 是 | 否 | 每表一个 | 行身份；InnoDB 聚簇索引 |
| 唯一索引 | 是 | 取决于列定义，通常允许多个 NULL | 可多个 | 保证业务唯一性并加速访问 |
| 普通索引 | 否 | 可以 | 可多个 | 提高查询、排序和连接效率 |

InnoDB 通常以主键构建聚簇索引，叶子节点保存完整行数据。二级索引的叶子节点通常保存二级索引列和主键值。

```sql
CREATE TABLE users (
    id    BIGINT PRIMARY KEY,
    email VARCHAR(100) NOT NULL,
    name  VARCHAR(100) NOT NULL,
    UNIQUE KEY uk_email (email),
    KEY idx_name (name)
);
```

联合唯一索引保证的是组合唯一：

```sql
UNIQUE KEY uk_tenant_order (tenant_id, order_no)
```

它不要求 `tenant_id` 和 `order_no` 各自唯一。

唯一索引在等值查询找到记录后可以确认没有第二条匹配记录，但选择唯一索引的首要原因应是业务约束，而不是夸大这一点性能差异。

面试回答：

> 主键唯一且非空，一张表只能有一个，InnoDB 通常将其作为聚簇索引；唯一索引保证业务键唯一，一张表可以有多个；普通索引不保证唯一，主要用于提高访问效率。

## 11. 为什么不建议 SELECT *

```sql
SELECT *
FROM orders
WHERE id = 100;
```

主要问题：

1. 读取不需要的列，增加数据页访问、网络传输和应用内存使用。
2. 可能破坏覆盖索引，导致额外回表。
3. 表新增大字段后，查询开销会在没有修改 SQL 的情况下上升。
4. 多表查询容易出现 `id`、`create_time` 等重名字段。
5. 可能把不应返回的敏感字段带到应用层。
6. 依赖列位置的代码容易受到表结构变化影响。

如果索引为：

```sql
KEY idx_user_status (user_id, status)
```

下面的查询可能由索引直接覆盖：

```sql
SELECT user_id, status
FROM orders
WHERE user_id = 10;
```

改成 `SELECT *` 后，其他字段不在索引中，通常需要回表。

这不是绝对禁令。临时排查，或者确实需要所有字段时可以合理使用。

面试回答：

> `SELECT *` 可能读取无用字段，增加 I/O、网络和内存开销，也可能破坏覆盖索引并导致回表。显式列出字段还能降低表结构变化、重名和敏感字段暴露的风险。

## 12. INT(11) 的含义

结论：`INT(11)` 中的 `11` 不限制数值范围，也不是 11 字节。

普通 `INT` 始终占 4 字节。有符号范围是：

```text
-2147483648 ～ 2147483647
```

`INT UNSIGNED` 范围是：

```text
0 ～ 4294967295
```

括号中的数字历史上表示显示宽度，曾主要配合 `ZEROFILL` 使用：

```sql
code INT(5) ZEROFILL
```

数值 `123` 可能显示为 `00123`，但实际仍是数值 123。现代 MySQL 中不应依赖整数显示宽度或 `ZEROFILL`。如果业务值需要保留前导零，应使用字符串类型。

面试回答：

> `INT(11)` 的 `11` 不是数值范围和存储长度。`INT` 占 4 字节，取值范围由是否使用 `UNSIGNED` 决定；`11` 历史上是显示宽度，现代设计中不应依赖它。

## 13. DATETIME 和 TIMESTAMP

| 对比项 | `DATETIME` | `TIMESTAMP` |
|---|---|---|
| 主要语义 | 日历日期时间 | 一个时间点 |
| 时区转换 | 通常不自动转换 | 按会话时区写入和读取转换 |
| 范围 | 约 1000 到 9999 年 | 传统范围较小，需注意 2038 问题和版本能力 |
| 典型场景 | 预约、合同生效、本地业务时间 | 创建、更新、日志、跨时区事件时间 |

如果东八区会话向 `TIMESTAMP` 写入：

```text
2026-08-30 12:00:00
```

UTC 会话读取时可能显示：

```text
2026-08-30 04:00:00
```

`DATETIME` 通常仍显示原来的日期时间。

现代 MySQL 中，两者都可以配置默认值和自动更新时间：

```sql
created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
updated_at DATETIME NOT NULL
    DEFAULT CURRENT_TIMESTAMP
    ON UPDATE CURRENT_TIMESTAMP
```

因此“只有 `TIMESTAMP` 才能自动更新”是过时结论。

选择建议：

- 表达全球统一事件时间：可以使用 `TIMESTAMP`，并统一数据库和应用时区规范。
- 表达与时区无关的日历时间：使用 `DATETIME`。
- 需要覆盖很远的未来日期：通常选择 `DATETIME`。
- 使用 UTC `DATETIME` 也可以，但必须由整个系统严格遵守统一约定。

面试回答：

> `DATETIME` 保存日期时间本身，通常不随会话时区变化且范围更大；`TIMESTAMP` 更接近时间点，会按会话时区进行转换。具体选择取决于业务时间语义，而不只是字段占用空间。

## 14. 高频组合题

### 14.1 查询订单数超过 10 的用户

```sql
SELECT user_id, COUNT(*) AS order_count
FROM orders
GROUP BY user_id
HAVING COUNT(*) > 10;
```

### 14.2 查询没有支付记录的订单

```sql
SELECT o.id, o.user_id
FROM orders o
WHERE NOT EXISTS (
    SELECT 1
    FROM payments p
    WHERE p.order_id = o.id
);
```

### 14.3 对比已支付订单数和支付时间非空数

```sql
SELECT
    COUNT(*) AS paid_order_count,
    COUNT(payment_time) AS payment_time_count
FROM orders
WHERE status = 'PAID';
```

两者不同可能说明已支付订单存在 `payment_time IS NULL` 的数据异常。

## 15. 阶段自测

1. `HAVING` 为什么通常不能代替 `WHERE`？
2. `TRUNCATE` 与无条件 `DELETE` 的事务行为有什么不同？
3. 为什么没有去重需求时使用 `UNION ALL`？
4. `LEFT JOIN` 右表条件放入 `WHERE` 为什么可能改变结果？
5. `COUNT(*)` 与 `COUNT(column)` 如何处理 `NULL`？
6. `VARCHAR(100)` 表示 100 字节还是 100 个字符？
7. 为什么 `NOT IN` 遇到 `NULL` 可能返回意外结果？
8. 为什么唯一索引中可能存在多个 `NULL`？
9. `SELECT *` 为什么可能导致回表？
10. `INT(11)` 的 `11` 是什么？
11. `TIMESTAMP` 为什么受会话时区影响？

