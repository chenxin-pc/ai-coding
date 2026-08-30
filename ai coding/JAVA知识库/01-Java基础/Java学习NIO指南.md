# Java 学习 NIO 指南

## 目录

- [1. 学习目标](#1-学习目标)
- [2. 第一阶段：IO 基础概念](#2-第一阶段io-基础概念)
  - [2.1 什么是 IO](#21-什么是-io)
  - [2.2 必须掌握的基础概念](#22-必须掌握的基础概念)
  - [2.3 阶段练习](#23-阶段练习)
  - [2.4 验收标准](#24-验收标准)
- [3. 第二阶段：操作系统 IO 基础](#3-第二阶段操作系统-io-基础)
  - [3.1 Java 程序如何完成 IO](#31-java-程序如何完成-io)
  - [3.2 核心概念](#32-核心概念)
  - [3.3 阶段练习](#33-阶段练习)
  - [3.4 验收标准](#34-验收标准)
- [4. 第三阶段：Java BIO 字节流](#4-第三阶段java-bio-字节流)
  - [4.1 核心抽象](#41-核心抽象)
  - [4.2 文件复制示例](#42-文件复制示例)
  - [4.3 阶段练习](#43-阶段练习)
  - [4.4 验收标准](#44-验收标准)
- [5. 第四阶段：字符流与编码](#5-第四阶段字符流与编码)
  - [5.1 核心抽象](#51-核心抽象)
  - [5.2 阶段练习](#52-阶段练习)
  - [5.3 验收标准](#53-验收标准)
- [6. 第五阶段：缓冲与装饰器](#6-第五阶段缓冲与装饰器)
  - [6.1 为什么需要缓冲](#61-为什么需要缓冲)
  - [6.2 从 BIO Buffer 过渡到 NIO Buffer](#62-从-bio-buffer-过渡到-nio-buffer)
  - [6.3 验收标准](#63-验收标准)
- [7. 第六阶段：BIO 网络编程](#7-第六阶段bio-网络编程)
  - [7.1 基本模型](#71-基本模型)
  - [7.2 阻塞发生在哪里](#72-阻塞发生在哪里)
  - [7.3 阶段练习](#73-阶段练习)
  - [7.4 验收标准](#74-验收标准)
- [8. 第七阶段：Java NIO 基础](#8-第七阶段java-nio-基础)
  - [8.1 ByteBuffer](#81-bytebuffer)
  - [8.2 Channel](#82-channel)
  - [8.3 FileChannel 示例](#83-filechannel-示例)
  - [8.4 阶段练习](#84-阶段练习)
  - [8.5 验收标准](#85-验收标准)
- [9. 第八阶段：NIO 网络编程](#9-第八阶段nio-网络编程)
  - [9.1 网络 Channel](#91-网络-channel)
  - [9.2 为什么需要 Selector](#92-为什么需要-selector)
  - [9.3 Selector 核心骨架](#93-selector-核心骨架)
  - [9.4 阶段练习](#94-阶段练习)
  - [9.5 验收标准](#95-验收标准)
- [10. 第九阶段：处理 TCP 数据边界](#10-第九阶段处理-tcp-数据边界)
  - [10.1 必须掌握的问题](#101-必须掌握的问题)
  - [10.2 推荐协议练习](#102-推荐协议练习)
  - [10.3 验收标准](#103-验收标准)
- [11. 第十阶段：操作系统 IO 模型与多路复用](#11-第十阶段操作系统-io-模型与多路复用)
  - [11.1 四种常见 IO 思想](#111-四种常见-io-思想)
  - [11.2 Java Selector 与操作系统](#112-java-selector-与操作系统)
  - [11.3 验收标准](#113-验收标准)
- [12. 第十一阶段：Reactor 模型](#12-第十一阶段reactor-模型)
  - [12.1 单 Reactor 单线程](#121-单-reactor-单线程)
  - [12.2 单 Reactor 多线程](#122-单-reactor-多线程)
  - [12.3 主从 Reactor 多线程](#123-主从-reactor-多线程)
  - [12.4 验收标准](#124-验收标准)
- [13. 第十二阶段：进入 Netty](#13-第十二阶段进入-netty)
- [14. 推荐实践项目](#14-推荐实践项目)
  - [14.1 文件复制工具](#141-文件复制工具)
  - [14.2 BIO 聊天室](#142-bio-聊天室)
  - [14.3 NIO 聊天室](#143-nio-聊天室)
  - [14.4 Netty 重写聊天室](#144-netty-重写聊天室)
- [15. 学习优先级](#15-学习优先级)
  - [15.1 第一梯队：必须吃透](#151-第一梯队必须吃透)
  - [15.2 第二梯队：理解并能使用](#152-第二梯队理解并能使用)
  - [15.3 第三梯队：按需深入](#153-第三梯队按需深入)
- [16. 学习时应该持续追问的问题](#16-学习时应该持续追问的问题)
- [17. 最终验收清单](#17-最终验收清单)

## 1. 学习目标

这份指南适合已经具备 Java 基础、希望从最基础的 IO 概念开始，逐步掌握 Java BIO、NIO、操作系统 IO、多路复用、Reactor 和 Netty 的开发者。

学习时不要只记 API，要始终围绕三个问题展开：

- 数据从哪里来，要到哪里去？
- 当前线程会不会阻塞，阻塞在哪里？
- 为什么要从上一种模型演进到下一种模型？

完整主线如下：

```text
bit / byte / char / 编码
        ↓
输入与输出
        ↓
操作系统 IO 基础
        ↓
Java 字节流与字符流
        ↓
缓冲流与装饰器模式
        ↓
BIO Socket 网络编程
        ↓
阻塞模型的并发问题
        ↓
ByteBuffer + Channel
        ↓
非阻塞 SocketChannel
        ↓
Selector + SelectionKey
        ↓
操作系统 IO 多路复用
        ↓
Reactor
        ↓
Netty
```

## 2. 第一阶段：IO 基础概念

### 2.1 什么是 IO

IO 是 Input/Output，本质是数据在设备、内存、进程或网络节点之间传输。

Java 中的输入和输出通常以“当前 Java 程序”为参照物：

```text
文件 ─────→ Java 程序：输入
Java 程序 ─────→ 文件：输出

网络 ─────→ Java 程序：输入
Java 程序 ─────→ 网络：输出
```

### 2.2 必须掌握的基础概念

- `bit`：位，计算机存储的最小单位，值为 0 或 1
- `byte`：字节，通常由 8 个 bit 组成
- `char`：字符的抽象表示，不等同于固定数量的字节
- ASCII：早期字符编码标准
- Unicode：统一字符集
- UTF-8：Unicode 的一种可变长度编码方式
- GBK：常见中文字符编码
- 编码：字符转换为字节
- 解码：字节转换为字符

字符串落盘或通过网络传输时，实际传输的是字节：

```text
字符串
  ↓ encode
字节序列
  ↓ 文件或网络
字节序列
  ↓ decode
字符串
```

乱码通常是因为编码和解码使用了不兼容的字符集，或者字节序列不完整。

### 2.3 阶段练习

- 使用 UTF-8 和 GBK 分别编码同一个中文字符串，比较字节数组长度
- 使用错误的字符集解码字节数组，观察乱码现象
- 解释 Java 中的输入和输出为什么必须先明确参照物

### 2.4 验收标准

能够独立解释：字节与字符的区别、编码与解码的过程、乱码产生的原因。

## 3. 第二阶段：操作系统 IO 基础

### 3.1 Java 程序如何完成 IO

Java 程序通常不会直接操作磁盘或网卡，而是通过 JVM 调用操作系统能力：

```text
Java 应用程序
      ↓
JVM / 本地方法
      ↓
系统调用
      ↓
操作系统内核
      ↓
文件系统 / 磁盘 / 网卡
```

### 3.2 核心概念

- 用户态：普通应用程序运行的受限环境
- 内核态：操作系统内核运行的高权限环境
- 系统调用：应用程序请求内核提供服务的入口
- 文件描述符：操作系统用于标识已打开文件或 Socket 的整数句柄
- 内核缓冲区：由操作系统内核管理的缓冲区
- 用户缓冲区：应用程序可访问的内存缓冲区

读取文件可以先用下面的简化模型理解：

```text
磁盘
  ↓
内核缓冲区
  ↓
用户空间
  ↓
JVM 中的 byte[] 或对象
```

### 3.3 阶段练习

- 画出 `FileInputStream.read()` 从 Java 代码到磁盘的大致调用路径
- 查清进程、线程、文件描述符之间的关系
- 思考一次读取一个字节为什么通常比批量读取效率低

### 3.4 验收标准

能够解释 Java IO 为什么依赖操作系统，以及用户态、内核态、系统调用和缓冲区分别是什么。

## 4. 第三阶段：Java BIO 字节流

### 4.1 核心抽象

字节流的两个顶层抽象：

```text
InputStream
    ├── FileInputStream
    ├── ByteArrayInputStream
    ├── BufferedInputStream
    └── ObjectInputStream

OutputStream
    ├── FileOutputStream
    ├── ByteArrayOutputStream
    ├── BufferedOutputStream
    └── ObjectOutputStream
```

字节流适合处理文件、图片、压缩包、音视频和网络数据等原始字节。

### 4.2 文件复制示例

```java
byte[] buffer = new byte[8192];
int length;

while ((length = input.read(buffer)) != -1) {
    output.write(buffer, 0, length);
}
```

需要理解：

- `read()` 返回 `int`，因为既要表示读取的字节值或数量，也要用 `-1` 表示流结束
- 一次读取不保证填满整个数组
- 最后一次读取的数据通常少于缓冲区容量
- 必须使用实际读取长度 `length` 写出，不能默认写出整个数组
- `try-with-resources` 可以确保流被可靠关闭

### 4.3 阶段练习

- 使用单字节方式复制文件
- 使用 `byte[]` 批量复制文件并比较耗时
- 实现文本、图片和压缩包复制
- 使用 `try-with-resources` 管理流

### 4.4 验收标准

能够独立实现不会丢数据、不会重复写入、能正确关闭资源的文件复制程序。

## 5. 第四阶段：字符流与编码

### 5.1 核心抽象

```text
Reader
  ├── FileReader
  ├── BufferedReader
  └── InputStreamReader

Writer
  ├── FileWriter
  ├── BufferedWriter
  └── OutputStreamWriter
```

字符流主要用于文本数据。字节流与字符流之间通过转换流连接：

```text
InputStream
      ↓ InputStreamReader + Charset
Reader

Writer
      ↓ OutputStreamWriter + Charset
OutputStream
```

推荐显式指定字符集：

```java
Reader reader = new InputStreamReader(
        inputStream,
        StandardCharsets.UTF_8
);
```

### 5.2 阶段练习

- 使用 `BufferedReader` 按行读取 UTF-8 文本
- 将 GBK 文本转换为 UTF-8 文本
- 比较字节流和字符流处理二进制文件的结果

### 5.3 验收标准

能够根据数据类型选择字节流或字符流，并能显式、正确地处理字符编码。

## 6. 第五阶段：缓冲与装饰器

### 6.1 为什么需要缓冲

频繁执行小块 IO 会增加系统调用、数据复制和方法调用开销。缓冲的核心思想是批量读写：

```text
低效方式：read → read → read → read

缓冲方式：底层一次读取一批数据 → 应用逐步消费
```

需要掌握：

- `BufferedInputStream`
- `BufferedOutputStream`
- `BufferedReader`
- `BufferedWriter`
- `flush()` 的作用
- 关闭输出流为什么通常也会触发刷新
- `byte[]` 为什么也可以作为手动缓冲区

### 6.2 从 BIO Buffer 过渡到 NIO Buffer

BIO 中的缓冲通常隐藏在流对象内部；NIO 中的 `ByteBuffer` 被提升为开发者直接管理的核心对象。

理解缓冲区之后，再学习 `ByteBuffer` 的 `position`、`limit` 和 `capacity` 会自然得多。

### 6.3 验收标准

能够解释缓冲为什么能提升性能、什么时候必须调用 `flush()`，并理解 Java IO 中装饰器模式的作用。

## 7. 第六阶段：BIO 网络编程

### 7.1 基本模型

```text
客户端 Socket ─────→ 服务端 ServerSocket
                           ↓ accept()
                      服务端 Socket

客户端 OutputStream ─────→ 服务端 InputStream
```

先完成最简单的请求与响应程序，再逐步支持多个客户端。

### 7.2 阻塞发生在哪里

```java
Socket socket = serverSocket.accept();
```

没有新连接时，线程可能阻塞在 `accept()`。

```java
int length = inputStream.read(buffer);
```

没有数据、连接又未关闭时，线程可能阻塞在 `read()`。

经典 BIO 并发模型通常为一个连接分配一个线程：

```text
Client A → Thread A
Client B → Thread B
Client C → Thread C
```

当连接数量很大且多数连接长期空闲时，大量线程会带来内存和调度开销。这是继续学习 NIO 的直接动机。

### 7.3 阶段练习

- 实现单客户端回显服务器
- 实现一个连接一个线程的多客户端服务器
- 使用线程池限制服务端线程数量
- 模拟客户端连接后长时间不发送数据，观察服务端线程状态

### 7.4 验收标准

能够指出 `accept()` 和 `read()` 的阻塞位置，并解释一个连接一个线程模型的优点与局限。

## 8. 第七阶段：Java NIO 基础

建议严格按照下面的顺序学习：

```text
ByteBuffer
    ↓
Channel
    ↓
FileChannel
    ↓
ServerSocketChannel / SocketChannel
    ↓
非阻塞模式
    ↓
Selector
    ↓
SelectionKey
```

### 8.1 ByteBuffer

创建缓冲区：

```java
ByteBuffer buffer = ByteBuffer.allocate(10);
```

三个核心属性：

- `capacity`：缓冲区总容量，创建后通常不变
- `position`：下一个要读或写的位置
- `limit`：当前模式下允许访问的边界

写入 3 个字节后：

```text
capacity = 10
position = 3
limit    = 10

0   1   2   3   4   5   6   7   8   9
+---+---+---+---+---+---+---+---+---+---+
| 1 | 2 | 3 |   |   |   |   |   |   |   |
+---+---+---+---+---+---+---+---+---+---+
            ↑                           ↑
         position                     limit
```

调用 `flip()` 后：

```text
capacity = 10
position = 0
limit    = 3
```

`flip()` 的本质是将缓冲区从写模式切换到读模式，让读取从位置 0 开始，并把可读边界设置为此前写入结束的位置。

重点方法：

```java
put()
get()
flip()
clear()
compact()
rewind()
remaining()
hasRemaining()
```

`clear()` 与 `compact()` 的区别：

- `clear()`：逻辑上丢弃全部未读数据，准备重新写入
- `compact()`：保留未读数据并移动到缓冲区前部，再从其后继续写入

### 8.2 Channel

BIO 与 NIO 文件读取模型：

```text
BIO：File → InputStream → byte[]

NIO：File ↔ FileChannel ↔ ByteBuffer
```

需要特别注意方法方向：

```java
channel.read(buffer);
```

表示 Channel 将数据写入 Buffer。

```java
channel.write(buffer);
```

表示 Channel 从 Buffer 读取数据并写向目标。

### 8.3 FileChannel 示例

```java
try (FileChannel source = FileChannel.open(
            Path.of("a.txt"),
            StandardOpenOption.READ);
     FileChannel target = FileChannel.open(
            Path.of("b.txt"),
            StandardOpenOption.CREATE,
            StandardOpenOption.TRUNCATE_EXISTING,
            StandardOpenOption.WRITE)) {

    ByteBuffer buffer = ByteBuffer.allocate(8192);

    while (source.read(buffer) != -1) {
        buffer.flip();

        while (buffer.hasRemaining()) {
            target.write(buffer);
        }

        buffer.clear();
    }
}
```

### 8.4 阶段练习

- 手工记录每次 `put()`、`get()`、`flip()` 后三个核心属性的值
- 分别使用 `clear()` 和 `compact()` 处理半包，观察差异
- 使用 `FileChannel` 复制文件
- 比较堆缓冲区和直接缓冲区的基本使用方式

### 8.5 验收标准

能够不靠死记解释 `flip()` 为什么存在，并能准确说明 Channel 与 Buffer 之间的数据方向。

## 9. 第八阶段：NIO 网络编程

### 9.1 网络 Channel

```text
BIO                    NIO

ServerSocket     →     ServerSocketChannel
Socket           →     SocketChannel
InputStream      →     Channel + Buffer
OutputStream     →     Channel + Buffer
```

先用阻塞模式熟悉 `ServerSocketChannel` 和 `SocketChannel`，再切换到非阻塞模式：

```java
serverChannel.configureBlocking(false);
socketChannel.configureBlocking(false);
```

非阻塞模式下：

- `accept()` 没有待处理连接时可能返回 `null`
- `read()` 返回 `0` 表示当前没有读到数据，但连接仍然有效
- `read()` 返回 `-1` 表示对端已正常关闭输出方向，通常需要关闭 Channel
- `write()` 可能只写出一部分数据，需要保存剩余数据并稍后继续写

### 9.2 为什么需要 Selector

非阻塞 Channel 虽然不会让线程一直卡住，但轮询成千上万个 Channel 同样低效：

```text
多个 Channel
     ↓ 注册
Selector
     ↓ 就绪事件
一个线程集中处理
```

Selector 主要关注四类事件：

```java
SelectionKey.OP_ACCEPT
SelectionKey.OP_CONNECT
SelectionKey.OP_READ
SelectionKey.OP_WRITE
```

可以直观理解为：

- `OP_ACCEPT`：服务端有新连接可以接收
- `OP_CONNECT`：客户端连接过程可以完成
- `OP_READ`：Channel 当前可读
- `OP_WRITE`：Channel 当前可继续写

### 9.3 Selector 核心骨架

```java
try (Selector selector = Selector.open();
     ServerSocketChannel serverChannel = ServerSocketChannel.open()) {

    serverChannel.configureBlocking(false);
    serverChannel.bind(new InetSocketAddress(8080));
    serverChannel.register(selector, SelectionKey.OP_ACCEPT);

    while (true) {
        selector.select();

        Iterator<SelectionKey> iterator =
                selector.selectedKeys().iterator();

        while (iterator.hasNext()) {
            SelectionKey key = iterator.next();
            iterator.remove();

            if (!key.isValid()) {
                continue;
            }

            if (key.isAcceptable()) {
                // 接收新连接并注册 OP_READ
            }

            if (key.isReadable()) {
                // 从 SocketChannel 读取数据
            }

            if (key.isWritable()) {
                // 继续写出此前未发送完的数据
            }
        }
    }
}
```

必须理解：

- `select()` 可以阻塞，但一个线程等待的是大量 Channel 的就绪事件
- `selectedKeys()` 是本次需要处理的就绪 Key 集合
- 处理完成后需要从 selected-key 集合移除 Key
- `SelectionKey` 可以通过 `attachment` 绑定连接状态
- Channel 关闭或发生异常时，需要取消 Key 并释放资源
- `OP_WRITE` 通常只在确实存在待发送数据时注册，否则会频繁触发空转

### 9.4 阶段练习

- 实现 Selector 版回显服务器
- 实现多个客户端同时连接的聊天室
- 为每个连接绑定独立 `ByteBuffer` 和消息状态
- 正确处理客户端主动断开、异常断开和部分写入

### 9.5 验收标准

能够从零写出 `Selector + ServerSocketChannel + SocketChannel` 服务端骨架，并解释每个事件的触发条件。

## 10. 第九阶段：处理 TCP 数据边界

NIO 网络编程真正困难的部分通常不是 Selector API，而是 Buffer、TCP 数据边界和连接状态管理。

### 10.1 必须掌握的问题

- TCP 没有消息边界
- 粘包与拆包
- 半包处理
- 定长消息、分隔符消息、长度字段消息
- ByteBuffer 扩容
- 字符跨多个字节包时的解码问题
- 写半包与待发送队列
- 背压与内存上限

### 10.2 推荐协议练习

先实现按换行符拆分消息的协议：

```text
客户端发送的 TCP 字节流：

Hello\nWorld\nJava\n
        ↓
服务端累计进 ByteBuffer
        ↓
按 \n 提取完整消息
        ↓
Hello
World
Java
```

读取完成后：

- 扫描缓冲区中的完整消息
- 消费完整消息
- 使用 `compact()` 保留不完整的尾部数据
- 下一次可读事件到来后继续拼接

### 10.3 验收标准

能够正确处理一条消息被拆成多次读取、多条消息一次读入，以及输出无法一次写完的情况。

## 11. 第十阶段：操作系统 IO 模型与多路复用

### 11.1 四种常见 IO 思想

- 阻塞 IO：调用线程等待 IO 条件满足和数据复制完成
- 非阻塞 IO：调用立即返回，应用需要反复判断当前是否可操作
- IO 多路复用：一个等待机制同时关注多个文件描述符的就绪事件
- 异步 IO：应用发起操作后继续工作，由系统在操作完成时通知结果

需要特别注意：

> 非阻塞 IO 不等于 IO 多路复用。非阻塞描述单次调用的行为，多路复用描述如何集中等待多个 IO 对象。

### 11.2 Java Selector 与操作系统

```text
Java Selector
       ↓
JDK 针对不同操作系统的实现
       ↓
Linux：通常基于 epoll
macOS/BSD：通常基于 kqueue
Windows：使用相应系统机制
```

Linux 下建议依次理解：

```text
select
  ↓
poll
  ↓
epoll
```

重点比较：

- 文件描述符集合如何传递和维护
- 每次等待后如何确定哪些描述符就绪
- 连接数量增加时扫描成本如何变化
- 水平触发和边缘触发的差别

不要简单背诵“epoll 一定更快”。它的优势与连接数量、活跃比例、事件分布和具体实现有关。

### 11.3 验收标准

能够解释 `select()` 自己也可能阻塞，为什么仍能提高大量连接场景的并发处理能力；能够说明 Java Selector 与操作系统多路复用机制的大致关系。

## 12. 第十一阶段：Reactor 模型

前面实现的 Selector 事件循环，本质上已经具备简单 Reactor 的雏形：

```text
              Reactor
                 │
              Selector
                 │
      ┌──────────┼──────────┐
      ↓          ↓          ↓
   Accept      Read       Write
      ↓          ↓          ↓
  Acceptor    Handler    Handler
```

依次学习：

### 12.1 单 Reactor 单线程

一个线程负责连接接入、IO 事件和业务处理。实现简单，但任何耗时任务都会阻塞整个事件循环。

### 12.2 单 Reactor 多线程

Reactor 线程负责 IO，耗时业务提交到工作线程池。需要处理任务结果如何安全返回事件循环的问题。

### 12.3 主从 Reactor 多线程

```text
Main Reactor
     │ accept
     ↓
Sub Reactor Group
     │ read / write
     ↓
Worker Thread Pool
     │
业务处理
```

主 Reactor 负责接收连接，从 Reactor 负责连接上的读写事件，工作线程负责可能耗时的业务逻辑。

### 12.4 验收标准

能够画出三种 Reactor 模型，解释接入、IO 和业务处理分别由哪些线程负责，以及为什么事件循环中不应执行长时间阻塞任务。

## 13. 第十二阶段：进入 Netty

完成 NIO 和 Reactor 后再学习 Netty，可以把框架概念与底层知识对应起来：

```text
Java NIO                 Netty

Channel           →      Channel
Selector          →      EventLoop 内部机制
线程 + Selector   →      EventLoop
线程组            →      EventLoopGroup
ByteBuffer        →      ByteBuf
事件处理逻辑       →      ChannelHandler
处理器链          →      ChannelPipeline
```

推荐学习顺序：

1. 使用 Netty 重写此前的 NIO 回显服务器
2. 理解 `EventLoop` 与 Channel 的线程绑定关系
3. 理解入站事件、出站事件和 `ChannelPipeline`
4. 学习 `ByteBuf` 的读写索引、引用计数和内存池
5. 学习常见编解码器以及长度字段协议
6. 学习心跳、空闲检测、断线重连和背压
7. 最后再阅读启动流程、事件循环和内存管理源码

## 14. 推荐实践项目

### 14.1 文件复制工具

分别使用以下方式实现并比较：

- 单字节 BIO
- 带 `byte[]` 的 BIO
- `BufferedInputStream` / `BufferedOutputStream`
- `FileChannel + ByteBuffer`
- `FileChannel.transferTo()` 或 `transferFrom()`

### 14.2 BIO 聊天室

目标：理解阻塞、线程模型、连接生命周期和消息边界。

### 14.3 NIO 聊天室

目标：掌握 Selector、连接状态、半包、写半包和异常处理。

建议至少支持：

- 多客户端连接
- 昵称注册
- 消息广播
- 按换行符拆包
- 客户端下线通知
- 慢客户端写队列限制

### 14.4 Netty 重写聊天室

使用 Netty 重写相同需求，对比：

- 手工事件循环与 `EventLoop`
- `ByteBuffer` 与 `ByteBuf`
- 手工拆包与 Netty Decoder
- 手工状态管理与 Pipeline/Handler

## 15. 学习优先级

### 15.1 第一梯队：必须吃透

- 字节、字符、编码与解码
- `InputStream` / `OutputStream`
- `Reader` / `Writer`
- 缓冲思想
- BIO Socket 阻塞模型
- `ByteBuffer`
- `Channel`
- `SocketChannel`
- `Selector`
- `SelectionKey`
- IO 多路复用
- TCP 消息边界
- Reactor

### 15.2 第二梯队：理解并能使用

- `DirectByteBuffer`
- `MappedByteBuffer`
- `FileChannel.transferTo()`
- Scatter/Gather
- `Pipe`
- 字符集增量解码

### 15.3 第三梯队：按需深入

- Linux `select` / `poll` / `epoll` 系统调用细节
- 水平触发与边缘触发
- JDK Selector 的平台实现
- 零拷贝与内存映射底层原理
- Netty 内存池和引用计数源码

## 16. 学习时应该持续追问的问题

不要停留在：

```java
buffer.flip();
```

要追问：为什么写完数据后需要切换读写边界？

不要停留在：

```java
serverChannel.configureBlocking(false);
```

要追问：原来阻塞在哪里，由谁让线程进入等待状态？

不要停留在：

```java
selector.select();
```

要追问：`select()` 也会阻塞，为什么一个阻塞的线程反而能管理大量连接？

继续追问：

- Java Selector 与 Linux epoll 是什么关系？
- epoll 为什么适合大量空闲连接？
- 为什么 `OP_WRITE` 不能一直注册？
- 一次 `read()` 为什么不等于读取一条完整消息？
- 为什么业务线程不能直接随意修改 EventLoop 管理的连接状态？
- Netty 为什么通常让一个 Channel 的 IO 事件固定由一个 EventLoop 处理？

## 17. 最终验收清单

完成这条学习路线后，应当能够：

- 解释字节流、字符流、编码和缓冲
- 使用 BIO 完成可靠的文件与网络读写
- 指出 `accept()`、`read()` 和 `select()` 的阻塞含义
- 熟练使用 `ByteBuffer`，理解其状态变化
- 使用 `FileChannel` 处理文件
- 使用 `Selector` 管理多个非阻塞 Channel
- 正确处理 TCP 粘包、拆包、半包和写半包
- 解释阻塞 IO、非阻塞 IO、IO 多路复用和异步 IO 的区别
- 解释 Reactor 三种典型线程模型
- 从 NIO 设计自然映射到 Netty 的核心组件
- 从零实现一个简易 NIO 聊天室，并使用 Netty 重写

最终目标不是背诵零散 API，而是建立下面这条因果链：

```text
传统 BIO
   ↓
一个连接一个线程的资源问题
   ↓
非阻塞 Channel
   ↓
大量 Channel 如何高效等待
   ↓
Selector 与 IO 多路复用
   ↓
事件分发与 Reactor
   ↓
Netty EventLoop 与 Pipeline
```
