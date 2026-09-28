# HTTP, HTTPS, and Web Communication

面试准备笔记：**English terminology + 中文解释**。重点是理解一次 request 从 client 到 server 的全过程，以及 caching、security 和 failure handling 如何影响实际行为。

前置知识：[TCP and Socket Programming](tcp.md)。TCP handshake、retransmission、socket buffer 和 transport-level timeout 已在该文件解释，本篇不重复推导。

**阅读顺序：**先读 1–6 建立基本模型，再读 7–12 理解 performance 和 security，最后通过 13–17 的 API design、code、troubleshooting 和 interview questions 串起来。

**Examples:** `http` 和 `text` blocks 是协议示意或 pseudocode；HTTP/1.1 示意图省略部分非关键 headers，实际 header lines 使用 CRLF。第 15 节有一个可独立运行的 Python example。本文介绍常见 methods，不是完整的 HTTP method registry。

**Contents**

- [1. What Is HTTP?](#1-what-is-http)
- [2. URL, Origin, and a Request Lifecycle](#2-url-origin-and-a-request-lifecycle)
- [3. HTTP Messages and Message Framing](#3-http-messages-and-message-framing)
- [4. HTTP Methods and Their Semantics](#4-http-methods-and-their-semantics)
- [5. Status Codes](#5-status-codes)
- [6. Headers, Content Negotiation, and Compression](#6-headers-content-negotiation-and-compression)
- [7. HTTP Caching](#7-http-caching)
- [8. Conditional Requests and Optimistic Concurrency](#8-conditional-requests-and-optimistic-concurrency)
- [9. Persistent Connections and HTTP Versions](#9-persistent-connections-and-http-versions)
- [10. HTTPS and TLS](#10-https-and-tls)
- [11. Cookies, Sessions, and Tokens](#11-cookies-sessions-and-tokens)
- [12. Same-Origin Policy, CORS, CSRF, and XSS](#12-same-origin-policy-cors-csrf-and-xss)
- [13. Reliable HTTP API Design](#13-reliable-http-api-design)
- [14. Streaming, Polling, SSE, and WebSocket](#14-streaming-polling-sse-and-websocket)
- [15. Runnable Python: GET, HEAD, ETag, and 304](#15-runnable-python-get-head-etag-and-304)
- [16. Troubleshooting and Performance](#16-troubleshooting-and-performance)
- [17. Common Interview Questions](#17-common-interview-questions)

## 1. What Is HTTP?

### 1.1 An Application-layer Protocol

**HTTP — Hypertext Transfer Protocol** 定义 client 和 server 如何表达 requests、responses，以及这些 messages 的含义。

例如 browser 想显示某个用户的 profile：

```text
Browser
    |
    | GET /users/42
    v
HTTP server
    |
    | 查询 application / database
    v
HTTP response: 200 + JSON
    |
    v
Browser 解析 JSON，更新页面
```

HTTP 负责表达“我要哪个 resource”“我打算进行什么 operation”“结果是什么”。至于 bytes 如何可靠传输，通常由下层协议负责。

| Layer / component | 主要负责什么 |
| --- | --- |
| HTTP | Methods、status codes、headers、representation semantics |
| TLS | Protected communication：confidentiality、integrity、peer authentication |
| TCP | Reliable ordered byte stream，HTTP/1.1 和 HTTP/2 常用 |
| QUIC | HTTP/3 使用的 secure multiplexed transport，建立在 UDP 上 |
| IP | 把 packets 送往 destination address |
| Application | Authentication policy、业务规则、database operations |

**HTTP 不等于 HTML，也不等于 JSON。** HTTP 可以传 HTML、JSON、images、video、binary files 等。

### 1.2 Resource vs. Representation

**Resource** 是 URI 标识的对象或概念；**representation** 是某次响应中传输的具体表达。

```text
Resource: user 42

Possible representations:
- JSON profile
- HTML profile page
- Different language versions
```

一个 resource 不必对应磁盘上的一个 file。`/users/42` 可以由 server 动态查询 database 后生成。

同一个 URI 返回的内容也不一定永远相同：可能随时间、身份、language 或 negotiated format 变化。这也是 cache key 和 authorization 必须考虑 request context 的原因。

### 1.3 Request / Response Does Not Mean One Connection per Request

```text
One connection:
    request A → response A
    request B → response B
    request C → response C
```

Connection 可以复用；HTTP/2 和 HTTP/3 还能在同一个 connection 上 multiplex 多个 exchanges。

一个 HTTP request 也不等于一个 packet。Request 可能被拆成多个 transport packets，多个小 requests 也可能进入同一批底层发送数据。

### 1.4 What Does Stateless Mean?

**Stateless** 指 HTTP 的 request semantics 不要求 server 依靠同一 connection 上之前的对话，才能知道当前 request 的基本含义。

例如：

```http
GET /orders/123 HTTP/1.1
Host: api.example.com
Authorization: Bearer example-token
```

当前 request 带上了 target 和 credentials；server 不需要假设“这个 connection 上的第 3 个 request 一定属于 Alice”。

但这不表示 server 没有任何 state：

- Database 保存 users、orders。
- Session store 保存 login sessions。
- Cache 保存 representations。
- TCP / TLS connection 本身也有 protocol state。

**HTTP stateless、application state 和 transport connection state 是三个不同层面的概念。**

### 1.5 Client, Origin Server, and Intermediaries

```text
Browser → CDN / reverse proxy → load balancer → application server → database
```

| Component | 作用 |
| --- | --- |
| User agent | 发起 HTTP requests，例如 browser、CLI、mobile app |
| Origin server | 对某个 resource 提供 authoritative responses |
| Forward proxy | 代表 client 访问其他 servers |
| Reverse proxy | 在 server 前接收 requests，再转发到 upstream |
| CDN | 在多个 locations 提供内容分发，通常包含 caching 和 proxying |
| Gateway | 与 upstream 交互，可能转换协议或聚合服务 |

Response 可能由 CDN cache 直接返回，根本没到 application server。排错时不能默认所有 status codes 都由自己的 application 生成。

## 2. URL, Origin, and a Request Lifecycle

### 2.1 Anatomy of a URL

```text
https://api.example.com:443/users/42?view=summary#details
  |          |          |     |          |        |
scheme      host       port  path       query   fragment
```

- **Scheme**：这里是 `https`。
- **Host**：用于识别目标 host；通常需要 DNS resolution。
- **Port**：这里显式写出 443；HTTPS 的 default port 是 443，HTTP 是 80。
- **Path**：选择 target resource。
- **Query**：附加参数；含义由 application 定义。
- **Fragment**：例如 page 内部 anchor，由 client 处理，**不会作为 HTTP request target 发给 server**。

以上 URL 对应常见 HTTP/1.1 request target：

```http
GET /users/42?view=summary HTTP/1.1
Host: api.example.com
```

通过 forward proxy 的某些 requests 会使用 absolute-form；`CONNECT` 有自己的 target form，不要把普通 origin-form 当作唯一格式。

### 2.2 Encoding Is Not Encryption

`%20`、URL encoding、Base64 都不提供 confidentiality。

Query parameters 可能进入 browser history、server logs、analytics 等位置。即使使用 HTTPS，也不要把 passwords 或长期 secrets 放进 URL。

Path / query 的编码和解析应交给 URL library。不要随手 string concatenation，否则 `&`、`=`、`#`、Unicode 等都可能改变含义。

### 2.3 Origin vs. Site

对常见 HTTP(S) URLs，**origin = scheme + host + effective port**。

| URL pair | Same origin? | 原因 |
| --- | --- | --- |
| `https://a.example.com/x` 与 `https://a.example.com/y` | Yes | Path 不参与 origin |
| `https://a.example.com` 与 `http://a.example.com` | No | Scheme 不同 |
| `https://a.example.com` 与 `https://b.example.com` | No | Host 不同 |
| `https://a.example.com` 与 `https://a.example.com:8443` | No | Port 不同 |
| `https://a.example.com` 与 `https://a.example.com:443` | Yes | Effective port 相同 |

**Site** 是另一种边界，常见 schemeful same-site 判断主要看 scheme 和 registrable domain。

例如 `https://app.example.com` 与 `https://api.example.com` 通常是 **same-site but cross-origin**。CORS 主要使用 origin；cookie 的 `SameSite` 使用 site。不能混用。

### 2.4 Typing a URL in the Browser

面试时按依赖关系说明，而不是死背每一步一定发生：

1. Browser 解析 URL，检查适用的本地策略、cache、service worker 等。
2. 如果需要 network access，解析 host；DNS result 可能已 cached。
3. 复用可用 connection，或建立新的 TCP/TLS connection；HTTP/3 使用 QUIC。
4. 根据 negotiated HTTP version 发送 request。
5. CDN、proxy 或 origin server 处理 request。
6. Client 收到 status、headers 和可能存在的 body。
7. Browser 根据 response 更新 cache、处理 redirects、解析内容。
8. 如果是 HTML，可能继续请求 CSS、JavaScript、images，并构建和渲染页面。

已有 fresh cache 可以省掉 network request；已有 connection 可以省掉新的 handshake。不能每次都把 DNS + TCP + TLS 的成本全部加一遍。

## 3. HTTP Messages and Message Framing

### 3.1 HTTP/1.1 Request

```http
POST /users HTTP/1.1
Host: api.example.com
Content-Type: application/json
Accept: application/json
Content-Length: 14

{"name":"Ada"}
```

这里展示的 JSON 如果实际末尾**没有 newline**，长度是 **14 bytes**：

```text
{"name":"Ada"} = 14 ASCII / UTF-8 bytes
```

Request 主要包含：

1. **Request line**：method、request target、version。
2. **Headers**：metadata。
3. **Empty line**：headers 结束。
4. **Optional body**：具体 content。

Header names case-insensitive；HTTP/2 和 HTTP/3 的 field names 使用 lowercase。Header values 是否 case-sensitive 要看具体 field，不能把全部 values 都统一 lowercase。

### 3.2 HTTP/1.1 Response

```http
HTTP/1.1 201 Created
Content-Type: application/json
Content-Length: 9
Location: /users/42

{"id":42}
```

- `201` 表示成功创建了 resource。
- `Location` 指向新 resource。
- `Content-Type` 说明 body format。
- `Content-Length` 说明 content 的 byte count。

Reason phrase，例如 `Created`，不是业务逻辑应依赖的内容；client 应按 numeric status 和协议 semantics 处理。HTTP/2 / HTTP/3 没有这种 textual status line。

### 3.3 Why Does HTTP Need Framing?

TCP 只给 byte stream。如果连续收到：

```text
response A bytes + response B bytes
```

Client 必须知道 A 在哪里结束，才能开始解析 B。不能依赖一次 `recv()` 恰好对应一个 response。

HTTP/1.1 的边界与 method、status 和 framing headers 有关，常见情况包括：

- 有效的 `Content-Length`。
- `Transfer-Encoding: chunked`。
- 某些 responses 通过 connection close 表示结束。
- 特殊 method/status 明确没有 response body。

普通 HTTP/1.1 request 没有 `Content-Length` 或 `Transfer-Encoding` 时，不能随意把后续所有 bytes 当作 body；通常解释为 zero-length body。

### 3.4 Content-Length Counts Bytes

对于 UTF-8：

```text
"hello" → 5 bytes
"你好"  → 6 bytes
```

Length 应按最终编码后的 bytes 计算，而不是按字符数。

如果 body 使用 `Content-Encoding: gzip`，`Content-Length` 描述的是 encoded content 的长度。它不是 uncompressed text 的字符数，也不包含 HTTP headers 或 chunk framing overhead。

### 3.5 Chunked Transfer Coding

当 HTTP/1.1 server 开始发送时还不知道完整 body 长度，可以使用 chunked：

```http
HTTP/1.1 200 OK
Content-Type: text/plain
Transfer-Encoding: chunked

5
hello
6
 world
0

```

实际格式中 chunk size、chunk data 等之间使用 CRLF。Size 使用 hexadecimal；这里 `5` 和 `6` 碰巧也等于 decimal 表示。

最终 content 是 `hello world`，总计 11 bytes。`0` chunk 标记结束，随后可以有 trailers，最后 empty line 结束。

**Chunk boundary 不等于 application message boundary。** 一个 JSON object 或 SSE event 可能横跨多个 chunks。

不要把 chunked 与 gzip 混淆：

- Chunked 解决 transfer framing。
- Gzip 解决 content compression。
- 两者在 HTTP/1.1 中可以同时出现。
- HTTP/2 / HTTP/3 使用自己的 frame / stream framing，不使用 HTTP/1.1 chunked transfer coding。

### 3.6 Responses Without a Body

几个必须记住的情况：

- **HEAD response**：没有 body，即使 `Content-Length` 表示对应 GET representation 的长度。
- **204 No Content**：没有 response content；不能发送 `Content-Length`。
- **304 Not Modified**：没有 body；client 复用已有 representation。
- **1xx responses**：interim information，没有普通 response body。
- Successful `CONNECT` 有 tunnel semantics，之后不再按普通 HTTP response body 解释。

**看到 Content-Length 不代表一定要读那么多 response bytes。** HEAD 和 304 是典型反例；304 的 Content-Length 若出现也受对应 representation 长度规则约束。

### 3.7 Ambiguous Framing Is Dangerous

不同 proxies / servers 如果对 message boundary 理解不同，可能产生 **request smuggling** 等问题。

因此不要自行接受 conflicting framing，不要构造同时使用 `Transfer-Encoding` 与 `Content-Length` 的普通 messages。Production 使用成熟 HTTP parsers，并确保 intermediaries 的 parsing policy 一致。

HTTP/1.1 的完整 framing precedence 比“有 length 就读 length”复杂，见 [RFC 9112: Message Body Length](https://www.rfc-editor.org/rfc/rfc9112.html#section-6.3)。

## 4. HTTP Methods and Their Semantics

### 4.1 Common Methods

| Method | 常见意图 | Example |
| --- | --- | --- |
| GET | 获取 selected representation | `GET /users/42` |
| HEAD | 获取 GET 对应的 metadata，不传 body | 检查 file metadata |
| POST | 让 target resource 处理提交的 content | 创建 order、提交 operation |
| PUT | 用提交的 representation 创建或替换 target resource 的 state | `PUT /profiles/42` |
| PATCH | 按 patch document 描述做 partial modification | 更新某些 fields |
| DELETE | 请求移除 target resource 与当前功能的关联 | 删除某个 resource |
| OPTIONS | 查询 communication options | CORS preflight |
| CONNECT | 建立 tunnel | 通过 HTTP proxy 建 HTTPS tunnel |

DELETE 不意味着 server 一定物理擦除 disk bytes；soft deletion、audit history 等属于 application implementation。

PUT 的 replacement semantics 针对 target resource，不代表要替换整张 database table，也不要求 server 原样存储所有输入 fields。

### 4.2 Safe

**Safe method** 的 intended semantics 基本上是读取，不要求 client 发起业务 state change。

GET、HEAD、OPTIONS 是常见 safe methods。

Server 仍然可以记录 access logs、更新 metrics 或 cache。这里的 safe 不表示“实现过程中没有任何 memory/disk write”。

因此下面的 API design 有问题：

```text
GET /transfer-money?amount=100
GET /delete-user?id=42
```

Browser prefetch、link preview 或 crawler 都可能触发 GET。不能要求“大家不要随便点链接”来维持业务正确性。

### 4.3 Idempotent

**Idempotent**：多次相同 requests 的 intended effect，与执行一次的 intended effect 相同。

```text
PUT /settings/theme  {"value":"dark"}
PUT /settings/theme  {"value":"dark"}

Final intended state: dark
```

与之对比：

```text
POST /counter/increment
POST /counter/increment

Counter may increase twice.
```

Idempotent 不要求每次 response 都完全相同：

```text
DELETE /users/42 → 204
DELETE /users/42 → 404

Resource 仍然保持不存在。
```

每次 request 产生独立 access log，也不自动破坏其 idempotent semantics。

### 4.4 Safe, Idempotent, and Cacheable Are Different

| Method | Safe? | Idempotent semantics? | 常见 caching 情况 |
| --- | --- | --- | --- |
| GET | Yes | Yes | 最常见，仍取决于 response / request rules |
| HEAD | Yes | Yes | 可用于 metadata 和 cache validation |
| POST | No | 通常 No | 特定条件允许，但许多 caches 不支持 |
| PUT | No | Yes | Responses 不按普通 HTTP cache reuse |
| PATCH | No | 不保证 | 取决于具体 method rules 和 explicit metadata；不要默认 |
| DELETE | No | Yes | Responses 不按普通 HTTP cache reuse |

不要背“只有 GET 才能 cache”，也不要反过来以为所有 GET responses 都能公开 cache。

Safe 和 idempotent 是 method contract；server 如果错误实现 GET 为扣款，HTTP 并不会自动阻止它。相关定义见 [RFC 9110: Common Method Properties](https://www.rfc-editor.org/rfc/rfc9110.html#section-9.2)。

### 4.5 PUT vs. PATCH

假设当前 profile：

```json
{"name":"Ada","city":"Toronto"}
```

PUT 通常提交 target 的完整期望 state；PATCH 提交 modification instructions。

```text
PATCH: set city = Vancouver
    重复通常仍然是 Vancouver，可能设计成 idempotent。

PATCH: append item to list
    重复可能 append 两次，不保证 idempotent。
```

Patch format 必须明确，例如 JSON Patch 与 JSON Merge Patch 的 semantics 不同。普通 JSON object 本身并没有自动定义 PATCH 的行为。

### 4.6 Does GET Have a Body?

不能简单说“HTTP grammar 禁止 GET body”。

更准确地说：GET request content 没有普遍定义的 semantics，许多 clients、intermediaries、servers 不支持或拒绝它，存在 interoperability 和 framing risk。

普通 API 把简单 read filters 放在 query 中；复杂搜索可定义清晰的 POST search endpoint。不要依赖 GET body 在整条 infrastructure 中都能正常工作。

### 4.7 Why Idempotence Matters for Retries

当 client 没收到 response 时，server 可能已经执行了 operation。

- Idempotent operation 更容易安全 retry。
- 非 idempotent operation 通常需要 application deduplication。
- 相同 method 名称不保证 business implementation 正确。
- Idempotence 也不解决 concurrent writers 的 lost update；第 8 节使用 conditional requests。

Transport uncertainty 的完整推导见 [TCP notes](tcp.md#15-retries-idempotency-and-application-level-reliability)。


## 5. Status Codes

### 5.1 The Five Classes

| Class | 含义 | 面试理解 |
| --- | --- | --- |
| 1xx | Informational | 通常还不是最终 response |
| 2xx | Successful | Request 按该 status 的 semantics 成功处理 |
| 3xx | Redirection | 需要进一步 action；304 是 cache validation 的特殊情况 |
| 4xx | Client error | 当前 request、credentials 或 resource state 存在问题 |
| 5xx | Server error | Server / upstream 未能完成有效 request |

不要把 4xx 理解为“所有责任一定在用户”。例如 application bug 生成了错误 request，也可能得到 400。

### 5.2 Common Success and Informational Codes

| Code | 解释 |
| --- | --- |
| 100 Continue | Client 可以继续发送 request body；常配合 `Expect: 100-continue` |
| 101 Switching Protocols | HTTP/1.1 protocol switch，例如经典 WebSocket handshake |
| 103 Early Hints | 在最终 response 前提供 hints，例如相关资源的 Link |
| 200 OK | Operation 成功；具体 content 含义取决于 method |
| 201 Created | 已创建 resource；常配合 Location |
| 202 Accepted | 已接受处理，**不代表 operation 已经完成或必然成功** |
| 204 No Content | 成功，但没有 response content |
| 206 Partial Content | 成功返回 requested range |

`202` 常用于 background jobs：

```text
POST /reports
    → 202 Accepted
    → Location: /jobs/abc

GET /jobs/abc
    → running / succeeded / failed
```

Job state 是 application response body 中定义的内容，不是 HTTP 自动替你维护的。

### 5.3 Redirects

| Code | Permanent? | Method / body 的要点 |
| --- | --- | --- |
| 301 Moved Permanently | Yes | Historical behavior 允许 POST 在 follow 时改为 GET |
| 302 Found | No | 同样可能把 POST 改成 GET |
| 303 See Other | No | 引导到 retrieval request，常见 POST 后 GET |
| 307 Temporary Redirect | No | 自动 follow 时保持 method / body |
| 308 Permanent Redirect | Yes | 自动 follow 时保持 method / body |

“Temporary / permanent”描述 resource relocation 的语义，不等同于绝对的 cache lifetime。

**POST / Redirect / GET**：

```text
POST /checkout
    → 303 Location: /orders/42
GET /orders/42
    → 200
```

这样刷新结果页面通常是重复 GET，而不是再次提交 checkout。但这不是完整的重复扣款防护；原始 POST 的重试仍需 idempotency design。

`304 Not Modified` 不是“跳转到另一个 URL”，通常没有 Location redirect，也不传 representation body。

### 5.4 Common Client Errors

| Code | 典型含义 / Example |
| --- | --- |
| 400 Bad Request | Malformed request 或无法接受的 request syntax |
| 401 Unauthorized | 缺少有效 authentication；应包含适用的 WWW-Authenticate challenge |
| 403 Forbidden | Server 理解 request，但拒绝执行；常见于权限不足 |
| 404 Not Found | Resource 不存在，或 server 不愿披露其存在 |
| 405 Method Not Allowed | Resource 不支持该 method；带 Allow |
| 409 Conflict | 与当前 resource state 冲突，例如重复业务状态转换 |
| 412 Precondition Failed | If-Match 等 precondition 不成立 |
| 413 Content Too Large | Request content 超过允许大小 |
| 415 Unsupported Media Type | 不支持提交的 Content-Type / content coding |
| 416 Range Not Satisfiable | 请求的 byte range 无法满足 |
| 422 Unprocessable Content | Content syntax 可理解，但 instructions 无法处理 |
| 429 Too Many Requests | Rate limit；可能带 Retry-After |
| 431 Request Header Fields Too Large | Headers 太大，例如异常大的 cookies |

**401 vs. 403** 的记忆法：

- 401：需要有效 authentication，或当前 authentication 无效。
- 403：这个 request 被拒绝；换成相同 credentials 再发通常没用。

403 并不严格意味着 server 一定已认证了你；这只是常见 application usage。对于不想暴露的 resource，也可能故意使用 404。

### 5.5 Common Server Errors

| Code | 谁遇到了什么问题 |
| --- | --- |
| 500 Internal Server Error | Server 内部 unexpected failure |
| 501 Not Implemented | Server 不支持完成 request 所需的功能，例如未知 method |
| 502 Bad Gateway | Gateway / proxy 从 upstream 收到无效 response |
| 503 Service Unavailable | 暂时无法提供服务，例如 overload 或 maintenance |
| 504 Gateway Timeout | Gateway / proxy 等待 upstream 超时 |

例子：

```text
Browser → reverse proxy → application

Application throws unexpected error → often 500
Proxy receives invalid upstream response → often 502
Service temporarily rejects work → often 503
Proxy waits too long for upstream → often 504
```

现实中的 connection failures 如何映射到 502 / 503 等，取决于 proxy implementation。排查时看生成 response 的组件及其 logs。

### 5.6 HTTP Success vs. Business Success

不要只看到 200 就认为所有 business work 都完成了。

例如：

```json
{"job_status":"failed","reason":"input file missing"}
```

这可以是 `GET /jobs/abc` 成功读到的 job state：HTTP retrieval 成功，job 本身失败。

另一方面，如果当前 request 本身因为 validation 被拒绝，却总是返回 200，就会让 monitoring、SDK 和 retry policy 更难正确工作。Status 与 application error model 应保持一致。

## 6. Headers, Content Negotiation, and Compression

### 6.1 A Practical Header Map

| Header | 常见方向 | 用途 |
| --- | --- | --- |
| Host | Request，HTTP/1.1 | 目标 authority，支持同 IP 多个 websites |
| Content-Type | Both | 当前 content 的 media type |
| Content-Length | Both | Content 的 byte length，注意特殊 responses |
| Accept | Request | Client 接受哪些 media types |
| Accept-Encoding | Request | Client 支持哪些 content codings |
| Content-Encoding | Response 常见 | Content 实际使用的 coding，例如 gzip |
| Accept-Language | Request | Preferred languages |
| Authorization | Request | Authentication credentials |
| WWW-Authenticate | Response | Authentication challenge |
| Cookie | Request | Client 发送适用 cookies |
| Set-Cookie | Response | Server 要求 client 设置 cookie |
| Cache-Control | Both | Cache directives；request / response semantics 有区别 |
| ETag / Last-Modified | Response | Validators |
| If-None-Match / If-Match | Request | Conditional request |
| Location | Response | 新 resource 或 redirect target |
| Range / Content-Range | Request / response | Partial content |
| Origin | Request | Browser 发起 request 的 origin context |
| Vary | Response | 哪些 request fields 参与选择 cached variant |

Headers 不是一个可以随意合并的普通 dictionary。某些 fields 可以多行出现并按规则合并；`Set-Cookie` 不能简单用逗号拼成一个值。

### 6.2 Content-Type vs. Accept

```http
POST /documents HTTP/1.1
Host: api.example.com
Content-Type: application/json
Accept: application/pdf
Content-Length: 14

{"name":"Ada"}
```

这表示：

- 我提交的是 JSON。
- 我希望返回 PDF。

两者可以不同。`Accept` 不保证 server 一定支持对应 representation；无法满足时可能返回 406，或按照协议允许的方式忽略 preference。

常见 formats：

- `application/json`：structured data。
- `text/html; charset=utf-8`：HTML。
- `application/octet-stream`：generic binary。
- `application/x-www-form-urlencoded`：form fields。
- `multipart/form-data; boundary=...`：多个 form parts，常用于 upload。

Multipart 的 boundary 是 body format 的组成部分，不是 TCP packet boundary。让 HTTP library 生成 boundary 和匹配的 Content-Type。

### 6.3 Content Compression

```http
Accept-Encoding: gzip, br
```

Server 选择 gzip 后可能返回：

```http
Content-Encoding: gzip
Vary: Accept-Encoding
```

Compression 可以降低 bytes transferred，但消耗 CPU，效果与 data type 有关。已经 compressed 的 JPEG 或 ZIP 通常没有同样的收益。

HPACK / QPACK 压缩 HTTP field sections；gzip / Brotli 压缩 content。它们解决的是不同部分的开销。

### 6.4 Range Requests

```http
GET /video.mp4 HTTP/1.1
Host: media.example.com
Range: bytes=0-999
```

Server 如果支持并满足这个 range，可以返回：

```http
HTTP/1.1 206 Partial Content
Content-Range: bytes 0-999/5000
Content-Length: 1000
```

Ranges 的 endpoints 是 inclusive，所以 0–999 共 1000 bytes。

这能支持 download resume、media seeking 等。Server 可以忽略 Range 并返回完整 200，client 必须处理这种情况。

Resume 时还要考虑 file 已变化：`If-Range` 可以帮助避免把旧 file 的前半段与新 file 的后半段拼在一起。它失败时通常转为发送完整 representation，而不是 412。

## 7. HTTP Caching

### 7.1 Why Cache?

```text
Without cache:
Client → network → server → database → network → client

With a reusable cached response:
Client → cache → client
```

Cache 不只是“把 body 存起来”。要决定是否能 reuse，还需要考虑：

- Target URI / method。
- Response status 和 cache directives。
- Freshness。
- Request headers 对应的 variants。
- Credentials、private/shared cache 的边界。

### 7.2 Private Cache vs. Shared Cache

| Cache | 谁会使用 | Example |
| --- | --- | --- |
| Private cache | 单个 user 的上下文 | Browser HTTP cache |
| Shared cache | 多个 users | CDN、shared proxy |

`/me` 这种 user-specific response，如果错误地被 shared cache 公开复用，可能把 Alice 的数据返回给 Bob。

**Cookie 的存在本身不是“任何 cache 都绝不会缓存”的保证。** 同样，不能以 URL 看起来像 API 为由，假设它不会被 cached。Personalized responses 要设计明确的 cache policy 和 key。

### 7.3 Freshness vs. Validation

两种常见 reuse 路径：

```text
Fresh cached response:
    直接 reuse，通常不访问 origin。

Stale cached response:
    可能进行 conditional request。
    未变化 → 304，reuse old body。
    已变化 → 200，获取 new body。
```

Stale 只表示 freshness lifetime 已过，不表示 data 一定发生变化。

反过来，在 max-age 期限内，origin data 也可能已经变了，但 cache 仍可以按 policy reuse。Cache policy 是对 freshness tradeoff 的约定。

### 7.4 Cache-Control Directives

| Response directive | 意义 |
| --- | --- |
| max-age=60 | Freshness lifetime 为 60 seconds |
| s-maxage=300 | 给 shared caches 的 freshness lifetime；优先于 max-age / Expires |
| public | 显式允许 shared caching，但仍受其他条件约束 |
| private | Shared cache 不应存储该 response；private cache 可以 |
| no-cache | 可以存储，但 reuse 前必须成功 validation |
| no-store | 不应存储该 request / response 的缓存内容 |
| must-revalidate | Stale 后，不能在未成功 validation 的情况下随意 reuse |
| immutable | Fresh 时表示 content 不会变化，适合 fingerprinted assets |
| stale-while-revalidate=30 | 允许在指定 stale window 内先返回旧内容并后台 revalidate |

`immutable` 和 `stale-while-revalidate` 是扩展 directives；支持与组合行为要看 cache implementation。

重点：**no-cache ≠ no-store**；**max-age=0 ≠ no-store**。后者仍可保存 representation 并在下次 revalidate。见 [MDN: Cache-Control](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control)。

### 7.5 Age, Date, and Expires

```http
Cache-Control: public, max-age=60
Age: 45
```

作为简化模型，这个 response 大约剩 15 seconds freshness，而不是 client 收到后重新获得完整的 60 seconds。

正式 age calculation 会考虑 Date、Age、network delay 和 cache residence time，不只是“当前时间减我收到 response 的时间”。

`Expires` 使用 absolute timestamp；`max-age` 使用 relative lifetime，优先级更高。Shared cache 还要考虑 `s-maxage`。

### 7.6 Vary and Cache Keys

假设同一个 URL 支持 English 和 French：

```http
Vary: Accept-Language
```

Cache 不能把 English response 直接复用给要求 French 的 request，需要区分相关 request field values。

类似地：

```http
Vary: Accept-Encoding
```

可以区分 compressed representations。

但 Vary 越多，variants 越多，cache hit rate 可能越低。尤其 `Vary: Cookie` 可能使 variants 非常多；它也不能替代完整的 authorization design。

### 7.7 Practical Policies

| Resource | 示例 policy | Tradeoff |
| --- | --- | --- |
| 带 content hash 的 public JS/CSS | `public, max-age=31536000, immutable` | 内容变化必须换 URL |
| 需要及时更新的 public HTML | `no-cache` + validator | 可 reuse body，但通常需要 round trip |
| 可在 browser 保存的 profile | `private, no-cache` | 每次 reuse 前验证，避免 shared storage |
| 不希望 HTTP caches 保存的敏感 response | `no-store` | 放弃 HTTP cache reuse |
| 允许短时间滞后的 public feed | 短 max-age，可结合 stale-while-revalidate | 以 freshness 换 latency / availability |

这些是设计示例，不是对所有 applications 的默认配置。

`no-store` 也不表示 server logs、screenshots、application 自己的 storage 或所有 browser history 都消失。它控制的是 HTTP caching 行为。

### 7.8 Invalidation and Negative Caching

如果 `/users/42` 改了，`/users?page=1` 的 cached response 可能也受影响。

HTTP cache 对成功 unsafe requests 有相关 invalidation rules，但它不会理解“这条 database row 影响了哪些其他 URLs”。跨 resources 的 invalidation 仍需要 application / CDN strategy。

404 等 responses 在一定条件下也可以被 cached。这叫 **negative caching**，因此“刚创建的页面仍然 404”有时是 cache issue。

更完整的 caching rules 见 [RFC 9111](https://www.rfc-editor.org/rfc/rfc9111.html)。

## 8. Conditional Requests and Optimistic Concurrency

### 8.1 ETag Is a Validator

```http
HTTP/1.1 200 OK
ETag: "profile-v7"
Cache-Control: private, no-cache
Content-Type: application/json
```

ETag 是 server 给 selected representation 的 opaque validator。可以由 version number、hash 或其他机制产生；client 不应解析它的内部格式。

ETag 不一定是 MD5，也不自动提供 authenticity 或 security。

### 8.2 If-None-Match and 304

第一次 GET：

```text
Client → GET /profile
Server → 200 + ETag "profile-v7" + body
Client → store body and ETag
```

再次 validation：

```http
GET /profile HTTP/1.1
Host: api.example.com
If-None-Match: "profile-v7"
```

如果 representation 没变：

```http
HTTP/1.1 304 Not Modified
ETag: "profile-v7"
Cache-Control: private, no-cache
```

Client 复用之前保存的 body，并按 rules 更新 cached metadata。

304 **节省 body transfer，不保证消除 RTT，也不保证 server 完全不访问 database**。生成或验证 ETag 的成本取决于 implementation。

### 8.3 Last-Modified and If-Modified-Since

另一个 validator：

```http
Last-Modified: Wed, 16 Sep 2026 12:00:00 GMT
```

Client 后续发送：

```http
If-Modified-Since: Wed, 16 Sep 2026 12:00:00 GMT
```

Timestamps 的 granularity 和修改时间管理可能限制精度。ETag 更容易表达“representation version 是否一致”。

GET 同时带 `If-None-Match` 与 `If-Modified-Since` 时，前者优先；不是要求两个条件都各自独立决定结果。

### 8.4 Strong vs. Weak ETags

```text
Strong: "abc123"
Weak:   W/"abc123"
```

- Strong validator 要满足更严格的 representation equivalence，可用于要求准确匹配的操作。
- Weak validator 允许语义等价但 bytes 不完全相同的 representations。
- GET cache validation 中的 If-None-Match 使用 weak comparison。
- If-Match 使用 strong comparison，因此 weak ETag 不适合它。

不要让不同 encoded variants 随便共用同一个 strong ETag，除非确实满足 strong validator 的要求。

### 8.5 The Lost Update Problem

```text
Initial: profile version 7

Alice GET → version 7
Bob   GET → version 7

Alice edits city and saves → version 8
Bob edits name using old copy and saves → may overwrite Alice's city
```

即使两次 PUT 都是 idempotent，也不能防止这个问题。Idempotence 讨论重复同一 operation 的效果，不是 concurrent modification control。

### 8.6 If-Match and 412

Bob 提交时带上他读到的 version：

```http
PUT /profile HTTP/1.1
Host: api.example.com
If-Match: "profile-v7"
Content-Type: application/json
Content-Length: 14

{"name":"Bob"}
```

如果 server 当前已经是 version 8：

```http
HTTP/1.1 412 Precondition Failed
Content-Length: 0
```

Bob 应重新 GET，看到最新数据后 merge 或让用户解决 conflict，而不是无条件覆盖。

**Version check + write 必须是 atomic operation。**

```sql
UPDATE profiles
SET name = 'Bob', version = version + 1
WHERE id = 42 AND version = 7;
```

如果 affected row count 为 0，可能是 version mismatch 或 resource 不存在；application 结合自己的语义处理。

不能先无保护地检查 version，再单独 update：两个 writers 可能同时通过检查。

### 8.7 Create Only If Absent

```http
PUT /documents/report-42 HTTP/1.1
Host: api.example.com
If-None-Match: *
Content-Length: 0
```

表示只在 target 没有 current representation 时执行这个 operation。

这个 condition 失败时是 precondition failure，通常为 412；**If-None-Match 不是所有 methods 失败时都返回 304**。304 对应 GET / HEAD validation 的情况。


## 9. Persistent Connections and HTTP Versions

### 9.1 HTTP/1.0 and HTTP/1.1

HTTP/1.0 的基础模型通常每次 exchange 后关闭 connection，也有 keep-alive extensions。

HTTP/1.1 默认支持 persistent connections：只要 framing 正确、双方允许且 connection 没有关闭，就可复用。

```text
New connection:
    TCP handshake → optional TLS handshake → request / response

Reused connection:
    request / response
```

Keep-alive 不意味着永久保持，也不意味着 application session。Client、proxy、server 都可能因 idle timeout、resource limits 或 shutdown 关闭 connection。

**HTTP connection persistence、TCP keepalive probes 和 login session 是不同概念。**

### 9.2 HTTP/1.1 Pipelining

Pipelining 允许不等前一个 response，就发送后面的 requests：

```text
Client sends:     request A, request B, request C
Server responds:  response A, response B, response C
```

Responses 仍按 requests 的顺序返回。A 很慢，即使 B 已准备好，也不能让 B 的 response 随意插到 A 前面。

这是 HTTP 层面的 head-of-line blocking。实践中 clients 也常通过多个 connections 并行请求，而不是依赖 pipelining。

### 9.3 HTTP/2

HTTP/2 保留 methods、status codes 等 HTTP semantics，改变 wire representation：

- Binary framing。
- 多个 streams multiplex 在一个 connection 上。
- HPACK 压缩 field sections。
- Stream-level 和 connection-level flow control。
- 使用 pseudo-headers，例如 `:method`、`:path`、`:authority`、`:status`。

```text
One HTTP/2 connection:
    Stream 1: request A → DATA A1, DATA A2
    Stream 3: request B → DATA B1
    Stream 5: request C → DATA C1, DATA C2

Frames from different streams can interleave.
```

Stream ID 让 receiver 知道哪个 frame 属于哪个 exchange。这里不再依赖 HTTP/1.1 textual message 连续排列的方式。协议定义见 [RFC 9113](https://www.rfc-editor.org/rfc/rfc9113.html)。

### 9.4 HTTP/2 Still Has TCP Head-of-Line Blocking

```text
TCP bytes carrying:
    stream A data
    missing TCP segment
    stream B data already arrived
```

TCP 必须按 sequence order 把 byte stream 交给上层。Missing bytes 后面的数据即使属于 HTTP/2 的另一个 stream，也可能暂时不能交付。

所以：

- HTTP/2 缓解 HTTP/1.1 response ordering 的阻塞。
- HTTP/2 over TCP 仍受 TCP ordered delivery 的跨 stream 影响。

另外，streams 共享 congestion control、bandwidth 和 connection resources，不代表每个 stream 获得独立网络容量。

### 9.5 HTTP/3

HTTP/3 把 HTTP semantics 映射到 **QUIC**，QUIC 使用 UDP 作为底层承载。

它不是“HTTP 改用 unreliable delivery”。Reliability、loss recovery、congestion control、secure streams 等由 QUIC 实现。

主要特点：

- 多个 independent streams。
- 某个 stream 缺少 bytes，不必要求其他 streams 等它补齐后才能交付自己的 data。
- 使用 QPACK 压缩 field sections。
- QUIC 集成 TLS 1.3 handshake。
- Connection IDs 支持在满足条件时进行 connection migration，例如 network path 改变。

这不意味着完全没有 blocking：单个 stream 内仍有 ordered delivery，streams 仍共享 bandwidth / congestion resources，QPACK dependencies 也可能引入等待。见 [RFC 9114](https://www.rfc-editor.org/rfc/rfc9114.html)。

### 9.6 Version Comparison

| Aspect | HTTP/1.1 | HTTP/2 | HTTP/3 |
| --- | --- | --- | --- |
| 常见 transport | TCP | TCP | QUIC over UDP |
| Wire framing | Textual start line / headers，body 可 binary | Binary frames | HTTP frames over QUIC streams |
| Concurrent exchanges on one connection | Pipelining 有 response ordering 限制 | Multiplexed streams | Multiplexed streams |
| Field compression | 无 HPACK / QPACK | HPACK | QPACK |
| Cross-stream transport HOL | TCP ordered delivery | TCP ordered delivery | 避免 TCP 式跨 stream delivery 阻塞 |
| Security in web deployment | 可 HTTP 或 HTTPS | Browser 部署通常使用 TLS | QUIC 使用 TLS 1.3 |

HTTP/3 不保证任何场景都更快。Loss pattern、RTT、implementation、CPU、connection reuse、UDP reachability 都会影响结果。

### 9.7 Version Negotiation and Connection Management

HTTPS 上，ALPN 可以协商例如 `h2` 或 `http/1.1`。HTTP/3 endpoints 可通过 Alt-Svc 或 DNS HTTPS records 等机制发现。

Browser 到 proxy 与 proxy 到 origin 可以使用不同 versions：

```text
Browser --HTTP/3--> CDN --HTTP/2--> origin
```

HTTP/2 / HTTP/3 不使用 HTTP/1.1 的 `Connection: keep-alive` 机制，也不能直接转发这些 connection-specific fields。

Graceful shutdown 时，HTTP/2 / HTTP/3 可使用 GOAWAY 表达 connection 不再接受某些新 work。Retry 是否安全仍取决于 request 是否可能被处理，以及 method / application semantics。

## 10. HTTPS and TLS

### 10.1 What HTTPS Adds

HTTPS 是 HTTP 使用 secured transport，而不是一个完全不同的 application data model。

TLS 主要提供：

- **Confidentiality**：旁观者不能直接读出 HTTP content。
- **Integrity**：篡改传输内容会被检测。
- **Authentication**：常见配置下 client 验证 server identity；也可配置 client certificates。

TLS 不自动保证 server 的业务逻辑正确，也不会替 application 决定 Alice 能否读取 Bob 的 orders。

### 10.2 A Simplified TLS 1.3 Handshake

以下是常见 certificate-based full handshake 的概念图：

```text
Client                                      Server
  | -- ClientHello + key share -------------> |
  | <--- ServerHello + key share ------------- |
  | <--- encrypted handshake messages ------- |
  |      certificate, proof, Finished         |
  | -- Finished ----------------------------> |
  | <===== protected application data ======> |
```

双方协商参数，通过 key exchange 派生 traffic keys。Certificate 和签名用于认证，不是每个 HTTP byte 都用 certificate public key 去加密。

Bulk application data 使用 symmetric authenticated encryption。不要把旧式 RSA key transport 的图机械套到 TLS 1.3 上。

Full TLS 1.3 handshake 通常可在一个 TLS RTT 后让 client 发送普通 application data；若基于新 TCP connection，还需考虑 TCP establishment。Session resumption、early data 和实际调度会改变 latency。协议细节见 [RFC 8446](https://www.rfc-editor.org/rfc/rfc8446.html)。

### 10.3 Certificate Validation

Client 需要检查：

- Certificate 是否能 chain 到 trusted authority。
- Requested hostname 是否匹配 certificate identity。
- Validity period。
- 适用的 usage constraints、签名和其他 validation policy。

只“拿到一张 certificate”不够。跳过 verification 会破坏 server authentication，因此示例 code 不应靠关闭 verification 来修复连接错误。

Certificate 验证的是相应 identity 与 key 的关系，不是对网站所有内容的可靠性背书。

### 10.4 What HTTPS Hides and Does Not Hide

加密后，普通网络旁观者不能直接看到 HTTP path、query、headers、body。

但 IP addresses、traffic timing、packet sizes 等通常仍可见。Hostname 是否暴露还取决于 DNS transport、TLS ClientHello 是否使用适用的保护机制等。

因此不要说“HTTPS 让任何人都不知道你连到了哪个服务”。

### 10.5 TLS Termination

```text
Browser --TLS A--> reverse proxy --TLS B--> application
```

Proxy 在终止 TLS A 后能读取 HTTP content，再建立独立的 TLS B。这里是两个 security connections。

如果第二段是 plaintext，第一段 HTTPS 并不会自动保护第二段。面试讨论 load balancer 时，应明确 TLS 在哪里终止、upstream 是否再次加密。

### 10.6 HSTS

Server 可以用 `Strict-Transport-Security` 告诉 browser，在 policy 有效期内使用 HTTPS 访问适用 host。

HSTS 可以减少后续被降级到 HTTP 的风险，但第一次访问还没获得 policy 时存在 bootstrap 问题；preload 是一种额外机制。

HSTS 不是让 server 支持 TLS 的替代品，也不是 authorization mechanism。定义见 [RFC 6797](https://www.rfc-editor.org/rfc/rfc6797.html)。

### 10.7 0-RTT and Replay

某些 resumed connections 可发送 early data，减少等待，但 **0-RTT data 有 replay risk**。

因此不能因为 transport 支持 early data，就把未经防护的扣款、下单操作随意放进去。Application 需要限制哪些 requests 能 early-send，并考虑 replay-safe handling。

## 11. Cookies, Sessions, and Tokens

### 11.1 Cookies

Server 返回：

```http
Set-Cookie: session_id=opaque-example; Path=/; Secure; HttpOnly; SameSite=Lax
```

Browser 保存后，在符合 cookie scope 和 policy 的后续 request 中发送：

```http
Cookie: session_id=opaque-example
```

Cookie 是 client storage + automatic request attachment mechanism。Cookie value 可以是 session ID，也可以是其他 application data。

Cookie 并不自动加密内容；HTTPS 保护传输，server 仍要验证其真实性和权限。

### 11.2 Cookie Attributes

| Attribute | 作用 | 不保证什么 |
| --- | --- | --- |
| Secure | Cookie 仅在符合 secure transport 规则时发送 | 不防止已在 origin 内执行的恶意 script 发 authenticated requests |
| HttpOnly | 不向 document.cookie 等 script API 暴露 | 不等于消除所有 XSS 后果 |
| SameSite | 限制 cross-site contexts 中的发送 | 不等于完整 CSRF defense |
| Domain | 控制适用 host scope | 不是 user authorization |
| Path | 按 request path 限制发送 | 不是安全隔离边界 |
| Max-Age / Expires | 控制 client-side lifetime | 不决定 server session 必须同时有效 |

没有 Domain 时通常是 host-only cookie；指定允许的 parent domain 可以扩大到 subdomains。Server 不能随意给无关 domain 设置 cookie。

`SameSite=None` 要求 `Secure`；browser 的第三方 cookie policy 仍可能进一步限制发送。见 [MDN: Using HTTP Cookies](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/Cookies)。

### 11.3 Server-side Sessions

```text
Browser cookie:
    session_id = random opaque value

Server session store:
    session_id → user_id, expiration, authentication state
```

流程：

1. Client 提交 login credentials。
2. Server 验证 credentials。
3. Server 创建 session，设置 session cookie。
4. 后续 requests 携带 session ID。
5. Server 查 session store，再做 resource-level authorization。

Session 可以存 memory、database 或 distributed store。多 application instances 时，需要考虑 shared session storage，或有意设计 routing / replication。

不要把“server 维护 session”与“每个 request 必须使用原来的 TCP connection”混淆。

### 11.4 Session Lifecycle

实际设计要考虑：

- Login / privilege change 时 rotation，降低 session fixation 风险。
- Idle timeout 和 absolute expiration。
- Logout 时 server-side invalidation。
- Password reset、account suspension 后如何 revoke。
- Cookie 删除时使用匹配的 scope。

只删除 browser cookie，不一定让其他持有该 session ID 的 clients 失效。Server session 的生命周期需要独立管理。

### 11.5 Bearer Tokens

```http
Authorization: Bearer example-access-token
```

Bearer 表示“持有这个 token 的一方可使用相应 authority”，所以 token 泄露具有直接风险。

Bearer token 可以是 opaque string，也可以是 JWT。**Bearer 是使用方式，JWT 是格式，不是同义词。**

Authentication 验证 token / session；authorization 还要检查当前 user 是否有权操作这个具体 resource。

### 11.6 JWT

常见 signed JWT 的 compact form：

```text
base64url(header).base64url(payload).signature
```

Payload 通常可被 decode，**signed 不等于 encrypted**。不要把 password 或 secret 放进去，误以为别人读不到。

Verifier 需要按自己的 trust policy 检查 signature、allowed algorithm、issuer、audience、expiration 等，而不是只 decode payload。

JWT 也可以使用其他保护形式，例如 encrypted JWT；不能把所有 JWT 都说成永远只有三段。JWT 定义见 [RFC 7519](https://www.rfc-editor.org/rfc/rfc7519.html)。

### 11.7 Sessions vs. Self-contained Tokens

| Aspect | Server-side session | Self-contained signed token |
| --- | --- | --- |
| 每次验证 | 常需 session store lookup | 可能本地验证 signature / claims |
| Immediate revocation | 删除 / 标记 session 较直接 | 常需 denylist、短 expiry 或其他 state |
| Payload update | Server state 可立刻更新 | 已发 token 的 claims 可能过时 |
| Size | Opaque ID 通常较小 | Claims / signature 增加 request size |
| Scaling tradeoff | Shared store availability / latency | Key distribution、rotation、revocation complexity |

JWT 不会让 application 完全无 state。Accounts、permissions、refresh token lifecycle、revocation 都可能仍然有 state。

“Cookie vs. JWT”也不是同一维度的比较：JWT 可以放在 cookie 里，opaque token 也可以放在 Authorization header 里。

### 11.8 OAuth and OpenID Connect

- **OAuth 2.0**：delegated authorization framework，例如允许某个 app 获得访问 API 的有限权限。
- **OpenID Connect**：建立在 OAuth 2.0 上的 identity layer，用于 authentication 等场景。
- **Access token**：供 resource server 判断 API access。
- **ID token**：供 client 理解 authentication event / identity claims，不能默认拿它替代 API access token。

面试先分清这几种职责，再解释实际 flow；不要把所有“第三方登录”都简化为“传一个 JWT”。详见 [OpenID Connect Core](https://openid.net/specs/openid-connect-core-1_0.html)。

### 11.9 Basic Authentication

`Authorization: Basic ...` 通常携带 Base64 编码的 username/password 组合。Base64 不是 encryption；没有 TLS 时 credentials 可被读出。

Basic 与 Bearer 都是 HTTP authentication schemes，但 credentials 的含义不同。无论采用哪种，都不能把 authentication 成功当作对所有 resources 自动具有 authorization。

## 12. Same-Origin Policy, CORS, CSRF, and XSS

### 12.1 Same-Origin Policy

Browser 对 scripts 跨 origin 访问资源施加限制。SOP 的重点之一是避免任意 website 的 script 读取另一个 origin 的 authenticated response。

它不是“任何 cross-origin network request 都禁止”。Images、forms、navigation 等存在不同规则，部分 requests 可以发送，只是 response 不会暴露给发起页面的 script。

### 12.2 CORS

假设：

```text
Frontend: https://app.example.com
API:      https://api.example.com
```

两者 cross-origin。Frontend script 想读取 API response，需要符合 CORS policy：

```http
Origin: https://app.example.com
```

API response 可以带：

```http
Access-Control-Allow-Origin: https://app.example.com
Vary: Origin
```

如果 server 动态根据 Origin 选择允许的 response policy，Vary: Origin 能帮助 shared cache 区分 variants。

Server 必须根据可信 allowlist 决定是否允许，不能无条件 echo 任意 Origin 然后开放 credentials。

### 12.3 Preflight

某些 cross-origin requests 需要先发 OPTIONS，例如使用 PUT、Authorization header 或 application/json content type 的常见 requests。

```http
OPTIONS /profile HTTP/1.1
Host: api.example.com
Origin: https://app.example.com
Access-Control-Request-Method: PUT
Access-Control-Request-Headers: content-type, authorization
```

Server 可以回复：

```http
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, PUT
Access-Control-Allow-Headers: Content-Type, Authorization
Vary: Origin
```

Browser 确认 policy 允许后，才发送实际 PUT。

Preflight 是 browser permission check，不是实际业务 operation。Actual response 仍然需要相应 CORS headers。

### 12.4 Credentialed Requests

如果 cross-origin fetch 需要 cookies，client 通常要设置：

```javascript
// Browser fragment; API must separately permit this origin and credentials.
fetch("https://api.example.com/profile", {
  credentials: "include"
});
```

Server 需要对应的：

```http
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Credentials: true
```

Credentialed CORS response 不能用 `Access-Control-Allow-Origin: *`。

Cookie 是否发送还受 SameSite、Secure、scope 和 browser policy 影响。`credentials: "include"` 不会绕过这些限制。Preflight request 本身通常不带 user credentials。

### 12.5 CORS Does Not Authenticate Callers

`curl`、backend service、mobile native clients 不按 browser SOP 的方式限制 API 调用。

因此：

- CORS 不能阻止任意 non-browser client 访问 public endpoint。
- Origin header 不是 API credential。
- Server 仍需 authentication 和 authorization。
- 对不需要 preflight 的 requests，CORS failure 可能发生在 server 已处理 request 之后。

如果需要从 cross-origin response 读取非 safelisted headers，例如 ETag，server 还可能需要 `Access-Control-Expose-Headers: ETag`。Browser CORS rules 见 [MDN: CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS)。

### 12.6 CSRF

**CSRF** 利用 browser 会自动附带某些 credentials 的特性，诱导用户从另一个 site 发起用户并不打算执行的 operation。

攻击者不一定需要读到 response。如果扣款 request 已经被接受，CORS 阻止 script 读取 body 也不等于撤销扣款。

常见 defenses：

- CSRF tokens，并在 server 验证。
- 根据用途配置 SameSite cookies。
- 对敏感 operations 校验 Origin / Referer 等 request context。
- 不用 GET 执行 state-changing operations。
- 对重要 actions 做额外确认或 re-authentication。

CSRF token 与 session cookie 不应被简单地当作“两个会自动发送的同类值”；关键是 server 验证攻击者不能正确提供的 request proof。具体模式见 [OWASP: CSRF Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)。

### 12.7 XSS

**XSS** 是不可信 content 被当作 script / executable markup，在受害 origin 的 context 中运行。

与 CSRF 不同，XSS code 已经处在被信任的页面 context 中。它可能读取 script-accessible data，或直接发 authenticated requests。

Defenses 包括 contextual output encoding、safe DOM APIs、适当的 sanitization 和 CSP 等。HttpOnly 能减少 cookie 被 script 直接读取的风险，但不能让整个 application 对 XSS 免疫。

### 12.8 Keep the Boundaries Separate

| Mechanism | 主要解决什么 |
| --- | --- |
| TLS | 传输过程中保护 communication / authenticate peer |
| Authentication | 当前 caller 是谁，credentials 是否有效 |
| Authorization | 当前 caller 是否能做这个 operation |
| CORS | Browser 是否允许 script 读取 / 发起某些 cross-origin exchanges |
| CSRF defenses | 防止利用自动 credentials 伪造用户意图 |
| XSS defenses | 防止不可信内容成为 trusted-origin executable code |

任何一个机制都不能替代其他所有机制。


## 13. Reliable HTTP API Design

### 13.1 HTTP vs. REST vs. JSON

- **HTTP** 是 protocol。
- **REST** 是 architectural style，包含 uniform interface、stateless interaction、cacheability 等约束。
- **JSON** 是 data format。

用 HTTP + JSON 不自动代表严格符合 REST。面试讨论具体 API 时，先把 resource model、semantics 和 failure behavior 讲清楚，比只贴“RESTful”标签更有用。

### 13.2 Resource-oriented Routes

```text
GET    /orders            → list / search orders
POST   /orders            → create order
GET    /orders/42         → retrieve order
PATCH  /orders/42         → modify allowed fields
DELETE /orders/42         → remove / cancel according to contract
```

不是所有 business actions 都能简单当成任意 field update。比如 payment capture 应有明确的 state machine、permissions 和 retry semantics。

Server 必须独立验证 resource ownership；不能因为 route 是 `/users/42/orders` 就信任 caller 是 user 42。

### 13.3 Idempotency Keys

Client 对同一个 logical operation 使用稳定 key：

```http
POST /payments HTTP/1.1
Host: api.example.com
Idempotency-Key: payment-attempt-abc
Content-Type: application/json
Content-Length: 14

{"amount":100}
```

这里 `Idempotency-Key` 代表 API 明确定义的 application contract；不能假设所有 HTTP servers 自动支持同样规则。

一个可用 implementation 需要考虑：

1. Key 的 scope，例如 user / tenant + endpoint + key。
2. Key 与 request payload 是否匹配。
3. 两个相同 key 的 concurrent requests 如何 atomic deduplicate。
4. Operation 正在处理时，第二个 request 返回什么。
5. 保存什么 result，以及保存多久。
6. Key expiration 后重试是否可能再次执行。
7. Database change 与外部 side effect 如何协调。

只用一个“先查 cache，没有就执行”的流程，可能被 concurrent requests 同时穿过。持久化 unique constraint / transaction 能保护本地 dedup record，但外部 payment call 仍需配套设计。

### 13.4 Retry Policy

| Situation | 常见处理方向 |
| --- | --- |
| Invalid syntax / validation error | 修复 request，通常不原样 retry |
| Invalid credentials | 按 auth flow 处理，避免无限刷新 / 重试 |
| 429 / some 503 | 遵循 Retry-After，结合有限 backoff |
| Connection loss / timeout | Outcome 可能 unknown；检查 idempotence / key contract |
| 409 / 412 | 重新读取 state，解决 conflict |
| 502 / 504 | 可能是 transient，也可能 operation 已执行；不要盲目重复写 |

`Retry-After` 可以是 delay seconds 或 HTTP-date。它不是允许无限 retry 的指令。

通常设置 total deadline、最大 attempts、exponential backoff 和 jitter。多个 layers 各自 retry 会放大流量：

```text
Client retries 3 times × gateway retries 3 times
    → one user operation may trigger up to 9 upstream attempts
```

表中“3 times”这里指各层总共最多 3 attempts；实际文档中要明确 retries 是否包含第一次 attempt。

### 13.5 Timeouts and Cancellation

区分：

- Connect timeout。
- TLS handshake timeout。
- Response headers timeout。
- Body read / idle timeout。
- Whole-operation deadline。

收到 headers 不代表 body 已完整。某些 streaming responses 本来就持续很久，不能套普通短请求相同的 idle policy。

Client timeout / cancellation 不保证 server operation 自动 rollback。Server 可能继续执行；需要结合 cancellation propagation 和业务状态查询。

### 13.6 Pagination

```text
Offset pagination:
    GET /orders?offset=100&limit=20

Cursor pagination:
    GET /orders?after=opaque-cursor&limit=20
```

Offset 易理解，但大 offsets 可能代价高，并且 concurrent insert/delete 会导致跨页重复或遗漏。

Cursor 通常基于稳定排序位置，例如 `created_at + unique_id`。它也不自动提供 snapshot consistency；如果需要“所有 pages 看的是同一时刻的数据”，要额外定义 snapshot / version semantics。

Limit 上限、排序规则、cursor expiry、authorization 都应明确。

### 13.7 Error Responses and Observability

一个 consistent error response 可以包含：

```json
{
  "code": "VERSION_CONFLICT",
  "message": "The profile changed. Reload before saving.",
  "request_id": "request-example"
}
```

- Machine-readable code 供 client 分支处理。
- Message 供开发者 / 用户理解。
- Request ID 用于关联 logs / traces。
- 不泄露 credentials、stack traces 或内部 connection strings。

HTTP status 仍应表达适当的协议层结果，不需要把所有 failures 都塞到 200 body 中。

### 13.8 Limits and Backpressure

HTTP server 不应无界接受 work。常见 limits 包括：

- Header size、body size、upload duration。
- Concurrent requests / streams。
- Queue depth、worker pool size。
- Per-user / per-tenant rate limits。
- Downstream connection pool capacity。

HTTP/2 multiplexing 让很多 requests 共用 connection，不会自动增加 database capacity。Thread pool、queues 和 backpressure 的基础见 [Process & Thread](../os/process-thread.md) 与 [Synchronization](../os/synchronization.md)。

## 14. Streaming, Polling, SSE, and WebSocket

### 14.1 Buffering vs. Streaming

Buffered response：先生成完整结果，再发送。

Streaming response：边产生 content，边发送。

Streaming 能降低 first-byte / first-result latency，避免把巨大 body 一次放进 memory，但增加 partial failure handling 的复杂度。

```text
Server sends 200 headers
Server sends half the content
Server fails
```

此时通常不能把已发送的 status 改成 500。Client 必须能识别 incomplete content，或由 application stream format 定义 error event。

Proxy buffering、compression buffers、client rendering behavior 都可能影响实际可见 latency。“Server 调了 write”不等于“用户立刻看到”。

### 14.2 Short Polling

```text
Every 5 seconds:
    GET /job/42
```

简单、容易与现有 infrastructure 集成，但更新 latency 与 polling interval 有关；大量空轮询也会浪费 requests。

### 14.3 Long Polling

Client 请求后，server 等有新 event 或 timeout 才返回；client 收到后再发下一次。

它减少空轮询，但需要管理 outstanding requests、timeouts 和 reconnect。一个 outstanding HTTP request 不一定占用一个 OS thread，取决于 server 是否使用 async I/O 等模型。

### 14.4 Server-Sent Events

**SSE** 是通过 HTTP response 持续发送 text events，主要用于 server → client updates。

```http
Content-Type: text/event-stream
Cache-Control: no-cache
```

Body 示例：

```text
id: 42
event: progress
data: {"percent":50}

id: 43
event: progress
data: {"percent":75}

```

Empty line 结束一个 event；不是一个 TCP packet 结束一个 event。

Browser 的 EventSource 支持 reconnect，并可携带 Last-Event-ID 帮助 server resume。Server 仍需决定保存多久的 history、如何 replay 和 deduplicate；这不自动保证 exactly-once delivery。

SSE 可以基于不同 HTTP versions 传输，不是 HTTP/2 server push。Event format 与 browser behavior 见 [WHATWG: Server-sent Events](https://html.spec.whatwg.org/multipage/server-sent-events.html)。

### 14.5 WebSocket

WebSocket 提供持续的 bidirectional messaging，适用于 interactive communication。

经典 HTTP/1.1 establishment：

```text
Client → HTTP Upgrade request
Server → 101 Switching Protocols
Then   → WebSocket frames in both directions
```

Upgrade 后，后续 frames 不再是普通 HTTP request/response messages。WebSocket 也有基于较新 HTTP versions 的不同 establishment mechanisms，不能把所有情况都说成必须 101。

WebSocket 定义 message framing，但底层仍可能分段，large message 也可能由多个 frames 构成。仍需 authentication、authorization、heartbeat、message size limits 和 reconnect policy。基础协议见 [RFC 6455](https://www.rfc-editor.org/rfc/rfc6455.html)。

### 14.6 Which One Should I Use?

| Need | 常见候选 |
| --- | --- |
| 偶尔查后台 job 状态 | Polling |
| 希望少发空 requests，又沿用普通 HTTP | Long polling |
| Server 持续向 browser 推送 progress / notifications | SSE |
| 双方高频交互，例如 collaborative editing | WebSocket |

这是按 communication pattern 选择的起点，不是绝对规定。

WebSocket 不天然更省资源；大量 idle connections 也需要 memory、connection tracking 和 load balancer support。

## 15. Runnable Python: GET, HEAD, ETag, and 304

### 15.1 What This Example Demonstrates

下面 code 可以整体保存为 `http_cache_demo.py`，使用 Python 3 直接运行。只监听 `127.0.0.1`，port 由 OS 自动选择，不访问外部 network，不修改 repository files。

它演示：

1. 第一次 GET 返回 200、JSON body 和 ETag。
2. Client 保存 body 和 ETag。
3. 相同 ETag 的 conditional GET 返回 304，没有 body。
4. Client 自己复用 cached body。
5. HEAD 没有 body，但 Content-Length 是对应 GET 的 byte count。
6. 不匹配的 ETag 返回完整 200。
7. 不存在的 route 返回 404。

**Scope:** 只实现本示例需要的单个、exact strong ETag comparison，没有实现完整 If-None-Match grammar（例如 lists / weak tags / wildcard）、自动 HTTP cache 或 production security。真实框架应使用完整的 conditional request implementation。

Python `http.server` 适合这样的本地 teaching demo，不作为 production server。见 [Python http.server documentation](https://docs.python.org/3/library/http.server.html)。

### 15.2 Complete Code

```python
import hashlib
import json
import threading
from http.client import HTTPConnection
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
from urllib.parse import urlsplit

# Encode first; Content-Length must count bytes, not Python characters.
BODY = json.dumps(
    {"name": "Ada", "city": "多伦多"},
    ensure_ascii=False,
    separators=(",", ":"),
).encode("utf-8")
ETAG = '"' + hashlib.sha256(BODY).hexdigest() + '"'


class Handler(BaseHTTPRequestHandler):
    protocol_version = "HTTP/1.1"

    def setup(self):
        super().setup()
        self.connection.settimeout(5)

    def log_message(self, format, *args):
        pass  # Keep the teaching output deterministic.

    def do_GET(self):
        self.respond(head_only=False)

    def do_HEAD(self):
        self.respond(head_only=True)

    def respond(self, head_only):
        if urlsplit(self.path).path != "/profile":
            body = b'{"error":"not found"}'
            self.send_response(404)
            self.send_header("Content-Type", "application/json")
            self.send_header("Cache-Control", "no-store")
            self.send_header("Content-Length", str(len(body)))
            self.end_headers()
            if not head_only:
                self.wfile.write(body)
            return

        # Deliberately supports only the exact tag sent by this demo client.
        if self.headers.get("If-None-Match") == ETAG:
            self.send_response(304)
            self.send_header("ETag", ETAG)
            self.send_header("Cache-Control", "private, no-cache")
            self.end_headers()
            return  # A 304 has no response body.

        self.send_response(200)
        self.send_header("Content-Type", "application/json")
        self.send_header("Cache-Control", "private, no-cache")
        self.send_header("ETag", ETAG)
        self.send_header("Content-Length", str(len(BODY)))
        self.end_headers()
        if not head_only:
            self.wfile.write(BODY)


def main():
    server = ThreadingHTTPServer(("127.0.0.1", 0), Handler)
    thread = threading.Thread(target=server.serve_forever, daemon=True)
    thread.start()
    conn = HTTPConnection("127.0.0.1", server.server_port, timeout=5)

    try:
        conn.request("GET", "/profile")
        response = conn.getresponse()
        cached_body = response.read()
        tag = response.getheader("ETag")
        assert response.status == 200
        assert cached_body == BODY
        assert int(response.getheader("Content-Length")) == len(BODY)
        assert tag == ETAG
        print("GET:", response.status, cached_body.decode("utf-8"))

        conn.request("GET", "/profile", headers={"If-None-Match": tag})
        response = conn.getresponse()
        received = response.read()
        assert response.status == 304
        assert received == b""
        assert response.getheader("ETag") == tag
        print("Conditional GET:", response.status, "body bytes =", len(received))
        print("Reused cache:", json.loads(cached_body)["city"])

        conn.request("HEAD", "/profile")
        response = conn.getresponse()
        received = response.read()
        assert response.status == 200
        assert received == b""
        assert int(response.getheader("Content-Length")) == len(BODY)
        print("HEAD:", response.status, "body bytes =", len(received))

        conn.request(
            "GET", "/profile", headers={"If-None-Match": '"old-version"'}
        )
        response = conn.getresponse()
        assert response.status == 200
        assert response.read() == BODY
        print("Changed validator:", response.status)

        conn.request("GET", "/missing")
        response = conn.getresponse()
        assert response.status == 404
        assert json.loads(response.read()) == {"error": "not found"}
        print("Missing route:", response.status)

    finally:
        conn.close()
        server.shutdown()
        server.server_close()
        thread.join(timeout=5)


if __name__ == "__main__":
    main()
```

预期 output：

```text
GET: 200 {"name":"Ada","city":"多伦多"}
Conditional GET: 304 body bytes = 0
Reused cache: 多伦多
HEAD: 200 body bytes = 0
Changed validator: 200
Missing route: 404
```

### 15.3 What the Code Is Actually Doing

`ThreadingHTTPServer` 在 background thread 中接受 connections。Client 使用同一个 `HTTPConnection` 顺序发送 requests，并在下一个 request 前读完前一个 response。

`response.read()` 在这里不会自己猜测 TCP packet boundary。HTTP library 根据 status、method 和 framing 处理 body，所以 HEAD 和 304 正确返回 empty bytes。

`http.client` 不是自动 browser cache。示例里的 `cached_body` 由 application 自己保存；304 后自己决定 reuse。这能清楚区分 **HTTP validation signal** 与 **client cache implementation**。

### 15.4 Why UTF-8 Matters Here

`多伦多` 是 3 个 Unicode characters，但在 UTF-8 中占 9 bytes。

Code 先 `.encode("utf-8")`，再 `len(BODY)`。如果对 JSON string 直接 `len()` 并作为 Content-Length，包含 non-ASCII content 时就可能错误。

### 15.5 Follow-up Exercises

- 把 code 中的 profile 内容改掉，再运行，观察 ETag 变化。
- 增加 conditional HEAD，验证 304 同样不传 body。
- 增加 PUT + If-Match，并用 lock 或 database conditional update 保证 check-and-write atomic。
- 将 response policy 改为 max-age，思考真正的 cache client 何时可以省掉 request。

示例每次启动都会重新创建 server，不能用它证明跨运行保留了 cache；要研究 persistent cache，需要另外保存 metadata 和 body。


## 16. Troubleshooting and Performance

### 16.1 Decompose Latency

```text
User-visible latency may include:

cache lookup
+ DNS, if needed
+ connection establishment, if needed
+ TLS, if needed
+ request transmission
+ proxy / server queueing
+ application / database processing
+ response transfer
+ client parsing / rendering
```

不要因为页面慢就直接得出“TCP 慢”。一个 SQL query 可能已经占了 90% 的时间。

**TTFB** 是 time to first byte。它不是单纯的 application execution time，具体工具的计时起点也要确认。

### 16.2 Browser DevTools

Network panel 中可以观察：

- Method、status、protocol version。
- Request / response headers。
- Redirect chain。
- Timing breakdown。
- Transferred size vs. resource size。
- Response 是否来自 memory cache、disk cache 或 service worker。
- Cookie / CORS errors。

排 cache 问题时注意 DevTools 的 **Disable cache** 会影响实验结果，通常只在 DevTools 打开时生效，且不等于清除所有 application storage 或 service worker caches。

### 16.3 curl on Windows

下面是 command templates，需要把 URL 换成正在运行的服务。第 15 节 demo 会自行退出，所以不能在它退出后继续访问。

PowerShell 中显式使用 `curl.exe`，避免某些环境把 `curl` 解析成其他命令的 alias：

```powershell
# Show response headers and body.
curl.exe -i http://localhost:8000/profile

# Make a HEAD request.
curl.exe -I http://localhost:8000/profile

# Show connection / request / response details.
curl.exe -v http://localhost:8000/profile

# Follow redirects.
curl.exe -L -i http://localhost:8000/old-path
```

Verbose output 可能包含 credentials 和 cookies，分享 debug logs 前应移除这些 values。

Curl 不执行 browser 的 CORS access checks。因此“curl 成功而 browser 失败”可能是 CORS 或 browser-specific policy，不能直接证明 server authentication 没问题。

### 16.4 A Symptom-driven Checklist

| Symptom | 优先检查 |
| --- | --- |
| Browser 报 CORS，但 server log 有 request | 是否 request 已执行，只是 response 不允许 script 读取 |
| 总是看到旧内容 | Cache-Control、Age、Vary、service worker、CDN invalidation |
| 304 但页面没数据 | Client 是否实际保存了对应 body，cache key 是否一致 |
| Content-Length mismatch | 是否按 characters 计数、compression 后 length 是否变化 |
| 401 | Credentials、expiry、issuer / audience、authentication challenge |
| 403 | Resource authorization、policy、origin / CSRF validation 等 |
| 502 / 504 | 哪层生成 response、upstream connectivity / latency、proxy timeout |
| 大量 429 / 503 | Rate limit、queue saturation、retry amplification |
| Streaming 很久才显示 | Proxy buffering、compression buffers、client consumption |
| 偶尔 duplicate order | Timeout 后 retry、idempotency key、atomic deduplication |

### 16.5 Optimize After Measuring

| Dominant cost | 可能有用的方向 |
| --- | --- |
| Repeated identical data transfer | Cache、ETag、appropriate freshness |
| Large text payload | Compression、减少不必要 fields |
| Many small resources | Connection reuse、HTTP/2 / HTTP/3、减少 requests |
| Slow database / application | Query optimization、避免重复 work、capacity analysis |
| Long physical distance | CDN、regional placement |
| Queueing under load | Backpressure、bounded concurrency、capacity / overload handling |

提高 HTTP version 不能修好所有 bottlenecks；把 server-side 2 seconds query 改成 HTTP/3，并不会让 query 本身变成 20 ms。

## 17. Common Interview Questions

这一节是短答练习；完整推导在前文。先自己口头回答，再对照要点，避免只记 keywords。

### Q1. What is HTTP?

**English answer:** HTTP is an application-layer request/response protocol that defines methods, status codes, fields, and the semantics of exchanging resource representations.

补充时说明 HTTP version 影响 wire format / transport mapping，而不是把所有 application semantics 全部重新定义。

### Q2. Why is HTTP called stateless if websites have login sessions?

HTTP 不要求依赖 connection 上此前的对话来解释当前 request。Application 可以通过 cookie / token 把当前 request 关联到 session state；两者不矛盾。

### Q3. Does every HTTP request create a new TCP connection?

No。HTTP/1.1 可以复用 connection；HTTP/2 在 connection 上 multiplex streams；HTTP/3 使用 QUIC，因此也不能笼统说所有 HTTP 都建立 TCP。

### Q4. GET vs. POST?

GET 请求 retrieval，具有 safe / idempotent semantics；POST 让 target 处理提交的 content，通常不保证 idempotence。区别不只是“参数放 URL 还是 body”。

### Q5. Safe vs. idempotent?

Safe 关注 intended semantics 是否要求业务 state change。Idempotent 关注重复相同 request 的 intended effect 是否与一次相同。DELETE 通常 idempotent，但不 safe。

### Q6. Why can DELETE be idempotent if it returns 204 and then 404?

Idempotence 不要求 responses 完全相同。执行一次和多次后的 intended state 都是 resource 不再存在。

### Q7. PUT vs. PATCH?

PUT 表达 target 的创建 / replacement；PATCH 表达 partial modifications。PATCH 是否 idempotent 取决于 patch operations，不能一概而论。

### Q8. What does 202 mean?

Request 已接受，处理可能尚未完成。需要 job status 或其他 application mechanism 得知最终成功或失败。

### Q9. 401 vs. 403?

401 表示需要有效 authentication，带适用 challenge；403 表示 request 被拒绝，常见原因是权限不足，但不局限于“已登录”。

### Q10. 301/302 vs. 307/308?

307/308 要求自动 follow 时保持 method。301/302 存在允许 POST 改 GET 的历史语义。Permanent pair 是 301/308。

### Q11. What is 304?

GET / HEAD conditional validation 表明 selected representation 不需要重新传 body。Client 使用自己已有的对应 body，不是把空 304 body 当成 resource 内容。

### Q12. no-cache vs. no-store?

no-cache 可以存储，但 reuse 前必须 validation。no-store 禁止相应 HTTP cache storage。两者解决不同需求。

### Q13. Why does a response with max-age=60 not always have 60 seconds left when it reaches the browser?

它可能已经在 shared cache 中停留。Age 与相关 timing rules 决定 current age；freshness 不会在每一跳自动归零。

### Q14. ETag vs. Last-Modified?

ETag 是 opaque representation validator；Last-Modified 基于 modification timestamp。ETag 可以表达 timestamp granularity 不易区分的 versions。

### Q15. How can HTTP prevent lost updates?

Client 提交 If-Match，server atomic check version + write；不匹配返回 412。HTTP condition 本身需要 application / storage 正确执行。

### Q16. Does 304 eliminate network latency?

No。Validation 仍然有 request / response，通常省掉 body transfer。Fresh cache 的本地 reuse 才可能完全避免这次 network request。

### Q17. What does Vary do?

告诉 cache 哪些 request fields 会影响 representation selection。忽略 Vary 可能把错误的 language 或 encoding variant 返回给 client。

### Q18. Why does HTTP/2 still experience head-of-line blocking?

它在 HTTP 层 multiplex streams，但底层 TCP 仍按 byte order 交付。一个 missing segment 可能影响同 connection 上多个 streams。

### Q19. How does HTTP/3 improve this?

QUIC 提供独立 streams 的 delivery，一条 stream 缺 data 不必阻止另一条交付自己的完整 data；仍有 shared bandwidth / congestion 和其他可能的 dependencies。

### Q20. What does HTTPS protect?

保护 transport confidentiality、integrity 和 peer authentication。它不代替 application authorization，也不保证 server 没有 vulnerabilities。

### Q21. Why is TLS not just public-key encryption for every byte?

Asymmetric mechanisms 用于 key establishment / authentication；bulk content 使用高效的 symmetric authenticated encryption。不同 TLS versions / handshake modes 的具体步骤不同。

### Q22. Cookie vs. session vs. JWT?

Cookie 是 browser storage / attachment mechanism；session 是 application interaction state；JWT 是 token format。它们可以组合使用。

### Q23. Does HttpOnly prevent XSS?

No。它限制 script 直接读取 cookie，但 origin 内执行的恶意 script 仍可能发 authenticated requests 或读取其他 data。

### Q24. CORS vs. CSRF?

CORS 控制 browser 的 cross-origin access；CSRF 关注攻击者利用自动 credentials 伪造用户操作。无法读 response 不代表无法造成 state change。

### Q25. Why does application/json often trigger preflight?

它不是 CORS-safelisted Content-Type 之一。还要结合 method / headers / origin 判断；不是所有 POST 都 preflight，也不是所有 GET 都免 preflight。

### Q26. Can I retry POST after a timeout?

先考虑 outcome unknown：server 可能已执行。需要明确的 idempotency / dedup contract，不能只根据“没收到 response”判断没有副作用。

### Q27. 502 vs. 504?

502 是 gateway / proxy 获取到无效 upstream response；504 是等待 upstream 超时。结合实际 proxy logs 判断具体原因。

### Q28. SSE vs. WebSocket?

SSE 适合 server → client text event stream，沿用 HTTP；WebSocket 提供持续 bidirectional messaging。两者都需要处理 disconnect / reconnect 和 application delivery semantics。

### Q29. Why can Content-Length be nonzero for HEAD even though there is no body?

它可以描述对应 GET representation 的 byte length。Client 必须结合 method semantics 解析，不能看到 length 就盲目等 body。

### Q30. A payment timed out, and the user clicks again. How would you design the API?

把同一 logical payment 关联到稳定 idempotency key；atomic dedup，明确 in-progress / completed behavior，协调 external payment side effect；client 查询 result 或按 contract retry，并设置 bounded backoff。

不要只回答“加个 mutex”：多个 processes / hosts 和 crash recovery 都可能参与。

### Q31. Two users edit the same profile. Is using PUT enough?

No。PUT idempotence 不防止 concurrent lost update。需要 version-based conditional write，例如 If-Match + atomic version check，或者其他 concurrency control。

### Q32. The browser is slow, but the API has only 20 ms of processing time. What next?

分解 DNS / connection / TLS / queueing / transfer / client rendering，检查 cache 和 request waterfall。先找到占主要时间的阶段，再选择 optimization。

### Q33. What is the difference between same-origin and same-site?

Origin 主要看 scheme + host + port；schemeful site 使用 scheme + registrable domain 等规则。两个 subdomains 可以 same-site 但 cross-origin，影响 CORS 与 SameSite 判断。

### Q34. Can a response be cached if it is not a 200?

可以。Cacheability 取决于 method、status 和 directives。某些 redirects、404 等也能在适用规则下被 cached。

### Q35. Does a successful HTTP response prove the data is durable?

不能普遍推出。要看该 API 对 success 的 contract：可能只是接受任务，也可能已经 commit transaction。HTTP status 本身不会让 application 自动获得 durability。

## References and Further Reading

本文以面试常见 semantics 为主，implementation-specific defaults 需要查所用 framework / proxy 的文档。

- [RFC 9110 — HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110.html)
- [RFC 9111 — HTTP Caching](https://www.rfc-editor.org/rfc/rfc9111.html)
- [RFC 9112 — HTTP/1.1](https://www.rfc-editor.org/rfc/rfc9112.html)
- [RFC 9113 — HTTP/2](https://www.rfc-editor.org/rfc/rfc9113.html)
- [RFC 9114 — HTTP/3](https://www.rfc-editor.org/rfc/rfc9114.html)
- [RFC 5789 — PATCH Method](https://www.rfc-editor.org/rfc/rfc5789.html)
- [RFC 8446 — TLS 1.3](https://www.rfc-editor.org/rfc/rfc8446.html)
- [RFC 6797 — HSTS](https://www.rfc-editor.org/rfc/rfc6797.html)
- [OpenID Connect Core](https://openid.net/specs/openid-connect-core-1_0.html)
- [Python http.client](https://docs.python.org/3/library/http.client.html)
