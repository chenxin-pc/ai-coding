# 跨域与 Web 安全（14—15）

[返回网络知识导航](./README.md)

## 14. 什么是跨域和 CORS？为什么浏览器失败而 Java 调用成功？

### 是什么

同源通常要求协议、主机、端口一致；路径不同不影响同源。浏览器同源策略限制网页脚本访问其他源的资源，CORS 通过服务端响应头声明哪些来源可读取响应。

跨源 fetch/XHR 使用 PUT、DELETE、Authorization、自定义请求头，或 Content-Type 为 application/json 等不满足简单请求条件的情况，通常会先发送 OPTIONS 预检。

### 为什么

浏览器可能同时持有多个网站的登录凭据，需要阻止恶意网页任意读取其他网站的私有响应。普通 Java HTTP 客户端不执行网页同源策略，因此不会仅因缺少 CORS 响应头就拒绝交付响应。

### 实际场景怎么用或排查

1. 在浏览器网络面板确认是预检失败，还是实际请求返回后响应被浏览器拦截。
2. 对照 Origin、方法和请求头，检查 Access-Control-Allow-Origin、Allow-Methods、Allow-Headers 等响应配置。
3. 携带 Cookie 时还要检查客户端 credentials、Cookie 属性和 Allow-Credentials；此时 Allow-Origin 不能使用星号。
4. 实际业务响应和错误响应也要有适用的 CORS 头，不能只给 OPTIONS 配置。
5. 比较浏览器与 Java 调用的 URL、认证信息和代理路径，不把“Java 调用成功”当成浏览器一定可用的证据。

允许来源应来自明确配置的列表，不能任意反射调用方 Origin 并允许凭据。动态返回允许源且经过共享缓存时，还要正确设置 Vary: Origin。

**易错点：** 不是所有跨域请求都会预检。简单请求可能已经执行，只是脚本读不到响应；预检失败通常不会继续发送实际请求。CORS 也不是服务端身份认证，Java 调用仍需要鉴权。

参考 [MDN CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS)和 [同源策略](https://developer.mozilla.org/en-US/docs/Web/Security/Defenses/Same-origin_policy)。

## 15. HTTPS 能否防止 CSRF、XSS？实际应该如何防护？

### 是什么

不能。HTTPS 保护传输，CSRF 与 XSS 发生在不同层面。

| 问题 | 核心机制 | 主要防护思路 |
|---|---|---|
| CSRF | 攻击者诱导浏览器利用已有凭据发起非用户本意的操作 | CSRF Token、适当的 SameSite、来源或 Fetch Metadata 校验，禁止用 GET 执行状态变更 |
| XSS | 不可信内容被浏览器当作可执行脚本处理 | 按输出上下文编码、富文本净化、安全模板与 DOM API、CSP 等纵深防御 |

### 为什么

CSRF 请求可以通过一条完全正常的 HTTPS 连接提交；XSS 脚本也可以通过 HTTPS 被完整传给浏览器。加密只能保护传输中的数据，不能判断应用操作是不是用户真实意愿，也不能修正不安全的页面渲染。

### 实际场景怎么用或排查

使用 Cookie 登录的写接口，检查框架的 CSRF 机制是否正确启用、Token 是否绑定并验证。单独设置 CORS 不足以防 CSRF，因为某些跨站请求无需读取响应就能产生影响。

展示用户昵称、评论、富文本时，区分 HTML 文本、属性、URL、脚本等上下文进行处理，不直接把不可信内容拼入可执行位置。HttpOnly 可降低 Cookie 被读取的风险，但 XSS 仍可能在页面上下文中代发操作。

**易错点：** Bearer Token 的 CSRF 风险取决于是否由浏览器自动附带；把 Token 换个名字放入 Cookie 不会消除 CSRF。防护依据见 [MDN CSRF](https://developer.mozilla.org/en-US/docs/Web/Security/Attacks/CSRF)和 [MDN XSS](https://developer.mozilla.org/en-US/docs/Web/Security/Attacks/XSS)。

---

[返回导航与复习清单](./README.md)
