# RocketMQ 学习路线

## 1. 学习总览

这条路线的目标不是只背 RocketMQ 的概念，而是建立一条完整的消息链路认知：

```text
Producer 发送消息
  -> NameServer 提供路由
  -> Broker 接收并存储
  -> CommitLog 保存消息主体
  -> ConsumeQueue 建立消费索引
  -> Consumer Group 分配 Queue
  -> Consumer 拉取并消费
  -> ACK / Offset 推进
  -> 失败重试
  -> 多次失败进入 DLQ
```

学习每一部分时，都按三个问题拆解：

```text
1. 它解决什么问题？
2. 核心流程是什么？
3. 生产环境会出什么问题？
```

## 2. 推荐学习顺序

```text
第1部分  RocketMQ 整体架构
第2部分  Topic / Queue / Broker / NameServer 核心模型
第3部分  Producer 消息发送流程
第4部分  CommitLog / ConsumeQueue / IndexFile 存储原理
第5部分  Consumer 消费流程
第6部分  Offset / ACK / 重试 / DLQ
第7部分  Rebalance
第8部分  顺序消息
第9部分  延迟消息
第10部分 事务消息
第11部分 主从复制 / 刷盘 / 高可用
第12部分 消息丢失 / 重复 / 积压
第13部分 Kafka vs RocketMQ
```

阶段目标：

| 阶段 | 内容 | 目标 |
|---|---|---|
| 第1-5部分 | 基本工作链路 | 讲清楚一条消息从发送到消费成功经历了什么 |
| 第6-7部分 | 消费可靠性和消费组行为 | 理解 Offset、ACK、重试、DLQ、Rebalance |
| 第8-10部分 | 高级消息能力 | 掌握顺序消息、延迟消息、事务消息，尤其事务消息 |
| 第11-12部分 | 生产问题排查 | 能定位消息丢失、重复、积压和 Broker 可靠性问题 |
| 第13部分 | 技术选型能力 | 能判断什么时候选 Kafka，什么时候选 RocketMQ |

## 第1部分 RocketMQ 整体架构

### 1. 它解决什么问题？

RocketMQ 解决的是系统之间如何可靠、异步、可扩展地传递消息。

例如订单系统下单后，不直接同步调用库存、积分、短信等系统，而是发送一条消息：

```text
订单系统 -> RocketMQ -> 库存服务 / 积分服务 / 短信服务 / 风控服务
```

这样可以降低系统耦合，提高吞吐，并在下游系统短暂不可用时缓冲消息。

### 2. 核心流程是什么？

RocketMQ 主要角色：

| 角色 | 作用 |
|---|---|
| Producer | 发送消息 |
| Consumer | 消费消息 |
| Broker | 存储和转发消息 |
| NameServer | 注册中心，保存 Broker 路由信息 |
| Topic | 消息主题 |
| Queue | Topic 下的队列，是负载均衡和顺序性的基本单位 |

核心链路：

```text
Broker 启动并注册到 NameServer
  -> Producer 从 NameServer 获取 Topic 路由
  -> Producer 选择 Broker 和 Queue
  -> Producer 发送消息
  -> Broker 写入磁盘
  -> Consumer 从 NameServer 获取路由
  -> Consumer 从 Broker 拉取消息
  -> Consumer 消费成功后提交 Offset
```

可以记住一句话：

> NameServer 管路由，Broker 管存储，Producer 管发送，Consumer 管消费，Queue 管分片。

### 3. 生产环境会出什么问题？

- NameServer 地址配置错误，Producer 或 Consumer 找不到 Broker。
- Broker 挂了，导致部分 Topic 不可用。
- Topic 队列数设置不合理，影响消费并发。
- Consumer Group 配错，导致重复消费或消费不到消息。
- Broker 磁盘满，消息无法写入。
- 网络抖动，Producer 发送超时或 Consumer 拉取失败。

## 第2部分 核心模型

### 1. 它解决什么问题？

核心模型解决的是 RocketMQ 如何组织消息、路由消息和分配消费。

要理解这些对象：

```text
Topic：消息分类
Queue：Topic 下的分片
Broker：真正存储消息的节点
NameServer：保存 Broker 和 Topic 路由
Producer Group：一类生产者
Consumer Group：一类消费者
```

### 2. 核心流程是什么？

一个 Topic 通常包含多个 Queue：

```text
OrderTopic
  Queue 0
  Queue 1
  Queue 2
  Queue 3
```

Producer 发送消息时，会从 Topic 的多个 Queue 中选择一个。Consumer Group 消费时，同组 Consumer 会分摊这些 Queue。

```text
Consumer A -> Queue 0, Queue 1
Consumer B -> Queue 2, Queue 3
```

同一个 Consumer Group 内，一条消息通常只被一个 Consumer 实例消费。不同 Consumer Group 会各自消费一份。

### 3. 生产环境会出什么问题？

- Queue 数太少，Consumer 扩容后仍然消费不快。
- Consumer Group 随便改名，导致从新消费进度开始消费。
- 同一个 Consumer Group 下订阅关系不一致，引发消费行为混乱。
- Topic 自动创建导致消息发到错误 Topic 后不容易发现。
- 测试环境和生产环境 NameServer 配错，消息发错集群。

## 第3部分 Producer 消息发送流程

### 1. 它解决什么问题？

Producer 解决的是业务系统如何把消息可靠地投递到 RocketMQ。

它需要处理：

```text
发到哪个 Broker？
发到哪个 Queue？
发送失败要不要重试？
发送结果如何确认？
使用同步、异步还是单向发送？
```

### 2. 核心流程是什么？

```text
Producer 启动
  -> 从 NameServer 拉取 Topic 路由
  -> 根据 Topic 找到 Broker 和 Queue
  -> 选择一个 Queue
  -> 发送消息到 Broker
  -> Broker 写入 CommitLog
  -> Broker 返回 SendResult
```

发送方式：

| 方式 | 特点 | 适用场景 |
|---|---|---|
| 同步发送 | 等待 Broker 返回结果，可靠性较高 | 订单、支付、核心业务事件 |
| 异步发送 | 通过回调处理结果，吞吐更高 | 日志、通知、非核心链路 |
| 单向发送 | 不等待结果，可靠性最低 | 可丢弃的监控或日志类数据 |

发送成功通常表示 Broker 已接收消息，并按配置完成写入或刷盘确认，不代表下游业务已经处理成功。

### 3. 生产环境会出什么问题？

- 发送超时：Broker 压力大、网络慢、磁盘慢。
- 发送失败：Topic 不存在、Broker 不可用、权限问题。
- 重试导致重复消息：Producer 不确定 Broker 是否已经写入成功。
- 消息体过大：影响网络、内存和 Broker 存储性能。
- 顺序消息没有按业务 key 选择 Queue，导致顺序被破坏。
- 异步发送没有处理失败回调，导致问题被吞掉。

生产经验：

> RocketMQ 可以提高消息投递可靠性，但业务必须接受消息可能重复。

## 第4部分 CommitLog / ConsumeQueue / IndexFile 存储原理

### 1. 它解决什么问题？

Broker 存储层解决的是如何高性能、可靠地保存和读取消息。

RocketMQ 不是把每个 Topic、每个 Queue 的完整消息单独存一份，而是采用：

```text
CommitLog：保存完整消息
ConsumeQueue：保存消费索引
IndexFile：支持按 key 查询消息
```

### 2. 核心流程是什么？

```text
Producer 发送消息
  -> Broker 接收消息
  -> 顺序追加写入 CommitLog
  -> 异步构建 ConsumeQueue
  -> Consumer 根据 ConsumeQueue 定位 CommitLog
  -> Broker 读取完整消息并返回给 Consumer
```

ConsumeQueue 里通常不存完整消息，只保存类似这样的信息：

```text
CommitLog offset
消息大小
Tag hash
```

可以记住：

> CommitLog 是消息本体，ConsumeQueue 是消费索引。

### 3. 生产环境会出什么问题？

- 磁盘 IO 慢，导致发送延迟升高。
- CommitLog 文件过多，占满磁盘。
- Broker 异常退出后需要恢复 CommitLog 和 ConsumeQueue。
- ConsumeQueue 构建延迟，导致消息已经写入但暂时不可消费。
- 消息堆积后磁盘读压力增大。
- IndexFile 只适合辅助排查，不适合当数据库频繁查询。

## 第5部分 Consumer 消费流程

### 1. 它解决什么问题？

Consumer 解决的是业务系统如何从 RocketMQ 获取消息，并完成业务处理。

它需要处理：

```text
多个消费者如何分摊消息？
消费进度如何记录？
消费失败怎么办？
消费顺序如何保证？
消费太慢导致积压怎么办？
```

### 2. 核心流程是什么？

消费模式：

| 模式 | 含义 |
|---|---|
| 集群消费 | 同一个 Consumer Group 内，一条消息只被一个 Consumer 消费 |
| 广播消费 | 同一个 Consumer Group 内，每个 Consumer 都会收到一份消息 |

最常见的是集群消费：

```text
Consumer 启动
  -> 向 Broker 注册
  -> 获取 Topic 路由
  -> 和同组 Consumer 做队列分配
  -> 从分配到的 Queue 拉取消息
  -> 执行业务消费逻辑
  -> 返回消费成功或失败
  -> 提交 Offset 或进入重试
```

虽然常说 PushConsumer，但 RocketMQ 的 Push 底层通常是拉取和长轮询：

```text
Consumer 发起拉取请求
  -> Broker 有消息就返回
  -> Broker 没消息就挂起一小段时间
  -> 有新消息后再返回
```

### 3. 生产环境会出什么问题？

- Consumer 处理慢，导致消息积压。
- Consumer 数量大于 Queue 数量，多出来的 Consumer 空闲。
- 消费逻辑异常，消息不断重试。
- Consumer Group 配置错误，导致消费进度混乱。
- Rebalance 期间可能出现短暂重复消费。
- 单条消息处理时间太长，拖慢整个队列。

生产经验：

> RocketMQ 的并发消费能力主要受 Queue 数量限制，不是 Consumer 越多越好。

## 第6部分 Offset / ACK / 重试 / DLQ

### 1. 它解决什么问题？

这一部分解决消费可靠性：

```text
Consumer 消费到哪里了？
消费成功如何确认？
消费失败如何重试？
一直失败的消息怎么办？
为什么会重复消费？
```

RocketMQ 默认更偏向至少一次投递：

```text
消息尽量不丢，但可能重复。
```

### 2. 核心流程是什么？

Offset 表示某个 Consumer Group 在某个 Topic 的某个 Queue 上消费到的位置：

```text
Topic: OrderTopic
Queue: 0
ConsumerGroup: order-service-group
Offset: 1024
```

正常消费：

```text
Consumer 拉取消息
  -> 执行业务逻辑
  -> 返回消费成功
  -> 提交 Offset
  -> 下次从新 Offset 继续消费
```

消费失败：

```text
Consumer 拉取消息
  -> 业务处理失败
  -> 返回消费失败
  -> Broker 将消息投递到重试队列
  -> 稍后再次投递
```

常见特殊队列：

```text
%RETRY%ConsumerGroup：重试队列
%DLQ%ConsumerGroup：死信队列
```

可以这样记：

```text
消费成功 -> ACK 成功 -> 推进 Offset
消费失败 -> 进入重试
重试耗尽 -> 进入 DLQ
```

### 3. 生产环境会出什么问题？

- 消费成功但 Offset 提交失败，可能重复消费。
- 业务处理成功，但 Consumer 返回失败，消息会重试。
- 消费失败一直重试，拖垮下游服务。
- DLQ 无人处理，问题消息被长期忽略。
- Offset 被错误重置，导致大量消息重复消费或跳过。
- Consumer Group 改名，相当于新消费组，会从新的消费策略位置开始。

幂等建议：

```text
使用业务唯一键去重，例如 orderId + eventType。
使用数据库唯一索引防重复。
使用消费记录表记录处理状态。
使用业务状态机判断是否允许重复执行。
```

## 第7部分 Rebalance

### 1. 它解决什么问题？

Rebalance 解决的是同一个 Consumer Group 中，多个 Consumer 如何分摊 Queue。

例如一个 Topic 有 4 个 Queue，同组有 2 个 Consumer：

```text
Consumer A -> Queue 0, Queue 1
Consumer B -> Queue 2, Queue 3
```

### 2. 核心流程是什么？

Rebalance 通常发生在：

```text
Consumer 启动
Consumer 下线
Consumer 崩溃
Consumer Group 内实例数量变化
Topic Queue 数量变化
Broker 路由变化
订阅关系变化
```

核心流程：

```text
Consumer 获取消费组内所有实例
  -> 获取 Topic 下所有 Queue
  -> 按负载均衡策略分配 Queue
  -> 每个 Consumer 只消费分给自己的 Queue
  -> 分配关系变化后释放旧 Queue，接管新 Queue
```

Rebalance 的本质：

> 重新分配 Queue 和 Consumer 的归属关系。

### 3. 生产环境会出什么问题？

- 短时间重复消费：旧 Consumer 未提交 Offset，新 Consumer 接管后重新消费。
- Consumer 数量超过 Queue 数量，部分实例空闲。
- 实例频繁重启或网络抖动，导致频繁 Rebalance。
- 同一个 Consumer Group 下订阅关系不一致，导致消费行为混乱。

生产建议：

```text
同一个 Consumer Group 保持相同订阅和相同消费逻辑。
扩容前先看 Queue 数量。
减少实例频繁重启和心跳异常。
业务侧必须幂等。
```

## 第8部分 顺序消息

### 1. 它解决什么问题？

顺序消息解决的是同一业务维度下，事件必须按顺序处理。

例如订单状态：

```text
创建订单 -> 支付成功 -> 发货 -> 确认收货
```

生产中通常追求局部顺序，而不是全局顺序：

```text
同一个订单有序，不同订单可以并发。
```

### 2. 核心流程是什么？

核心是相同业务 key 的消息进入同一个队列，并按顺序消费。

```text
Producer 设置顺序 key / MessageGroup
  -> RocketMQ 将同一组消息写入同一 Queue
  -> Consumer 按 Queue 顺序消费
  -> 前一条成功后，后一条继续
```

示例：

```text
orderId=1001 -> Queue 0
orderId=1002 -> Queue 1
orderId=1001 -> Queue 0
```

### 3. 生产环境会出什么问题？

- 顺序 key 太粗，造成热点。
- 消费失败会阻塞后续消息。
- Consumer 收到消息后丢到线程池异步处理，业务自己打乱顺序。
- 追求全局顺序导致吞吐明显下降。

生产建议：

```text
只保证必要的局部顺序。
顺序 key 要足够分散。
顺序消费逻辑不要随便异步并发处理。
失败消息要有告警和人工处理机制。
```

## 第9部分 延迟消息

### 1. 它解决什么问题？

延迟消息解决的是消息发送后，不立即消费，而是在未来某个时间点再消费。

典型场景：

```text
订单超时关闭
支付结果延迟检查
优惠券到期提醒
任务延后执行
失败任务补偿
```

### 2. 核心流程是什么？

```text
Producer 发送延迟消息
  -> Broker 接收，但暂时不投递
  -> 消息进入延迟存储
  -> 到达指定时间
  -> Broker 将消息变为可消费
  -> Consumer 正常消费
```

订单超时关闭示例：

```text
用户创建订单
  -> 发送一条 30 分钟后的延迟消息
  -> 30 分钟后 Consumer 收到
  -> 查询订单状态
  -> 未支付则关闭，已支付则忽略
```

版本差异：

```text
RocketMQ 4.x：常见方式是使用固定延迟等级。
RocketMQ 5.x：更偏向设置未来投递时间戳，并使用 Delay 类型 Topic。
```

### 3. 生产环境会出什么问题？

- 延迟消息不是精准定时器，Broker 压力大或堆积时可能晚于预期。
- 大量消息同一时间触发，造成消费洪峰。
- 收到延迟消息后没有重新校验业务状态，导致误操作。
- 把延迟消息当长期定时任务系统使用，维护成本高。

生产建议：

```text
消费延迟消息时必须二次查询业务状态。
避免大量消息设置完全相同的触发时间，可以适当打散。
不要假设投递时间精确到毫秒。
长期周期性任务优先考虑调度系统。
```

## 第10部分 事务消息

### 1. 它解决什么问题？

事务消息解决的是本地事务和发送消息之间的一致性问题。

例如支付成功后通知下游：

```text
支付服务更新支付单为 SUCCESS
  -> 发送消息通知订单、积分、营销等系统
```

普通消息容易出现两个风险：

```text
本地事务成功，但消息发送失败，下游不知道。
消息发送成功，但本地事务失败，下游误以为成功。
```

事务消息解决的是：

> 本地事务成功，消息最终能被投递；本地事务失败，消息不能被投递。

注意，它解决的是 Producer 本地事务和消息发送的一致性，不直接保证 Consumer 消费一定成功。

### 2. 核心流程是什么？

事务消息核心是两阶段加回查：

```text
Producer 发送半事务消息
  -> Broker 持久化消息，但暂不投递给 Consumer
  -> Broker 返回发送成功
  -> Producer 执行本地事务
  -> Producer 根据本地事务结果提交二次确认
  -> Broker 根据结果 Commit 或 Rollback
```

本地事务成功：

```text
业务数据库事务成功
  -> Producer 发送 Commit
  -> Broker 将半事务消息变为可投递
  -> Consumer 消费消息
```

本地事务失败：

```text
业务数据库事务失败
  -> Producer 发送 Rollback
  -> Broker 丢弃半事务消息
  -> Consumer 看不到这条消息
```

Producer 宕机或网络异常时，Broker 会进行事务回查：

```text
Broker 发现半事务消息没有最终状态
  -> 向 Producer Group 中的 Producer 发起回查
  -> Producer 查询本地事务状态
  -> 返回 Commit / Rollback / Unknown
```

### 3. 生产环境会出什么问题？

- 把事务消息误认为分布式强一致事务。
- 事务回查逻辑写错，依赖内存、缓存或临时变量。
- 没有业务唯一键，回查时不知道查哪条业务记录。
- 长时间返回 Unknown，导致事务状态悬挂。
- Consumer 端仍然可能重复消费，没有做幂等。

正确回查方式：

```text
根据 message key / transactionId / orderId 查询数据库最终状态。
状态成功 -> Commit
状态失败或不存在 -> Rollback
状态仍处理中 -> Unknown
```

生产经验：

> 事务消息不是分布式强一致事务，它解决的是本地事务和消息发送之间的最终一致性。

## 第11部分 主从复制 / 刷盘 / 高可用

### 1. 它解决什么问题？

这一部分解决 Broker 层面的可靠性问题：

```text
消息写到 Broker 后会不会丢？
Broker 宕机后消息还能不能消费？
Master 挂了，Slave 有没有完整数据？
磁盘满了，消息还能不能写？
```

核心关注：

```text
刷盘：消息有没有真正落到磁盘。
复制：消息有没有同步到副本。
高可用：Broker 故障后系统能否继续服务。
```

### 2. 核心流程是什么？

```text
Producer 发送消息
  -> Master Broker 接收消息
  -> 写入 CommitLog
  -> 根据 flushDiskType 决定是否等待刷盘
  -> 根据 brokerRole 决定是否等待 Slave 同步
  -> 返回 SendResult 给 Producer
```

关键配置：

| 配置 | 方式 | 说明 |
|---|---|---|
| flushDiskType | ASYNC_FLUSH | 写入 PageCache 后返回，性能高，极端宕机可能丢少量消息 |
| flushDiskType | SYNC_FLUSH | 刷到磁盘后返回，可靠性高，但延迟更高 |
| brokerRole | ASYNC_MASTER | Master 写成功后返回，之后异步同步 Slave |
| brokerRole | SYNC_MASTER | Master 等 Slave 同步后再返回，可靠性更高 |
| brokerRole | SLAVE | 从节点，保存副本 |

常见组合：

```text
高吞吐：ASYNC_MASTER + ASYNC_FLUSH
高可靠：SYNC_MASTER + SYNC_FLUSH
折中型：SYNC_MASTER + ASYNC_FLUSH
```

### 3. 生产环境会出什么问题？

- Broker 磁盘满，无法写入新消息。
- 刷盘慢，Producer 发送超时。
- Slave 落后 Master，主从切换后少量消息不可见。
- Master 宕机，异步复制下可能丢失未同步消息。
- CommitLog 清理过早，历史消息无法回溯。

排查重点：

```text
Broker 磁盘使用率
Broker 写入耗时
刷盘耗时
主从复制延迟
Broker 日志
Producer 发送超时日志
```

生产经验：

> 刷盘决定消息是否落盘，主从复制决定 Broker 挂了后是否还有副本，高可用决定故障后业务能不能继续跑。

## 第12部分 消息丢失 / 重复 / 积压

### 12.1 消息丢失

#### 1. 它解决什么问题？

排查这类问题：

```text
Producer 说发了，Consumer 说没收到，消息到底去哪了？
```

#### 2. 核心排查流程是什么？

按三段查：

```text
Producer -> Broker -> Consumer
```

Producer 侧：

```text
是否发送成功？
是否打印 SendResult？
是否有 message key？
是否发错 Topic？
是否发错环境？
是否发送超时后被业务吞掉异常？
```

Broker 侧：

```text
能否按 topic + key 查到消息？
Broker 是否有写入异常？
CommitLog 是否正常？
磁盘是否满？
消息是否过期被清理？
```

Consumer 侧：

```text
Consumer Group 是否正确？
订阅 Topic / Tag 是否正确？
Offset 是否已经跳过？
是否被同组其他实例消费？
是否进入重试队列？
是否进入 DLQ？
```

#### 3. 生产环境常见原因是什么？

- 发送失败但没记录。
- 发错 Topic 或环境。
- Tag 过滤导致消费不到。
- Consumer Group 配错。
- Offset 被重置。
- 消息已经被同组消费者消费。
- 消息过期被清理。
- Broker 异步刷盘或异步复制时发生极端宕机。

### 12.2 消息重复

#### 1. 它解决什么问题？

排查这类问题：

```text
订单为什么处理了两次？
库存为什么扣了两次？
短信为什么发了两条？
```

RocketMQ 默认更接近至少一次投递：

```text
尽量不丢，但可能重复。
```

#### 2. 核心流程是什么？

Consumer 端重复：

```text
Consumer 消费成功
  -> 业务数据库已经更新
  -> ACK / Offset 提交失败
  -> Broker 认为没成功
  -> 消息再次投递
```

Producer 端重复：

```text
Producer 发送消息
  -> Broker 写入成功
  -> Producer 等响应超时
  -> Producer 重试
  -> 产生两条业务相同的消息
```

Rebalance 重复：

```text
Consumer A 正在消费
  -> 还没提交 Offset
  -> 发生 Rebalance
  -> Consumer B 接管 Queue
  -> 从旧 Offset 再消费一次
```

#### 3. 生产环境怎么处理？

核心是业务幂等：

```text
数据库唯一索引
消费记录表
业务状态机
Redis setnx 去重
orderId + eventType 作为幂等键
```

不要只依赖 msgId，更推荐业务唯一键：

```text
orderId + PAY_SUCCESS
paymentId
transactionId
```

### 12.3 消息积压

#### 1. 它解决什么问题？

排查这类问题：

```text
Producer 一直发
Consumer 消费不过来
Broker 上未消费消息越来越多
业务延迟越来越高
```

#### 2. 核心排查流程是什么？

先看指标：

```text
生产 TPS
消费 TPS
消费耗时
消息堆积量
Consumer 实例数
Topic Queue 数
重试消息数量
DLQ 数量
下游数据库 / Redis / HTTP 耗时
```

再判断积压形态：

```text
所有 Queue 都积压：
整体消费能力不足，可能是下游慢、线程少、消费逻辑重。

某几个 Queue 积压：
可能是热点 key、顺序消息阻塞、某个 Consumer 异常。
```

#### 3. 生产环境怎么处理？

短期止血：

```text
扩容 Consumer
提高消费线程数
临时降级非核心逻辑
批量消费
修复慢接口
隔离异常消息
必要时重置 Offset 跳过低价值历史消息
```

长期治理：

```text
增加 Topic Queue 数
优化单条消息处理耗时
减少 DB / RPC 调用
热点 key 打散
失败消息进入独立补偿流程
监控重试队列和 DLQ
```

生产经验：

> 消息积压不是只靠加机器解决，先看 Queue 数、消费耗时、失败重试和下游容量。

## 第13部分 Kafka vs RocketMQ

### 1. 它解决什么问题？

这一部分解决技术选型问题。

选型时要问：

```text
消息主要是业务事件，还是数据事件？
是否需要事务消息？
是否大量使用延迟消息？
是否需要消息回放？
是否需要流计算生态？
团队更熟哪个中间件？
运维和监控体系是否已经具备？
```

### 2. 核心对比是什么？

| 维度 | Kafka | RocketMQ |
|---|---|---|
| 核心定位 | 事件流平台 | 分布式消息中间件 |
| 典型场景 | 日志、埋点、数据流、流计算、数据同步 | 订单、支付、交易、异步解耦、事务消息 |
| 存储模型 | Topic + Partition + Log | Topic + Queue + CommitLog + ConsumeQueue |
| 消费模型 | Consumer Group 分配 Partition | Consumer Group 分配 Queue |
| 顺序能力 | Partition 内有序 | Queue / MessageGroup 内有序 |
| 延迟消息 | 原生能力较弱，通常依赖扩展 | 原生支持延迟 / 定时消息 |
| 事务能力 | 偏 Kafka 链路和流处理一致性 | 偏本地事务和消息发送一致性 |
| 生态 | Kafka Connect、Kafka Streams、Flink / Spark 生态强 | 业务消息语义更完整 |

Kafka 更适合：

```text
日志采集
用户行为埋点
实时数据管道
大数据平台
流式计算
CDC 数据同步
指标采集
多个系统订阅同一批事件流
```

RocketMQ 更适合：

```text
订单状态变更
支付成功通知
库存扣减
营销发券
异步解耦
延迟任务
事务消息
业务补偿
顺序业务事件
```

### 3. 生产环境选型会出什么问题？

- 只看吞吐，不看业务语义，导致后续补偿、延迟、事务能力需要自己实现。
- 只看功能，不看团队运维能力，导致故障时没人能排查。
- 把 RocketMQ 当纯日志流平台，生态能力不如 Kafka 自然。
- 把 Kafka 当业务事务消息中间件，需要额外建设很多补偿能力。
- 没有考虑消息回放、保留周期、监控、DLQ、重试策略等生产治理能力。

选型结论：

```text
Kafka：更适合事件流、日志流、数据管道、流计算生态。
RocketMQ：更适合业务消息、交易链路、事务消息、延迟消息、业务补偿。
```

更成熟的表达：

> 如果系统核心是数据流和实时计算，我优先考虑 Kafka；如果系统核心是业务解耦、事务消息、延迟消息和补偿机制，我优先考虑 RocketMQ。最终还要结合团队运维能力、现有生态和故障恢复方案。

## 3. 生产配置关注点

### 3.1 Producer 核心配置

| 配置 | 作用 | 生产注意点 |
|---|---|---|
| producerGroup | 生产者组名 | 同一类发送业务用同一个组 |
| namesrvAddr | NameServer 地址 | 配多个 NameServer，避免单点 |
| sendMsgTimeout | 发送超时时间 | 结合业务链路设置，避免过短或过长 |
| retryTimesWhenSendFailed | 同步发送失败重试次数 | 重试可能导致重复消息 |
| retryTimesWhenSendAsyncFailed | 异步发送失败重试次数 | 异步发送必须处理失败回调 |
| retryAnotherBrokerWhenNotStoreOK | 存储失败时是否换 Broker 重试 | 高可靠场景可开启，但要接受重复风险 |
| maxMessageSize | 单条消息最大大小 | 不建议发大消息 |
| compressMsgBodyOverHowmuch | 消息压缩阈值 | 节省网络但增加 CPU 消耗 |

### 3.2 Broker 核心配置

| 配置 | 作用 | 生产注意点 |
|---|---|---|
| brokerClusterName | Broker 所属集群 | 多环境隔离 |
| brokerName | Broker 名称 | 主从节点 brokerName 要一致 |
| brokerId | Broker ID | 0 是 Master，非 0 是 Slave |
| namesrvAddr | NameServer 地址 | Broker 启动后向 NameServer 注册 |
| brokerIP1 | 对客户端暴露的 IP | 多网卡、容器、跨机房时显式配置 |
| storePathRootDir | 存储根目录 | 建议独立磁盘 |
| storePathCommitLog | CommitLog 路径 | 磁盘性能很关键 |
| fileReservedTime | 文件保留时间 | 结合磁盘容量和回溯需求 |
| diskMaxUsedSpaceRatio | 磁盘使用率阈值 | 必须监控和告警 |
| brokerRole | Broker 角色 | SYNC_MASTER 更可靠，ASYNC_MASTER 性能更好 |
| flushDiskType | 刷盘方式 | SYNC_FLUSH 更可靠，ASYNC_FLUSH 性能更好 |
| autoCreateTopicEnable | 是否自动创建 Topic | 生产建议关闭 |
| aclEnable | 是否开启 ACL | 生产环境建议开启 |

### 3.3 Consumer 核心配置

| 配置 | 作用 | 生产注意点 |
|---|---|---|
| consumerGroup | 消费者组名 | 极其重要，不要随意变更 |
| namesrvAddr | NameServer 地址 | 配多个地址 |
| messageModel | 消费模式 | 常用 CLUSTERING，广播消费慎用 |
| consumeFromWhere | 首次启动从哪里消费 | 新组上线时要特别小心 |
| subscription | 订阅 Topic 和 Tag | 同组订阅关系要一致 |
| consumeThreadMin / consumeThreadMax | 消费线程数 | 结合下游容量设置 |
| consumeMessageBatchMaxSize | 单次批量消费数量 | 批量提升吞吐，但失败粒度变粗 |
| pullBatchSize | 单次拉取数量 | 太大会增加内存和处理压力 |
| maxReconsumeTimes | 最大重试次数 | 超过后进入 DLQ |
| consumeTimeout | 消费超时时间 | 消费太慢会触发重试或堆积 |

## 4. 生产上线检查清单

```text
NameServer 是否至少 2 个？
Broker 是否多副本？
Topic 是否提前创建？
autoCreateTopicEnable 是否关闭？
Topic Queue 数量是否满足消费并发？
Producer 是否处理发送失败？
重要消息是否设置业务 key？
Consumer Group 命名是否稳定？
同一个 Consumer Group 订阅关系是否一致？
Consumer 是否做了幂等？
重试次数是否合理？
DLQ 是否有告警和处理流程？
Broker 磁盘容量是否有告警？
CommitLog 磁盘 IO 是否可承载峰值？
是否监控发送 TPS、消费 TPS、堆积量、失败率、重试量？
```

## 5. 总结口诀

```text
基本链路：Producer -> NameServer -> Broker -> Consumer。
存储核心：CommitLog 存消息，ConsumeQueue 存索引。
消费可靠性：Offset 定进度，ACK 定成败，失败进重试，耗尽进 DLQ。
消费组行为：Consumer Group 分摊 Queue，Rebalance 会带来重复消费风险。
高级消息：顺序管顺序，延迟管时间，事务管本地事务和消息发送的一致性。
生产排查：丢消息按三段查，重复消息靠幂等，消息积压看 Queue、耗时和下游。
技术选型：数据流优先 Kafka，业务消息优先 RocketMQ。
```
