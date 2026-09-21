# HTTP 基础与请求方法（01—04）

[返回网络知识导航](./README.md)

## 01. HTTP 是什么？它与 TCP、HTTPS 有什么关系？

### 是什么

HTTP 是应用层的请求—响应协议，规定资源标识、方法、状态码和消息语义。TCP 提供可靠、有序的字节流；HTTPS 是通过安全连接传输 HTTP，提供通信机密性、完整性和身份验证。

常见协议关系：

| 使用方式 | 协议关系 |
|---|---|
| 普通 HTTP/1.1 | HTTP → TCP → IP |
| HTTPS 上的 HTTP/1.1、HTTP/2 | HTTP → TLS → TCP → IP |
| HTTP/3 | HTTP → QUIC（集成 TLS 1.3 握手）→ UDP → IP |

### 为什么

分层让应用不用自行实现可靠传输和加密。HTTP 决定“查询订单是什么意思”，传输协议负责交付数据，TLS 负责保护通信。HTTP 的方法语义可以保持不变，而底层传输和编码持续演进。

### 实际场景怎么用或排查

接口不可用时先按阶段分类：域名解析失败、建连失败、TLS 验证失败、收到 HTTP 错误、业务返回失败，定位方式各不相同。连接拒绝时通常还没有 HTTP 响应；收到 401 则说明某个 HTTP 服务或网关已经处理了请求。

**易错点：** 不能说所有 HTTP 都基于 TCP，也不能把“收到 TCP ACK”等同于“订单已经提交”。协议分层参考 [HTTP 概述](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Guides/Overview)，HTTP/3 的传输关系参考 [RFC 9114](https://www.rfc-editor.org/rfc/rfc9114.html)。

## 02. HTTP 请求、响应和常见头部由什么组成？

### 是什么

HTTP/1.1 请求由请求行、头部、空行和可选消息体组成；响应由状态行、头部、空行和可选消息体组成。HTTP/2、HTTP/3 使用帧表达消息，不使用同样的文本起始行。

```http
POST /orders HTTP/1.1
Host: api.example.com
Content-Type: application/json
Accept: application/json
Content-Length: 20

{"sku":"A1","qty":1}
```

```http
HTTP/1.1 201 Created
Content-Type: application/json
Content-Length: 11

{"id":1001}
```

示例中的长度分别是 20 和 11 个 UTF-8 字节，不包含代码块展示用的末尾换行；实际报文行结束使用 CRLF。

| 常见字段 | 作用 |
|---|---|
| Host | 请求要访问的主机与可选端口 |
| Content-Type | 当前消息内容的格式，例如 JSON |
| Accept | 客户端希望收到的响应格式 |
| Authorization | 认证凭据，例如 Bearer Token |
| Cookie / Set-Cookie | 客户端携带 Cookie / 服务端设置 Cookie |
| Content-Length | 内容长度，按字节计算 |
| Cache-Control | 缓存策略 |

### 为什么

消息体承载数据，头部描述解析、认证、缓存等处理规则。Content-Type 与 Accept 不能互换：前者描述“发的是什么”，后者表达“希望收到什么”。

### 实际场景怎么用或排查

参数为空时依次核对方法、路径、Content-Type、实际请求体以及 Java 接口的参数绑定方式。415 通常检查输入媒体类型，406 检查响应内容协商，400 继续检查格式和参数解析错误。

用浏览器网络面板或客户端日志逐项比较请求和响应，记录前先隐藏 Authorization、Cookie 等凭据。不能只修改 Content-Type 就认为表单自动变成 JSON，也不要按字符数手算传输长度。

参考 [HTTP 消息](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Guides/Messages)和 [Content-Type](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Type)。

## 03. GET 与 POST 有什么区别？POST 是否更安全？

### 是什么

GET 请求获取资源的表示；POST 提交内容，由目标资源按约定处理，常用于创建订单、执行操作。GET 的协议语义是安全且幂等，POST 不作这两项保证。

GET 常将筛选条件放在查询字符串，POST 常使用消息体，但 POST 同样可以包含查询参数。GET 请求体没有普遍定义的语义，不适合作为通用接口契约。

### 为什么

方法语义会影响预取、缓存、爬虫、自动重试和客户端行为。若用 GET 执行扣款，浏览器预加载、链接扫描或重试都可能触发不期望的操作。

### 实际场景怎么用或排查

查询订单使用 GET，提交订单通常使用 POST 并增加业务幂等控制。密码、Token 不放 URL，以免进入历史记录或访问日志；消息体也可能被应用日志记录，需要脱敏。两种方法均应通过 HTTPS 保护传输。

**易错点：** POST 不自动加密；HTTP 没有“GET 统一限长 2 KB”的通用规定，实际限制来自浏览器、网关和服务器配置。不要把“常见实现习惯”当成协议强制要求。方法语义参考 [HTTP 请求方法](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Methods)。

## 04. 什么是安全方法和幂等性？为什么 DELETE 幂等而 POST 通常不幂等？

### 是什么

“安全”表示客户端没有通过该方法请求改变服务端资源状态；“幂等”表示重复相同请求，对服务器产生的预期效果与执行一次相同。这里的安全不是信息安全或加密。

| 方法 | 安全 | 幂等 |
|---|---|---|
| GET、HEAD、OPTIONS | 是 | 是 |
| PUT、DELETE | 否 | 是 |
| POST、PATCH | 否 | 不保证，可由业务约束实现 |

### 为什么

这两个性质帮助客户端判断哪些请求适合预取和重试。服务端记录访问日志、计数等附带行为不必完全相同，也不因此否定方法的安全或幂等语义。

### 实际场景怎么用或排查

DELETE 首次成功返回 204、再次返回 404，目标都处于已删除状态，因此不要求返回码相同。POST 创建订单则可能每次生成新订单，需要业务机制实现幂等。

审查接口时，先问“业务预期效果是什么”，再看重放是否会重复创建、扣减或通知。连续两次 GET 返回不同库存，可能只是资源在两次请求之间变化，并不说明 GET 不幂等。

**易错点：** 幂等不保证每次状态码和响应体相同，也不等于并发安全。重试还应检查请求体能否重放、认证是否仍有效、整体时限是否足够。定义参考 [RFC 9110 的安全与幂等方法](https://www.rfc-editor.org/rfc/rfc9110.html#section-9.2)。

---

[返回导航与复习清单](./README.md)
