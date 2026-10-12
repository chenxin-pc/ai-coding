# Netty 核心知识点

面向刚开始学习 Netty 的 Java 开发者，汇总从 NIO、EchoServer 到线程模型、Pipeline、异步结果和内存管理的学习内容。

本文以 **Netty 4.1 的 NIO TCP 模型**为主要背景。默认配置、内部计数和内存分配策略可能随版本变化，使用时以项目实际依赖为准。代码片段用于说明已有示例，不构成完整项目。

前置知识：[Java 学习 NIO 指南](./Java学习NIO指南.md)。

## 目录

- [1. Netty 与 Java NIO](#1-netty-与-java-nio)
- [2. 整体架构和核心组件](#2-整体架构和核心组件)
- [3. EchoServer 请求处理流程](#3-echoserver-请求处理流程)
- [4. 两种 SocketChannel 的区别](#4-两种-socketchannel-的区别)
- [5. EventLoop、Selector 与任务队列](#5-eventloopselector-与任务队列)
- [6. 一个 Channel 为什么只绑定一个 EventLoop](#6-一个-channel-为什么只绑定一个-eventloop)
- [7. Selector 空转与 Netty 的恢复机制](#7-selector-空转与-netty-的恢复机制)
- [8. ChannelPipeline 和事件传播](#8-channelpipeline-和事件传播)
- [9. ChannelHandler 类型与选择](#9-channelhandler-类型与选择)
- [10. ChannelFuture 与异步结果](#10-channelfuture-与异步结果)
- [11. ByteBuf 与编解码注意事项](#11-bytebuf-与编解码注意事项)
- [12. 引用计数和内存池](#12-引用计数和内存池)
- [13. ByteBuf 零拷贝](#13-bytebuf-零拷贝)
- [14. TCP 粘包拆包与分帧](#14-tcp-粘包拆包与分帧)
- [15. 易错点与复习问题](#15-易错点与复习问题)

## 1. Netty 与 Java NIO

Java NIO 提供底层网络编程接口；Netty 是异步、事件驱动的网络框架，提供线程调度、通道管理、处理器链、编解码和缓冲区管理。

```text
业务：聊天、RPC、网关、自定义协议
                  ↓
Netty：Channel、EventLoop、Pipeline、ByteBuf
                  ↓
Java NIO：Channel、Selector、ByteBuffer
                  ↓
操作系统网络能力
```

上图针对 NIO 传输。Netty 还支持其他传输实现，不能把 Netty 完全等同于 Java NIO 的包装。

| 工作 | 直接使用 Java NIO | 使用 Netty |
|---|---|---|
| 监听连接与读写 | 编写 Selector 循环 | 由 EventLoop 驱动 |
| 线程与连接分配 | 自己实现 | EventLoopGroup 组织 |
| 字节存储 | ByteBuffer | ByteBuf |
| 消息边界 | 自己累计、判断、截取 | 配置解码器或实现自定义解码器 |
| 组织处理步骤 | 自己设计 | Pipeline 与 Handler |
| 写不完的数据 | 自己保存并等待可写 | 框架管理待发送数据 |
| 异步结果通知 | 自己维护 | ChannelFuture |

非阻塞描述操作是否必须等待；异步描述操作结果如何通知。Netty 可以基于 NIO 非阻塞操作提供异步接口。Netty 的价值在于成熟封装与较低开发成本，不能推导出任何场景都比手写 NIO 快。[官方入门指南](https://netty.io/wiki/user-guide-for-4.x.html)

## 2. 整体架构和核心组件

| 组件 | 职责 |
|---|---|
| ServerBootstrap / Bootstrap | 装配并启动服务端 / 客户端 |
| Channel | 表示网络通道，提供连接、写入、关闭等操作 |
| EventLoopGroup | 管理多个 EventLoop，为 Channel 分配执行者 |
| EventLoop | 处理注册通道的 I/O，以及提交的任务 |
| ChannelPipeline | 组织 Handler，传播事件和操作 |
| ChannelHandler | 完成一个具体处理步骤 |
| ChannelHandlerContext | 当前 Handler 所在节点的上下文，提供传播和通道操作 |
| ByteBuf | 存放和操作字节数据 |
| ChannelFuture | 观察一次异步操作的状态和结果 |
| ChannelPromise | 在 Future 的基础上，还允许设置成功或失败结果 |

典型 TCP 服务端的职责分工：

```text
客户端建立连接
      ↓
boss EventLoop 接受连接
      ↓
创建连接 Channel，注册给一个 worker EventLoop
      ↓
worker 处理读写，驱动 Pipeline
      ↓
解码 → 业务处理 → 编码 → 返回响应
```

boss 和 worker 是常见角色名称，不是固定类名。worker 在这里是 I/O 事件循环，不等于另设的业务线程池。常见客户端只需要负责连接与读写的 EventLoopGroup。[官方服务端示例](https://netty.io/wiki/user-guide-for-4.x.html)

## 3. EchoServer 请求处理流程

本次讨论的示例监听 `9090`，收到字符串后回复 `echo: ` 加原消息。

| 示例中的配置或调用 | 含义 |
|---|---|
| `new NioEventLoopGroup(1)` | boss 组包含一个 EventLoop |
| `new NioEventLoopGroup()` | worker 使用相应版本和配置的默认数量 |
| `.group(boss, worker)` | 分别指定监听通道与连接子通道的执行组 |
| `.channel(NioServerSocketChannel.class)` | 选择 NIO 服务端监听通道 |
| `.childHandler(...)` | 配置新接受连接的处理器 |
| `initChannel(SocketChannel ch)` | 为这个新连接初始化 Pipeline |
| `.bind(9090)` | 发起监听端口绑定 |
| 第一个 `.sync()` | 等待绑定完成，失败时抛出失败原因 |
| `.channel().closeFuture().sync()` | 等待服务端监听通道关闭 |
| `shutdownGracefully()` | 发起线程组优雅关闭 |

`initChannel()` 是每个连接初始化时执行，不是每条消息到来时执行。每个连接有自己的 Pipeline。

```text
接收：网络 → ByteBuf → StringDecoder → String → channelRead0()
发送：channelRead0() → StringEncoder → ByteBuf → 当前客户端
```

已有业务调用 `ctx.writeAndFlush("echo: " + msg)` 只回复当前连接，不会广播。`exceptionCaught()` 中的 `ctx.close()` 也只关闭当前连接。

主线程等待 `closeFuture()` 不会阻止 worker 工作。`shutdownGracefully()` 本身是异步操作；`finally` 执行时才会发起关闭，不能认为任意进程终止方式都会执行该代码。

原示例只是文本回显演示，只有 StringDecoder，没有分帧器。要按完整的一行处理消息，应采用 UTF-8，并让两端约定换行结束，接收链改为“按行分帧 → 字符串解码 → 业务处理”。[StringDecoder 用法](https://netty.io/4.1/api/io/netty/handler/codec/string/StringDecoder.html)

## 4. 两种 SocketChannel 的区别

| 对比 | NioServerSocketChannel | NioSocketChannel |
|---|---|---|
| 表示什么 | TCP 监听端点 | 一条 TCP 连接的本地端点 |
| 职责 | 监听端口、接受连接 | 实际收发业务字节 |
| 使用位置 | 服务端 | 服务端和客户端 |
| 底层 JDK 通道 | ServerSocketChannel | SocketChannel |
| 服务端典型归属 | boss EventLoop | worker EventLoop |

```text
一个 NioServerSocketChannel 监听 9090
         ├── 接受连接后创建 NioSocketChannel A ↔ 客户端 A
         ├── 接受连接后创建 NioSocketChannel B ↔ 客户端 B
         └── 接受连接后创建 NioSocketChannel C ↔ 客户端 C
```

NioSocketChannel 并非“客户端专用”。在该服务端模型中，Netty 接受连接后自动创建它，并传入 `initChannel()`。

`ServerBootstrap.handler(...)` 面向监听通道，`childHandler(...)` 面向连接子通道；`option(...)` 与 `childOption(...)` 也遵循这个区分。

关闭一个连接子通道只影响对应连接；关闭监听通道停止接入新连接，不会仅因父通道关闭就自动关闭所有已有连接。[监听通道](https://netty.io/4.1/api/io/netty/channel/socket/nio/NioServerSocketChannel.html)、[连接通道](https://netty.io/4.1/api/io/netty/channel/socket/nio/NioSocketChannel.html)

## 5. EventLoop、Selector 与任务队列

在 NioEventLoop 中，Selector 与任务队列是两种工作来源，由同一个事件循环线程处理。其他 EventLoop 实现不一定使用 Java Selector。

```text
网络通道 → Selector：发现就绪状态 ─┐
                                ├→ EventLoop 线程
程序提交 → 任务队列：保存待执行代码 ─┘
```

| 对比 | Selector 的 I/O 事件 | 任务队列 |
|---|---|---|
| 来源 | 通道底层就绪状态 | 应用或 Netty 提交的任务 |
| 含义 | 某个通道可以进行某种 I/O 操作 | 请执行一段代码 |
| 载体 | SelectionKey 和就绪标记 | Runnable 等任务 |
| 示例 | 收到数据，发现读就绪 | 其他线程请求向通道写入 |

四种常见就绪事件：

- `OP_ACCEPT`：可以接受新连接。
- `OP_CONNECT`：非阻塞建连可以继续完成。
- `OP_READ`：读取就绪，执行读取也可能发现连接结束或错误。
- `OP_WRITE`：可以尝试继续写入，不表示“收到一条发送请求”。

Selector 提供就绪信息，不保存完整业务消息，也不代替实际读写。[Java Selector 文档](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/nio/channels/Selector.html)

默认情况下，网络读取可以直接驱动 Pipeline 和业务回调，不必把每条入站消息先包装成 Runnable。跨线程发起 Channel 操作时，需要切换执行线程的部分会调度到对应执行器。

任务队列中的任务也可以执行网络操作；“I/O 事件”和“任务”的区别在于触发来源，不是“网络”与“业务”的划分。

简化循环：等待或检查 I/O → 处理就绪事件 → 执行任务与到期定时任务 → 下一轮。定时任务按截止时间另外管理。有任务时应避免一直等待 I/O；外部提交任务时，必要的 `wakeup()` 使事件线程结束等待。

把慢 SQL 放进同一个 `eventLoop.execute(...)` 并没有换成业务线程，仍会阻塞该 EventLoop。默认 Handler 中的慢调用同样会拖慢它管理的其他连接。[事件循环接口](https://netty.io/4.1/api/io/netty/channel/EventLoop.html)

## 6. 一个 Channel 为什么只绑定一个 EventLoop

准确含义是：同一时刻，一个已注册 Channel 归属一个 EventLoop；一个 EventLoop 可以管理多个 Channel。

这样设计便于串行管理连接状态、解码累计数据、发送缓冲和关闭流程，减少锁竞争与线程协调。如果多个线程同时处理同一条连接，就必须额外协调这些共享状态。

```text
EventLoop 1 → Channel A、B
EventLoop 2 → Channel C、D
```

需要区分四个边界：

1. 外部线程可以调用 Channel 的公开操作，Netty 会安排相关操作的执行归属。
2. Handler 可以配置独立执行器；共享 Handler 或共享业务对象仍需考虑线程安全。
3. 业务线程池并行执行后，响应完成顺序可能改变；多个外部线程并发提交也没有预先约定的业务顺序。
4. 支持重新注册的通道可在完成注销、协调在途操作后注册到兼容 EventLoop，通常不需要业务主动迁移。

Netty 拒绝对已注册通道的重复注册，并有重新注册处理逻辑。[注册源码](https://netty.io/4.1/xref/io/netty/channel/AbstractChannel.html#464)

这不是 Java NIO 的硬限制：JDK 的 SelectableChannel 可以注册到多个 Selector。[JDK 文档](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/nio/channels/SelectableChannel.html)

## 7. Selector 空转与 Netty 的恢复机制

正常情况下，没有事件时 `select()` 可以阻塞等待，不会一直消耗 CPU。非阻塞 Channel 与可以阻塞的 Selector 并不矛盾。

异常空转是本应等待，却持续立即返回且没有有效工作，导致循环高速运行，可能占满一个 CPU 核心。

返回 0 本身不证明异常：超时、主动唤醒、中断，以及本来就不等待的 `selectNow()` 都要区分。持续监听不需要的可写事件，也可能因真实就绪通知而忙循环。

Netty 4.1 的 NioEventLoop 实现会结合是否有就绪事件、是否执行任务和线程中断等情况统计连续无有效工作的循环。达到配置阈值后重建 Selector；本次核对的实现默认阈值为 **512**，配置项为 `io.netty.selectorAutoRebuildThreshold`。不是进程累计 512 次返回 0 就重建。

```text
创建新 Selector
   ↓
迁移有效通道，保留关注事件与关联对象
   ↓
更新 SelectionKey，替换 Selector
   ↓
关闭旧 Selector，继续处理
```

迁移成功的连接通常无需重新建立 TCP 连接；迁移失败的个别通道可能被关闭。这是异常状态恢复措施，不能修复业务死循环或错误的忙轮询逻辑。[NioEventLoop 源码](https://netty.io/4.1/xref/io/netty/channel/nio/NioEventLoop.html)

## 8. ChannelPipeline 和事件传播

每个 Channel 自带一条 Pipeline，按顺序组织 Handler。Pipeline 决定处理路径，EventLoop 负责执行；Pipeline 不是线程池或任务队列。

以原 EchoServer 为例：

```text
排列：Head → StringDecoder → StringEncoder → BusinessHandler → Tail

入站：网络 → StringDecoder → BusinessHandler
出站：BusinessHandler → StringEncoder → 网络
```

入站向尾部传播，出站向头部传播，跳过方向不匹配的 Handler。连接事件、异常、用户事件也可以传播，不只有业务数据。

| 调用 | 传播方式 |
|---|---|
| `ctx.fireChannelRead(msg)` | 从当前上下文继续寻找后续入站 Handler |
| `ctx.write(msg)` | 从当前上下文向前寻找出站 Handler |
| `channel.write(msg)` | 从 Pipeline 尾部开始寻找出站 Handler |

Handler 可以转换、消费消息或继续传播；重写回调后，不应假定事件会自动走完整条链。入站 Handler 可以发起写操作，出站 Handler 才负责拦截出站操作。[Pipeline 文档](https://netty.io/4.1/api/io/netty/channel/ChannelPipeline.html)

## 9. ChannelHandler 类型与选择

| 类型 | 常用基类 | 典型职责 |
|---|---|---|
| 入站 | ChannelInboundHandlerAdapter | 通用连接事件、消息转发与生命周期管理 |
| 入站 | SimpleChannelInboundHandler<T> | 消费匹配类型的业务消息 |
| 出站 | ChannelOutboundHandlerAdapter | 拦截 write、flush、connect、close 等操作 |
| 双向 | ChannelDuplexHandler | 同时处理入站和出站 |

核心接口是 ChannelInboundHandler 与 ChannelOutboundHandler，双向处理器同时具备两者能力。[Handler 分类](https://netty.io/4.1/api/io/netty/channel/ChannelHandler.html)

| 对比 | ChannelInboundHandlerAdapter | SimpleChannelInboundHandler<T> |
|---|---|---|
| 消息回调 | channelRead(ctx, Object msg) | channelRead0(ctx, T msg) |
| 类型匹配 | 自己处理 | 自动匹配泛型类型 |
| 自动释放 | 不自动释放消费的消息 | 默认尝试释放已处理的引用计数消息 |

普通 String 无须引用计数释放；ByteBuf、FullHttpRequest 等需要明确所有权。使用 SimpleChannelInboundHandler 时，继续转发或异步保存引用计数消息，要正确 retain/release。[SimpleChannelInboundHandler 文档](https://netty.io/4.1/api/io/netty/channel/SimpleChannelInboundHandler.html)

常见回调：`channelActive()`、`channelInactive()`、`channelRead()` / `channelRead0()`、`userEventTriggered()`、`exceptionCaught()`。

| 功能 | 常用 Handler |
|---|---|
| 字符串转换 | StringDecoder、StringEncoder |
| 按行、分隔符、固定长度分帧 | LineBasedFrameDecoder、DelimiterBasedFrameDecoder、FixedLengthFrameDecoder |
| 按长度字段分帧 | LengthFieldBasedFrameDecoder |
| 自定义字节解码 | ByteToMessageDecoder |
| 自定义对象编码 | MessageToByteEncoder<T> |
| 收发调试 | LoggingHandler |
| 空闲检测 | IdleStateHandler |
| TLS | SslHandler |
| HTTP 编解码与聚合 | HttpServerCodec、HttpObjectAggregator |
| 初始化 Pipeline | ChannelInitializer |

IdleStateHandler 触发空闲事件，不会自动发送心跳；业务要自行处理事件。ChannelInitializer 完成初始化后会从 Pipeline 移除自身。[空闲检测文档](https://netty.io/4.1/api/io/netty/handler/timeout/IdleStateHandler.html)

## 10. ChannelFuture 与异步结果

ChannelFuture 表示一次 Channel 异步操作的完成状态。发起操作返回时，操作可能已经完成，也可能尚未完成。它是 `Future<Void>`，不保存对端业务响应。

| 方法 | 含义 |
|---|---|
| addListener | 注册完成回调，不阻塞等待 |
| isDone | 已完成，包括成功、失败和取消 |
| isSuccess | 已成功完成 |
| cause | 失败原因，未完成时可能为 null |
| channel | 操作关联通道 |
| sync | 等待完成，操作失败时重新抛出原因 |
| await | 等待完成，不主动重新抛出操作失败原因 |
| await(timeout, unit) | 有限等待，返回是否已完成 |

sync 和 await 都可能因线程中断而抛出 InterruptedException。操作完成后再注册监听器也会收到通知；回调可能很快运行，不能假定一定晚于注册之后的其他代码。[Future 文档](https://netty.io/4.1/api/io/netty/util/concurrent/Future.html)

### 10.1 EchoServer 中的用法

已有发送调用可获得本次写入的 Future：

```java
ChannelFuture future = ctx.writeAndFlush("echo: " + msg);
```

结果处理顺序：注册监听器 → 成功时记录完成 → 取消时按策略处理 → 失败时读取 cause 并决定是否关闭连接。调试与错误日志使用 SLF4J，不输出不必要的原始用户数据。

已有启动链路：

```java
.bind(9090).sync().channel().closeFuture().sync();
```

第一个 Future 对应绑定，第二个对应通道关闭通知。`closeFuture()` 不主动关闭通道；`close()` 才发起关闭。[ChannelFuture 文档](https://netty.io/4.1/api/io/netty/channel/ChannelFuture.html)

### 10.2 必须注意的边界

- 不在 EventLoop 中阻塞等待未完成 Future，否则它可能无法继续完成所等操作；Netty 通常通过 BlockingOperationException 阻止这种等待。main 中等待绑定则是常见用法。
- `isSuccess() == false` 可能仅表示未完成。判断失败应在完成回调中进行，或先确认 isDone，并区分取消。
- `await(3, SECONDS)` 超时只表示“不再等待”，不等于底层操作失败或自动取消。连接超时使用 CONNECT_TIMEOUT_MILLIS，业务响应超时单独设计。
- 回调通常在相关 I/O 线程运行，不能在监听器里执行慢 SQL 或阻塞式远程调用。[监听器文档](https://netty.io/4.1/api/io/netty/channel/ChannelFutureListener.html)
- `write()` 不主动 flush；没有刷新时，写 Future 可能迟迟不完成。`writeAndFlush()` 合并了两步。[写入接口](https://netty.io/4.1/api/io/netty/channel/ChannelOutboundInvoker.html)
- 本地写入成功不等于对端收到、解析完成或业务成功。请求响应协议需要在入站路径接收业务响应，通常使用请求 ID 关联。

## 11. ByteBuf 与编解码注意事项

ByteBuf 是字节容器，通常位于网络读写与对象编解码之间。独立读写索引使读、写进度可以分别管理。

```text
0          readerIndex           writerIndex          capacity
| 已读区域 |       可读区域        |      可写区域       |
```

写入 10 字节再读取 4 字节，可读数据为 6 字节。`capacity()` 是容量，不是消息长度；`clear()` 只重置索引，不擦除内容，也不释放内存。

| 操作 | 对索引的影响 |
|---|---|
| read / skip | 推进 readerIndex |
| write | 推进 writerIndex |
| get | 按位置读取，不推进索引 |
| set | 按位置修改，不推进索引 |

不能假定所有 ByteBuf 都有可直接访问的 byte[]；直接内存或组合缓冲区不一定支持 array()。[ByteBuf 文档](https://netty.io/4.1/api/io/netty/buffer/ByteBuf.html)

编解码时检查：

1. **字符集**：两端明确 UTF-8 等编码，不依赖系统默认值。
2. **字节长度**：`"你好"` 的 Java length 为 2，UTF-8 编码后为 6 字节；长度字段通常按协议中的字节长度计算。
3. **消息格式**：字段顺序、字段宽度、大小端、长度是否包含消息头必须一致。
4. **完整性**：先判断字节是否足够，不足则保留数据；不要错误推进索引。
5. **上限**：校验消息长度，限制最大帧大小，避免无限累计。
6. **输出所有权**：MessageToByteEncoder 提供的 out 由框架管理，正常编码只向它写入，不自行 release。[编码器文档](https://netty.io/4.1/api/io/netty/handler/codec/MessageToByteEncoder.html)

## 12. 引用计数和内存池

### 12.1 引用计数

普通新分配 ByteBuf 初始引用计数为 1。retain 增加计数，release 减少计数，降到 0 后释放或回收资源。不能因为 Java 对象还有引用，就认为底层内存仍可使用。

| 使用方式 | 生命周期责任 |
|---|---|
| 自己消费入站消息 | 自己释放，或由自动释放 Handler 负责 |
| 继续传给下游 | 转交责任，不立即释放已转交的那份引用 |
| 交给标准出站发送流程 | 通常由框架在写操作结束后释放 |
| 异步持有 | 保留必要引用，最终释放，并处理任务提交失败等路径 |

漏释放可能造成内存泄漏；提前释放或重复释放可能触发 IllegalReferenceCountException。retain 延长生命周期，不保证内容或索引的并发安全。框架自动释放与手动释放不能重复承担同一份引用。[引用计数指南](https://netty.io/wiki/reference-counted-objects.html)

### 12.2 池化

池化复用内存区域，减少频繁底层分配和释放：

```text
内存池 → 分配区域 → 使用 → 归还 → 后续请求复用
```

PooledByteBufAllocator 使用内存区域管理与缓存降低成本，代价是池保留内存及管理开销。release 后进程内存没有立即下降，可能只是内存保留在池内，并不必然是泄漏；也不能仅凭这一点排除泄漏。[池化分配器](https://netty.io/4.1/api/io/netty/buffer/PooledByteBufAllocator.html)

两个维度不能混淆：

| 维度 | 分类 |
|---|---|
| 是否复用 | 池化 / 非池化 |
| 存储位置 | 堆内存 / 直接内存 |

两类存储都可以池化。需要缓冲区时，可通过 `ctx.alloc()` 使用当前配置的分配器，不要假定所有运行环境的默认分配策略相同。[ByteBufAllocator](https://netty.io/4.1/api/io/netty/buffer/ByteBufAllocator.html)

## 13. ByteBuf 零拷贝

这里的零拷贝主要指共享或组合已有存储，省去应用层不必要的字节复制，不保证整个网络链路没有拷贝。

| 方式 | 原理 | 注意事项 |
|---|---|---|
| slice / duplicate | 创建共享底层数据的视图 | 索引独立，内容共享 |
| CompositeByteBuf | 多个缓冲区逻辑上视为一个 | 组件合并等操作可能实际复制 |
| wrappedBuffer | 包装已有数组或缓冲区 | 原存储的修改会影响视图 |
| copy / copiedBuffer | 复制出独立数据 | 不是零拷贝 |

普通 slice / duplicate 不自动增加引用计数，需要独立持有时可使用 retainedSlice 等形式并配对释放。修改共享内容会互相影响。[派生缓冲区](https://netty.io/4.1/api/io/netty/buffer/ByteBuf.html)

组合示例：消息头和消息体已经分别存在时，可让 CompositeByteBuf 引用两者，不必为了拼接而复制成新数组。包装数组后，也不能随意修改尚在发送的数据。[组合缓冲区](https://netty.io/4.1/api/io/netty/buffer/CompositeByteBuf.html)、[包装与复制](https://netty.io/4.1/api/io/netty/buffer/Unpooled.html)

直接内存可能减少某些中间缓冲复制，但不意味着没有用户态与内核间的数据传递。文件传输的 FileRegion / transferTo 属于另一类优化，能否利用操作系统零拷贝路径还取决于平台及传输条件。[直接缓冲区](https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/nio/ByteBuffer.html)、[DefaultFileRegion](https://netty.io/4.1/api/io/netty/channel/DefaultFileRegion.html)

## 14. TCP 粘包拆包与分帧

### 14.1 为什么出现

TCP 是字节流，不保留应用每次 write 的消息边界。发送两次，不保证接收端读两次。所谓“包”在这里指应用消息，不等同于 TCP 报文段。

发送端依次写入 HELLO、WORLD，接收端可能读到：

| 每次读取内容 | 常见称呼 |
|---|---|
| HELLO / WORLD | 边界恰好一致 |
| HELLOWORLD | 粘包 |
| HE / LLO / WORLD | 拆包、半包 |
| HEL / LOWO / RLD | 两者同时出现 |

字节顺序没有改变，变化的是每次读取的范围。发送缓冲、TCP 分段、接收数据到达情况、读取时机与缓冲容量共同影响这一范围。关闭 Nagle、调整缓冲区、每次 flush，都不能保证一发一收对应。[TCP 规范](https://www.rfc-editor.org/rfc/rfc9293.html#section-3.7)

### 14.2 Netty 如何处理

```text
收到字节 → 累计 → 根据协议判断完整性
                    ├── 不完整：保留，等待下次
                    └── 完整：输出一条，再解析剩余数据
```

ByteToMessageDecoder 提供累计字节和驱动解码的基础机制，具体分帧规则由实现决定。若消息不足，不能破坏后续重新解析所需的读取位置。[解码器机制](https://netty.io/4.1/api/io/netty/handler/codec/ByteToMessageDecoder.html)

| 协议规则 | 解码器 | 使用边界 |
|---|---|---|
| 固定字节长度 | FixedLengthFrameDecoder | 适合长度固定的消息 |
| 换行结束 | LineBasedFrameDecoder | 正文换行需要约束或转义 |
| 自定义分隔符 | DelimiterBasedFrameDecoder | 处理分隔符与正文冲突 |
| 消息头含长度 | LengthFieldBasedFrameDecoder | 明确长度字段位置、宽度及含义 |

按行示例，协议数据是 `HELLO\nWORLD\n`：

| 本次收到 | 输出给后续 Handler | 保留内容 |
|---|---|---|
| HEL | 无 | HEL |
| LO\nWO | HELLO | WO |
| RLD\n | WORLD | 无 |

采用剥离行结束符的配置后，StringDecoder 转换完整帧，业务处理完整的一行。请求和响应都要遵守同一套边界约定。[LineBasedFrameDecoder](https://netty.io/4.1/api/io/netty/handler/codec/LineBasedFrameDecoder.html)

长度字段示例：`[4 字节消息体长度 N][N 字节消息体]`。先等完整头部，再等累计达到 4 + N 字节。LengthFieldBasedFrameDecoder 支持配置字段偏移、宽度、长度修正和剥离字节数，并限制最大帧长。[长度字段解码器](https://netty.io/4.1/api/io/netty/handler/codec/LengthFieldBasedFrameDecoder.html)

Netty 根据协议还原消息，不能自动猜测业务边界。StringDecoder、ByteBuf 池化、TCP_NODELAY 都不能替代分帧协议。

## 15. 易错点与复习问题

| 容易误解 | 正确理解 |
|---|---|
| NioSocketChannel 只给客户端用 | 服务端每条已接受连接也使用它 |
| 一个连接一个 EventLoop 就是一个连接一个线程 | 多个连接可以共享同一个 EventLoop |
| eventLoop.execute 可以隔离耗时业务 | 提交给原 EventLoop 仍占用原 I/O 线程 |
| Pipeline 是处理任务的线程池 | 它组织处理链，不负责提供线程 |
| 入站 Handler 不能回复数据 | 可以发起写操作，由出站链处理 |
| Channel 串行就代表所有业务都线程安全 | 跨连接共享状态及外部执行器仍可能并发 |
| isDone 等于成功 | 失败、取消也属于完成 |
| 写 Future 成功等于对端业务成功 | 业务成功需要协议响应确认 |
| await 超时会取消网络操作 | 等待时间和 I/O 超时是两回事 |
| clear 会释放 ByteBuf | clear 只重置索引 |
| retain 让 ByteBuf 线程安全 | retain 只影响生命周期 |
| 池化等于直接内存 | 复用方式与存储位置是独立维度 |
| 零拷贝意味着完全不复制 | 要明确省掉哪一层、哪一次复制 |
| 一次 write 对应一次 channelRead | TCP 不保留应用消息边界 |
| select 返回 0 就是空转 Bug | 需要排除超时、唤醒等正常情况 |

建议按以下问题口述复习：

1. 画出 EchoServer 从监听端口到返回响应的全过程。
2. 区分 Channel、EventLoop、Pipeline、Handler 和 Context。
3. 为什么同一连接默认串行处理，不同连接又可以并行？
4. Selector 的就绪事件与任务队列中的 Runnable 有何区别？
5. StringEncoder 放在业务 Handler 前面，为什么能处理响应？
6. Future 未完成、失败与业务失败分别怎么识别？
7. ByteBuf 谁分配、谁使用、谁释放？切片又会改变什么？
8. 用 HELLO/WORLD 演示分帧器如何处理半包和多消息。
9. 如何区分正常等待、Selector 异常空转和业务线程忙循环？

相关专题：[Java 学习 NIO 指南](./Java学习NIO指南.md)、[网络知识](../08-网络知识/README.md)、[JVM 问题排查](../03-JVM/JVM问题排查思路.md)。
