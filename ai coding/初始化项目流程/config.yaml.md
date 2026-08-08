schema: spec-driven

# Project context — shown to AI when creating artifacts
context: |
  语言：中文（简体）
  所有产出物必须用简体中文撰写
  # 技术栈
  - JDK: JDK 1.8
  - 框架: Spring Boot 2.5.X 或 1.5.X, Spring Cloud Dalston.SR1
  - 数据库: MySQL 5.7, MyBatis 3.4.X
  - 配置中心: Apollo / SpringCloud Config
  - 服务调用: Feign + Hystrix (内部微服务优先使用 Feign 调用)
  - MQ: Kafka (framework-kafka) + RocketMQ (framework-rocketmq)
  - 缓存: Redis (framework-redis-cache) + Redisson + Guava Cache
  - 代码风格: 阿里巴巴 Java 代码规范

  # 架构规范
  - 分层职责: Controller不写业务逻辑，必须下沉到 Service；Service 不直接操作数据库，必须通过 Mapper
  - 不允许跨层调用（如 Controller 直接调用 mapper)
  - Mapper/Service: 禁用 MyBatis Plus 的 Wrapper、ServiceImpl、BaseMapper，统一使用 GenericMapper / GenericService
  - 日志: 禁止 System.out.println，必须使用 SLF4J 日志框架
  - 跨服务: 禁止跨服务直接访问对方数据库，必须通过 API 调用
  - 性能: 禁止在循环中操作数据库/远程接口/ES,redis等中间件

  # 数据库规范
  - 主键统一 id bigint AUTO_INCREMENT
  - 表名/字段名: 小写，多单词用下划线连接 (如 user_info, user_name)
  - 必含审计字段: trace_id, created_by, creation_date, updated_by, enabled_flag, updation_date
  - 禁止使用外键约束、存储过程、视图、触发器

  # 安全规范
  - 禁止硬编码敏感信息，禁止 SQL 注入(${}拼接)
  - 外部请求必须有超时和失败策略，TLS 校验不得关闭
  - 日志禁止输出密码/Token 等敏感信息

# Per-artifact rules
rules:
  proposal:
    - 提案必须包含：Why(变更动机和业务价值)、What Changes(变更内容)、Capabilities(新增/修改的能力)、Impact(影响范围：模块、API、数据库)
    - 必须列出受影响的 controller、service、mapper 类及方法
    - 列出需要依赖外部功能
  specs:
    - 每条需求使用RFC2119关键词表达强度等级：MUST(必须)、SHOULD(应当)、MAY(可以)
    - 场景使用 Given/When/Then 格式描述业务场景，每个需求必须包含至少一个验收场景
    - 包含边界条件和异常场景，注重代码健壮性
    - 涉及状态流转的需求必须覆盖所有合法状态和边界状态
    - 安全敏感改动必须标注风险点、安全控制、剩余风险
    - MODIFIED 必须包含完整更新内容，不能只写差异
  design:
    - 设计必须覆盖：Context、Goals/Non-Goals、Decisions(含替代方案及选型理由)、Risks/Trade-offs、Migration Plan
    - 跨模块变更或新架构模式时必须写 design.md
    - 聚焦架构和方案，不是逐行实现细节
    - 必须说明数据库设计
    - 必须提供API设计说明（路径、方法、请求/响应格式）
    - 必须说明事务边界和异常处理策略
    - 必须提供关键代码结构和类关系图
    - 外部调用必须标注超时、失败策略、目标限制
  tasks:
    - 任务按分层拆解：例如Controller 任务 → Service 任务 → Mapper 任务
    - 每个任务使用 - [ ] X.Y 格式，描述格式：- [ ] X.Y 完成 {层级}.{类名}.{方法名} 功能
    - 任务粒度控制在 2 小时以内
    - 需要人工完成的任务使用 [MANUAL] 前缀标注
    - 任务按依赖排序，先做基础再叠加功能
    - 任务执行后需要确保代码可以编译通过