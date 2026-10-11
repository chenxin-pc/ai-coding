# 场景题复习：大数据量加字段——MySQL 5.7 与 Elasticsearch

> 整理日期：2026-10-11。
>
> 复习主线：是否重写历史数据 → 是否影响在线业务 → 选择执行方式 → 控制负载与增量一致性 → 校验和发布。
>
> MySQL 部分以原版 MySQL 5.7、InnoDB 普通字段为前提；ES 示例使用不带 mapping type 的 API。实际执行前应核对精确版本、表或索引结构、拓扑与业务要求，下面的示例不是未经验证即可上线的脚本。

## 目录

- [1. 先记住核心区别](#1-先记住核心区别)
- [2. MySQL 5.7 大表加字段](#2-mysql-57-大表加字段)
- [3. gh-ost 是什么，如何选型](#3-gh-ost-是什么如何选型)
- [4. 大厂公开实践与可借鉴的流程](#4-大厂公开实践与可借鉴的流程)
- [5. ES 一亿条数据只新增字段](#5-es-一亿条数据只新增字段)
- [6. ES 历史数据必须补齐](#6-es-历史数据必须补齐)
- [7. ES 新索引迁移与别名切换](#7-es-新索引迁移与别名切换)
- [8. 发布顺序与监控](#8-发布顺序与监控)
- [9. 高频误区与面试回答](#9-高频误区与面试回答)
- [10. 关联笔记](#10-关联笔记)

## 1. 先记住核心区别

**MySQL 5.7 新增普通列需要重建表；ES 只添加字段 mapping 不会重写历史文档，给历史文档补值才产生大规模读写。**

| 场景 | 主要成本 | 优先考虑 |
| --- | --- | --- |
| MySQL 5.7 新增普通列 | 全表重建、锁等待、I/O、临时空间和复制延迟 | 评估原生 Online DDL；需要控制复制节奏时评估在线迁移工具 |
| ES 只让新文档携带字段 | 更新 mapping，后续文档正常索引 | 显式添加 mapping，修改写入程序 |
| ES 历史文档允许缺少字段 | 查询和展示的兼容处理 | 不回填，明确定义缺失值语义 |
| ES 历史文档必须有值 | 被更新文档重新索引、副本写入、段合并 | 按必要范围分批回填 |
| ES 更改已有字段类型或重做索引结构 | 新旧索引共存、全量迁移和增量追平 | 创建新索引、校验后切换别名 |

“一亿条”只能说明规模，不能单独决定方案。还要确认数据和索引体积、单条数据大小、写入量、资源余量、允许的延迟，以及是否真的要修改全部历史数据。

## 2. MySQL 5.7 大表加字段

### 2.1 原生 Online DDL 能做什么

```sql
-- 使用专用连接，与 ALTER 在同一个会话中执行。
-- 5 秒仅为示例，按业务容忍度调整。
SET SESSION lock_wait_timeout = 5;

ALTER TABLE order_info
    ADD COLUMN external_ref varchar(64) NULL COMMENT '外部关联编号',
    ALGORITHM=INPLACE,
    LOCK=NONE;
```

- `ALGORITHM=INPLACE`：对于新增普通列仍要重建表，并不是只改元数据。
- `LOCK=NONE`：要求支持并发读写，不代表整个操作没有锁。
- 显式指定算法和锁选项，不支持时应报错，避免自动采用限制更强的执行方式。
- `lock_wait_timeout` 控制元数据锁等待，每次获取锁分别计时；它不是整条 DDL 的总执行时间限制，也不同于 `innodb_lock_wait_timeout`。
- 字段放在末尾、允许 `NULL`、不设置默认值，都不能让 5.7 新增普通列免于重建。新增自增列、生成列等要单独判断。

MySQL 5.7 没有 `ALGORITHM=INSTANT`。MySQL 8.0.12 开始支持符合条件的瞬间加列，8.0.29 扩大了列位置等支持范围，但这些能力不能套用到 5.7。[5.7 Online DDL 操作](https://docs.oracle.com/cd/E17952_01/mysql-5.7-en/innodb-online-ddl-operations.html)、[8.0 Online DDL 操作](https://dev.mysql.com/doc/refman/8.0/en/innodb-online-ddl-operations.html)、[锁等待参数](https://docs.oracle.com/cd/E17952_01/mysql-5.7-en/server-system-variables.html#sysvar_lock_wait_timeout)

### 2.2 为什么“在线”也可能影响业务

| 风险 | 机制与影响 | 处理方向 |
| --- | --- | --- |
| 元数据锁 MDL | 长事务持有锁，DDL 排队，后续请求也可能被阻塞，进而耗尽连接池 | 检查长事务与锁等待；限制会话等待时间，避免高频自动重试 |
| I/O、CPU 与磁盘空间 | 重建数据和索引占用资源；临时文件可能耗尽文件系统 | 分别评估数据目录、临时目录和增长余量，不只计算新增字段大小 |
| 并发修改过多 | 在线变更日志可能超过 `innodb_online_alter_log_max_size` 导致失败 | 评估写入速率和执行时长；调大上限也可能延长收尾阶段锁定时间 |
| 主从复制延迟 | 从库还要执行 DDL，后续变更可能积压 | 监控所有关键从库，确认完成并追平再启用新功能 |
| 取消或失败成本 | 已执行的大量工作需要回滚或清理 | 不把“业务变慢就杀 DDL”当成低成本退路 |

5.7 原生 Online DDL 没有内置的暂停或按负载限制 I/O、CPU 的机制。磁盘预算应覆盖重建中间表、排序临时文件和日志，不能用一个固定倍数保证所有环境安全。[5.7 Online DDL 限制](https://docs.oracle.com/cd/E17952_01/mysql-5.7-en/innodb-online-ddl-limitations.html)、[空间需求](https://docs.oracle.com/cd/E17952_01/mysql-5.7-en/innodb-online-ddl-space-requirements.html)

### 2.3 尽量减少重复重建

同一发布范围内已经确定的多个字段，可以合并到一次 `ALTER TABLE` 中。字段默认值、是否允许 `NULL`、索引需求应提前设计，并结合兼容性决定是否合并执行。

如果历史记录有合理的固定默认值，可以评估直接添加最终的 `NOT NULL DEFAULT ...` 定义；不要机械地先加可空列再改成 `NOT NULL`，后者在 5.7 中可能再次重建。需要计算历史业务值时，再采用可空字段、分批回填、校验的流程。

## 3. gh-ost 是什么，如何选型

**gh-ost 是 GitHub 开源的 MySQL 在线表结构迁移工具，不是 MySQL 自带的功能或 SQL 命令，需要独立部署运行。**

```text
创建影子表，在空表上修改结构
    → 分批复制原表历史数据
    → 读取 binlog，同步业务新增、修改和删除
    → 复制完成且增量追平
    → 在合适时机切换表名
```

它仍要复制全表，主要价值是迁移速度、暂停和切换时机更容易控制。其增量同步不依赖触发器，可根据负载、复制延迟或人工指令暂停迁移。[gh-ost 项目](https://github.com/github/gh-ost)、[负载与限速控制](https://github.com/github/gh-ost/blob/master/doc/throttle.md)

使用前确认：

- 原表和新表有可用于迁移的共同主键或唯一键，迁移键不能包含实际为 `NULL` 的值。
- 工具读取的 binlog 满足所用版本的要求，常见前提为 `ROW` 与 `FULL` 行镜像。
- 外键、触发器、复制拓扑、权限和云数据库限制都经过预检查。
- 空间足够保存影子表、索引和新增日志；binlog 保留窗口覆盖迁移与暂停时间。
- 增量应用能力能跟上业务写入，否则难以完成追平和切换。

最终切换仍需获取锁。保留旧表不代表可以随时无损回切，因为切换后的新写入不会自动回到旧表。[使用要求与限制](https://github.com/github/gh-ost/blob/master/doc/requirements-and-limitations.md)

| 实际条件 | 方案倾向 |
| --- | --- |
| 低峰重建耗时、资源和从库延迟都可接受 | 原生 `INPLACE + LOCK=NONE`，实现和维护更简单 |
| 持续高写入，要求可暂停、可限速、控制切换窗口 | 满足前提时评估 gh-ost 或已有平台的在线迁移引擎 |
| 能安排停写维护窗口 | 在窗口中执行验证过的原生方案 |
| 实例已接近资源上限或磁盘不足 | 先增加资源或降低负载，工具本身不能消除全量迁移成本 |

`pt-online-schema-change` 也是常见工具，但依赖触发器同步增量。当前工作区的数据库规范禁止触发器，所以不把它作为该规范下的默认方案；这是项目约束，不是 MySQL 的通用限制。[Percona 工具说明](https://docs.percona.com/percona-toolkit/pt-online-schema-change.html)

## 4. 大厂公开实践与可借鉴的流程

公开资料支持的共同方向是：将变更接入平台，配合预检查、测试、负载控制、监控和切换管理，再按场景选择原生执行或影子表迁移，不能概括成“所有大厂都用 gh-ost”。

| 公开案例 | 已公开的做法 | 证据边界 |
| --- | --- | --- |
| GitHub | gh-ost，配合自动化迁移测试和数据校验 | 2017 年工程文章，不能据此推断当前全部内部架构。[工程实践](https://github.blog/engineering/infrastructure/mysql-testing-automation-at-github/) |
| 阿里云 DMS | 自研无锁结构变更引擎，也可配置原生 Online DDL 优先 | 产品能力和实际实例配置要区分，DMS 自研引擎不能直接等同 gh-ost。[执行策略](https://help.aliyun.com/zh/dms/enable-the-lock-free-schema-change-feature) |
| Facebook / Meta | 自研 OnlineSchemaChange，配合一致性检查和集成测试 | 2017 年历史实践，不作为当前所有业务使用方式的证明。[工程实践](https://engineering.fb.com/2017/05/05/production-engineering/onlineschemachange-rebuilt-in-python/) |

可借鉴的落地流程：提交变更 → 检查结构、容量和兼容性 → 演练 → 小范围执行 → 按指标控制负载 → 追平并切换 → 校验 → 启用新功能。分库分表应分批推进，应用需要兼容不同分片暂时处于不同结构的阶段。

“通过平台提交 SQL”不等于“使用了影子表迁移”。执行前应确认本次工单真正采用的引擎；产品称为“无锁”，也不能理解为不存在最终切换的元数据锁。

## 5. ES 一亿条数据只新增字段

### 5.1 新增 mapping 不会重写全部文档

```http
PUT /orders_v1/_mapping
{
  "properties": {
    "customer_level": {
      "type": "keyword"
    }
  }
}
```

上述操作添加字段定义，不会给旧文档自动填值，也无需扫描和重写一亿条数据。新旧文档可以同时存在：

```json
{"order_id": "1001"}
```

```json
{"order_id": "1002", "customer_level": "vip"}
```

生产中先显式定义字段类型，再调整写入程序。mapping 更新仍涉及集群元数据传播，大量索引或字段会增加负担，不能据此推断任意 mapping 变更都零成本。[Mapping 说明](https://www.elastic.co/docs/manage-data/data-store/mapping)

按天建索引或使用 rollover 时，还要更新相应模板，保障后续索引具有新字段。**修改模板不会自动修改已有索引**，已存在的目标索引需要另行更新 mapping。[索引模板](https://www.elastic.co/docs/manage-data/data-store/templates)

### 5.2 历史数据不一定要补值

如果旧订单缺少客户等级只影响页面展示，可以在应用中显示“未知”，避免一次全量回填。如果要按该值过滤、排序或聚合，则查询也要明确处理缺失值，或者进行字段回填。

- `null_value` 不会自动给缺失字段或历史文档补默认值，也不修改 `_source`；它用于索引时处理显式 `null`。
- 给已有 `text` 增加 `keyword` 子字段，历史文档也不会立刻获得该子字段的索引值，需要更新或重新索引。
- runtime field 可在支持的版本中提供查询时计算，但会把部分成本转移到查询，不能作为高频、大范围聚合的免费替代方案。

[null_value](https://www.elastic.co/docs/reference/elasticsearch/mapping-reference/null-value)、[新增多字段说明](https://www.elastic.co/docs/manage-data/data-store/mapping/update-mappings-examples)、[Runtime fields](https://www.elastic.co/guide/en/elasticsearch/reference/current/runtime.html)

## 6. ES 历史数据必须补齐

### 6.1 按字段值来源选择回填方式

| 新值来源 | 处理方式 |
| --- | --- |
| 固定值，或能从 ES 文档当前字段计算 | `_update_by_query` 配合脚本 |
| 来自 MySQL、其他服务或离线计算 | 从权威数据源分批读取，通过 Bulk API 更新 |
| 字段类型和整体索引结构都要调整 | 评估新建索引并重建数据 |

**ES 更新一个字段也会重新索引该文档，并不是只修改一块字段存储。** 回填一亿条文档会消耗 I/O、CPU、堆内存、副本写入和段合并资源；即使在原索引更新，也要预留磁盘增长和合并所需空间。[Update API](https://www.elastic.co/docs/api/doc/elasticsearch/operation/operation-update)

### 6.2 回填执行要点

1. 先让新的写入和更新逻辑正确维护该字段，再开始历史回填。
2. 优先回填业务真正需要的范围，例如最近几个月；先以小范围验证值和性能。
3. 按时间、业务范围或现有物理索引拆分，记录进度。脚本和批处理要幂等，不覆盖并发产生的新值。
4. `_update_by_query` 可设置 `wait_for_completion=false` 异步执行，通过 `requests_per_second` 限速和 `_rethrottle` 调速；只有资源允许时才增加 `slices`。
5. 监控查询 P95/P99、写入拒绝、CPU、GC、磁盘余量和段合并压力，并处理任务失败与版本冲突。
6. 校验业务范围内仍缺少字段的文档、失败记录和业务值正确性，完成后再启用依赖完整历史字段的功能。

注意三个边界：

- `_update_by_query` 根据开始时的快照查找文档；并发修改可能触发版本冲突。
- `conflicts=proceed` 只代表跳过并记录冲突，不代表那些文档已更新成功，需要补处理。
- 失败或取消不会回滚已完成的更新；异步执行也不会降低任务本身的资源消耗。

[Update by query API](https://www.elastic.co/docs/api/doc/elasticsearch/v8/operation/operation-update-by-query)

如果 ES 由 MySQL 同步而来，应同时调整同步转换逻辑。否则 ES 回填后，下一次来自上游的整文档覆盖可能再次删除新字段。批处理也要避免旧值晚到覆盖新值；需要根据同步链路使用顺序控制、业务版本或乐观并发控制。

## 7. ES 新索引迁移与别名切换

只新增字段不必创建新索引。需要改变已有字段类型、调整索引结构，或希望在切换前完整校验一份重建数据时，再考虑新索引方案。

```text
创建 orders_v2，配置 mapping、分片、副本等
    → 全量导入历史数据，并可靠保留迁移期间的变更
    → 按正确顺序追平新增、修改、删除
    → 校验数据、查询结果和性能
    → 切换读写别名
    → 观察稳定后按保留策略清理旧索引
```

关键边界：

- `_reindex` 不会自动复制源索引的 mapping、分片、副本等配置，目标索引应提前创建并配置；通常依赖可用的 `_source`。
- `_reindex` 是一次数据复制任务，不像 gh-ost 那样自动持续追平业务增量。直接“reindex 完成 → 切别名”，可能漏掉迁移期间的变更。
- 持续写入时，要设计可靠的 CDC/MQ 回放、顺序和版本控制；也可以选择能够保证一致性的停写窗口。双写本身不自动解决失败补偿、旧值覆盖和删除同步。
- 一次别名变更请求可以原子切换路由，但不能代替数据追平；一个别名关联多个索引也不会自动双写。
- 新旧索引共存需要额外容量；同集群重建仍会与在线查询竞争资源。
- 保留旧索引不代表可以无损回切。切换后新写入需要专门同步或补偿，才能保障回退时数据完整。

[Reindex API](https://www.elastic.co/guide/en/elasticsearch/reference/8.19/docs-reindex.html)、[Aliases](https://www.elastic.co/docs/manage-data/data-store/aliases)

## 8. 发布顺序与监控

推荐顺序：兼容性检查 → 完成字段定义变更 → 确认相关副本或索引生效 → 上线正确的新写入逻辑 → 按需回填历史数据 → 校验 → 启用新功能。

| 系统 | 重点观察 |
| --- | --- |
| MySQL | 长事务、MDL 等待、活跃连接、I/O、磁盘余量、主从延迟、业务响应时间 |
| ES | 搜索延迟、写入拒绝、CPU、GC、磁盘水位、段合并、任务进度、失败和冲突数量 |
| 同步链路 | 积压、丢失或重复事件、字段转换、版本顺序、删除同步和端到端延迟 |

MySQL 应检查不带列名的 `INSERT ... VALUES (...)`、依赖列顺序的映射等兼容风险；ES 应检查缺失值语义、模板和整文档覆盖逻辑。功能回退与存储结构回退分开考虑，新增兼容字段一般可以保留，不能依靠普通事务 `ROLLBACK` 撤销已完成的 MySQL DDL 或 ES 批量变更。

## 9. 高频误区与面试回答

| 常见说法 | 正确理解 |
| --- | --- |
| MySQL 5.7 用 INPLACE 加字段就不重建表 | 新增普通列仍会重建 |
| LOCK=NONE 或“无锁变更”完全不拿锁 | 仍有元数据锁或切换锁要求 |
| gh-ost 是 MySQL 原生能力 | 它是独立的开源迁移工具 |
| 所有大厂都用同一种工具 | 根据版本、负载、拓扑与平台选择执行方式 |
| ES 一亿条数据加字段一定要重建索引 | 只新增 mapping 不需要重写历史文档 |
| ES 局部更新只重写那个字段 | 被更新的整个文档会重新索引 |
| reindex 会自动跟上迁移期间的写入 | 需要额外设计增量同步和一致性方案 |
| 保留旧表或旧索引就能随时无损回退 | 切换后的新写入必须被同步或补偿 |

**MySQL 一分钟回答**：

> 我会先确认版本、表结构、数据和索引体积、写入量、磁盘余量及主从架构。MySQL 5.7 的普通加列即使用 INPLACE 也要重建，LOCK=NONE 也有 MDL 风险。低峰重建成本可接受时使用显式指定算法和锁级别的原生 DDL；高并发且需要可暂停、可限速时评估 gh-ost 或现有平台。上线前演练，期间监控锁等待、资源和复制延迟，追平并校验后再启用新字段功能。

**ES 一分钟回答**：

> 我会先区分添加字段定义和回填历史值。ES 只加 mapping 不需要重写一亿条旧文档，优先让新数据写入、旧数据兼容缺失。业务必须补齐历史值时，再按值来源选 update_by_query 或外部批处理，分范围、限速并处理并发冲突。需要修改已有字段类型或重做索引时才评估新索引迁移，并单独解决迁移期间的新增、修改和删除；别名原子切换不等于数据自动追平。

## 10. 关联笔记

- [MySQL 复习路线](../06-数据库/MYSQL知识点/README.md)
- [SQL 优化与场景题](../06-数据库/MYSQL知识点/05-SQL优化与场景题.md)
- [Elasticsearch 类型、常用命令与核心原理](../07-中间件/Elasticsearch类型命令与原理.md)
