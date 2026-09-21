# Java 链路追踪

## 目录

- [1. 链路追踪解决什么问题](#1-链路追踪解决什么问题)
- [2. 核心概念](#2-核心概念)
- [3. Trace ID 如何生成和传递](#3-trace-id-如何生成和传递)
- [4. MDC 是什么](#4-mdc-是什么)
- [5. 最小实现：Trace ID + 请求头 + MDC](#5-最小实现trace-id--请求头--mdc)
- [6. 异步与线程池为什么会导致链路丢失](#6-异步与线程池为什么会导致链路丢失)
- [7. OpenTelemetry 是什么](#7-opentelemetry-是什么)
- [8. OpenTelemetry 与 Pinpoint](#8-opentelemetry-与-pinpoint)
- [9. Java 开发者需要掌握到什么程度](#9-java-开发者需要掌握到什么程度)
- [10. 线上排查方法](#10-线上排查方法)
- [11. 常见误区](#11-常见误区)
- [12. 面试回答模板](#12-面试回答模板)
- [13. 参考资料](#13-参考资料)

## 1. 链路追踪解决什么问题

在单体应用中，一次请求通常只经过一个进程，通过日志就比较容易定位问题。微服务系统中，一次请求可能经过网关、订单、库存、支付、数据库和消息队列：

```text
用户
  -> 网关
  -> 订单服务
       -> 库存服务 -> 数据库
       -> 优惠券服务
       -> 支付服务
```

当用户反馈“下单用了 5 秒并且失败”时，只看各个服务独立的日志会遇到几个问题：

- 无法确定哪些日志属于同一次请求；
- 不清楚请求经过了哪些服务；
- 不知道时间主要消耗在哪一步；
- 上游只看到“调用失败”，难以确认真正的异常源头；
- 跨线程、消息队列和重试会进一步打乱日志顺序。

链路追踪的作用，就是把分散在不同进程、线程和组件中的操作关联起来，还原一次请求的完整执行路径。

链路追踪主要回答三个问题：

```text
请求经过了哪里？
每一步花了多长时间？
异常最早出现在哪里？
```

它与日志、指标的分工如下：

| 数据 | 主要回答的问题 |
|---|---|
| Trace | 某一次请求经过哪里、在哪一步慢或失败 |
| Log | 某个位置具体发生了什么、异常细节是什么 |
| Metric | 系统整体是否异常，例如错误率、吞吐量、P99 耗时 |

三者需要结合使用。链路可以缩小范围，日志解释具体原因，指标判断影响范围和变化趋势。

## 2. 核心概念

### 2.1 Trace

`Trace` 表示一次完整的分布式请求。链路中的所有操作共享同一个 `Trace ID`。

### 2.2 Span

`Span` 表示链路中的一次具体操作，例如：

- 接收一个 HTTP 请求；
- 调用一个下游服务；
- 执行一条 SQL；
- 发送或消费一条消息；
- 执行一个重要业务步骤。

每个 Span 通常包含：

- `Trace ID`；
- `Span ID`；
- 父 Span ID；
- 操作名称；
- 开始时间和结束时间；
- 状态、异常和属性；
- 所属服务和实例信息。

例如：

```text
Trace ID：4bf92f3577b34da6a3ce929d0e0e4736

订单请求：3000ms
├── 参数校验：10ms
├── 查询数据库：80ms
├── 调用库存服务：200ms
└── 调用支付服务：2700ms
    └── 调用银行接口：2600ms，超时
```

这条链路只能证明主要耗时出现在支付服务调用银行接口的阶段。要判断是网络、连接池、银行接口还是代码问题，还需要继续检查子 Span、日志和指标。

### 2.3 上下文传播

服务之间必须传递 Trace 上下文，下游才能加入同一条链路：

```text
服务 A --HTTP Header--> 服务 B
服务 A --RPC Metadata--> 服务 B
服务 A --Message Header--> MQ --> 服务 B
```

如果只在入口生成 Trace ID，却没有向下游传递，那么每个服务都会生成自己的 Trace，链路就会断开。

## 3. Trace ID 如何生成和传递

### 3.1 UUID

简单系统可以使用 UUID：

```java
String traceId = UUID.randomUUID()
        .toString()
        .replace("-", "");
```

结果是 32 个十六进制字符：

```text
4bf92f3577b34da6a3ce929d0e0e4736
```

`UUID.randomUUID()` 适合用作简单的请求标识。与自增 ID 相比，它不依赖数据库，也适合多实例部署。

### 3.2 其他生成方式

| 方式 | 说明 | 使用建议 |
|---|---|---|
| UUID v4 | 实现简单、冲突概率极低 | 适合简单日志关联 |
| 128 bit 随机数 | 可直接生成 32 位十六进制字符串 | 适合遵循追踪格式的实现 |
| 雪花算法 | 带时间和机器信息，常用于业务主键 | 没有必要专门用于 Trace ID |
| 数据库自增 ID | 依赖数据库并增加中心节点 | 不建议用于 Trace ID |
| 追踪框架自动生成 | 同时管理 Trace、Span、采样和传播 | 生产微服务优先选择 |

如果要遵循 W3C Trace Context，Trace ID 是 16 字节、32 个小写十六进制字符，并且不能全部为 `0`。不要只满足长度，还要遵循完整协议校验规则。

### 3.3 放在哪里

Trace ID 属于技术上下文，通常放在 HTTP 请求头、RPC 元数据或消息 Header 中，而不是业务请求体中。

简单的自定义请求头可以是：

```http
X-Trace-Id: 4bf92f3577b34da6a3ce929d0e0e4736
```

标准链路追踪通常使用 W3C `traceparent`：

```http
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
```

其主要内容是：

```text
00                                 版本
4bf92f3577b34da6a3ce929d0e0e4736 Trace ID
00f067aa0ba902b7                   Parent Span ID
01                                 Trace Flags，例如是否采样
```

生产系统不应无条件信任公网请求传入的自定义 Trace ID。通常由可信入口校验格式、限制长度，并在需要时重新生成，以避免伪造和日志污染。

## 4. MDC 是什么

MDC 全称是 `Mapped Diagnostic Context`，可以理解为日志框架提供的“当前线程日志上下文”。常见入口类是：

```java
import org.slf4j.MDC;
```

设置上下文：

```java
MDC.put("traceId", traceId);
MDC.put("spanId", spanId);
```

业务代码正常记录日志：

```java
log.info("开始创建订单");
```

日志格式配置了 `%X{traceId}` 后，输出就可以自动携带标识：

```text
[4bf92f3577b34da6a3ce929d0e0e4736] 开始创建订单
```

MDC 常用操作：

```java
MDC.put("traceId", traceId);
String traceId = MDC.get("traceId");
MDC.remove("traceId");
MDC.clear();
```

MDC 通常基于线程上下文实现，所以要注意两个边界：

1. 线程池中的线程会被复用，请求结束时必须在 `finally` 中清理；
2. 切换到另一个线程后，MDC 默认不一定会自动传播。

MDC 只负责给日志附加上下文，不负责创建调用树、采集耗时或跨服务传递，因此它不是完整的链路追踪系统。

## 5. 最小实现：Trace ID + 请求头 + MDC

下面的过滤器适合帮助理解最小实现。正式的微服务链路追踪优先使用成熟框架。

```java
@Component
public class TraceIdFilter extends OncePerRequestFilter {

    private static final String TRACE_ID_HEADER = "X-Trace-Id";
    private static final String TRACE_ID_KEY = "traceId";

    @Override
    protected void doFilterInternal(
            HttpServletRequest request,
            HttpServletResponse response,
            FilterChain filterChain) throws ServletException, IOException {

        String traceId = request.getHeader(TRACE_ID_HEADER);
        if (traceId == null || traceId.isBlank()) {
            traceId = UUID.randomUUID().toString().replace("-", "");
        }

        try {
            MDC.put(TRACE_ID_KEY, traceId);
            response.setHeader(TRACE_ID_HEADER, traceId);
            filterChain.doFilter(request, response);
        } finally {
            MDC.remove(TRACE_ID_KEY);
        }
    }
}
```

调用下游服务时还需要继续传递：

```java
requestHeaders.set("X-Trace-Id", MDC.get("traceId"));
```

这个方案只能完成基础日志关联，不能自动提供：

- Span 父子关系；
- SQL 和 RPC 耗时；
- 自动采样；
- 调用拓扑；
- 完整的异步上下文传播；
- Collector、存储和查询页面。

## 6. 异步与线程池为什么会导致链路丢失

下面的代码可能在线程池的另一个线程中执行：

```java
MDC.put("traceId", "abc123");

CompletableFuture.runAsync(() -> {
    log.info("异步处理订单");
});
```

如果上下文没有被复制，异步日志可能拿不到 `abc123`。常见断链位置包括：

- `CompletableFuture`；
- `@Async`；
- 自建线程和线程池；
- Reactor 等响应式流程；
- 定时任务；
- Kafka、RocketMQ 等消息生产和消费；
- 自定义 HTTP、RPC 客户端。

处理原则是：

```text
提交任务前捕获上下文
  -> 在线程池工作线程中恢复上下文
  -> 执行任务
  -> finally 清理工作线程上下文
```

只复制 MDC 可以解决日志字段问题，但完整链路还需要传播 OpenTelemetry Context 或具体追踪框架的上下文。不要把 MDC 复制等同于 Trace 上下文传播。

## 7. OpenTelemetry 是什么

OpenTelemetry，简称 OTel，是一套厂商中立的可观测性标准和工具集，用于生成、采集、处理和导出：

- Trace；
- Metric；
- Log。

典型结构如下：

```text
Java 应用
  │ OpenTelemetry Java Agent / SDK
  ▼
OpenTelemetry Collector
  ▼
Jaeger / Tempo / Prometheus / Elastic / 商业平台
```

OpenTelemetry 可以负责：

- 自动生成 Trace ID 和 Span ID；
- 创建 HTTP、RPC、JDBC、消息等 Span；
- 跨服务注入和提取追踪上下文；
- 记录耗时、状态、异常和属性；
- 对数据进行采样、处理和导出；
- 将 Trace 与日志、指标关联。

它本身不是完整的存储和查询平台。链路的长期存储、查询和可视化通常交给 Jaeger、Tempo、Elastic 或其他后端。

Java 项目可以使用 Java Agent 自动埋点，也可以通过 API 对关键业务步骤进行手动埋点。自动埋点负责通用框架，手动埋点补充真正重要的业务阶段。不要把每个普通方法都创建成 Span，否则会增加数据量和排查噪声。

## 8. OpenTelemetry 与 Pinpoint

Pinpoint 是一套完整的 APM 系统，Java 能力成熟，包含 Agent、Collector、存储方案和 Web 查询界面。OpenTelemetry 更偏向标准、采集和传输体系。

| 对比项 | OpenTelemetry | Pinpoint |
|---|---|---|
| 定位 | 可观测性标准、SDK、Agent 和采集协议 | 完整 APM 产品 |
| 数据范围 | Trace、Metric、Log | 侧重链路、Java/JVM 和应用性能 |
| 展示界面 | 需要搭配可观测性后端 | 自带 Pinpoint Web |
| 存储 | 自身不负责长期存储 | 配套 HBase、Pinot 等组件 |
| Java 自动埋点 | OpenTelemetry Java Agent | Pinpoint Java Agent |
| 跨语言 | 多语言生态更统一 | Java 能力最成熟 |
| 数据开放性 | OTLP、W3C Trace Context 等开放标准 | 主要围绕 Pinpoint 数据链路 |
| 后端选择 | 可以选择或更换多个后端 | 与 Pinpoint 平台结合较紧 |

选择建议：

- 公司已有 Pinpoint 平台，并且系统以 Java 为主，可以直接使用 Pinpoint；
- 系统包含多种语言，或者希望以后自由更换存储和展示平台，优先考虑 OpenTelemetry；
- 从零建设长期统一的可观测性体系，通常更适合先统一 OpenTelemetry 标准，再选择后端；
- 同一个应用一般不要同时启用两套完整自动埋点 Agent，以免产生重复 Span、上下文冲突和额外开销。

Pinpoint 正在增加 OTLP 相关能力，但不同版本、不同信号的支持范围可能不同。采用“OpenTelemetry Agent + Pinpoint 后端”前，需要按照实际 Pinpoint 版本验证 Trace、Metric 和 Log 的兼容范围。

## 9. Java 开发者需要掌握到什么程度

普通 Java 业务开发者的目标是：

> 能看懂链路、能利用链路排查问题，写代码时不会把链路弄断。

### 9.1 必须掌握

- 能解释 Trace、Span、Trace ID、Span ID 和父子关系；
- 能通过 Trace ID 关联不同服务的日志；
- 会在链路页面定位慢 Span 和异常 Span；
- 知道 HTTP、RPC、消息队列如何传递上下文；
- 知道 MDC 的作用、线程复用风险和清理方式；
- 知道异步线程、线程池和消息队列可能造成断链；
- 知道链路、日志和指标各自解决什么问题。

### 9.2 建议掌握

- 会使用团队已有的 Pinpoint、Jaeger、SkyWalking、Tempo 或其他平台；
- 理解自动埋点与手动埋点的边界；
- 理解采样，知道查不到 Trace 不代表请求一定没有发生；
- 能结合链路、日志、指标和代码定位性能问题；
- 知道如何给关键业务步骤补充 Span 和必要属性；
- 知道不能把密码、Token、身份证号等敏感信息写入 Span 属性或 Baggage。

### 9.3 平台或中间件开发者再深入

- OpenTelemetry API、SDK 和 Java Agent；
- W3C Trace Context、OTLP 和 Baggage；
- 字节码增强和 Agent 插件开发；
- Context Propagation；
- Collector Pipeline；
- 头部采样与尾部采样；
- Span Processor 和 Exporter；
- 链路存储、高基数和容量治理；
- 跨语言、跨协议上下文兼容。

普通业务开发者通常不需要自己实现 Collector、Agent 或 Trace ID 算法。

## 10. 线上排查方法

收到某个接口慢或失败的反馈后，可以按以下顺序排查：

```text
1. 获取时间、接口、用户或订单等检索条件
2. 从入口日志、响应头或监控平台找到 Trace ID
3. 打开完整调用链，确认总耗时和错误状态
4. 找到最慢或最早报错的 Span
5. 继续查看该 Span 的子调用、日志、SQL和异常
6. 对照服务指标判断是单次异常还是系统性问题
7. 回到代码、配置或外部依赖验证根因
8. 修复后再次观察链路、日志和指标
```

例如：

```text
订单接口耗时 3000ms
├── 订单服务本地处理：50ms
├── 数据库：100ms
└── 支付服务：2800ms
```

这能把排查重点转移到支付调用，但还需要进一步确认：

- 支付服务内部哪个 Span 最慢；
- 是否发生连接池等待；
- 是否调用外部接口超时；
- 是否存在重试；
- 同期 P95/P99 和错误率是否升高。

## 11. 常见误区

### 11.1 有 Trace ID 就等于有链路追踪

只有 Trace ID 时，可以关联日志，但看不到完整 Span 树、各步骤耗时和父子调用关系。

### 11.2 把 Trace ID 放进业务请求体

Trace ID 属于技术上下文，应该通过请求头、RPC 元数据或消息 Header 传递，避免污染业务协议。

### 11.3 只设置 MDC，不清理

Web 容器和线程池会复用线程。不清理 MDC，后一个请求可能带上前一个请求的 Trace ID。

### 11.4 只复制 MDC 就认为异步链路完整

MDC 只影响日志字段。完整链路还要传播当前 Span 和追踪上下文。

### 11.5 看到最慢服务就直接认定根因

慢服务可能正在等待数据库、连接池、锁或外部接口。要继续展开子 Span，并结合日志和指标确认。

### 11.6 所有请求都必须永久全量采集

全量采集会增加 CPU、网络、存储和查询成本。是否全量采集应根据流量、故障分析需求和成本决定。

### 11.7 在链路标签中放大量业务数据

高基数字段会增加索引和存储压力；敏感字段还会造成安全风险。只记录排查真正需要且符合数据规范的属性。

## 12. 面试回答模板

### 12.1 什么是链路追踪

> 链路追踪用于还原一次请求在分布式系统中的完整调用路径。它通过 Trace ID 标识整次请求，通过 Span 表示其中的 HTTP、RPC、SQL 或消息操作，并记录父子关系、耗时、状态和异常。它主要解决跨服务日志难关联、调用关系不清楚以及慢点和异常源头难定位的问题。

### 12.2 Trace ID 如何实现

> 简单场景可以使用 UUID 生成 Trace ID，在请求入口创建，通过 HTTP Header、RPC Metadata 或消息 Header 向下游传递，并放入 MDC 关联日志。但这只能完成基础日志关联。完整链路还需要 Span、上下文传播、采样、数据采集和查询平台，因此生产微服务通常使用 OpenTelemetry、Pinpoint 等成熟方案。

### 12.3 MDC 是什么

> MDC 是日志框架提供的线程级上下文，可以把 Trace ID 等公共字段自动附加到当前线程产生的日志。由于线程池会复用线程，请求结束时必须清理；异步切换线程时还需要显式传播上下文。MDC 负责日志关联，不等于完整链路追踪。

### 12.4 OpenTelemetry 与 Pinpoint 的区别

> OpenTelemetry 是厂商中立的可观测性标准和采集体系，需要搭配存储与展示后端；Pinpoint 是包含 Agent、Collector、存储方案和 Web 页面的完整 APM 产品。Java 为主且已有 Pinpoint 平台时可直接使用 Pinpoint；多语言或希望降低后端绑定时，更适合统一采用 OpenTelemetry。

## 13. 参考资料

- [OpenTelemetry：What is OpenTelemetry](https://opentelemetry.io/docs/what-is-opentelemetry/)
- [OpenTelemetry Java 文档](https://opentelemetry.io/docs/languages/java/)
- [OpenTelemetry Java Agent](https://opentelemetry.io/docs/zero-code/java/agent/)
- [W3C Trace Context](https://www.w3.org/TR/trace-context/)
- [Pinpoint 官方文档](https://pinpoint-apm.gitbook.io/pinpoint/)
- [SLF4J MDC API](https://www.slf4j.org/api/org/slf4j/MDC.html)
