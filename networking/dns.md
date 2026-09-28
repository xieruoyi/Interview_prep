# DNS and Name Resolution

面试准备笔记：**English terminology + 中文解释**。先建立 namespace 和 resolution 的模型，再理解 caching、transport、security，以及 DNS 如何影响真实 application。

前置知识：[TCP and Socket Programming](tcp.md)、[HTTP, HTTPS, and Web Communication](http.md)。TCP handshake、HTTP caching 和 TLS 的细节在对应 notes 中；本篇专注 DNS 本身。

**Examples:** 协议和 zone-file snippets 用于说明概念，不是 deployment configuration。Documentation addresses 如 `192.0.2.20`、`2001:db8::20` 不能当作 public services 访问。第 14 节有两个可以独立运行的 Python examples，其中 TTL example 完全离线。

**Contents**

- [1. What Is DNS?](#1-what-is-dns)
- [2. Domain Names, Hierarchy, and Zones](#2-domain-names-hierarchy-and-zones)
- [3. The Participants in DNS Resolution](#3-the-participants-in-dns-resolution)
- [4. Recursive and Iterative Resolution](#4-recursive-and-iterative-resolution)
- [5. Resource Records and Record Types](#5-resource-records-and-record-types)
- [6. Delegation, Glue, and Zone Operations](#6-delegation-glue-and-zone-operations)
- [7. DNS Caching and TTL](#7-dns-caching-and-ttl)
- [8. DNS Messages and Transport](#8-dns-messages-and-transport)
- [9. Results, Errors, and Failure Semantics](#9-results-errors-and-failure-semantics)
- [10. DNS-based Traffic Routing and Availability](#10-dns-based-traffic-routing-and-availability)
- [11. DNS Security and DNSSEC](#11-dns-security-and-dnssec)
- [12. Encrypted DNS and Query Privacy](#12-encrypted-dns-and-query-privacy)
- [13. How Applications Use DNS](#13-how-applications-use-dns)
- [14. Runnable Python Examples](#14-runnable-python-examples)
- [15. Troubleshooting on Windows and Other Systems](#15-troubleshooting-on-windows-and-other-systems)
- [16. Common Interview Questions and Scenarios](#16-common-interview-questions-and-scenarios)

## 1. What Is DNS?

### 1.1 A Distributed Naming System

**DNS — Domain Name System** 是一个 hierarchical、distributed naming system。

最常见的用途是查询 domain name 对应的 IP addresses：

```text
www.example.com
        |
        | DNS resolution
        v
192.0.2.20
        |
        | application establishes a connection
        v
HTTP / TLS / other application protocol
```

但 DNS 并不只是一个 `name → IP` dictionary。它还保存 mail routing、delegation、service information、verification data 等不同类型的信息。

更准确的 query 模型是：

```text
(name, record type, class) → relevant DNS data or an error / negative result

(www.example.com., A, IN)    → IPv4 addresses
(www.example.com., AAAA, IN) → IPv6 addresses
(example.com., MX, IN)       → mail exchange information
```

`IN` 是 Internet class；面试中的普通 DNS 大部分都在讨论这个 class。

### 1.2 Why Not Use IP Addresses Directly?

Name 与 address 分离后：

- 用户 / application 使用较稳定、可理解的 name。
- Service 可以更换 IP，而不要求所有 users 修改配置。
- 同一个 name 可以映射到多个 addresses。
- 同一个 IP 可以为多个 names 提供服务。
- 不同 administrative domains 可以独立管理自己的 namespace。

但 DNS 不保证 address 永远有效，也不保证该 address 上的 application 健康。

```text
DNS success ≠ TCP connection success
DNS success ≠ TLS certificate validation success
DNS success ≠ HTTP request success
```

### 1.3 DNS Is Not a Proxy

普通 DNS resolution 后，client 直接连接得到的 destination：

```text
Client ---- DNS query ----> Resolver
Client <--- IP address ---- Resolver

Client ---- HTTPS connection ----> Web service
```

Resolver 通常不在之后的 HTTP data path 中。

CDN provider 可能同时运营 DNS 和 reverse proxies，但这是两个不同 roles。不能因为 DNS 返回了 CDN address，就说“HTTP response 经过 DNS server”。

### 1.4 DNS Is Not a URL Redirect

DNS 通常处理 name，不处理 URL 的 path、query 或 fragment。

```text
https://shop.example.com/products/42?color=blue#reviews
        ----------------
        DNS 关心这里的 host name
```

DNS CNAME 不会把 browser address bar 自动改成另一个 URL，也不能直接实现 `/old-path → /new-path`。后者通常是 HTTP redirect 或 application routing。

### 1.5 Why a Distributed Hierarchy?

如果所有 names 都放在一台 central server：

- 容易成为 availability bottleneck。
- 更新需要集中协调。
- Query volume 和 network distance 难以承受。
- 各组织不能独立维护自己的 data。

DNS 使用 delegation 分散 ownership，用 replication 提高 availability，用 caching 降低重复查询成本。

理解重点是 **authority 分层、查询可分步完成、结果可缓存**。基础模型见 [RFC 1034](https://www.rfc-editor.org/rfc/rfc1034.html)。

## 2. Domain Names, Hierarchy, and Zones

### 2.1 Labels and the Root

```text
www.api.example.com.
 |   |     |     | |
 labels    |    TLD root
```

从右往左看 hierarchy：

```text
.                       root
└── com.
    └── example.com.
        └── api.example.com.
            └── www.api.example.com.
```

最后的 dot 表示 root label。日常 URLs 经常省略它，但 DNS notation 中 `www.example.com.` 是 explicit absolute name。

**FQDN — Fully Qualified Domain Name** 指完整表达 namespace 位置的 name。在配置文件和 troubleshooting commands 中使用 trailing dot，可以避免某些 relative-name / search-suffix 歧义。

### 2.2 Names Are Not Ordinary Case-sensitive Strings

DNS label comparison 对 ASCII letters 不区分大小写：

```text
WWW.Example.COM. ≈ www.example.com.
```

这不表示 URL 中的 path 也不区分大小写。DNS host naming 与 HTTP resource path 是不同层次。

常规 DNS labels 最长 63 octets；完整 encoded name 最长 255 octets，包括 label length bytes 和 root terminator。不要把 wire-format 的 255 直接当成任何 Unicode name 都能写 255 characters。

Internationalized names 需要 IDNA 处理，常见 ASCII form 带 `xn--`。Display name 和 DNS 使用的 representation 不一定完全相同。

### 2.3 Domain vs. Zone

**Domain** 是 namespace 中某个 node 及其下方 subtree。

**Zone** 是一组由某个 authority 管理的数据，边界由 delegation 决定。

例子：

```text
example.com domain
├── www.example.com
├── mail.example.com
└── dev.example.com
    ├── api.dev.example.com
    └── db.dev.example.com
```

如果 `dev.example.com` 被 delegated：

```text
example.com zone:
    example.com
    www.example.com
    mail.example.com
    delegation information for dev.example.com

dev.example.com zone:
    dev.example.com
    api.dev.example.com
    db.dev.example.com
```

Parent zone 不再负责 child zone 中所有 records，但仍保留把查询引向 child 的 delegation information。

**Subdomain 不一定是独立 zone。** 只有 name 层级更深，不代表存在新的 authoritative boundary。

### 2.4 Zone Apex

Zone 起点称为 **zone apex**：

```text
Zone: example.com.
Apex: example.com.
```

Zone apex 通常有 SOA 和 NS records。这与后面“为什么普通 CNAME 不能随便放在 apex”有关。

### 2.5 Registrar, Registry, and DNS Hosting

| Role | 做什么 |
| --- | --- |
| Registrant | 持有 / 使用注册 domain 的个人或组织 |
| Registrar | 提供 domain registration / renewal 等服务 |
| Registry | 管理某个 TLD 下的 registration data |
| DNS hosting provider | 托管 authoritative zone data |
| Web hosting provider | 提供 website / application infrastructure |

这些 roles 可以由同一公司提供，也可以完全分离。

“我改了 domain 的 nameservers”是在改变 delegation；“我改了 A record”是在改 zone data；“我换了 web hosting”则是 service deployment。三者可能有关联，但不是同一操作。

## 3. The Participants in DNS Resolution

### 3.1 Application and Stub Resolver

Application 通常调用系统或 runtime API：

```text
Browser / application
        |
        | getaddrinfo("www.example.com", ...)
        v
OS / runtime name-resolution path
        |
        v
Configured recursive resolver
```

**Stub resolver** 通常不会自己从 root 一路追查，而是把复杂 resolution work 交给 recursive resolver。

但 browser 也可能使用自己的 DNS cache / DoH resolver，绕过部分 OS DNS path。具体 route 取决于 configuration。

### 3.2 Recursive Resolver

Recursive resolver 接收“帮我找到最终结果”的 query，然后：

- 查 cache。
- 必要时查询其他 DNS servers。
- 处理 referrals 和 aliases。
- 根据配置进行 DNSSEC validation。
- 返回 result 或 failure。

来源可以是 ISP、enterprise network、local router 转发服务、public DNS provider，或本机运行的 resolver。

注意“recursive”描述它给 client 提供的服务；它向 authoritative servers 发的 queries，通常是 iterative resolution 的一部分。

### 3.3 Authoritative Nameserver

Authoritative server 对自己负责的 zones 提供 authoritative data。

例如管理 `example.com.` zone 的 servers 可以回答其中的 A、MX、TXT 等 records，或返回 child delegation。

它不一定帮 client 查询其他无关 domains。很多 public authoritative servers 会关闭 unrestricted recursion。

### 3.4 Root Servers

Root servers 对 root zone authoritative，帮助 resolver 找到 TLD nameservers。

它们通常不会直接返回任意 website 的最终 IP。

```text
Question: Where can I continue resolving something under .com?
Root:     Here are the .com nameservers.
```

Root system 有 13 个 named server identities，不是全世界只有 13 台 physical computers。它们通过 distributed instances / anycast 提供服务。列表见 [IANA Root Servers](https://www.iana.org/domains/root/servers)。

### 3.5 TLD Nameservers

TLD servers 对 `.com.`、`.org.` 等相应 zones authoritative，返回下面 domains 的 delegation information。

```text
Root → .com nameservers
.com → example.com nameservers
example.com → www.example.com address
```

这是常见路径示意。不同 zones、aliases、cache state 和 forwarding arrangements 会让实际查询次数不同。

### 3.6 Roles Are Logical, Not Necessarily Separate Machines

一台 server 可以承担多个 roles；同一个 role 也可以由很多 machines 实现。

但在解释时先分清职责：

| Role | “它主要知道什么？” |
| --- | --- |
| Stub | 我应该问哪个 resolver |
| Recursive resolver | 如何查下去，以及已有 cache |
| Root authoritative | Root zone 和 TLD delegations |
| TLD authoritative | TLD zone 和其下 delegations |
| Domain authoritative | 该 zone 的 records / child delegations |

不要把 resolver 与 authoritative server 都简称“DNS server”后，就默认它们做完全相同的事情。

## 4. Recursive and Iterative Resolution

### 4.1 A Cold-cache Lookup

假设 resolver 没有相关 cached data，client 要查 `www.example.com. A`：

```text
Client              Recursive resolver       Authoritative hierarchy
  |                          |                         |
  | -- final answer please ->|                         |
  |                          | -- ask root ----------> |
  |                          | <- referral to .com --- |
  |                          | -- ask .com ----------> |
  |                          | <- referral to example -|
  |                          | -- ask example.com ---> |
  |                          | <- A 192.0.2.20 -------- |
  | <- A 192.0.2.20 ---------|                         |
```

Diagram 展示 authority progression，不要求每一跳都发送完全相同的完整 QNAME；启用 QNAME minimization 时会尽量减少向上层暴露的 name 信息。

这不是“root 替你问 TLD，TLD 再替你问 domain”这条服务器调用链。常见情况下，是 recursive resolver 逐个跟进 referrals。

### 4.2 Recursive Query

Client 向 recursive resolver 表达：

> 请你帮我完成 resolution，返回结果或失败。

Wire format 中 `RD` 表示 Recursion Desired；server 的 `RA` 表示 Recursion Available。

RD 是 request intention，不是强制命令；server 可以因为 policy、availability 或其他原因不提供 recursion。

### 4.3 Iterative Resolution

Iterative server response 可能告诉 requester：

> 我不能直接提供最终答案，但这里有下一步应该问的 authoritative servers。

Requester 自己继续 query。

**Referral 不是 HTTP redirect。** 它不包含让 browser follow 的 URL，而是 DNS delegation information。

### 4.4 Warm-cache Lookup

如果 resolver 已缓存最终 answer：

```text
Client → resolver cache → answer
```

如果只缓存了 `example.com` 的 delegation：

```text
Client → resolver → example.com authoritative → answer
```

因此每次 DNS lookup 不一定经过 root 和 TLD。大量重复 queries 都由不同层的 caches 处理。

### 4.5 Forwarding Resolvers

Corporate resolver 或 home router 可能把 query 转发到另一个 recursive resolver：

```text
Client → router DNS forwarder → ISP / public recursive resolver
```

它自己不一定完整执行 root-to-authoritative iteration。Forwarding 和 full recursion 是 deployment choices。

### 4.6 Bootstrap: How Does a Resolver Find the Root?

完整 recursive resolver 通常带有 root hints，包含已知 root-server addresses，并通过 priming 等机制获得 root information。

如果必须先通过同一个未知 DNS path 查询“DNS server 的 IP”，就会循环依赖。因此 resolver endpoint / bootstrap addresses / local configuration 必须有适当起点。

Encrypted DNS 也没有自动消除 bootstrap 问题：DoH hostname 对应的 endpoint 仍需要可达路径。

### 4.7 Alias Chasing

```text
www.example.com.       CNAME edge.example.net.
edge.example.net.     A     192.0.2.30
```

Resolver 可能需要沿 target 查下去，甚至跨到另一个 zone。

最终 response 可能包含 CNAME chain 和 address records；也可能需要额外 queries。不是每个 CNAME hop 都必然增加一个 network RTT，因为相关 data 可能已经 cached 或一起返回。

Aliases 形成 loop 或过长 chain 时，resolution 可能失败。设计上应避免无意义的 chain，而不是假设 resolver 会无限追下去。


## 5. Resource Records and Record Types

### 5.1 Record Structure

Zone-file notation 中常见形式：

```text
NAME                 TTL    CLASS  TYPE  RDATA
www.example.com.     300    IN     A     192.0.2.20
```

- **NAME / owner name**：record 属于哪个 name。
- **TTL**：可正常缓存的 lifetime，以 seconds 表示。
- **CLASS**：这里为 IN。
- **TYPE**：数据种类。
- **RDATA**：该 type 对应的具体 data。

相同 owner name、class、type 的 records 组成 **RRset**。

```text
www.example.com.  300 IN A 192.0.2.20
www.example.com.  300 IN A 192.0.2.21
```

这是一个包含两个 A records 的 RRset。RRset 应使用一致的 TTL；它不是两个互不相关、可任意混合版本的 fields。

### 5.2 A and AAAA

| Type | 保存什么 | Example |
| --- | --- | --- |
| A | IPv4 address | `192.0.2.20` |
| AAAA | IPv6 address | `2001:db8::20` |

A 和 AAAA 是不同 RRsets，有独立 query / cache state。

```text
A query succeeds
AAAA query returns NODATA
```

可以仅仅表示该 name 有 IPv4，没有 IPv6；不表示整个 domain 不存在。

一个 name 可以同时有 A 和 AAAA。Client 决定具体尝试哪些 addresses、顺序如何、是否并行，DNS 不替它建立 connection。

### 5.3 CNAME

```text
www.example.com.  300 IN CNAME origin.example.net.
```

CNAME 表示 owner name 是另一个 canonical name 的 alias。

要点：

- Target 是 name，不是 URL，不包含 `https://` 或 path。
- CNAME owner 通常不能同时有其他普通 data，例如 A、MX；DNSSEC-related metadata 有特定例外。
- CNAME 不改变 HTTP request 的原始 hostname identity。
- CNAME chain 的每个 RRset 有自己的 TTL。

如果 browser 访问 `www.example.com`，DNS CNAME 最终找到 address，TLS 仍需验证 `www.example.com` 对应 identity；HTTP routing 也仍需正确处理原始 authority。

### 5.4 Why Not a CNAME at the Zone Apex?

Apex 必须承载 SOA / NS 等 data，与普通 CNAME 的 exclusive ownership rule 冲突。

某些 providers 提供 `ALIAS`、`ANAME`、CNAME flattening 等 features，让 apex 看起来能指向另一个 name，但其实现可能是代为解析并返回 A / AAAA。

这些 provider features 不等于所有 DNS clients 都支持一个通用的“apex CNAME 例外”。应检查 provider 的 update、TTL 和 DNSSEC behavior。

### 5.5 MX

```text
example.com.  300 IN MX 10 mail1.example.com.
example.com.  300 IN MX 20 mail2.example.com.
```

MX 用于 mail routing：

- Number 是 preference，通常较小值优先。
- Target 是 mail exchanger hostname，不是 email address。
- Sender 还需要解析 exchanger 的 A / AAAA。
- 同 preference 可用于分配 mail delivery attempts；不是严格按比例的通用 load balancer。
- MX target 应直接有适当 address records，不应依赖它本身是 CNAME。

MX priority 与“IP packet priority”无关。

### 5.6 NS

```text
example.com.  86400 IN NS ns1.example.net.
example.com.  86400 IN NS ns2.example.net.
```

NS 指定 authoritative nameservers。

Parent 的 delegation NS 和 child apex 的 NS 应保持一致，但它们是处于不同 zones 的数据，可能在 migration 时出现不同更新进度。

NS target 是 hostname，resolver 仍需要找到其 IP。某些情况下由 glue 帮助 bootstrap，见第 6 节。

### 5.7 SOA

**SOA — Start of Authority** 保存 zone 的重要 metadata：

```text
example.com. IN SOA ns1.example.net. hostmaster.example.com. (
    2026092701 ; serial
    3600       ; refresh
    600        ; retry
    1209600    ; expire
    300        ; minimum / negative caching input
)
```

| Field | 作用 |
| --- | --- |
| MNAME | Zone 的 primary source 相关 name |
| RNAME | Responsible mailbox 的 DNS notation |
| Serial | Zone version，用于 secondary 判断更新 |
| Refresh | Secondary 正常检查更新的 interval |
| Retry | 更新检查失败后的 retry interval |
| Expire | 无法成功刷新达到该期限后，secondary 不应继续正常 authoritative serving |
| MINIMUM | 现代 negative caching calculation 的一个 input |

这里 `hostmaster.example.com.` 表示 mailbox notation，概念上是 `hostmaster@example.com`。含特殊字符的 mailbox 需要额外 escaping。

Serial 的日期形式只是常见 convention，不是必须格式。它也不是所有 clients 都持有的 global version。

**SOA MINIMUM 不是所有 records 的默认 TTL。** Zone-file 默认 TTL 通常由 `$TTL` 等机制指定。

### 5.8 TXT

```text
example.com.  300 IN TXT "verification=example-value"
```

TXT 保存 text strings，具体含义由使用它的 protocol / application 定义。常见用途包括 domain verification、SPF、DKIM-related data 等。

TXT 不是 secret store。Public zone 中的数据通常可以被 queries 读取；不要放 passwords 或 private keys。

一个 TXT RR 的 RDATA 可以包含多个 character-strings。不要把“单个 string 的长度限制”误记成“整个 TXT record 永远只能这么长”。

### 5.9 PTR and Reverse DNS

Forward lookup：

```text
www.example.com. → 192.0.2.20
```

IPv4 reverse lookup：

```text
20.2.0.192.in-addr.arpa. PTR www.example.com.
```

IPv4 octets 反转是为了适应 DNS hierarchy。IPv6 使用 `ip6.arpa.`，按 hexadecimal nibbles 反向组织。

PTR 通常由 address space 的管理方控制，不一定由 forward domain 的 owner 直接控制。

A / AAAA 不自动生成 PTR，PTR 也不自动证明某个 caller 的可信身份。Reverse DNS 不能代替 authentication。

### 5.10 SRV

```text
_service._tcp.example.com. 300 IN SRV 10 20 8443 host.example.com.
```

RDATA 包含：

```text
priority  weight  port  target
```

SRV 支持知道如何使用它的 applications 发现 service endpoints。

- Priority 较小通常优先。
- 同 priority 的 records 按 weight 参与选择。
- Port 是 service port。
- Target 是 hostname，随后还需 address resolution。

普通 browser 不会因为你给任意 website 配置了 SRV，就自动把 HTTPS port 改掉。Client 必须支持对应 discovery protocol。参见 [RFC 2782](https://www.rfc-editor.org/rfc/rfc2782.html)。

### 5.11 CAA

CAA 允许 domain owner 表达哪些 certificate authorities 被授权为其签发相应 certificates。

它参与 certificate issuance policy，不是浏览器访问时用来代替 TLS certificate validation 的记录，也不直接阻止 HTTP requests。参见 [RFC 8659](https://www.rfc-editor.org/rfc/rfc8659.html)。

### 5.12 HTTPS and SVCB

HTTPS / SVCB records 能发布 service binding 和连接参数，例如 supported ALPN 或 alternative endpoint information。

其意义是减少 client 在连接前缺失的信息，而不仅仅是返回一个 IP。

Address hints 也不意味着永远替代 A / AAAA resolution。Client 的支持、record mode 和参数 semantics 很重要。

不要把 HTTPS record 当成“在 DNS 中存整个 HTTPS response”。定义见 [RFC 9460](https://www.rfc-editor.org/rfc/rfc9460.html)。

### 5.13 A Small Zone-file Example

以下是 isolated teaching zone，不会自动成为 public DNS：

```text
$ORIGIN example.test.
$TTL 300

@ IN SOA ns1.example.test. hostmaster.example.test. (
    2026092701 3600 600 1209600 300
)
@       IN NS    ns1.example.test.
@       IN NS    ns2.example.test.
@       IN A     192.0.2.20
@       IN AAAA  2001:db8::20

ns1     IN A     192.0.2.53
ns2     IN A     192.0.2.54
www     IN CNAME example.test.
mail    IN A     192.0.2.25
@       IN MX    10 mail.example.test.
@       IN TXT   "verification=example-only"
```

`@` 表示 current origin；`www` 这种 relative owner 会展开为 `www.example.test.`。

Name-valued RDATA 的 trailing dot 很重要。遗漏它，zone-file parser 可能把 current origin 再拼上去，得到并非你想要的 target。

## 6. Delegation, Glue, and Zone Operations

### 6.1 Delegation

Parent zone 中：

```text
dev.example.com. IN NS ns1.dev.example.com.
```

这告诉 resolver：关于 `dev.example.com` subtree 的 authority，应继续找 child nameserver。

Child zone 则有自己的 SOA、apex NS 和 data。

Delegation 不意味着 parent 把 child 的所有 records 复制到自己这里，也不意味着每个 query 都要先联系 parent。

### 6.2 The Circular Dependency

如果 child 的 nameserver 在 child 内部：

```text
Need: api.dev.example.com address
  → ask ns1.dev.example.com
  → need ns1.dev.example.com address
  → ask dev.example.com authority
  → need ns1.dev.example.com address
  → ...
```

仅提供 NS hostname 无法启动查询。

### 6.3 Glue Records

Parent referral 可以提供用于 bootstrap 的 addresses：

```text
Delegation:
dev.example.com.      NS ns1.dev.example.com.

Glue:
ns1.dev.example.com.  A  192.0.2.53
```

Glue 让 resolver 能先联系 child authoritative server。Referral 中的 glue requirements 见 [RFC 9471](https://www.rfc-editor.org/rfc/rfc9471.html)。

需要区分：

- Parent 提供的 glue 是 delegation support information。
- Child zone 中关于该 nameserver 的 A / AAAA 是其 authoritative address data。
- 两边应协调更新，否则 resolver 可能被引向旧 server。

不在 child 内的 nameserver name 通常可以沿其他 delegation chain 独立解析，不一定有同样的 circular dependency。

### 6.4 Bailiwick

Resolver 不能把 response 里出现的所有 additional addresses 都无条件加入可信 cache。

例如收到某个 zone 的 referral，不代表 sender 可以顺便替完全无关的 domain 指定任意地址。

**Bailiwick checking** 根据 authority / delegation context 限制可接受的附加信息，是减少 cache poisoning 风险的一部分。它不替代 DNSSEC cryptographic validation。

### 6.5 Primary and Secondary Servers

Zones 通常有多个 authoritative servers，以减少单点故障。

```text
Zone update source / primary
          |
          | replication / zone transfer
          v
Secondary authoritative servers
```

Secondary 持有同步后的 zone data，仍可提供 authoritative answers；不是简单的 recursive cache。

“Secondary”不意味着 client 必须 primary 失败后才能问它。Resolver 可以从可用 authoritative servers 中选择。

### 6.6 AXFR, IXFR, and NOTIFY

| Mechanism | 用途 |
| --- | --- |
| AXFR | Full zone transfer，传统 DNS 中使用 TCP |
| IXFR | Incremental zone transfer；transport / fallback 与 transfer size、implementation 有关 |
| NOTIFY | 通知 secondary 可能有更新，促使它检查 |
| SOA serial checks | 判断 secondary 是否需要更新 |

Zone transfer 用于 authoritative replication，不是 ordinary clients 每次 lookup 都下载整个 zone。

Public authoritative query service 和 zone transfer access 是不同能力。Production 常限制 transfers 的 allowed peers，并使用适当认证。

Managed DNS 也可能通过 provider 自己的 control plane 分发 records，不一定把内部 replication 暴露为这些传统机制。

### 6.7 Changing Nameservers Safely

Nameserver migration 涉及至少两类 cached data：

- Parent delegation / glue。
- Child authoritative records。

合理顺序通常是先让新 servers 正确 serving，再更新 delegation，保留旧 servers 一段 overlap，并检查所有 authoritative copies。

只修改 child apex NS，不一定改变 parent referral；只更新 registrar，也不保证 child data 已准备好。

不能只看一个 recursive resolver 已返回新值，就判断全体 users 都已迁移。

## 7. DNS Caching and TTL

### 7.1 Where Caches Can Exist

```text
Application / browser cache
        ↓
OS / local resolver cache
        ↓
Router / forwarder cache
        ↓
Recursive resolver cache
```

不要求每个 deployment 都经过以上全部 layers。Browser DoH、enterprise VPN 和 custom runtimes 都会改变路径。

此外，application 还可能保存 resolved addresses 或长期复用 connections，这不完全等于 DNS RR cache。

### 7.2 What TTL Means

TTL 描述 record 在正常 caching rules 下还能被 reuse 多久。

例子：

```text
t = 0:   resolver obtains A=192.0.2.20, TTL=300
t = 100: cached response has roughly 200 seconds remaining
t = 300: entry is stale / expired for ordinary reuse
```

TTL 不是：

- IP address 的有效期。
- Domain registration 的剩余寿命。
- TCP connection 的 timeout。
- Server 必须在这段时间一直存活的承诺。
- 每过 TTL，所有 clients 一起收到 update 的 push schedule。

### 7.3 Cache Hits Do Not Normally Reset TTL

```text
t = 0:  TTL = 60
t = 20: cache hit, remaining TTL ≈ 40
t = 40: cache hit, remaining TTL ≈ 20
```

不能每次 cache hit 就重新给 60 seconds，否则热门旧 records 可能永远不 expire。

不同 cache layers 传递的是相应 remaining lifetime，而不是无条件把 authoritative original TTL 在每层重置。

### 7.4 TTL Tradeoffs

| Short TTL | Long TTL |
| --- | --- |
| 更快让后续 lookups 获得 changes | 更多 cache hits |
| 更多 resolver / authoritative traffic | 更低 lookup latency 和 query load |
| 更依赖 DNS infrastructure availability | 正常缓存期内更能承受短时 authoritative failure |
| 无法强制终止已有 connections | Changes 需要更长时间逐步被观察到 |

TTL 为 0 通常不允许常规跨 requests cache reuse，但仍可用于当前 transaction。它不是解决 failover 的免费按钮。

### 7.5 Why DNS Changes Seem to Propagate Slowly

DNS 不是把每次 update 挨个 push 给全世界所有 caches。

常见过程：

1. Authoritative data 被更新。
2. Provider 的 authoritative copies 完成同步。
3. 已持有旧 data 的 caches 继续按旧 remaining TTL 使用。
4. Cache miss / refresh 后才查询新 data。
5. Applications 根据自己的 lookup / connection policy 使用结果。

因此所谓“DNS propagation”经常同时包含 **authoritative replication** 和 **cache expiration**，不能用一个固定全球倒计时解释。

### 7.6 Lower TTL Before a Migration

假设旧 TTL = 3600，计划在 12:00 切换 IP。

在 12:00 同时把 IP 改掉、TTL 改成 60，不能让 11:59 缓存的旧 answer 自动在 12:01 expire。它之前已经拿到了长 TTL。

应提前降低 TTL，并留出足够时间让旧长 TTL entries 正常过期，再切换 addresses。

仍需考虑 application caches、serve-stale、connection reuse 和不同 providers 的 behavior。低 TTL 是 migration 手段之一，不是严格的全网同步保证。

### 7.7 Negative Caching

失败结果也可能被缓存：

- **NXDOMAIN**：name 不存在。
- **NODATA**：name 存在，但没有所查 type 的 data。

Authoritative negative response 中的 SOA 提供 negative caching 所需的信息。传统 negative TTL 由 SOA record TTL 与 SOA MINIMUM 的较小值确定，并受 resolver policy 等因素影响。

因此“刚创建一个 name，为什么还有人看到不存在”可能是之前的 negative cache 尚未 expire。规则见 [RFC 2308](https://www.rfc-editor.org/rfc/rfc2308.html)。

### 7.8 Negative Cache Keys

Positive RRsets 需要区分 name、type 和 class。

NODATA 通常需要区分 type：

```text
example.com AAAA → no IPv6 data
example.com A    → may still succeed
```

NXDOMAIN 则针对 name 的不存在，不只是“不存在 A record”。某些 resolver 还能利用相关规则推断其下 names 的不存在。

不要把 SERVFAIL 随意当成 NXDOMAIN cache。它描述的是 resolution failure，可能暂时不可用，而不是 name 被证明不存在。

### 7.9 Serve Stale and Prefetch

一些 resolvers 为改善 availability，支持在受控 policy 下暂时返回 expired data，同时尝试 refresh，例如 authoritative servers 不可达时。

这叫 **serve stale**，不是正常 TTL 被无限延长。实现需要限制 stale lifetime、refresh behavior 等。见 [RFC 8767](https://www.rfc-editor.org/rfc/rfc8767.html)。

Prefetch 则是在热门 data 接近 expiration 时主动 refresh，减少下一次用户 query 的等待。它也不是所有 resolvers 的统一默认行为。

### 7.10 DNS Cache vs. HTTP Cache

| Aspect | DNS cache | HTTP cache |
| --- | --- | --- |
| 缓存内容 | RRsets / negative results | Responses / representations |
| Key 核心 | Name、type、class 等 | Method、URI、Vary 等 |
| 常见 freshness | DNS TTL | Cache-Control / Expires / Age |
| 更新判断 | 通常重新 query | 可用 ETag / conditional request |
| 影响 | 下一次如何定位 endpoint | 是否重新获取 resource content |

浏览器页面仍旧，可能是 HTTP cache；连接仍指向旧 IP，可能是 DNS cache 或 existing connection。不要把所有 stale behavior 都归因于 DNS。


## 8. DNS Messages and Transport

### 8.1 DNS Is an Application-layer Protocol

DNS 本身定义 query / response format，可以通过多种 transports 传输。

| Transport | 常见 endpoint | 主要特点 |
| --- | --- | --- |
| DNS over UDP | UDP 53 | Datagram exchange，常见于传统 DNS |
| DNS over TCP | TCP 53 | Reliable byte stream，需要 DNS message framing |
| DNS over TLS — DoT | TCP 853 | DNS over TLS |
| DNS over HTTPS — DoH | HTTPS，通常 443 | DNS query / response 通过 HTTP exchanges 承载 |
| DNS over QUIC — DoQ | UDP 853 | 使用 QUIC secure streams |

这是常见标准 ports，不代表 infrastructure 绝不使用 custom ports。DoQ 与 DoH over HTTP/3 也不是同一个 protocol mapping。

### 8.2 Basic Message Sections

```text
DNS message
├── Header
├── Question
├── Answer
├── Authority
└── Additional
```

| Section | 常见内容 |
| --- | --- |
| Header | ID、flags、RCODE、各 section counts |
| Question | QNAME、QTYPE、QCLASS |
| Answer | 对 question 的 relevant answer data，例如 A / CNAME |
| Authority | Referral 的 NS，或 negative response 的 SOA 等 |
| Additional | Glue、相关附加 addresses、EDNS OPT 等 |

Authority section 有内容，不等于这个 response 一定是 final authoritative answer。它也可能在告诉你下一步去哪里查。

### 8.3 Important Header Fields

| Field / flag | 含义 |
| --- | --- |
| ID | 16-bit transaction identifier |
| QR | Query 还是 response |
| AA | Authoritative Answer |
| TC | Message 被截断 |
| RD | Recursion Desired |
| RA | Recursion Available |
| RCODE | Result code，例如 NOERROR、NXDOMAIN、SERVFAIL |
| AD | Authentic Data，涉及 DNSSEC validation status |
| CD | Checking Disabled，影响 validating resolver 的处理 |

AD 与 AA 不同：

- AA 说明 authoritative answer context。
- AD 表示相关 validation assertion。
- 不应通过不可信 plaintext channel 盲信别人设置的 AD bit。

AA 不等于 cryptographic proof，RA 不等于当前这次 query 一定成功执行了完整 recursion。

### 8.4 A Small Wire-format Example

普通 query 的 header 为 12 bytes。以下 query 问 `www.example.com. A IN`：

```text
Header:
12 34    transaction ID = 0x1234
01 00    standard query, RD = 1
00 01    QDCOUNT = 1
00 00    ANCOUNT = 0
00 00    NSCOUNT = 0
00 00    ARCOUNT = 0

QNAME:
03 77 77 77                         "www"
07 65 78 61 6d 70 6c 65             "example"
03 63 6f 6d                         "com"
00                                  root terminator

QTYPE:  00 01                       A
QCLASS: 00 01                       IN
```

这里完整 DNS message 是 33 bytes，不包括 UDP / IP headers。

Wire format 的 name 不是直接把 dots 连同 string 一起发送，而是 length-prefixed labels，最后以 root terminator 结束。

Responses 可以使用 name compression pointers 引用同一 message 中之前出现的 name。例如 `c0 0c` 可指向 offset 12；它不是两个普通 label characters。

实际 parser 必须处理 bounds、pointer loops、truncation 等，不应把教学示意当作完整 parser。格式基础见 [RFC 1035](https://www.rfc-editor.org/rfc/rfc1035.html)。

### 8.5 Why UDP Is Common

很多 DNS exchanges 比较小：

```text
query datagram → response datagram
```

UDP 无需为每个新的 exchange 先建立 TCP connection，适合低开销查询。

但 UDP 不提供自动 retransmission / reliable delivery。Resolver 需要自己管理 timeout、retry、alternative servers 和 transaction matching。

不能把“DNS 用 UDP”推导成“DNS 不需要可靠性”。可靠性来自 application retry、multiple servers、cache 和其他 protocol choices 的组合。

### 8.6 The 512-byte Rule and EDNS

传统、没有 EDNS 的 DNS over UDP message limit 是 512 bytes。这个值不是现代所有 DNS responses 的硬上限，也不等于所有网络的 MTU。

**EDNS(0)** 用 OPT pseudo-record 扩展协议，允许 requestor 宣告能接收的 UDP payload size，并传递其他 options / flags。

OPT 不是一个可以像 A record 那样普通缓存的 zone data record。EDNS 是逐跳协商，不代表 stub、recursive、authoritative 三段一定使用相同 size。

Advertise 更大的 size，不保证 path 一定能安全传输那么大的 unfragmented packet。规则见 [RFC 6891](https://www.rfc-editor.org/rfc/rfc6891.html)。

### 8.7 Truncation and TCP Fallback

Response 无法放入适用 UDP size 时，server 可设置 TC。Client / resolver 发现 truncated response 后，通常通过 TCP 重试以获得完整结果。

```text
UDP query
    ↓
UDP response: TC = 1
    ↓
TCP connection / reuse
    ↓
DNS query over TCP
    ↓
Complete response
```

但 packet 因 fragmentation / firewall 丢失时，client 可能只看到 timeout，不一定先收到 TC=1。

只允许 UDP 53、阻止 TCP 53，会让 DNS 在一些小 answers 上看起来正常，却在较大 answers、DNSSEC 等场景中失败。General-purpose DNS implementations 必须支持 TCP；见 [RFC 7766](https://www.rfc-editor.org/rfc/rfc7766.html)。

### 8.8 DNS over TCP Framing

TCP 没有 message boundaries，所以 DNS over TCP 给每个 message 加两字节 length prefix：

```text
[2-byte length][DNS message][2-byte length][DNS message]...
```

Length 不包含这个 prefix 自身。Reading 时仍需处理 partial reads；一次 recv 不保证读满一个 message。

TCP connections 可以复用，不必为每次 DNS query 都重新 handshake。Transport buffering 的基础见 [TCP notes](tcp.md)。

### 8.9 Matching Responses

Resolver 不应只看“这是一个看起来像 DNS 的 packet”就接受。

通常要检查 transaction context，例如 ID、question、expected peer / ports 等。随机化 transaction ID 和 source port 有助于降低某些 spoofing 风险，但不是 cryptographic authentication。

DNSSEC 与 encrypted transport 解决的边界在后面两节解释。

## 9. Results, Errors, and Failure Semantics

### 9.1 NOERROR

NOERROR 表示 DNS query handling 的 result code 没报相应错误。

它不保证 Answer section 一定含所需 IP：

- 可能成功获得 A / AAAA。
- 可能只有 alias，需要继续处理。
- 可能是 referral。
- 可能是 NODATA。

因此 parser / diagnostic tool 必须结合 sections 和 resolution context，不能只看 RCODE=0 就宣布“找到了 address”。

### 9.2 NXDOMAIN vs. NODATA

| Result | 含义 | Example |
| --- | --- | --- |
| NXDOMAIN | Queried name 不存在 | `missing.example.com.` 不存在 |
| NODATA | Name 存在，但没有所查 type 的可用 data | 有 A，没有 AAAA |

NODATA 通常表现为 NOERROR 加上相应 negative-answer structure，**不是单独的 RCODE**。

CNAME resolution 中还可能出现 alias 本身存在，但 target 不存在的情形。诊断时要看 chain 到底在哪个 name 失败。

### 9.3 SERVFAIL

SERVFAIL 说明 server 未能完成 resolution。可能原因包括：

- Authoritative servers unreachable。
- DNSSEC validation failure。
- Broken delegation / alias loop。
- Resolver internal failure。
- Resolution process 超过资源或时间限制。

它不是“不存在”的同义词。

同样，DNSSEC failure 导致 resolver 返回 SERVFAIL，并不能证明域名被攻击；expired signatures 或 deployment mistakes 也可能造成它。

### 9.4 REFUSED and FORMERR

- **REFUSED**：Server 因 policy 拒绝 operation，例如不给不被允许的 clients 提供 recursion。
- **FORMERR**：Server 无法正确理解 query format 等。

Resolver 不一定对所有来源都开放。能访问 UDP 53，不代表有权请求 recursive service。

### 9.5 Timeout Is Not an RCODE

Timeout 表示 client 没在 deadline 内收到可接受 response。

可能发生在：

```text
Query lost
Response lost
Firewall drop
Server overloaded
Wrong destination
EDNS / fragmentation issue
```

你没有收到 response，就没有从那次 exchange 读到一个“timeout status code”。

Application-facing error 也可能把多个底层 outcomes 合并成一个 exception。要知道工具展示的是 DNS RCODE、OS error，还是 runtime abstraction。

### 9.6 Retry Without Amplifying Failure

Resolver 通常会尝试 alternative servers、有限 retries 和适当 timeout。

但“失败就立即无限 retry”会增加 load。大量 clients 同时 cache miss、短 TTL、authoritative outage，可能形成 query storm。

需要 bounded work、request coalescing、cache policy、rate limits 和 observability。尤其不要把 DNS retry 与 application connection retry 叠加后忘记 total deadline。

## 10. DNS-based Traffic Routing and Availability

### 10.1 Multiple Addresses

```text
api.example.com. 60 IN A 192.0.2.20
api.example.com. 60 IN A 192.0.2.21
```

DNS 可以返回多个 candidates，但不保证：

- 每个 client 都使用相同 selection strategy。
- 一半 requests 去第一个，一半去第二个。
- Resolver 保持 response order。
- Client 对失败 address 立即尝试所有 alternatives。

所以 DNS round-robin 是粗粒度分流，不是精确 per-request load balancing。

### 10.2 DNS Routing vs. Reverse Proxy Load Balancing

```text
DNS routing:
    Choose which endpoint addresses a client learns.

Reverse proxy:
    Receive application traffic and choose an upstream.
```

| Aspect | DNS routing | Reverse proxy / L7 load balancer |
| --- | --- | --- |
| 决策时机 | Name lookup / cache refresh | Connection / request handling |
| 看到的信息 | DNS query context | 可看到适用的 HTTP / application metadata |
| Existing connections | 通常不主动迁移 | 可以按自身 routing / draining policy 管理 |
| Change visibility | 受 DNS / application caches 影响 | 受 proxy configuration / connection behavior 影响 |

两者常组合：

```text
DNS selects regional load balancer
    → regional load balancer selects application instance
```

### 10.3 GeoDNS

Authoritative service 可以根据 query context 返回不同 geographic endpoints。

但它通常看到的是 recursive resolver 的 source address，不一定是 end user's address。

如果 user 在 Toronto，却使用另一地区的 resolver，基于 resolver location 的判断可能不完全符合 user location。

“返回最近 server”也不能只按地理距离；network routing、capacity、health 和 provider policy 都有影响。

### 10.4 EDNS Client Subnet

**ECS — EDNS Client Subnet** 允许携带 client network prefix，帮助 authoritative systems 作出更贴近 client 的回答。

Tradeoffs：

- 可能改善 location-based responses。
- 暴露额外 client network information。
- 增加 cache variants，降低共享程度。
- Resolver / authoritative 双方必须正确处理 scope。

不是每个 resolver 都发送 ECS，也不应假设它包含完整 client IP。定义见 [RFC 7871](https://www.rfc-editor.org/rfc/rfc7871.html)。

### 10.5 DNS Failover

Provider 可以通过 health checks，把新 answers 从坏 endpoint 切到健康 endpoint。

但切换速度还受这些因素影响：

```text
Failure detection interval
+ decision / publication delay
+ authoritative replication
+ remaining cached TTL
+ application lookup / retry behavior
```

已有 TCP / QUIC connections 不会因为 DNS answer 改了就自动迁移到另一个 arbitrary service endpoint。Application 可能需要 reconnect；QUIC connection migration 也不等于 DNS 自动把 session 转给别的服务实例。

因此 DNS failover 不适合承诺“所有 requests 在某个秒数内无缝切换”。

### 10.6 Anycast

Anycast 让多个 network locations announce 同一个 service IP prefix。Routing 决定 query 到达哪个 instance。

```text
One service IP
   ├── instance in region A
   ├── instance in region B
   └── instance in region C
```

这是 network routing 技术，不是“DNS 返回多个 IP”的另一种叫法。

Anycast 常用于 authoritative / recursive DNS。它能改善 distribution 和 resilience，但同一个 IP 不意味着固定的一台 machine，route changes 也可能影响 stateful connections。

### 10.7 Split-horizon DNS

同一个 name，根据 network / view 返回不同 answers：

```text
Inside company network:
    api.example.com → private address

Outside:
    api.example.com → public address or no answer
```

用途包括 private services、VPN 和 environment separation。

Troubleshooting 时，如果 public resolver 与 corporate resolver 返回不同结果，不一定是其中一个坏了；可能就是预期 policy。

但 DNS naming 不能代替 network access control。知道 private hostname 或得到 private IP，并不意味着应该被授权访问 service。

### 10.8 Wildcards and Search Suffixes

DNS wildcard 例如 `*.example.com` 具有 DNS tree-based matching rules，不是简单的 string glob。Existing names、empty non-terminals 和 zone cuts 都可能影响结果，不能默认匹配所有 depth。具体规则见 [RFC 4592](https://www.rfc-editor.org/rfc/rfc4592.html)。

Search suffix 则是 client-side name expansion，例如把 `db` 尝试为 `db.corp.example.com`。

两者是完全不同的机制。FQDN 可以减少 search-suffix 歧义；wildcard 也不会自动创建对应 TLS certificate coverage。


## 11. DNS Security and DNSSEC

### 11.1 Why Plain DNS Answers Need Care

传统 DNS over UDP / TCP 默认没有 cryptographic data-origin authentication 或 encryption。

风险可能包括：

- Spoofed responses。
- Cache poisoning。
- On-path modification。
- Misconfigured resolver / delegation。
- Domain / DNS-provider account compromise。

这里有不同 trust boundaries：有的是网络传输问题，有的是 authoritative data 本身被改了，有的是 application 错误使用结果。

### 11.2 Cache Poisoning

Cache poisoning 的核心是让 resolver 缓存并返回不正确的数据。

如果错误 answer 被 shared resolver reuse，影响可能扩散到多个 clients。

防护包括适当 transaction matching、source-port / ID randomization、bailiwick rules、成熟 resolver implementations，以及在可用 trust chain 下进行 DNSSEC validation。

只说“使用 TCP 就不会 poisoning”不完整：TCP 改变了某些攻击条件，但不提供 DNS data 的 cryptographic authenticity。

### 11.3 What DNSSEC Provides

**DNSSEC** 为 DNS data 提供 data-origin authentication、integrity，以及适用的 authenticated denial of existence。

它签名的是 DNS RRsets，不是把所有 DNS messages 加密。

```text
DNSSEC:
    Can I validate that this RRset is authenticated
    under the DNS trust chain?

Encrypted DNS:
    Can observers on this transport path read or alter
    this client–resolver exchange?
```

DNSSEC 不隐藏 queried name，不保证 authoritative server 可用，也不保证被合法签名的 IP 上运行的是健康、安全的 application。概念见 [RFC 4033](https://www.rfc-editor.org/rfc/rfc4033.html)。

### 11.4 Key Record Types

| Type | 作用 |
| --- | --- |
| DNSKEY | Zone 使用的 public keys |
| RRSIG | 对 RRset 的 digital signature |
| DS | Parent 中用于建立 child trust relationship 的 digest information |
| NSEC / NSEC3 | 用于 authenticated denial of existence 等 |

RRSIG 还涉及 validity interval。即使 RRset TTL 未过，signature 过期也会影响 validation；validator 的 clock 错误同样可能造成问题。

### 11.5 Chain of Trust

简化图：

```text
Configured trust anchor
        |
        v
Validate root DNSKEY / signatures
        |
        v
Authenticated parent DS for child
        |
        v
Match child DNSKEY and validate child signatures
        |
        v
Repeat toward target zone
        |
        v
Validate target RRset
```

Parent 中的 DS 与 child key 建立链接；validator 不能只看到 child 发来一把 public key，就认为它一定可信。

常见 public DNS validation 以 root trust anchor 为起点。Private deployments 也可以配置不同 trust anchors。

### 11.6 Secure, Insecure, Bogus, and Indeterminate

| State | 概念 |
| --- | --- |
| Secure | 按适用 trust chain 验证成功 |
| Insecure | 已确定此处没有 DNSSEC protection chain，例如经过认证的 unsigned delegation |
| Bogus | 按预期应当可验证，但 validation 失败 |
| Indeterminate | 无法确定适用 trust relationship |

Unsigned 并不自动等于 bogus。DNSSEC 是逐步部署的系统，validator 必须区分“没有签名保护”和“预期签名保护坏了”。

Validating resolver 通常会对 bogus data 返回 failure，而不是照样返回可疑 answer。具体诊断可能需要 resolver logs 或 extended error information。

### 11.7 Signed Nonexistence

只收到“没有这个 name”的 unsigned statement，并不能建立 authenticated denial。

DNSSEC 使用 NSEC / NSEC3 等 mechanisms 对 namespace 中的不存在事实提供 proof。这也是 negative answers 可能比直觉上“一个错误码”更大的原因。

不能用“NOERROR + empty answer”单独推断 DNSSEC validation 已经完成。

### 11.8 Common Operational DNSSEC Failures

- Parent DS 指向旧 child key。
- Key rollover 顺序不正确。
- Signatures 过期或尚未生效。
- Authoritative copies 未同步。
- Validator clock 不正确。
- 大 responses 遇到 UDP fragmentation / TCP fallback 问题。

所以“只有 validating resolver 查不出来”是重要线索，而不是应该直接关闭 validation 的理由。

### 11.9 DNSSEC and HTTPS Are Complementary

DNSSEC 验证 DNS data 的 trust chain；TLS 验证 connection peer 的 identity 并保护 application traffic。

即使 DNS 错误把用户引到别处，正确 TLS hostname verification 也能阻止许多冒充目标 website 的连接。但这不是 DNSSEC 所有用途的替代品，也不能让 DNS failure 消失。

反过来，DNSSEC-valid address 不会自动让 TLS certificate 合法。

### 11.10 Availability and Abuse

DNS 的 small-query / larger-response 特性可能被滥用于 reflection / amplification；public recursive service 应有适当 exposure policy 和 abuse controls。

Authoritative services 则需要考虑 query floods、random-name traffic、capacity、multiple locations 和 monitoring。

这是 availability 问题，不是 DNSSEC 签名成功就能解决的问题。DNSSEC 本身也会增加部分 responses 的大小与 validation work。

## 12. Encrypted DNS and Query Privacy

### 12.1 DoT

**DNS over TLS** 用 TLS 保护 DNS transport，常见 endpoint 为 TCP 853。

Client 验证所连接 resolver 的 identity，然后在 secured connection 上发送 DNS messages。

它保护该 client–resolver hop；resolver 若继续通过 plaintext DNS 查询 authoritative servers，后面的 hop 不会自动被第一段 TLS 加密。见 [RFC 7858](https://www.rfc-editor.org/rfc/rfc7858.html)。

### 12.2 DoH

**DNS over HTTPS** 使用 HTTPS exchanges 携带 DNS queries / responses。

Standard wire-format mapping 使用 `application/dns-message`；不是所有 DoH 都是 vendor-specific JSON API。

可能使用 GET 或 POST：

- GET 把 encoded query 放入适用 URL parameter。
- POST 把 DNS message 放在 request body。
- Response 仍需要解释 DNS-level result，不只看 HTTP status。

```text
HTTP status: 200
DNS RCODE:  NXDOMAIN
```

可以表示 HTTP exchange 成功，而所查 name 不存在。协议定义见 [RFC 8484](https://www.rfc-editor.org/rfc/rfc8484.html)。

### 12.3 DoQ

**DNS over QUIC** 使用 QUIC secure streams 传输 DNS，标准 service port 为 UDP 853。

它与 DoH over HTTP/3 都使用 QUIC，但前者不是把 DNS 包在 HTTP request/response 里。

这种区别和“QUIC 与 HTTP/3 不是同一个协议层”一致。见 [RFC 9250](https://datatracker.ietf.org/doc/html/rfc9250)。

### 12.4 Encryption Does Not Make the Resolver Blind

```text
Client --encrypted DNS--> Resolver
```

中间网络观察者受到限制，但 resolver 自己仍需要知道 query 并提供 answer。

因此 encrypted DNS 改变的是 trust / visibility boundary，不是完全 anonymity。Resolver selection、logging policy、traffic metadata 和后续 connections 仍影响 privacy。

### 12.5 QNAME Minimization

完整 target 可能是：

```text
private-service.team.example.com.
```

在 resolution 的上层阶段，并不一定需要把完整 name 都发给 root / TLD。QNAME minimization 尽量只提供该阶段需要的 suffix information。

简化理解：

```text
Root needs to help find .com
.com needs to help find example.com
Deeper authority gets the more specific question
```

实际 query types 和 fallback behavior 由 algorithm 决定；不要把这个示意当成每个 resolver 的逐包固定序列。见 [RFC 9156](https://www.rfc-editor.org/rfc/rfc9156.html)。

### 12.6 DNSSEC vs. DoH / DoT

| Question | DNSSEC | DoH / DoT |
| --- | --- | --- |
| 隐藏 client–resolver query content? | No | 对该 transport path 提供 encryption |
| 验证 signed DNS data 的来源与完整性? | Yes，需完整 validation / trust chain | 本身不执行 DNSSEC validation |
| Resolver 能读 query? | Yes | Yes |
| Authoritative hops 自动被加密? | No | No |
| 保证 endpoint application 健康? | No | No |

它们可以组合使用。

如果 stub 依赖 recursive resolver 代做 DNSSEC validation，还需要可信地获取该 resolver 的 validation result；encrypted、authenticated channel 有助于保护这条信任边界。

### 12.7 Enterprise DNS and Private Names

Browser 使用外部 DoH，可能与 OS / VPN 使用的 corporate resolver 不同。

后果可能包括：

- Internal name 在 browser 里失败，系统工具却成功。
- Split-horizon answers 不同。
- Private queries 被发给不应接收它们的 external resolver。

处理时先查 effective configuration 和 organization policy，不应把所有解析差异都当成 cache corruption。

## 13. How Applications Use DNS

### 13.1 getaddrinfo Is Name Resolution, Not a Raw DNS Query

Application 常用 `getaddrinfo` 获得 connectable socket addresses。

这个 API 可能经过：

- Hosts file。
- OS cache。
- Configured DNS resolvers。
- Platform-specific naming mechanisms。
- Search suffix / policy rules。

所以 `getaddrinfo` 成功不一定证明刚刚发生了一个 DNS packet exchange。

它的返回结果也通常不是原始 DNS message，不直接提供每个 RRset 的 TTL、authority section 或 DNSSEC proof。

### 13.2 Name Resolution and connect Are Separate Steps

```text
getaddrinfo(name, service)
        ↓
list of candidate socket addresses
        ↓
socket()
        ↓
connect(selected address)
        ↓
TLS / application protocol
```

Resolve success 只完成 endpoint discovery 的一部分。

同一个 name 的多个 candidates 中，某些 addresses 可能不可达。成熟 clients 会使用适当 selection / fallback policy，而不是永远只尝试列表第一个。

### 13.3 IPv4, IPv6, and Happy Eyeballs

Dual-stack services 可能同时返回 A 和 AAAA。

如果 client 先尝试不可达 IPv6 并等待很长 timeout，再尝试 IPv4，会让用户感觉“域名很慢”，但真正问题可能在 connection path。

**Happy Eyeballs** 通过合理调度不同 address-family attempts 来降低这种等待，不是简单地“永远优先 IPv4”，也不是无限并发全部 addresses。见 [RFC 8305](https://www.rfc-editor.org/rfc/rfc8305.html)。

DNS query completion 和 connection race 是关联但不同的阶段。

### 13.4 TTL Does Not Close Connections

```text
t = 0:   resolve api.example.com → 192.0.2.20, TTL 60
t = 1:   open persistent HTTPS connection to 192.0.2.20
t = 61:  DNS entry expired
t = 120: same live connection may still be reused
```

TTL expiration 不会发送“close”命令给 socket。

这解释了：即使 DNS 已更新，某些 clients 仍继续访问旧 backend。Connection pool lifetime、draining 和 reconnect policy 需要独立设计。

### 13.5 Application DNS Caches

不同 runtimes / libraries 对 lookup frequency 和 cache lifetime 的策略不同。不能假设每次 `connect` 都重新 query authoritative DNS，也不能假设所有 language runtimes 都精确遵守同一 caching default。

Troubleshooting 时要确定 application 实际调用哪个 resolver API，是否 cache，是否复用 connections。

### 13.6 Hosts Files and Local Names

Hosts file 可以在本地把 name 映射到 address，通常只影响对应 machine / resolution path。

它不修改 public authoritative zone，不会自动让其他 users 得到相同 answer。

`.local` 常用于 mDNS，而不是 ordinary public unicast DNS。mDNS 使用 local-link multicast 等机制；普通 DNS notes 里的 root → TLD 模型不能直接套用。参见 [RFC 6762](https://www.rfc-editor.org/rfc/rfc6762.html)。

### 13.7 DNS and Security-sensitive Endpoint Selection

如果 application 接受 user-provided URLs，然后由 server 发请求，仅检查 hostname string 不够。

Name 的 addresses 可能变化，可能同时含多种 address families，也可能在 redirects 后换 destination。DNS rebinding / SSRF defenses 需要把解析、实际 connection target、network policy 和 redirect policy 一起考虑。

同样，不要用 PTR name 代替 identity verification。Security decision 不能只建立在“这个 name 看起来像公司域名”上。

## 14. Runnable Python Examples

### 14.1 Example A: System Name Resolution

保存为 `resolve_localhost.py` 后用 Python 3 运行。它只解析 `localhost`，不创建 outbound application connection，也不依赖某个 public site's records。

注意：这是 **system resolution API example**，不证明一定使用 DNS；localhost 通常由 local mechanisms 处理。

```python
import ipaddress
import socket


def main():
    results = socket.getaddrinfo(
        "localhost",
        443,
        family=socket.AF_UNSPEC,
        type=socket.SOCK_STREAM,
        proto=socket.IPPROTO_TCP,
    )

    seen = set()
    for family, socktype, proto, canonname, sockaddr in results:
        # IPv4 sockaddr: (address, port)
        # IPv6 sockaddr: (address, port, flowinfo, scope_id)
        address, port = sockaddr[:2]
        key = (family, address, port)
        if key in seen:
            continue
        seen.add(key)

        assert ipaddress.ip_address(address).is_loopback
        assert port == 443
        assert socktype == socket.SOCK_STREAM
        assert proto == socket.IPPROTO_TCP

        label = "IPv6" if family == socket.AF_INET6 else "IPv4"
        print(label, address, port)

    assert seen, "Expected at least one localhost address."
    print("Resolution complete; no application connection was opened.")


if __name__ == "__main__":
    main()
```

可能输出：

```text
IPv6 ::1 443
IPv4 127.0.0.1 443
Resolution complete; no application connection was opened.
```

Address families、order 和条目数取决于本机配置；可能只得到其中一种。

Port 443 是 caller 传给 API 的 service port，不是 A / AAAA record 中保存的内容。API 返回的 sockaddr 把 address 和 port 组合在一起。

接口定义见 [Python socket.getaddrinfo](https://docs.python.org/3/library/socket.html#socket.getaddrinfo)。

### 14.2 Example B: TTL and Negative Cache Simulation

下面是完全离线的教学模型，可以保存为 `dns_cache_demo.py`。使用手动 clock，不需要真的等 60 seconds。

它演示：

- A 和 AAAA 使用不同 keys。
- Cache hit 不重置 TTL。
- Authoritative data 改了，fresh cached answer 仍可返回旧值。
- NODATA 可被缓存；之后创建相应 record，不会自动清除已有 negative cache。

**Scope:** 只模拟 ASCII names、IN class、positive answers 和 type-specific NODATA；不实现 NXDOMAIN subtree rules、CNAME、DNSSEC、serve-stale、network transport 或 concurrent access。

```python
from dataclasses import dataclass


def key_for(name, qtype):
    return (name.lower().rstrip(".") + ".", qtype.upper(), "IN")


@dataclass(frozen=True)
class Answer:
    status: str
    records: tuple[str, ...]
    ttl: int


@dataclass
class Entry:
    answer: Answer
    expires_at: int


class TTLCache:
    def __init__(self):
        self.entries = {}

    def get(self, key, now):
        entry = self.entries.get(key)
        if entry is None:
            return None
        if now >= entry.expires_at:
            del self.entries[key]
            return None
        remaining = entry.expires_at - now
        return entry.answer, remaining

    def put(self, key, answer, now):
        if answer.ttl <= 0:
            self.entries.pop(key, None)
            return
        self.entries[key] = Entry(answer, now + answer.ttl)


def main():
    name = "api.example.test."
    a_key = key_for(name, "A")
    aaaa_key = key_for(name, "AAAA")

    authority = {
        a_key: Answer("NOERROR", ("192.0.2.20",), 60),
        aaaa_key: Answer("NODATA", (), 30),
    }
    cache = TTLCache()
    upstream_queries = 0

    def resolve(qtype, now):
        nonlocal upstream_queries
        key = key_for(name, qtype)
        hit = cache.get(key, now)

        if hit is not None:
            answer, remaining = hit
            source = "cache"
        else:
            upstream_queries += 1
            answer = authority[key]
            cache.put(key, answer, now)
            remaining = answer.ttl
            source = "authority"

        print(
            f"t={now:02d} {qtype:<4} {source:<9} "
            f"{answer.status:<7} {answer.records} ttl={remaining}"
        )
        return answer, remaining, source

    old, remaining, source = resolve("A", 0)
    assert old.records == ("192.0.2.20",)
    assert (remaining, source) == (60, "authority")

    # Change authoritative data while the old cache entry is still fresh.
    authority[a_key] = Answer("NOERROR", ("192.0.2.21",), 60)
    old, remaining, source = resolve("A", 20)
    assert old.records == ("192.0.2.20",)
    assert (remaining, source) == (40, "cache")

    # Exactly at expiration: fetch the new authoritative data.
    new, remaining, source = resolve("A", 60)
    assert new.records == ("192.0.2.21",)
    assert (remaining, source) == (60, "authority")

    missing, remaining, source = resolve("AAAA", 65)
    assert missing.status == "NODATA"
    assert (remaining, source) == (30, "authority")

    # Adding AAAA does not invalidate an already cached NODATA answer.
    authority[aaaa_key] = Answer("NOERROR", ("2001:db8::21",), 60)
    missing, remaining, source = resolve("AAAA", 70)
    assert missing.status == "NODATA"
    assert (remaining, source) == (25, "cache")

    present, remaining, source = resolve("AAAA", 95)
    assert present.records == ("2001:db8::21",)
    assert (remaining, source) == (60, "authority")

    assert upstream_queries == 4

    # TTL=0 can be used now, but is not retained for later reuse here.
    zero_key = key_for("zero.example.test", "A")
    cache.put(zero_key, Answer("NOERROR", ("192.0.2.99",), 0), 100)
    assert cache.get(zero_key, 100) is None

    assert key_for("API.EXAMPLE.TEST", "a") == a_key
    print("All assertions passed; upstream queries =", upstream_queries)


if __name__ == "__main__":
    main()
```

预期 output：

```text
t=00 A    authority NOERROR ('192.0.2.20',) ttl=60
t=20 A    cache     NOERROR ('192.0.2.20',) ttl=40
t=60 A    authority NOERROR ('192.0.2.21',) ttl=60
t=65 AAAA authority NODATA  () ttl=30
t=70 AAAA cache     NODATA  () ttl=25
t=95 AAAA authority NOERROR ('2001:db8::21',) ttl=60
All assertions passed; upstream queries = 4
```

### 14.3 Read the Timeline

```text
A cache:
t=0  -------------------------------- t=60
     old address is reusable              refresh sees new address

AAAA negative cache:
t=65 ------------------------------- t=95
      NODATA remains reusable             refresh sees new AAAA
```

这就是为什么“我已经改了 authoritative data”与“所有 clients 都已看到”不是同一件事。

### 14.4 What a Real Resolver Must Add

这个小 cache 不具备 production resolver 的能力。真实 implementation 还需要：

- DNS message encoding / parsing。
- Delegation、glue、alias chasing。
- Query coalescing、timeouts、alternative servers。
- Positive / negative caching rules 和 resource bounds。
- Transaction validation、DNSSEC、transport security。
- Thread safety、metrics、configuration 和 policy。

学习时应先理解每一层的职责，production 则使用成熟 resolver / DNS libraries，而不是从这个模拟直接扩展成 public service。


## 15. Troubleshooting on Windows and Other Systems

### 15.1 Start by Separating the Layers

```text
1. Did name resolution succeed?
2. Which resolver / naming mechanism supplied the result?
3. Are the addresses expected?
4. Can the client connect to those addresses?
5. Does TLS validation succeed?
6. Does the application return the expected response?
```

如果已经 resolve 到正确 IP，却 TCP connect timeout，继续反复 flush DNS cache 可能完全没有帮助。

同样，“直接输入 IP 无法打开 HTTPS website”也不能证明 IP 不通，因为 hostname-based routing 和 certificate validation 可能需要原来的 hostname。

### 15.2 Resolve-DnsName

以下 commands 是手动诊断示例，不是第 14 节的 offline tests。Public DNS data 会变化，不应把 output 写成固定预期 IP。

```powershell
# Query specific types using the system's configured resolution path.
Resolve-DnsName -Name example.com. -Type A -DnsOnly
Resolve-DnsName -Name example.com. -Type AAAA -DnsOnly
Resolve-DnsName -Name example.com. -Type NS -DnsOnly
Resolve-DnsName -Name example.com. -Type SOA -DnsOnly

# Force TCP for comparison.
Resolve-DnsName -Name example.com. -Type A -TcpOnly -DnsOnly

# Inspect configured DNS server addresses.
Get-DnsClientServerAddress

# Inspect the Windows DNS client cache.
Get-DnsClientCache
```

`-DnsOnly` 限制使用 DNS，而不是让查询继续依赖 LLMNR / NetBIOS fallback；它不等于“强制绕过所有 caches”。

可以用 `-Server` 指定你有意查询的 resolver / authoritative server。查询 authoritative server 时，结合 `-NoRecursion` 检查其 zone data，而不是要求它替你查完整 Internet。

可用 parameters 见 [Microsoft Resolve-DnsName](https://learn.microsoft.com/en-us/powershell/module/dnsclient/resolve-dnsname)。

### 15.3 nslookup

```powershell
nslookup -type=A example.com.
nslookup -type=AAAA example.com.
nslookup -type=NS example.com.
nslookup -type=SOA example.com.
```

`Non-authoritative answer` 不等于 answer 不正确，通常说明回答来自 recursive / cached resolution，而非该 zone 的 authoritative server。

工具显示的 DNS server name 如果 unavailable，也不一定意味着 resolver 本身坏了：它的 reverse lookup 可能没有 PTR。

### 15.4 dig

在已安装 BIND tools 等包含 dig 的环境中：

```bash
dig example.com. A
dig example.com. AAAA
dig example.com. NS
dig example.com. SOA
dig example.com. A +tcp
dig example.com. A +trace
```

`+trace` 用于观察 iterative resolution path；它不是显示当前 browser 之前经历过的真实 query history。

直接向 root / authoritative 发 queries 可能被 network policy 限制，因此 trace failure 也不一定表示 normal recursive lookup 不工作。

`dig +dnssec` 主要请求 DNSSEC-related data，**不等于 dig 自动完成完整 cryptographic validation**。要区分获取 signatures、查看 AD 和独立 validation。

### 15.5 Compare Recursive and Authoritative Answers

如果用户看到旧 address：

1. 找到当前 authoritative nameservers。
2. 分别直接查询它们，检查 data / SOA serial 是否一致。
3. 查询用户实际使用的 recursive resolver。
4. 查看 remaining TTL、negative cache 和 DNSSEC failures。
5. 检查 OS / browser / application 是否使用不同路径。
6. 最后检查 existing connection reuse。

如果 authoritative servers 自己就返回不同 values，问题可能在 replication / deployment，而不只是 client cache。

### 15.6 Flush Cache Is a Diagnostic Action, Not a Universal Fix

```powershell
# Optional manual diagnostic action: clears the local Windows DNS client cache.
Clear-DnsClientCache
```

它会修改本机 cache state，但不会清除：

- Public recursive resolver 的 cache。
- Authoritative records。
- Browser 自己的所有 caches。
- Application 保存的 addresses。
- 已经建立的 TCP / QUIC connections。

排错时最好先记录旧 answer 和 TTL，再决定是否清 cache，否则可能丢掉关键证据。

### 15.7 A Symptom Table

| Symptom | 优先检查 |
| --- | --- |
| NXDOMAIN | Name 拼写、zone / delegation、negative cache、search suffix |
| NOERROR 但没有 A | NODATA、alias chain、是否仅有 AAAA、是否为 referral |
| SERVFAIL | DNSSEC、authoritative reachability、delegation、resolver logs |
| UDP 看起来正常，较大 answers 失败 | EDNS size、fragmentation、TCP 53 reachability |
| TCP query 正常，UDP query timeout | UDP filtering、path / size issues |
| VPN 内正常，外部失败 | Split-horizon、private routing、resolver selection |
| 系统工具正常，browser 失败 | Browser DoH / cache、proxy、TLS、HTTP-layer error |
| Address 已改但旧 backend 仍有流量 | Remaining TTL、application cache、connection reuse |
| 某些 regions 才失败 | Anycast / authoritative instance、routing、GeoDNS、resolver differences |
| Authoritative 正确，recursive 仍报不存在 | Negative cache 或 validation / forwarding issue |

### 15.8 Useful Metrics

- Query latency percentiles，而不只 average。
- Cache hit / miss rate。
- RCODE distribution。
- Timeout / retry counts。
- TCP fallback rate。
- DNSSEC validation failures。
- Authoritative response consistency。
- Query volume by name / type / client group。
- Connection reuse 与 upstream resource saturation。

单纯“DNS requests per second 很高”不足以判断用户体验。高 hit rate 下的 workload 与大量 random-name cache misses 对 authoritative 的压力完全不同。

## 16. Common Interview Questions and Scenarios

以下用于练习短答。详细机制在前文；面试时根据追问再展开。

### Q1. What is DNS?

**English answer:** DNS is a hierarchical, distributed naming system that maps domain names to typed resource data, including IP addresses. It uses delegation and caching to make resolution scalable.

不要只停在“把 URL 变成 IP”：它处理 domain names，不解析完整 URL，records 也不限于 addresses。

### Q2. What happens when you resolve www.example.com?

先检查适用 local / recursive caches。Cache miss 时，recursive resolver 跟进 root、TLD、domain authoritative referrals，处理 aliases，取得 answer，按 policy cache 并返回。实际 queries 数量取决于 cache 和 delegation。

### Q3. Recursive vs. iterative?

Recursive service 帮 client 完成 resolution。Iterative exchange 可以返回 referral，让 requester 自己继续。Recursive resolver 通常用 iterative queries 完成自己的工作。

### Q4. Resolver vs. authoritative server?

Resolver 为 client 找结果并 cache；authoritative server 对自己负责的 zone 提供 data。两者是不同 roles，即使部署在同一 machine 上也应区分。

### Q5. Does every lookup reach the root?

No。已有 final answer 或 delegation cache 时，可以直接返回或从更下层开始。

### Q6. Are there only 13 root-server machines?

No。13 指 named server identities；实际由大量 distributed instances 提供服务，常用 anycast。

### Q7. Domain vs. zone?

Domain 是 namespace subtree。Zone 是由 delegation boundaries 划分的 authoritative data 管理范围。一个 domain 可以包含多个 delegated zones。

### Q8. Why do we need glue?

当 child nameserver 的 name 位于被 delegated child 内部时，resolver 需要其 address 才能问 child。Parent glue 提供 bootstrap address，打破 circular dependency。

### Q9. A vs. AAAA?

A 存 IPv4，AAAA 存 IPv6。它们是不同 types / RRsets；一个缺失不意味着另一个也缺失。

### Q10. CNAME vs. HTTP redirect?

CNAME 让 DNS name 指向另一个 name 以继续 resolution；HTTP redirect 由 HTTP response 引导 client 访问另一个 URL。CNAME 不自动改 browser address bar。

### Q11. Why can't a normal CNAME be used freely at the apex?

Apex 需要 SOA / NS，普通 CNAME 与这些其他普通 records 的共存规则冲突。Provider flattening / alias features 是额外实现。

### Q12. Does an A record contain a port?

No。A record 只含 IPv4 address。Port 通常来自 URL、application defaults / configuration，或 SRV / HTTPS 等适用 discovery mechanisms。

### Q13. What does TTL control?

正常 DNS cache reuse lifetime。它不控制 domain registration、IP validity 或 existing connection lifetime。

### Q14. Does a cache hit refresh TTL?

通常不会。Fresh cached data 按剩余 lifetime 返回，不能每次 hit 都无条件重置为 original TTL。

### Q15. Why doesn't lowering TTL at cutover immediately update everyone?

已经缓存旧 record 的 clients 仍持有旧 TTL；新 TTL 不会被 push 到那些 cache entries。应提前降低并等待旧 lifetime 过去。

### Q16. Can negative results be cached?

Yes。NXDOMAIN / NODATA 可以按 negative caching rules 被缓存；新建 record 后仍可能暂时收到旧 negative answer。

### Q17. NXDOMAIN vs. NODATA?

NXDOMAIN 表示 name 不存在。NODATA 表示 name 存在但缺少所查 type；它通常是 NOERROR response 的一种 negative-result semantics，不是独立 RCODE。

### Q18. Is SERVFAIL the same as NXDOMAIN?

No。SERVFAIL 是 resolution failure，可能是 DNSSEC、network、delegation 等问题。不能当成确定不存在。

### Q19. Does DNS only use UDP?

No。传统 DNS 使用 UDP 和 TCP；还有 DoT、DoH、DoQ。General-purpose DNS implementations 需要 TCP support。

### Q20. Why does DNS need TCP?

处理无法通过适用 UDP size 完整返回的数据、zone transfer 等，也可以直接用 TCP。TCP 上仍需要两字节 DNS length framing 和适当 connection management。

### Q21. Is 512 bytes the maximum DNS response size?

No。它是传统无 EDNS 的 UDP message limit。EDNS 可以宣告更大 UDP payload，TCP / encrypted transports 也有不同 framing / limits。

### Q22. What if UDP packets are too large?

可能发生 truncation 并通过 TC 提示 TCP retry；也可能 fragmentation / filtering 导致 packet loss，只看到 timeout。两种现象要区分。

### Q23. Does DNSSEC encrypt queries?

No。它为 DNS data 提供 validation-related authenticity / integrity 和 authenticated denial；query privacy 需要其他 mechanisms。

### Q24. DNSSEC vs. DoH?

DNSSEC 验证 data trust chain；DoH 保护与 HTTPS resolver 的 transport exchange。DoH endpoint 仍能看到 query，并不自动意味着 answer 经 DNSSEC 验证。

### Q25. How does DNSSEC know which key to trust?

通过 configured trust anchor 和 parent DS → child DNSKEY 等链路，逐层验证。不是相信 response 自带的任意 key。

### Q26. Does an unsigned zone always fail DNSSEC validation?

No。要区分 properly established insecure delegation 与本应 secure 却验证失败的 bogus state。

### Q27. Does DNS round-robin guarantee equal traffic?

No。Caches、client selection、connection reuse 和 retries 都影响分配；DNS 不是 per-request scheduler。

### Q28. Can DNS failover instantly move active connections?

No。DNS 改变后续 resolution 得到的 candidates。已有 connections、cache entries 和 application reconnect policy 需要另行考虑。

### Q29. GeoDNS sees whose location?

通常先看到 recursive resolver 的 source address。ECS 等 mechanisms 可以提供额外 client-prefix context，但有 privacy 和 caching tradeoffs。

### Q30. Anycast vs. multiple A records?

Anycast 是多个 locations 通过 routing 提供同一个 IP。Multiple A records 是 DNS 返回多个 address candidates。它们属于不同层次，可以同时使用。

### Q31. Why can getaddrinfo work while a direct DNS query differs?

System name resolution 可能使用 hosts、local caches、search suffix、platform policy 或其他 mechanisms。Browser / direct query tool 也可能使用不同 resolvers。

### Q32. Why does HTTPS still fail after DNS resolves?

Address reachability、TCP / QUIC、TLS hostname verification、server routing、HTTP/application 都是后续独立步骤。DNS success 只解决了部分 endpoint discovery。

### Q33. What is split-horizon DNS?

同一个 name 根据 resolver view / network context 返回不同 data，例如 internal private address 与 external public address。先确认 policy，再判断 differences 是否异常。

### Q34. How would you migrate a service to a new IP?

提前降低 TTL 并等待旧 entries 正常过期，验证新 endpoint / TLS / routing，更新 authoritative data，监控多个 resolvers 与真实 traffic，保留旧 endpoint overlap，并处理 connection draining / rollback。

### Q35. A new AAAA record exists, but some users still see no IPv6 address. Why?

可能是之前 AAAA NODATA 的 negative cache，authoritative replication 未完成，query path 不同，或 application cache。先分层比较 authoritative 和用户实际 resolver 的 answers。

### Q36. How would you debug intermittent SERVFAIL?

记录 resolver、query name/type、时间、TCP vs. UDP、DNSSEC status；比较 authoritative instances、delegation / DS / signatures 和 network path，不把 SERVFAIL 直接归结为“domain 不存在”。

### Q37. Does clearing the OS cache guarantee a fresh authoritative answer?

No。Upstream resolver、browser、application caches 可能还在，existing connections 也不受影响。需要明确实际 resolution path。

### Q38. Does DNSSEC replace HTTPS?

No。两者保护不同层。DNSSEC-valid data 不能证明 application peer 的 TLS identity 或保护 HTTP content；TLS 也不解决所有 DNS data validation / availability 问题。

### Q39. A record has TTL 60. At t=50 the authority changes it. What happens at t=55?

若 client 使用 t=0 获得的 fresh cache，通常仍可返回旧 value，约剩 5 seconds。Authority update 不自动 invalidate 那个 cache。过期后的 refresh 才有机会看到新值。

### Q40. Why can DNS be a distributed system interview topic?

它组合了 delegation、replication、caching、staleness、negative results、partial failures、trust boundaries 和 availability tradeoffs。用具体 resolution / migration 场景解释这些机制，比只背 record names 更有说服力。

## References and Further Reading

- [RFC 1034 — DNS Concepts](https://www.rfc-editor.org/rfc/rfc1034.html)
- [RFC 1035 — DNS Implementation and Specification](https://www.rfc-editor.org/rfc/rfc1035.html)
- [RFC 2308 — Negative Caching](https://www.rfc-editor.org/rfc/rfc2308.html)
- [RFC 2782 — SRV Records](https://www.rfc-editor.org/rfc/rfc2782.html)
- [RFC 4033 — DNSSEC Introduction](https://www.rfc-editor.org/rfc/rfc4033.html)
- [RFC 4592 — DNS Wildcards](https://www.rfc-editor.org/rfc/rfc4592.html)
- [RFC 6891 — EDNS(0)](https://www.rfc-editor.org/rfc/rfc6891.html)
- [RFC 7766 — DNS over TCP](https://www.rfc-editor.org/rfc/rfc7766.html)
- [RFC 8659 — CAA Records](https://www.rfc-editor.org/rfc/rfc8659.html)
- [RFC 9471 — Glue Requirements](https://www.rfc-editor.org/rfc/rfc9471.html)
- [IANA Root Server List](https://www.iana.org/domains/root/servers)
