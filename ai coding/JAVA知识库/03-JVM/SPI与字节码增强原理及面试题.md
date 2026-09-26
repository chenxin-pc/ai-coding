# SPI 与字节码增强：原理、使用与面试题

## 目录

- [1. 核心区别](#1-核心区别)
- [2. SPI 的原理与使用](#2-spi-的原理与使用)
- [3. Spring IoC 中的多个实现](#3-spring-ioc-中的多个实现)
- [4. 字节码增强的原理与使用](#4-字节码增强的原理与使用)
- [5. 动态代理与字节码增强](#5-动态代理与字节码增强)
- [6. SkyWalking、Pinpoint 与性能影响](#6-skywalkingpinpoint-与性能影响)
- [7. 面试题与回答要点](#7-面试题与回答要点)
- [8. 延伸阅读](#8-延伸阅读)

## 1. 核心区别

**SPI 负责发现和加载扩展实现；字节码增强负责生成或修改类的执行逻辑。** 两者可以配合使用，但互不依赖。

| 对比项 | SPI | 字节码增强 |
|---|---|---|
| 解决的问题 | 有哪些实现可以使用？ | 如何给类或方法增加行为？ |
| 核心机制 | 接口约定、提供者注册、发现与实例化 | 解析、修改或生成字节码 |
| 典型用途 | 插件扩展、组件替换 | APM 监控、方法追踪、代理、覆盖率统计 |
| 是否依赖 Spring | 不依赖 | 不依赖 |

## 2. SPI 的原理与使用

### 2.1 基本原理

SPI 的全称是 **Service Provider Interface，服务提供者接口**。它让框架依赖抽象接口，将具体实现交给外部扩展。

```text
框架定义接口
    → 第三方提供实现
    → 按约定注册实现
    → 框架发现并实例化
    → 通过接口调用
```

例如，序列化框架定义 `Serializer`，不同插件提供 `JsonSerializer`、`XmlSerializer`。框架无需写死具体实现类。

### 2.2 JDK 原生 SPI 如何使用

JDK 原生 SPI 主要使用 `ServiceLoader`。传统 classpath 方式需要在 JAR 中提供以下配置文件：

```text
META-INF/services/com.example.Serializer
```

文件名是接口的全限定名，文件内容是实现类的全限定名，每行一个：

```text
com.example.JsonSerializer
com.example.XmlSerializer
```

调用 `ServiceLoader.load(Serializer.class)` 获得加载器，再遍历取得实现对象。默认使用当前线程的上下文类加载器。

classpath 方式下，提供者类需要满足可访问性要求，并具有 `public` 无参构造器。Java 9 及以上的显式模块还支持通过 `uses`、`provides ... with ...` 声明服务关系，其提供者实例化规则与传统 classpath 方式有所不同。

来源：[ServiceLoader 官方文档](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/ServiceLoader.html)。

### 2.3 多个实现是否自动加载和调用

需要区分“发现、实例化、选择、调用”四个步骤。

| 操作 | 实际行为 |
|---|---|
| 只调用 `ServiceLoader.load()` | 不会立即实例化全部提供者，也不会调用业务方法 |
| 完整遍历加载器 | 在没有加载错误的情况下，逐步取得全部可发现的实现，并缓存实例 |
| 遍历时对每个对象调用业务方法 | 每个被遍历并调用的实现都会执行 |
| 找到符合条件的对象后调用并退出 | 只执行选中的实现 |

使用时要记住：

- **注册才能被发现**：仅实现接口，不会自动被原生 SPI 找到。
- **惰性加载**：`load()` 不立即实例化全部提供者，迭代时逐步加载并缓存。
- **发现不等于选择**：多个实现存在时，按配置、名称或 `supports()` 选择，需要调用方定义。
- **发现不等于调用**：遍历时主动调用每个实现的方法，才会全部执行。
- **第一个不等于默认实现**：不应将提供者发现顺序当作业务优先级。

来源：[ServiceLoader 加载与迭代行为](https://docs.oracle.com/javase/8/docs/api/java/util/ServiceLoader.html)。

### 2.4 适用场景

SPI 适合框架开放扩展点，例如协议处理、存储实现、序列化组件和插件加载。

如果业务需要“根据订单类型选择处理器”，SPI 可以发现处理器，但业务路由规则仍需自行实现。Spring、Dubbo 等框架可能提供额外的默认实现、名称选择或自动激活规则，这些属于框架能力。

## 3. Spring IoC 中的多个实现

假设 `MessageSender` 有两个实现，并且都通过组件扫描或显式配置注册为 Bean：

```text
Spring 容器
├── emailSender → EmailSender 实例
└── smsSender   → SmsSender 实例
```

### 3.1 创建对象与注入对象是两件事

默认 `ApplicationContext` 通常会在初始化期间创建非懒加载的单例 Bean。因此，这两个对象可以都已创建，但某个依赖注入点只引用其中一个。

`@Lazy`、作用域和条件注册等配置会影响行为。懒加载 Bean 如果是非懒加载单例的直接依赖，也可能因满足依赖而在启动时创建。

来源：[Spring Bean 初始化说明](https://docs.spring.io/spring-framework/reference/core/beans/dependencies/factory-lazy-init.html)。

### 3.2 单个依赖如何选择

| 常见情况 | 注入结果 |
|---|---|
| 只有一个符合条件的 Bean | 直接注入 |
| 多个候选，其中一个标记 `@Primary` | 优先注入该 Bean |
| 用 `@Qualifier` 明确限定 | 注入符合限定条件的 Bean；仍需能唯一确定 |
| 没有其他选择标记，字段名匹配某个 Bean 名 | 可以通过名称匹配确定 |
| 仍有多个候选，无法唯一确定 | 注入失败，通常导致启动失败 |

无法唯一确定依赖时会涉及 `NoUniqueBeanDefinitionException`。Spring 不会随便选第一个。以上是常见规则的概括，并非所有版本的完整候选解析算法。

**选择通常发生在依赖注入阶段。** 注入完成后，调用的是该引用指向的对象；Spring 不会根据每次调用的业务参数自动切换实现。使用自定义路由代理等机制属于额外设计。

来源：[Spring 注入选择规则](https://docs.spring.io/spring-framework/reference/core/beans/annotation-config/autowired-qualifiers.html)。

### 3.3 同时使用多个实现

注入 `List<MessageSender>` 可以获得所有符合条件的候选；注入 `Map<String, MessageSender>` 时，键是 Bean 名称，值是对应 Bean。

`@Primary` 用于单值依赖的优先选择，不会让集合注入只保留一个 Bean。得到集合后，执行哪些对象的方法，仍由业务代码决定。

仅在 `META-INF/services` 中注册的实现不会自动成为 Spring Bean。原生 `ServiceLoader` 创建的对象不会自动获得 Spring 依赖注入和 AOP，需要额外整合。

来源：[Spring 集合注入说明](https://docs.spring.io/spring-framework/reference/core/beans/annotation-config/autowired.html)。

## 4. 字节码增强的原理与使用

### 4.1 Class 文件是可解析的数据结构

Java 编译后的 Class 格式包含常量池、字段表、方法表等结构。普通 Java 方法的指令主要位于方法的 `Code` 属性中。

```text
Class 文件
├── 类名、父类、接口信息
├── 常量池
├── 字段表
└── 方法表
    └── Code 属性
        ├── 字节码指令
        ├── 最大操作数栈深度
        ├── 局部变量槽位数量
        └── 异常处理表等
```

增强工具解析这些结构，找到目标方法，插入指令，再维护跳转、异常处理表、栈映射帧等信息，输出合法的新字节码。

来源：[JVM Class 文件规范](https://docs.oracle.com/javase/specs/jvms/se21/html/jvms-4.html)。

### 4.2 增加耗时统计的例子

```text
增强前：
执行业务 → 返回

增强后：
记录开始时间
    → 执行业务
    → 正常结束或异常退出时记录耗时
    → 保留原返回值或继续抛出异常
```

实际改写涉及插入计时调用、分配局部变量槽位、处理正常返回和异常出口等。要保持原业务语义，还需要正确保留返回值、异常和控制流，并处理监控逻辑自身的异常。

### 4.3 加载期增强链路

```text
Java Agent 启动
    → 获取 Instrumentation
    → 注册 ClassFileTransformer
    → JVM 加载目标类时提供原始 byte[]
    → ASM / Byte Buddy 等工具修改字节码
    → 返回新的 byte[]
    → JVM 校验并使用增强后的类定义
```

| 组件 | 职责 |
|---|---|
| Java Agent | 接入 JVM 的入口 |
| `Instrumentation` | 提供注册转换器、重转换类等能力 |
| `ClassFileTransformer` | 接收类字节，返回转换后的类字节 |
| ASM、Byte Buddy 等 | 解析、生成或修改字节码内容 |

转换可以在内存中完成，不必修改磁盘上的原始 `.class` 文件。转换器不能直接修改传入的原始字节数组；无需转换时可以返回 `null`，需要转换时返回新的字节数组。

**方法调用时不会每次重新增强。** 转换发生在类加载、重转换或重定义等时机，后续方法调用执行已经包含增强逻辑的方法实现。

来源：[ClassFileTransformer 官方说明](https://docs.oracle.com/en/java/javase/21/docs/api/java.instrument/java/lang/instrument/ClassFileTransformer.html)。

### 4.4 增强时机与工具

| 时机 | 使用方式 |
|---|---|
| 构建期间 | 部署前处理编译生成的 Class 文件 |
| 类加载期间 | 类定义之前转换字节码，常见于启动时挂载 Agent |
| 类已加载后 | 使用重转换或重定义能力更新类定义，受 JVM 限制 |

常见工具：

- [ASM](https://asm.ow2.io/)：提供较底层的类结构和字节码操作。
- [Javassist](https://www.javassist.org/)：支持类似源码的修改方式，也提供字节码级 API。
- [Byte Buddy](https://bytebuddy.net/partial/tutorial.partial.html)：提供较高层的类型匹配、方法拦截和 Agent API。

使用 `-javaagent` 启动时，JVM 在应用 `main` 前调用 Agent 的 `premain`。动态加载 Agent 通常使用 `agentmain`，是否允许取决于 JVM 版本和配置。

来源：[Java Agent 启动机制](https://docs.oracle.com/en/java/javase/21/docs/api/java.instrument/java/lang/instrument/package-summary.html)。

### 4.5 已加载类的更新与限制

- `retransformClasses()`：基于 JVM 保留的初始类字节，按规则重新转换；不支持重转换的转换器复用此前结果，支持重转换的转换器重新执行。
- `redefineClasses()`：接收调用方提供的新类字节来更新类定义；该过程也可以触发转换器。
- 标准 JVM 通常允许修改方法体，但限制增删字段、方法和改变继承关系等结构变更，不能任意替换类。
- 能否更新还取决于 JVM 能力、Agent 能力声明和目标类是否可修改。
- 成功更新方法体后，已有对象后续调用也能使用新实现，通常不需要重新创建对象。
- 已经执行中的方法栈帧继续运行旧版本；类初始化器不会因此重新执行。

来源：[Instrumentation 官方文档](https://docs.oracle.com/javase/8/docs/api/java/lang/instrument/Instrumentation.html)。

## 5. 动态代理与字节码增强

动态代理强调通过代理对象拦截调用；字节码技术可以用于生成代理类，也可以直接修改目标类的方法。

| 方式 | 生效路径 |
|---|---|
| Spring AOP 代理 | 调用者 → 代理对象 → 拦截逻辑 → 目标对象 |
| 直接织入目标方法 | 调用者 → 已经包含增强指令的目标方法 |

Spring AOP 使用 JDK 动态代理或 CGLIB 代理。CGLIB 生成子类，并不等于直接改写了目标类的方法体。

Spring 代理模式下，目标对象内部的 `this.xxx()` 调用会绕过代理。直接织入目标方法字节码的方式没有同样的代理绕过问题。

如果增强直接织入目标类的方法，那么该类实例执行对应方法时即可运行增强逻辑，不要求实例必须由 Spring 创建。

来源：[Spring 代理与字节码织入说明](https://docs.spring.io/spring-framework/reference/core/aop/proxying.html)。

## 6. SkyWalking、Pinpoint 与性能影响

### 6.1 如何使用增强

| Agent | 如何接入方法执行 |
|---|---|
| SkyWalking | 使用 Byte Buddy，按插件规则匹配类和方法，织入拦截逻辑 |
| Pinpoint | 在类加载时向目标方法注入拦截器，记录调用前后和异常等信息 |

它们通常按插件规则增强特定类和方法，不会默认把所有业务方法都增强一遍。

一个监控框架可以把 SPI 与增强结合起来：

```text
SPI 发现增强插件
    → 插件声明目标类和目标方法
    → Agent 注册转换逻辑
    → 类加载时织入监控指令
    → 业务调用时采集链路数据
```

其中 SPI 负责发现插件，字节码增强负责改变方法执行行为。Pinpoint 的插件开发文档描述了通过 `ServiceLoader` 加载 `ProfilerPlugin` 的机制；不同框架的插件发现协议应以各自实现为准。

来源：[SkyWalking 字节码处理说明](https://skywalking.apache.org/docs/main/v9.4.0/en/faq/compatible-with-other-javaagent-bytecode-processing/)、[Pinpoint 技术说明](https://github.com/pinpoint-apm/pinpoint-apm.github.io/blob/main/want-a-quick-tour/techdetail.md)、[Pinpoint 插件开发说明](https://github.com/pinpoint-apm/pinpoint-apm.github.io/blob/main/documents/plugin-dev-guide.md)。

### 6.2 启动开销与运行开销

| 阶段 | 额外工作 |
|---|---|
| 启动或类加载阶段 | Agent 初始化、插件加载、规则匹配、字节码解析与转换、辅助类加载 |
| 运行阶段 | 计时、上下文维护、异常和参数采集、链路数据处理与上报 |

类如果直到第一次请求才加载，其增强成本可能体现在首次请求的延迟中。

采集越细，产生的数据和运行开销通常越多。SkyWalking 官方明确说明，部分缓存、Sentinel 等插件会产生大量 Span，具有潜在性能影响，因此被放在可选插件中。

来源：[SkyWalking 可选插件说明](https://skywalking.apache.org/docs/skywalking-java/v9.1.0/en/setup/service-agent/java-agent/optional-plugins/)。

### 6.3 调整方式的差异

| 调整方式 | 主要影响 |
|---|---|
| 减少插件、缩小增强范围 | 有机会同时降低启动和运行开销 |
| 降低链路采样量 | 主要降低运行期间的数据采集和上报开销 |
| 减少参数、SQL 参数等详细采集 | 主要降低运行期间的处理量和数据量 |

**降低采样量，不等于启动时少增强同等比例的类。** 从增强与采样的阶段划分可以推知：方法通常仍需先被增强，运行时再判断是否采样，因此不能单靠降低采样量解决启动变慢。

具体配置应以所用 Agent 版本为准，不能把不同版本的参数和默认值混用。

来源：[SkyWalking 采样配置](https://skywalking.apache.org/docs/skywalking-java/v9.3.0/en/setup/service-agent/java-agent/configurations/)、[Pinpoint 采样配置](https://pinpoint-apm.gitbook.io/pinpoint/getting-started/installation)。

### 6.4 如何测量

用相同应用、JDK 和资源配置，分别测试“不挂 Agent”“挂 SkyWalking”“挂 Pinpoint”，让采集范围与采样策略尽量相当。

- 启动：从 Java 进程启动到应用真正就绪，多次测量。
- 运行：预热后，在相同负载下比较吞吐量、P95/P99 延迟、CPU、内存和 GC。
- 区分首次请求与稳定运行阶段，避免混淆类加载和稳态采集成本。
- 不直接给出固定损耗比例，也不在采集范围不一致时断言某个 Agent 更快。

不要只看 Spring Boot 日志里的 `Started ... in ... seconds`：Agent 的 `premain` 在应用 `main` 之前执行，这部分耗时可能不包含在 Spring Boot 自己的计时中。

## 7. 面试题与回答要点

| 面试题 | 回答要点 |
|---|---|
| 1. API 和 SPI 有什么区别？ | API 侧重向调用方提供功能；SPI 侧重定义扩展契约，由提供者实现，再由框架使用。 |
| 2. JDK SPI 的实现原理是什么？ | 按服务注册信息定位提供者，通过类加载器加载，在需要时实例化，由 `ServiceLoader` 管理发现过程。 |
| 3. SPI 有多个实现，会自动全部执行吗？ | 不会。完整遍历可以逐步取得全部可发现实现，但业务方法是否执行、执行哪些，由调用方决定。 |
| 4. `ServiceLoader.load()` 会立即创建全部对象吗？ | 不会。它采用惰性加载；迭代时逐步加载和实例化，并在该加载器内缓存。 |
| 5. SPI 找不到实现，排查什么？ | 配置文件路径和名称、实现类名称、打包时注册文件是否被覆盖、类加载器可见性，以及构造器和依赖是否满足要求。 |
| 6. SPI 实现会自动成为 Spring Bean 吗？ | 不会。原生 `ServiceLoader` 创建的对象不会自动纳入 Spring 的依赖注入和 AOP，需要额外整合。 |
| 7. Spring 中一个接口有多个 Bean，如何注入？ | 单值依赖按 `@Qualifier`、`@Primary`、名称匹配等规则确定；无法唯一确定则失败。集合或 Map 可以获得多个候选。 |
| 8. 字节码增强为什么不需要修改源码？ | 工具直接处理 JVM 使用的 Class 格式数据，将额外逻辑写入字节码，源码可以保持不变。 |
| 9. 动态代理与字节码增强有什么区别？ | 动态代理通过代理对象拦截调用；字节码技术可以用于生成代理，也可以直接改写目标方法。Spring AOP 自调用会绕过代理，直接织入方法体没有同样的问题。 |
| 10. `premain` 和 `agentmain` 有什么区别？ | `-javaagent` 启动时在应用 `main` 前调用 `premain`；动态加载 Agent 通常使用 `agentmain`，是否允许取决于 JVM 和配置。 |
| 11. 已加载的类可以任意修改吗？ | 不可以。标准 JVM 通常允许修改方法体，但限制增删字段、方法和改变继承关系等结构变更。 |
| 12. 重转换和重定义有什么区别？ | `retransformClasses()` 基于 JVM 保留的初始类字节按规则重新转换；`redefineClasses()` 接收调用方提供的新类字节来更新定义。 |
| 13. 增强后必须重新创建对象吗？ | 成功更新已有类的方法体后，已有对象后续调用也能使用新实现；正在执行的方法栈帧继续运行旧版本。 |
| 14. 每次调用方法都会重新增强吗？ | 不会。转换发生在类加载、重转换等时机；每次调用执行的是已经插入的增强逻辑。 |
| 15. Agent 对性能的影响怎么评估？ | 分别测启动耗时和预热后的吞吐量、P95/P99、CPU、内存及 GC。启动从进程启动计到应用就绪，覆盖 `premain` 阶段。 |
| 16. 降低采样量能解决 Agent 启动慢吗？ | 通常不能直接解决。采样主要影响运行阶段采集量，而启动时通常仍需执行类匹配和字节码转换。 |
| 17. SPI 和字节码增强如何配合？ | SPI 发现插件实现，插件定义增强范围和逻辑，Agent 在类加载等时机应用字节码转换。 |

## 8. 延伸阅读

- [Java 链路追踪](../09-可观测性/01-Java链路追踪.md)
- [JVM 线上问题排查思路](JVM问题排查思路.md)
- [Java 代理模式](../04-设计模式/Java代理模式.md)
- [Spring 核心原理与扩展点](../05-Spring/Spring核心原理与扩展点.md)
