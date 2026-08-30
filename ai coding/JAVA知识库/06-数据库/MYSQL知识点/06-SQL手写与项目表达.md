# 阶段六：SQL 手写与项目表达

## 目录

- [1. 阶段目标](#1-阶段目标)
- [2. 练习表结构](#2-练习表结构)
- [3. 第二高工资](#3-第二高工资)
  - [3.1 不考虑并列，只取排序后的第二行](#31-不考虑并列只取排序后的第二行)
  - [3.2 第二高的不同工资](#32-第二高的不同工资)
- [4. 每个部门工资最高的员工](#4-每个部门工资最高的员工)
- [5. 查询重复手机号](#5-查询重复手机号)
- [6. 删除重复数据，只保留一条](#6-删除重复数据只保留一条)
- [7. 每个用户最近一笔订单](#7-每个用户最近一笔订单)
- [8. 有订单但没有支付记录的用户](#8-有订单但没有支付记录的用户)
- [9. 每日新增用户和累计用户](#9-每日新增用户和累计用户)
- [10. 每个分类销量前三的商品](#10-每个分类销量前三的商品)
- [11. 连续登录三天的用户](#11-连续登录三天的用户)
- [12. 条件聚合](#12-条件聚合)
- [13. 同比和环比基础](#13-同比和环比基础)
- [14. SQL 手写题答题流程](#14-sql-手写题答题流程)
- [15. 项目案例准备模板](#15-项目案例准备模板)
  - [15.1 示例骨架](#151-示例骨架)
- [16. 案例高频追问](#16-案例高频追问)
- [17. 一分钟与三分钟回答](#17-一分钟与三分钟回答)
  - [一分钟版](#一分钟版)
  - [三分钟版](#三分钟版)
- [18. 最终模拟面试](#18-最终模拟面试)
- [19. 阶段自测](#19-阶段自测)

## 1. 阶段目标

这一阶段把知识转化为面试输出：

```text
读懂题目
  -> 明确重复、NULL、并列和时间边界
  -> 写出正确 SQL
  -> 解释索引和复杂度
  -> 把真实项目经历组织成证据充分的案例
```

示例以 MySQL 8.x 为主，因此会使用窗口函数和 CTE。

## 2. 练习表结构

```sql
CREATE TABLE departments (
    id   BIGINT PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);

CREATE TABLE employees (
    id            BIGINT PRIMARY KEY,
    department_id BIGINT NOT NULL,
    name          VARCHAR(100) NOT NULL,
    salary        DECIMAL(12, 2) NOT NULL,
    KEY idx_department_salary (department_id, salary)
);

CREATE TABLE users (
    id          BIGINT PRIMARY KEY,
    phone       VARCHAR(30),
    create_time DATETIME NOT NULL,
    KEY idx_create_time (create_time)
);

CREATE TABLE orders (
    id          BIGINT PRIMARY KEY,
    user_id     BIGINT NOT NULL,
    status      VARCHAR(20) NOT NULL,
    amount      DECIMAL(12, 2) NOT NULL,
    create_time DATETIME NOT NULL,
    KEY idx_user_time (user_id, create_time, id),
    KEY idx_status_time (status, create_time)
);

CREATE TABLE login_log (
    id         BIGINT PRIMARY KEY,
    user_id    BIGINT NOT NULL,
    login_time DATETIME NOT NULL,
    KEY idx_user_login (user_id, login_time)
);
```

## 3. 第二高工资

### 3.1 不考虑并列，只取排序后的第二行

```sql
SELECT salary
FROM employees
ORDER BY salary DESC
LIMIT 1 OFFSET 1;
```

### 3.2 第二高的不同工资

```sql
SELECT MAX(salary) AS second_highest_salary
FROM employees
WHERE salary < (
    SELECT MAX(salary)
    FROM employees
);
```

或者使用窗口函数：

```sql
SELECT salary
FROM (
    SELECT
        salary,
        DENSE_RANK() OVER (ORDER BY salary DESC) AS salary_rank
    FROM employees
) t
WHERE salary_rank = 2
LIMIT 1;
```

面试时要先问：相同工资是否算并列？如果最高工资有两人，“第二行”和“第二高的不同工资”不是一回事。

## 4. 每个部门工资最高的员工

需要保留并列第一：

```sql
SELECT department_id, id, name, salary
FROM (
    SELECT
        e.*,
        DENSE_RANK() OVER (
            PARTITION BY department_id
            ORDER BY salary DESC
        ) AS salary_rank
    FROM employees e
) t
WHERE salary_rank = 1;
```

如果每个部门只允许返回一个人，即使工资相同，也要规定稳定规则：

```sql
SELECT department_id, id, name, salary
FROM (
    SELECT
        e.*,
        ROW_NUMBER() OVER (
            PARTITION BY department_id
            ORDER BY salary DESC, id ASC
        ) AS row_num
    FROM employees e
) t
WHERE row_num = 1;
```

区别：

- `ROW_NUMBER`：每行序号不同。
- `RANK`：并列后名次跳号，例如 1、1、3。
- `DENSE_RANK`：并列后名次连续，例如 1、1、2。

## 5. 查询重复手机号

```sql
SELECT phone, COUNT(*) AS duplicate_count
FROM users
WHERE phone IS NOT NULL
GROUP BY phone
HAVING COUNT(*) > 1;
```

如果空字符串也视为无效值：

```sql
WHERE phone IS NOT NULL
  AND phone <> ''
```

找到重复手机号只是第一步。业务上通常还要确认是否应增加唯一索引，以及历史重复数据如何合并。

## 6. 删除重复数据，只保留一条

假设手机号重复时保留最小 `id`：

```sql
DELETE u1
FROM users u1
JOIN users u2
  ON u1.phone = u2.phone
 AND u1.id > u2.id
WHERE u1.phone IS NOT NULL;
```

生产执行前必须：

1. 先把 `DELETE` 改成 `SELECT u1.*` 验证目标行。
2. 备份或建立可恢复方案。
3. 大表分批删除。
4. 处理引用这些用户 ID 的关联数据。
5. 清理后增加业务唯一约束，防止再次产生重复数据。

## 7. 每个用户最近一笔订单

```sql
SELECT id, user_id, status, amount, create_time
FROM (
    SELECT
        o.*,
        ROW_NUMBER() OVER (
            PARTITION BY user_id
            ORDER BY create_time DESC, id DESC
        ) AS row_num
    FROM orders o
) t
WHERE row_num = 1;
```

加入 `id DESC` 是为了在多个订单时间完全相同时得到稳定结果。

如果查询只针对一个用户：

```sql
SELECT id, user_id, status, amount, create_time
FROM orders
WHERE user_id = ?
ORDER BY create_time DESC, id DESC
LIMIT 1;
```

联合索引 `(user_id, create_time, id)` 可以服务过滤和排序。

## 8. 有订单但没有支付记录的用户

推荐使用 `EXISTS` 和 `NOT EXISTS` 表达存在性：

```sql
SELECT u.id
FROM users u
WHERE EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.user_id = u.id
)
AND NOT EXISTS (
    SELECT 1
    FROM orders o
    JOIN payments p ON p.order_id = o.id
    WHERE o.user_id = u.id
);
```

题目可能有两种语义，应先确认：

- 用户从未有任何支付记录。
- 用户至少存在一个尚未支付的订单。

第二种语义应该写成：

```sql
SELECT DISTINCT o.user_id
FROM orders o
WHERE NOT EXISTS (
    SELECT 1
    FROM payments p
    WHERE p.order_id = o.id
);
```

## 9. 每日新增用户和累计用户

```sql
WITH daily AS (
    SELECT
        DATE(create_time) AS register_date,
        COUNT(*) AS new_users
    FROM users
    GROUP BY DATE(create_time)
)
SELECT
    register_date,
    new_users,
    SUM(new_users) OVER (
        ORDER BY register_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS total_users
FROM daily
ORDER BY register_date;
```

如果查询时间范围很大且频率很高，不应只依赖在线全表聚合，可以建立日汇总表或使用离线分析系统。

## 10. 每个分类销量前三的商品

假设订单明细表包含 `category_id`、`product_id`、`quantity`：

```sql
WITH product_sales AS (
    SELECT
        category_id,
        product_id,
        SUM(quantity) AS sales
    FROM order_items
    GROUP BY category_id, product_id
), ranked AS (
    SELECT
        category_id,
        product_id,
        sales,
        DENSE_RANK() OVER (
            PARTITION BY category_id
            ORDER BY sales DESC
        ) AS sales_rank
    FROM product_sales
)
SELECT category_id, product_id, sales
FROM ranked
WHERE sales_rank <= 3;
```

使用 `DENSE_RANK` 时并列第三可能返回超过三个商品。如果题目要求每个分类严格三行，使用 `ROW_NUMBER` 并指定第二排序键。

## 11. 连续登录三天的用户

先对同一用户同一天的多次登录去重，再利用“日期减去行号后分组”识别连续区间：

```sql
WITH login_days AS (
    SELECT DISTINCT
        user_id,
        DATE(login_time) AS login_date
    FROM login_log
), numbered AS (
    SELECT
        user_id,
        login_date,
        ROW_NUMBER() OVER (
            PARTITION BY user_id
            ORDER BY login_date
        ) AS row_num
    FROM login_days
), grouped AS (
    SELECT
        user_id,
        login_date,
        DATE_SUB(login_date, INTERVAL row_num DAY) AS group_key
    FROM numbered
)
SELECT user_id
FROM grouped
GROUP BY user_id, group_key
HAVING COUNT(*) >= 3;
```

原理：连续日期和连续行号同步增加，两者相减得到相同分组键。

追问时要确认：

- 按哪个时区划分自然日？
- 连续三天是至少三天还是恰好三天？
- 一天多次登录是否只算一天？

## 12. 条件聚合

统计每天已支付和已取消订单：

```sql
SELECT
    DATE(create_time) AS order_date,
    SUM(CASE WHEN status = 'PAID' THEN 1 ELSE 0 END) AS paid_count,
    SUM(CASE WHEN status = 'CANCELLED' THEN 1 ELSE 0 END) AS cancelled_count,
    SUM(CASE WHEN status = 'PAID' THEN amount ELSE 0 END) AS paid_amount
FROM orders
GROUP BY DATE(create_time)
ORDER BY order_date;
```

`CASE WHEN` 是面试手写 SQL 的高频工具，可用于条件计数、行转列和分段统计。

## 13. 同比和环比基础

先按月汇总，再使用 `LAG` 取得上月值：

```sql
WITH monthly AS (
    SELECT
        DATE_FORMAT(create_time, '%Y-%m-01') AS month_start,
        SUM(amount) AS total_amount
    FROM orders
    WHERE status = 'PAID'
    GROUP BY DATE_FORMAT(create_time, '%Y-%m-01')
)
SELECT
    month_start,
    total_amount,
    LAG(total_amount) OVER (ORDER BY month_start) AS previous_amount,
    CASE
        WHEN LAG(total_amount) OVER (ORDER BY month_start) IS NULL THEN NULL
        WHEN LAG(total_amount) OVER (ORDER BY month_start) = 0 THEN NULL
        ELSE (
            total_amount - LAG(total_amount) OVER (ORDER BY month_start)
        ) / LAG(total_amount) OVER (ORDER BY month_start)
    END AS month_on_month_rate
FROM monthly
ORDER BY month_start;
```

真实报表还要补齐没有数据的月份，否则 `LAG` 得到的是“上一个有数据的月份”，未必是自然月的上个月。

## 14. SQL 手写题答题流程

拿到题目先确认：

```text
表之间是一对一、一对多还是多对多？
是否允许 NULL？
是否存在重复记录？
并列名次如何处理？
时间范围是否包含边界？
结果要求一行还是多行？
MySQL 版本是否支持窗口函数和 CTE？
```

写完后用极端数据自测：

- 空表。
- 只有一条记录。
- 多条并列记录。
- 字段为 `NULL`。
- 同一天多条记录。
- 月末、年末和跨时区时间。

最后再讨论索引。不要为了提前谈性能而先写出错误 SQL。

## 15. 项目案例准备模板

准备至少一个真实、可量化且能承受追问的案例：

```text
1. 业务背景：哪个接口或任务，为什么重要。
2. 现象：RT、超时率、CPU、扫描行数等发生了什么。
3. 数据规模：表行数、日增量、QPS、结果集大小。
4. 定位证据：慢日志、执行计划、监控、锁等待。
5. 根因：索引、SQL、数据分布、锁或架构中的具体问题。
6. 方案选择：为什么选这个方案，放弃了哪些方案。
7. 实施：如何测试、灰度、回滚和控制风险。
8. 结果：优化前后的可验证指标。
9. 复盘：如何通过规范或监控防止复发。
```

### 15.1 示例骨架

> 订单列表接口在订单表增长到约某个规模后，P99 从某值上升到某值。慢日志显示主要耗时来自按用户和时间分页的查询，执行计划中实际扫描行数远高于返回行数，并存在大量回表。结合查询模式，我调整了联合索引，使过滤和排序顺序匹配，同时将较深页面改为基于 `create_time, id` 的游标分页。上线前用脱敏生产数据压测并观察写入开销，灰度后 P99 和扫描行数下降到某值。随后增加了慢查询告警和分页深度限制。

其中所有“某值”都必须替换成自己真实经历中的数字。如果没有精确数字，应诚实说明当时记录的指标范围，不要编造。

## 16. 案例高频追问

准备回答：

1. 表结构和原索引是什么？
2. 原 SQL 与新 SQL 分别是什么？
3. 执行计划具体哪里发生变化？
4. 为什么不只建立一个单列索引？
5. 新索引增加了多少空间和写入成本？
6. 测试数据是否接近生产分布？
7. 如何避免在线加索引阻塞业务？
8. 如果优化无效，下一步查什么？
9. 为什么不用缓存、ES 或分库分表？
10. 方案发生故障时如何回滚？

## 17. 一分钟与三分钟回答

同一个问题准备两个版本：

### 一分钟版

```text
结论
  -> 两个核心机制
  -> 一个例子
```

适合面试官快速确认基础。

### 三分钟版

```text
结论
  -> 内部流程
  -> 示例
  -> 性能代价
  -> 适用边界
  -> 项目关联
```

面试官没有追问时不要无限展开；面试官追问时也不能只重复第一句话。

## 18. 最终模拟面试

建议计时 60 分钟：

```text
10 分钟：基础 SQL 与字段类型
15 分钟：索引、联合索引和 EXPLAIN
15 分钟：事务、MVCC、锁和日志
10 分钟：慢 SQL、库存与主从延迟场景
10 分钟：手写两道 SQL 并讲项目案例
```

每次模拟后只记录三类问题：

- 回答错误。
- 知道但表达混乱。
- 无法应对追问。

下一次模拟前优先修复这三类问题，不再无限扩充题库。

## 19. 阶段自测

1. `ROW_NUMBER`、`RANK`、`DENSE_RANK` 如何处理并列？
2. 第二高工资为什么需要先确认并列语义？
3. 连续登录题为什么要先按自然日去重？
4. `NOT EXISTS` 适合表达什么问题？
5. 删除重复记录之前必须做哪些安全检查？
6. 按非唯一时间排序分页，为什么还要加入主键？
7. 为什么月度环比需要补齐空月份？
8. SQL 写完后应该用哪些极端数据验证？
9. 一个可信的优化案例至少需要哪些量化证据？
10. 为什么项目案例不能只说“加了索引，性能提升很多”？

