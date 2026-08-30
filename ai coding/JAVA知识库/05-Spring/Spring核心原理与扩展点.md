# Spring 核心原理与扩展点

## 目录

- [1. Spring 是什么](#1-spring-是什么)
- [2. IoC 与 DI](#2-ioc-与-di)
  - [2.1 IoC](#21-ioc)
  - [2.2 DI](#22-di)
- [3. BeanDefinition 与 Bean](#3-beandefinition-与-bean)
- [4. BeanFactory 与 ApplicationContext](#4-beanfactory-与-applicationcontext)
  - [4.1 BeanFactory](#41-beanfactory)
  - [4.2 ApplicationContext](#42-applicationcontext)
- [5. Spring 容器启动流程](#5-spring-容器启动流程)
  - [5.1 准备环境：prepareRefresh](#51-准备环境preparerefresh)
  - [5.2 获取 BeanFactory：obtainFreshBeanFactory](#52-获取-beanfactoryobtainfreshbeanfactory)
  - [5.3 配置 BeanFactory：prepareBeanFactory](#53-配置-beanfactorypreparebeanfactory)
  - [5.4 子类扩展：postProcessBeanFactory](#54-子类扩展postprocessbeanfactory)
  - [5.5 执行 BeanFactoryPostProcessor](#55-执行-beanfactorypostprocessor)
  - [5.6 注册 BeanPostProcessor](#56-注册-beanpostprocessor)
  - [5.7 初始化消息组件](#57-初始化消息组件)
  - [5.8 初始化事件广播器](#58-初始化事件广播器)
  - [5.9 初始化特殊基础设施](#59-初始化特殊基础设施)
  - [5.10 注册监听器](#510-注册监听器)
  - [5.11 创建非懒加载单例 Bean](#511-创建非懒加载单例-bean)
  - [5.12 发布容器刷新完成事件](#512-发布容器刷新完成事件)
- [6. Bean 生命周期](#6-bean-生命周期)
- [7. Bean 作用域与线程安全](#7-bean-作用域与线程安全)
- [8. AOP 与动态代理](#8-aop-与动态代理)
  - [8.1 同类内部调用问题](#81-同类内部调用问题)
- [9. Spring 事务](#9-spring-事务)
- [10. 循环依赖与三级缓存](#10-循环依赖与三级缓存)
- [11. Spring 核心扩展点](#11-spring-核心扩展点)
  - [11.1 Environment 与配置绑定](#111-environment-与配置绑定)
  - [11.2 @Configuration 与 @Bean](#112-configuration-与-bean)
  - [11.3 Spring 事件](#113-spring-事件)
  - [11.4 自定义 AOP](#114-自定义-aop)
  - [11.5 生命周期扩展](#115-生命周期扩展)
  - [11.6 Converter 与 Validator](#116-converter-与-validator)
  - [11.7 BeanPostProcessor](#117-beanpostprocessor)
  - [11.8 BeanFactoryPostProcessor](#118-beanfactorypostprocessor)
  - [11.9 动态注册 BeanDefinition](#119-动态注册-beandefinition)
  - [11.10 FactoryBean](#1110-factorybean)
  - [11.11 Aware](#1111-aware)
- [12. 扩展点选择原则](#12-扩展点选择原则)
- [13. 高频面试问题](#13-高频面试问题)
  - [13.1 Spring 如何启动](#131-spring-如何启动)
  - [13.2 BeanFactoryPostProcessor 与 BeanPostProcessor 的区别](#132-beanfactorypostprocessor-与-beanpostprocessor-的区别)
  - [13.3 Spring AOP 为什么会失效](#133-spring-aop-为什么会失效)
  - [13.4 @Transactional 为什么会失效](#134-transactional-为什么会失效)
  - [13.5 Spring 单例 Bean 是否线程安全](#135-spring-单例-bean-是否线程安全)
  - [13.6 Spring 为什么使用三级缓存](#136-spring-为什么使用三级缓存)
- [14. 核心源码入口](#14-核心源码入口)
- [15. 总结](#15-总结)

## 1. Spring 是什么

Spring 的核心可以概括为一句话：

> Spring 是一个管理对象及其依赖关系的容器，并通过代理和扩展点，在不侵入业务代码的情况下加入事务、日志、缓存等通用能力。

Spring Framework 最核心的能力包括：

- IoC：统一创建和管理对象
- DI：为对象注入依赖
- AOP：通过代理增强方法调用
- 事务：通过 AOP 管理事务边界
- 事件：在应用内部发布和监听事件
- 扩展点：允许框架或应用介入容器启动和 Bean 创建过程

## 2. IoC 与 DI

### 2.1 IoC

IoC 全称为 Inversion of Control，即控制反转。

没有 Spring 时，对象通常自己创建依赖：

```java
class OrderService {
    private UserService userService = new UserService();
}
```

这种方式会导致：

- 对象之间强耦合
- 实现类难以替换
- 单元测试不方便
- 生命周期难以统一管理

使用 Spring 后，对象只声明自己需要什么：

```java
@Service
class OrderService {
    private final UserService userService;

    OrderService(UserService userService) {
        this.userService = userService;
    }
}
```

对象的创建权和依赖管理权被交给 Spring，这就是控制反转。

### 2.2 DI

DI 全称为 Dependency Injection，即依赖注入。

IoC 是设计思想，DI 是 Spring 实现 IoC 的主要手段。常见注入方式有：

- 构造器注入
- Setter 注入
- 字段注入

实际开发优先使用构造器注入，因为依赖关系明确、方便测试，也能较早发现构造器循环依赖。

依赖查找大致遵循：

```text
按类型查找候选 Bean
→ 只有一个，直接注入
→ 存在多个，结合 @Primary、@Qualifier、名称筛选
→ 仍无法唯一确定，抛出异常
```

## 3. BeanDefinition 与 Bean

Spring 不会在发现 `@Service` 后立刻创建对象，而是先把类解析成 BeanDefinition。

BeanDefinition 是 Bean 的创建说明书，主要记录：

```text
beanClass
scope
lazyInit
primary
dependsOn
constructorArguments
propertyValues
initMethod
destroyMethod
```

两者的区别：

```text
BeanDefinition：描述对象应该怎样创建
Bean：根据 BeanDefinition 创建出来的真实对象
```

BeanDefinition 的主要来源包括：

- XML 配置
- `@Component`、`@Service` 等组件扫描
- `@Configuration` 和 `@Bean`
- `@Import`
- `ImportSelector`
- `ImportBeanDefinitionRegistrar`
- `BeanDefinitionRegistryPostProcessor`

## 4. BeanFactory 与 ApplicationContext

### 4.1 BeanFactory

`BeanFactory` 是 Spring 最基础的 IoC 容器，负责：

- 注册 BeanDefinition
- 创建和获取 Bean
- 管理单例对象
- 解析依赖关系
- 完成自动装配

常用实现是：

```text
DefaultListableBeanFactory
```

### 4.2 ApplicationContext

`ApplicationContext` 在 BeanFactory 基础上增加了：

- 国际化
- 事件发布
- 资源加载
- Environment 和配置管理
- 自动发现并注册各种后处理器
- 生命周期管理

可以这样理解：

```text
BeanFactory：Bean 管理引擎
ApplicationContext：以 BeanFactory 为核心的完整应用容器
```

## 5. Spring 容器启动流程

Spring 容器启动的核心入口是：

```java
AbstractApplicationContext#refresh()
```

核心流程如下：

```java
prepareRefresh();
ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
prepareBeanFactory(beanFactory);
postProcessBeanFactory(beanFactory);

invokeBeanFactoryPostProcessors(beanFactory);
registerBeanPostProcessors(beanFactory);

initMessageSource();
initApplicationEventMulticaster();
onRefresh();
registerListeners();

finishBeanFactoryInitialization(beanFactory);
finishRefresh();
```

### 5.1 准备环境：prepareRefresh

主要工作：

- 记录启动时间
- 设置容器 active 状态
- 初始化 PropertySource
- 校验必须存在的配置
- 保存早期监听器
- 创建 earlyApplicationEvents，暂存事件系统初始化前产生的事件

核心组件是 `Environment`：

```text
Environment
├── Profiles：dev、test、prod 等环境
└── PropertySources：系统变量、JVM 参数、配置文件等属性来源
```

### 5.2 获取 BeanFactory：obtainFreshBeanFactory

内部逻辑可以概括为：

```java
refreshBeanFactory();
return getBeanFactory();
```

不同 ApplicationContext 的实现有所区别：

```text
XML 容器：refresh 时创建 BeanFactory 并读取 XML
注解容器：构造 Context 时已创建 BeanFactory，refresh 时继续解析配置类
```

### 5.3 配置 BeanFactory：prepareBeanFactory

这一步把一个通用 BeanFactory 改造成能够支撑 ApplicationContext 的 BeanFactory。

主要工作：

- 设置类加载器
- 设置 Spring EL 表达式解析器
- 注册资源属性编辑器
- 注册 `ApplicationContextAwareProcessor`
- 注册 ApplicationContext、Environment 等可解析依赖
- 注册 environment、systemProperties 等单例对象

`ApplicationContextAwareProcessor` 负责处理：

```text
ApplicationContextAware
EnvironmentAware
ResourceLoaderAware
ApplicationEventPublisherAware
```

### 5.4 子类扩展：postProcessBeanFactory

这是留给 ApplicationContext 子类的模板方法。

子类可以在这里：

- 注册 Web 作用域
- 增加 Servlet 相关依赖
- 添加特殊 BeanPostProcessor
- 调整自动装配规则

它不是 `BeanFactoryPostProcessor`，而是 ApplicationContext 子类可以重写的钩子方法。

### 5.5 执行 BeanFactoryPostProcessor

对应方法：

```java
invokeBeanFactoryPostProcessors(beanFactory);
```

`BeanFactoryPostProcessor` 操作的是 BeanDefinition，而不是 Bean 实例。

重要实现：

- `ConfigurationClassPostProcessor`：解析配置类、`@ComponentScan`、`@Import` 和 `@Bean`
- `PropertySourcesPlaceholderConfigurer`：处理 `${...}` 属性占位符
- 自定义处理器：批量调整 BeanDefinition

执行顺序大致为：

```text
BeanDefinitionRegistryPostProcessor
→ BeanFactoryPostProcessor
```

同一类型内部再按照：

```text
PriorityOrdered
→ Ordered
→ 无排序接口
```

### 5.6 注册 BeanPostProcessor

对应方法：

```java
registerBeanPostProcessors(beanFactory);
```

这一步会找到所有 BeanPostProcessor，提前创建并注册到 BeanFactory。

重要实现包括：

- `AutowiredAnnotationBeanPostProcessor`：处理 `@Autowired`、`@Value`
- `CommonAnnotationBeanPostProcessor`：处理 `@Resource`、`@PostConstruct`、`@PreDestroy`
- `ApplicationContextAwareProcessor`：处理各种 Aware 接口
- `AbstractAutoProxyCreator`：判断是否需要生成 AOP 代理

两类后处理器的区别：

```text
BeanFactoryPostProcessor：修改 BeanDefinition
BeanPostProcessor：处理 Bean 实例
```

### 5.7 初始化消息组件

对应方法：

```java
initMessageSource();
```

Spring 查找名称为 `messageSource` 的 Bean，用于：

- 国际化
- 错误信息管理
- 参数校验提示

### 5.8 初始化事件广播器

对应方法：

```java
initApplicationEventMulticaster();
```

事件体系中的主要角色：

```text
ApplicationEventPublisher：发布事件
ApplicationEvent：事件内容
ApplicationListener：监听事件
ApplicationEventMulticaster：将事件分发给监听器
```

没有自定义实现时，通常使用 `SimpleApplicationEventMulticaster`。

### 5.9 初始化特殊基础设施

对应方法：

```java
onRefresh();
```

这是模板方法，普通上下文可能不做处理，Web 上下文可以在此初始化 Web 相关基础设施。

### 5.10 注册监听器

对应方法：

```java
registerListeners();
```

主要处理：

- 通过 API 提前添加的监听器
- 容器中实现 `ApplicationListener` 的 Bean
- 重新发布早期暂存的事件

### 5.11 创建非懒加载单例 Bean

对应方法：

```java
finishBeanFactoryInitialization(beanFactory);
```

主要工作：

- 设置 ConversionService
- 添加占位符解析器
- 冻结 BeanDefinition 配置
- 调用 `preInstantiateSingletons()`
- 创建所有非懒加载单例 Bean

一次 `getBean()` 的核心路径：

```text
doGetBean
→ 查询单例缓存
→ 处理 dependsOn
→ createBean
→ doCreateBean
→ 实例化
→ 提前暴露引用
→ 属性填充
→ 初始化
→ BeanPostProcessor
→ 注册销毁逻辑
→ 放入单例池
```

### 5.12 发布容器刷新完成事件

对应方法：

```java
finishRefresh();
```

主要工作：

- 清理启动过程中的临时缓存
- 初始化 LifecycleProcessor
- 启动 Lifecycle、SmartLifecycle 组件
- 发布 `ContextRefreshedEvent`

在 Spring Boot 中，`ContextRefreshedEvent` 不完全等于应用已经可以对外服务。后面还可能执行 Runner，最终再发布 `ApplicationReadyEvent`。

## 6. Bean 生命周期

一个普通单例 Bean 的主要生命周期如下：

```text
实例化
→ 属性填充和依赖注入
→ Aware 接口回调
→ BeanPostProcessor 前置处理
→ @PostConstruct
→ InitializingBean#afterPropertiesSet
→ 自定义 init-method
→ BeanPostProcessor 后置处理
→ 可能生成 AOP 代理
→ Bean 可以使用
→ @PreDestroy、DisposableBean、destroy-method
```

三个容易混淆的概念：

```text
实例化：调用构造器，在内存中创建对象
属性填充：向对象注入依赖
初始化：依赖注入完成后，执行各种初始化回调
```

容器最终对外提供的不一定是原始对象。如果 Bean 匹配 AOP 切点，后处理阶段可能返回代理对象。

## 7. Bean 作用域与线程安全

常见作用域：

- `singleton`：一个 Spring 容器中通常只有一个实例，默认作用域
- `prototype`：每次获取时创建新实例
- `request`：一次 HTTP 请求一个实例
- `session`：一次 HTTP Session 一个实例

Spring 单例不等于线程安全。Spring 只保证对象通常只有一份，不保证多个线程同时操作该对象时没有竞争。

无状态 Service 通常是安全的：

```java
@Service
class OrderService {
    public int calculate(int price, int count) {
        return price * count;
    }
}
```

如果把请求用户、订单状态等数据放在成员变量中，就可能产生线程安全问题。通常应该让单例 Bean 保持无状态。

## 8. AOP 与动态代理

AOP 用于统一处理分散在多个业务方法中的公共逻辑，例如：

- 事务
- 日志
- 权限
- 监控
- 幂等
- 限流
- 分布式锁

核心调用链：

```text
调用方
→ 代理对象
→ 拦截器链
→ 事务、日志等增强逻辑
→ 目标对象
```

Spring 常用两种代理方式：

- JDK 动态代理：基于接口生成代理对象
- CGLIB：通过创建目标类的子类生成代理对象

CGLIB 无法正常增强 `final` 类、`final` 方法和 `private` 方法，因为这些方法不能被子类重写。

### 8.1 同类内部调用问题

```java
public void createOrder() {
    this.saveOrder();
}

@Transactional
public void saveOrder() {
}
```

`this.saveOrder()` 调用的是当前原始对象，没有经过外部代理对象，因此事务拦截器可能没有执行。

判断 AOP 是否生效的关键不是方法上有没有注解，而是：

> 这次方法调用是否经过 Spring 代理对象。

## 9. Spring 事务

`@Transactional` 本质上建立在 AOP 之上：

```text
代理对象
→ TransactionInterceptor
→ TransactionManager
→ 开启或加入事务
→ 执行业务方法
→ 提交或回滚
```

Spring 通常会把当前事务使用的数据库连接绑定到当前线程，使同一线程中的多个 DAO 操作共享事务上下文。

常见传播行为：

- `REQUIRED`：有事务就加入，没有就新建，默认值
- `REQUIRES_NEW`：暂停外层事务，创建独立事务
- `NESTED`：在当前事务中创建保存点
- `SUPPORTS`：有事务就加入，没有就非事务执行
- `MANDATORY`：必须在已有事务中执行
- `NOT_SUPPORTED`：暂停已有事务，以非事务方式执行
- `NEVER`：存在事务时抛出异常

常见事务失效原因：

- 对象不是 Spring Bean
- 同类内部通过 `this` 调用
- 方法不可被代理有效增强
- 异常被捕获后没有继续抛出
- 抛出受检异常但没有配置 `rollbackFor`
- 数据库或存储引擎不支持事务
- 开启了新线程，事务上下文没有自动传递

分析事务问题时，应围绕三个问题：

```text
调用是否经过代理
事务上下文是否存在
异常是否满足回滚规则
```

## 10. 循环依赖与三级缓存

假设单例 Bean A 依赖 B，B 又依赖 A：

```text
创建 A
→ A 需要 B
→ 创建 B
→ B 又需要 A
```

Spring 单例缓存可以简化理解为：

- 一级缓存 `singletonObjects`：完整单例 Bean
- 二级缓存 `earlySingletonObjects`：提前暴露的 Bean 引用
- 三级缓存 `singletonFactories`：能够生成早期引用的工厂

解决流程：

```text
实例化 A
→ 将 A 的早期引用工厂放入三级缓存
→ 给 A 注入 B，因此创建 B
→ B 需要 A，从三级缓存获得 A 的早期引用
→ B 创建完成并注入 A
→ A 创建完成
```

三级缓存中的工厂可以决定提前暴露原始对象还是与 AOP 相关的早期代理引用，从而尽量保持对象引用一致。

Spring 不能解决所有循环依赖：

- 构造器相互依赖通常不能解决
- prototype Bean 循环依赖通常不能解决
- 复杂代理和生命周期组合可能创建失败

循环依赖通常说明职责划分存在问题，应优先调整设计，而不是依赖容器解决。

## 11. Spring 核心扩展点

### 11.1 Environment 与配置绑定

适合：

- 多环境配置
- 第三方客户端参数
- 功能开关
- 超时和重试参数

优先使用结构化配置：

```java
@ConfigurationProperties(prefix = "payment")
public class PaymentProperties {
    private int timeout;
    private String endpoint;
}
```

### 11.2 @Configuration 与 @Bean

适合注册无法添加 `@Component` 的第三方对象：

```java
@Configuration
public class PaymentConfig {

    @Bean
    public PaymentClient paymentClient(PaymentProperties properties) {
        return new PaymentClient(
                properties.getEndpoint(),
                properties.getTimeout());
    }
}
```

典型对象包括 HTTP 客户端、Redis 客户端、对象存储客户端、线程池和第三方 SDK。

### 11.3 Spring 事件

适合单体应用内部业务解耦：

```java
public record OrderPaidEvent(Long orderId) {
}
```

```java
publisher.publishEvent(new OrderPaidEvent(orderId));
```

监听事件：

```java
@EventListener
public void addPoints(OrderPaidEvent event) {
}
```

事务提交后再处理：

```java
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void handle(OrderPaidEvent event) {
}
```

需要注意：

- 普通 Spring 事件默认通常同步执行
- 监听器异常可能影响发布方
- 跨服务可靠事件应使用消息队列、事务消息或本地消息表

### 11.4 自定义 AOP

适合日志、权限、幂等、限流、分布式锁等横切逻辑。

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface Idempotent {
    String key();
}
```

```java
@Aspect
@Component
public class IdempotentAspect {

    @Around("@annotation(idempotent)")
    public Object around(
            ProceedingJoinPoint point,
            Idempotent idempotent) throws Throwable {
        try {
            // 获取锁或校验幂等键
            return point.proceed();
        } finally {
            // 释放资源
        }
    }
}
```

如果逻辑只在一个方法中使用，直接写业务代码通常更清晰；只有跨多个模块的公共逻辑才适合抽成切面。

### 11.5 生命周期扩展

`@PostConstruct` 适合当前 Bean 自身的轻量初始化：

```java
@PostConstruct
public void init() {
    // 构建本地映射或校验配置
}
```

`SmartInitializingSingleton` 适合所有普通单例创建完成后执行：

```java
public class HandlerRegistry implements SmartInitializingSingleton {
    @Override
    public void afterSingletonsInstantiated() {
        // 收集并注册所有策略处理器
    }
}
```

`SmartLifecycle` 适合管理需要启动和停止的长期运行组件：

- 消息消费者
- 长连接客户端
- 后台任务
- 自定义服务器
- 需要优雅停机的资源

### 11.6 Converter 与 Validator

`Converter` 负责类型转换：

```java
@Component
public class StringToOrderStatusConverter
        implements Converter<String, OrderStatus> {

    @Override
    public OrderStatus convert(String source) {
        return OrderStatus.valueOf(source.toUpperCase());
    }
}
```

`Validator` 或 Bean Validation 负责数据合法性校验：

```java
public class CreateOrderRequest {
    @NotNull
    private Long productId;

    @Positive
    private Integer count;
}
```

### 11.7 BeanPostProcessor

它可以介入每一个 Bean 的创建过程，适合开发框架级能力：

- 自定义注解注入
- RPC 客户端代理
- 自动埋点
- SDK 统一增强
- 自动注册处理器

普通业务逻辑不建议直接使用，因为影响范围大、执行时机早、排查难度高。

### 11.8 BeanFactoryPostProcessor

适合：

- 批量修改 BeanDefinition
- 动态调整作用域或懒加载
- 处理自定义属性占位符
- 根据平台规则修改 Bean 创建方式

它修改的是对象创建规则，普通业务项目很少需要自行实现。

### 11.9 动态注册 BeanDefinition

常见扩展接口：

```text
BeanDefinitionRegistryPostProcessor
ImportBeanDefinitionRegistrar
ImportSelector
```

适合：

- 扫描接口并生成 RPC 代理
- 根据配置注册多个数据源
- 开发自定义 Starter
- 实现 `@EnableXxx`
- 批量生成客户端 Bean

MyBatis Mapper、Feign Client 等框架都使用了类似思想。

### 11.10 FactoryBean

`FactoryBean` 适合封装复杂对象的创建过程：

- 代理对象
- RPC 客户端
- MyBatis SqlSessionFactory
- 根据元数据动态生成的对象

获取方式：

```java
getBean("clientFactoryBean");   // 获取生产出来的对象
getBean("&clientFactoryBean");  // 获取 FactoryBean 自身
```

普通第三方对象优先使用 `@Bean`，只有创建逻辑需要框架化复用时才考虑 FactoryBean。

### 11.11 Aware

Aware 接口允许 Bean 获取容器相关能力，但业务代码应谨慎使用。

大量调用 `ApplicationContext#getBean()` 会隐藏依赖关系，退化为 Service Locator 模式。优先使用正常依赖注入，只在动态插件、框架适配或遗留代码中使用。

## 12. 扩展点选择原则

可以按照下面的顺序选择：

```text
普通依赖注入
→ @Configuration/@Bean
→ 事件或 AOP
→ 生命周期接口
→ FactoryBean
→ BeanPostProcessor
→ BeanFactoryPostProcessor
→ 动态 BeanDefinition 注册
```

越往后越接近 Spring 容器底层，能力越强，但影响范围和维护成本也越高。

选择扩展点前，先判断自己需要操作的是什么：

```text
配置和运行环境：Environment、ConfigurationProperties
对象创建规则：BeanDefinition、BeanFactoryPostProcessor
对象实例：BeanPostProcessor
方法调用：AOP
容器启动和停止：Lifecycle、事件
复杂对象生产：FactoryBean
```

## 13. 高频面试问题

### 13.1 Spring 如何启动

回答主线：

```text
准备 Environment
→ 获取 BeanFactory
→ 加载 BeanDefinition
→ 执行 BeanFactoryPostProcessor
→ 注册 BeanPostProcessor
→ 初始化事件等基础设施
→ 创建非懒加载单例 Bean
→ 发布 ContextRefreshedEvent
```

### 13.2 BeanFactoryPostProcessor 与 BeanPostProcessor 的区别

```text
BeanFactoryPostProcessor：普通 Bean 创建前修改 BeanDefinition
BeanPostProcessor：在 Bean 创建过程中处理或包装 Bean 实例
```

### 13.3 Spring AOP 为什么会失效

核心判断：调用有没有经过 Spring 代理对象。

常见原因：

- 同类内部调用
- 对象不是 Spring Bean
- 方法不可被代理增强
- 手动创建了对象
- 切点表达式没有匹配

### 13.4 @Transactional 为什么会失效

除了 AOP 代理问题，还需要检查：

- 异常是否被捕获
- 异常类型是否满足回滚规则
- 事务传播行为是否正确
- 是否切换了线程
- 数据库是否支持事务

### 13.5 Spring 单例 Bean 是否线程安全

Spring 只负责作用域和生命周期，不自动保证线程安全。无状态单例通常安全，有可变成员状态时需要重新设计或使用并发控制。

### 13.6 Spring 为什么使用三级缓存

用于解决部分单例属性循环依赖，并通过三级缓存中的工厂处理早期代理引用，尽量保证最终注入对象的一致性。

## 14. 核心源码入口

```text
AbstractApplicationContext#refresh
DefaultListableBeanFactory
ConfigurationClassPostProcessor
PostProcessorRegistrationDelegate
AbstractBeanFactory#doGetBean
AbstractAutowireCapableBeanFactory#createBean
AbstractAutowireCapableBeanFactory#doCreateBean
AbstractAutowireCapableBeanFactory#populateBean
AbstractAutowireCapableBeanFactory#initializeBean
DefaultSingletonBeanRegistry
AbstractAutoProxyCreator
TransactionInterceptor
```

阅读源码时建议始终围绕一条主线：

```text
BeanDefinition 从哪里来
→ Bean 在哪里创建
→ 依赖在哪里注入
→ BeanPostProcessor 在哪里执行
→ AOP 代理在哪里生成
→ 最终对象在哪里进入单例池
```

## 15. 总结

Spring 的核心主线是：

> Spring 启动时读取配置并注册 BeanDefinition，通过 BeanFactoryPostProcessor 修改定义；随后实例化 Bean、完成依赖注入和初始化，并通过 BeanPostProcessor 扩展生命周期。AOP 后处理器可以把符合条件的 Bean 包装成代理对象，事务建立在代理拦截器之上。对于部分单例属性循环依赖，Spring 通过三级缓存提前暴露引用。

真正掌握 Spring 的标准，不是记住所有接口，而是能从 `ApplicationContext#refresh()` 开始，完整解释 BeanDefinition、Bean 创建、依赖注入、后处理器、AOP 代理、事务和扩展点之间的关系。
