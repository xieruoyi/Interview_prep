# TCP and Socket Programming

面试准备笔记：**English terminology + 中文解释**。先理解端到端的数据路径，再学习协议机制和 application-level consequences。

前置知识：[Process & Thread](../os/process-thread.md)、[Memory Management](../os/memory.md)。DNS 与 HTTP 只在需要定位层次时提及，详细内容留给各自笔记。

本篇以 **ordinary unicast TCP connections** 为主。握手图省略了 extensions 和部分异常路径；Reno-style congestion control 用作教学模型，不代表所有 TCP implementations 都采用完全相同策略。OS-specific API、defaults 和 timer values 会单独说明，避免把平台行为当成协议常量。

**Examples:** `text` blocks 是示意图或 pseudocode。两个标为 Runnable Python 的示例可独立保存为 `.py` 运行；socket 示例只使用 localhost 和 OS 自动分配的 port，不访问外部服务。

## 1. Where Does TCP Fit?

### 1.1 A Practical Layering Model

```text
Application: HTTP, database protocol, custom messages
                         ↓
Transport: TCP byte stream
                         ↓
Network: IP packets and routing
                         ↓
Link: Ethernet / Wi-Fi frames
                         ↓
Physical transmission
```

发送时，各层增加自己的控制信息；接收时，各层根据相关 headers 处理，再交给上层。

| Layer / component | 主要回答 |
| --- | --- |
| **Application protocol** | Bytes 表达什么，如何划分 request/response |
| **TCP** | 如何建立两个 endpoints 之间的 reliable ordered byte stream |
| **IP** | Packet 应发往哪个 address，经哪些 routes 转发 |
| **Link layer** | 在当前 link 上如何传递 frame |
| **DNS** | Name 如何解析成相关 records，例如 IP addresses；不是每个 TCP packet 都查 DNS |

Routing 可以变化；TCP connection 不是预先保留一条固定的物理线路。

### 1.2 IP Address, Port, and Socket

- **IP address:** 网络层用于定位 interfaces/endpoints 等。一个 host 可以有多个 addresses；NAT/proxy 也会改变外部看到的地址关系。
- **Port:** Transport endpoint 的 16-bit 编号，用于 host 内 demultiplexing。它不是 PID。
- **Socket:** Application 操作通信 endpoint 的 OS/API abstraction，可通过 FD/handle 等引用。

端口号 443 是常用 HTTPS convention，不代表这个端口上的所有 bytes 都必然是 HTTPS。TCP port 53 与 UDP port 53 属于不同 transport namespaces。

`127.0.0.1` 是 IPv4 loopback，连接的是本机。`0.0.0.0` 常用于 server bind 表示监听本机所有 IPv4 interfaces，不是一个应当普遍交给远端 client 的具体目标地址。

### 1.3 Identifying a Connection

在指定 transport/address-family 和网络 context 下，TCP connection 常用 **4-tuple** 区分：

```text
(source IP, source port, destination IP, destination port)
```

例如同一 server 的 port 443：

```text
Client A: 192.0.2.10:51000 → Server: 198.51.100.20:443
Client B: 192.0.2.11:51000 → Server: 198.51.100.20:443
Client C: 192.0.2.10:51001 → Server: 198.51.100.20:443
```

这些是不同 connections。示例地址是 documentation addresses，不是供练习连接的服务。

Network monitoring 中还常见加入 protocol 的 **5-tuple**。Established connection 的 identity，与 listening socket 只根据 local bind endpoint 等接受连接，是不同层面。

### 1.4 Client Ports and Resource Limits

Client 通常不主动 bind，而由 OS 选择 local address 和 ephemeral source port。具体 port range 取决于 OS/configuration。

**16-bit port space 不等于 server 总共只能接 65,535 个 TCP connections。** 同一 listening port 可对应大量不同 remote tuples；实际还受 memory、FD limits、CPU、backlogs、source-port allocation 和 NAT 等限制。

Client 连同一个 destination 时，可用 local tuples 可能成为瓶颈；连接不同 destinations 时的 tuple reuse 规则也不能简单化成“每个 host 一个 port 永远只能用一次”。

## 2. What Does TCP Guarantee?

### 2.1 Connection-oriented, Reliable, Ordered, Full-duplex

TCP 向 application 提供 **connection-oriented, reliable, ordered byte-stream service**：

- **Connection-oriented:** 通信双方维护 connection state。
- **Reliable:** 使用 sequence space、acknowledgments 和 retransmission 等处理丢失；连接无法继续时可能失败。
- **Ordered:** 每个方向交付给 application 的 stream 按相应 byte order 排列。
- **Full-duplex:** 两个方向可独立发送，同时各自维护 sequence/ACK/window state。
- **Byte stream:** 不提供任意 application message boundaries。

TCP base protocol 由 [RFC 9293](https://www.rfc-editor.org/rfc/rfc9293.html) 描述；loss recovery、congestion control 和 extensions 还有各自规范。

### 2.2 Reliability Is Not an Infinite Delivery Promise

“Reliable”不表示：

- 无论断网多久都最终成功。
- Request 在固定时间内完成。
- 对端 application 已读取、验证、持久化或执行数据。
- Connection 中断后可自动恢复任意业务 session。
- Application retries 天然只执行一次。

TCP ACK 主要确认 transport 层收到相关 sequence-space data；business completion 需要 application protocol 明确表达。

### 2.3 Byte Stream Has No Message Boundaries

发送方执行：

```text
send("HELLO")
send("WORLD")
```

接收方可能观察到：

```text
recv() → "HELLOWORLD"
```

或者：

```text
recv() → "HE"
recv() → "LLOW"
recv() → "ORLD"
```

只要整个 stream 的 bytes 与顺序符合协议，这些都正常。中文教程里的“粘包/拆包”通常是在描述 application framing 没有正确处理 stream boundaries，不是 TCP 把消息弄坏。

解决方法见 Section 12：application 自己定义 length prefix、delimiter 等 framing，而不是假设一次 recv 就等于一条 message。

### 2.4 What TCP Does Not Provide

| Requirement | TCP alone? | 应由谁补上 |
| --- | --- | --- |
| Message framing | 否 | Application protocol |
| Encryption / peer authentication | 否 | TLS 等安全协议 |
| Business success acknowledgment | 否 | Application response |
| Exactly-once business effect | 否 | Deduplication、idempotency、transactional design |
| Broadcast / multicast transport service | 普通 TCP 不提供 | 使用其他机制并设计可靠性 |
| Hard real-time deadline | 否 | System/application design，并接受失败处理 |

Checksum 用于发现某些传输损坏，不是 cryptographic integrity 或 authentication；不能把它当成防篡改保证。

### 2.5 TCP vs UDP vs QUIC

| Aspect | TCP | UDP |
| --- | --- | --- |
| Application interface | Ordered byte stream | Datagrams |
| Delivery/order handling | 协议内提供相应机制 | 不内置同等可靠、有序交付 |
| Connection establishment | 普通 TCP 有 handshake | UDP 协议本身无 TCP 式 handshake |
| Flow/congestion control | TCP 的核心组成 | UDP 本身不提供同等机制，application/上层需负责 |
| Message boundaries | 不保留 | 保留 datagram boundaries；过小 receive buffer 可能截断 |
| Typical choice | 需要 stream 的通信 | 自定义 transport、允许丢弃过时数据、部分实时通信 |

UDP 不是“永远更快”，TCP 也不是“只适合网页”。选择要看 latency、loss tolerance、ordering、security 和 implementation requirements。

**QUIC** 是构建在 UDP 上的 transport，提供自身的可靠 streams、congestion control 和安全握手等机制；HTTP/3 使用 QUIC。不能因为 UDP 本身无可靠性，就说 QUIC 没有可靠传输。[RFC 9000](https://www.rfc-editor.org/rfc/rfc9000.html)、[HTTP/3](https://www.rfc-editor.org/rfc/rfc9114.html)

## 3. Segments, Headers, MTU, and MSS

### 3.1 Segment vs Packet vs Frame

- **TCP segment:** TCP header 加 payload。
- **IP packet:** IP header 加其 payload，例如 TCP segment。
- **Link-layer frame:** Link header/trailer 加上层 packet 等。

日常讨论常把它们都叫 packet，但计算 overhead、MTU 或分析 captures 时要区分。

Application 一次 write 的 data 可以被分成多个 TCP segments。多个小 writes 也可能被组合；offloading 还会影响 host capture 看到的形态。

### 3.2 Important TCP Header Fields

| Field / flag | Meaning |
| --- | --- |
| **Source / destination port** | Endpoints 的 transport identifiers |
| **Sequence number** | 当前 segment 在发送方向 sequence space 的位置 |
| **Acknowledgment number** | 累计确认对向 stream 的位置，通常表示下一个期待的 sequence number |
| **Data offset** | TCP header 长度 |
| **Window** | Receiver advertised window，供对方实施 flow control |
| **Checksum** | 错误检测，包含 TCP header/data 与 IP pseudo-header 信息 |
| **SYN** | 同步 initial sequence space，建立 connection |
| **ACK** | Acknowledgment field 有效 |
| **FIN** | 发送方在这个方向不再发送新的 stream data |
| **RST** | Reset/abort 等异常终止相关控制 |
| **PSH** | 提示及时向接收 user 交付数据，不是 message delimiter |
| **ECE / CWR** | 与特定 ECN/congestion signaling 机制有关 |
| **Options** | MSS、window scale、timestamps、SACK 等 |

TCP 基本 header 是 20 bytes；options 增加 header 长度，常规 header 最大 60 bytes。IP header 长度也不一定固定。

不必把所有 flags 都背成一行口诀；优先解释 SYN、ACK、FIN、RST 如何改变 state。

### 3.3 MTU vs MSS

**MTU** 常用于描述某个 link 能承载的最大 network-layer packet size。**Path MTU** 受路径上相关 links 的限制。

**MSS** 描述 TCP data payload 的大小上限，不包括 TCP/IP headers；它不是整个 packet 大小。

教学计算，假设 MTU=1500 bytes，IPv4 无 options，TCP 无 options：

```text
1500 - 20-byte IPv4 header - 20-byte TCP header = 1460 bytes
```

若为普通 40-byte IPv6 header、同样 20-byte TCP header：

```text
1500 - 40 - 20 = 1440 bytes
```

实际 payload 还要考虑 TCP/IP options、extension headers、tunnels、peer MSS 和 path constraints。不能把“任何网络 MSS 都是 1460”当成事实。

### 3.4 MSS Negotiation Is Directional

SYN 中的 MSS option 表达发送这个 option 的 endpoint 愿意接受的 segment data size。双方各自声明，两个方向不一定完全相同。

Sender 还要符合实际 path constraints，不是只看 peer 声明就可以无限按那个大小发送。

IP fragmentation 与 TCP segmentation 不同。前者是 IP 层对 packet 的处理；后者是把 stream data 放进 TCP segments。一个 oversized packet 被拆成 IP fragments，不等于拆成多个独立 TCP segments。

### 3.5 Path MTU Problems and Offloading

若路径某处不接受大 packets，而相关控制信息又无法正常返回，可能出现“小请求正常，大传输卡住”的现象。需要检查 PMTU discovery、encapsulation、firewall 和实际 packet sizes；不要只归因于 application buffer。

TSO/GSO/GRO 等 offloading/aggregation 可能让本机 capture 显示比普通 wire MTU 更大的 chunks，或显示尚未由 NIC 完成的 checksum。分析前先确认 capture point，不要立即断言线上发送了非法 packets。

## 4. Connection Establishment: The Three-way Handshake

### 4.1 A Concrete Example

Client initial sequence number 为 1000，server 为 9000：

```text
Client                                      Server
CLOSED                                      LISTEN
   |
   | SYN, seq=1000
   |---------------------------------------->
SYN-SENT                                  SYN-RECEIVED
   |                  SYN+ACK, seq=9000, ack=1001
   |<----------------------------------------
   | ACK, seq=1001, ack=9001
   |---------------------------------------->
ESTABLISHED                               ESTABLISHED
```

Diagram 展示最终状态；client 通常在收到有效 SYN+ACK 并发送 final ACK 时进入 established，server 在收到有效 final ACK 后进入 established。

SYN 消耗一个 sequence number，所以第一次普通 data 的起点分别是 1001 与 9001。没有 payload 的普通 ACK 不额外消耗 sequence number。

### 4.2 What the Three Messages Establish

- Client SYN：提出连接并提供 client initial sequence space 与部分 options。
- Server SYN+ACK：确认 client SYN，同时提供自己的 initial sequence space。
- Client ACK：确认 server SYN，让 server 知道 client 收到了它的回应。

第三步不是单纯“再客气确认一次”。它完成双向 sequence synchronization，并帮助避免把某些旧的重复 connection requests 误当成双方都已认可的新连接。

不要用“数学上所有通信永远至少三次”解释：不同协议有不同 assumptions，TCP 也有 extensions 和特殊 paths。这里说的是普通 TCP handshake 的设计。

### 4.3 Why Initial Sequence Numbers Matter

ISNs 不是所有 connections 固定从 0 开始。不同 connection incarnations 的 sequence spaces、旧 segments，以及安全考虑都影响 ISN selection。

Wireshark 常显示相对 sequence numbers，例如 SYN=0、first data=1，是为了便于阅读，不说明 wire 上实际 ISN 为 0。

Sequence numbers 按 32-bit space 运算，比较需要考虑 wraparound；不能对所有情况直接套普通无符号整数大小比较。

### 4.4 What If a Handshake Message Is Lost?

| Lost message | Typical consequence |
| --- | --- |
| First SYN | Client 等待后重传，最终也可能超时失败 |
| SYN+ACK | Server/client 对应 retransmission paths 会尝试恢复 |
| Final ACK | Client 可能认为已 established，server 暂时仍等待确认；server 重传 SYN+ACK，client 再确认，后续带有效 ACK 的 data 也可能完成确认 |

普通 pure ACK 没有自己的“可靠 ACK-of-ACK 链”。协议通过对需要确认的 sequence-space events 的重传与后续 acknowledgments 恢复。

Retries 次数、timer progression 和 connect timeout 取决于 implementation/configuration，不能背成所有 OS 的固定秒数。

### 4.5 Listening, Backlogs, and accept

Server 常先 `bind`、`listen`，kernel 处理 handshake；application 调用 `accept` 取出完成的 connection，获得新 connected socket。

**accept 不是额外的第四次握手，也不要求 server application 每处理一个 SYN 都亲自运行一次。**

Linux 中需要区分 incomplete requests 与等待 application accept 的 established connections；`listen(backlog)` 对应队列的具体含义、上限与历史行为依 OS。[Linux listen](https://man7.org/linux/man-pages/man2/listen.2.html)

### 4.6 SYN Flood and SYN Cookies

大量未完成 handshakes 可能消耗 state。SYN cookies 是在某些情况下减少 server 为未验证请求保留 state 的技术，但不能消除 bandwidth、CPU 或所有形式的 overload。

这里作为 availability 背景理解即可；它不是加密通信，也不提供 application authentication。

普通握手从 client 看约一个 RTT 后可以发送普通 application data；TLS/application handshakes 可能增加额外 startup cost。Connection reuse 和 TCP Fast Open 等机制会改变部分流程，面试先明确是否考虑 extensions。


## 5. Sequence Numbers, ACKs, and Ordered Delivery

### 5.1 TCP Counts Bytes, Not Application Messages

假设 sender 下一 byte 的 sequence number 是 1001，发送 500 bytes：

```text
Segment: seq=1001, payload length=500
Covers bytes: 1001 through 1500
Next expected byte: 1501
```

Receiver 可以返回 `ACK=1501`，表示已经连续收到该 stream 中直到 1500 的 bytes，并期待后续 bytes 从 1501 开始。

它不是“第 1501 个 packet”，也不表示 application 完成了第 1501 个 request。

### 5.2 Cumulative ACK

**Cumulative ACK** 确认一个连续前缀。后面的 bytes 到达，但前面仍有 hole 时，累计 ACK 通常停在 hole 起点：

```text
Received: [1001,1501)
Missing:  [1501,2001)
Received: [2001,2501)

Cumulative ACK = 1501
```

本篇用半开区间 `[start,end)` 表示 bytes：包含 start，不包含 end，长度为 end-start。

当 missing range 也到达，receiver 就可以把连续前缀向后推进；它不需要为已经缓存好的后续 bytes再等一份相同数据。

### 5.3 Out-of-order Arrival vs Ordered Delivery

IP network 可以让 segments 丢失、重复或乱序。TCP receiver 可以保留 out-of-order data，按 sequence space 合并并去除重复，随后按序交付。

```text
Network arrival:   part A → part C → part B
Application sees:  part A → part B → part C
```

part C 已经在 receiver memory 中，也不代表 application 可以跳过 B 直接从该 TCP stream 中读取 C。

这会造成 **transport-level head-of-line blocking**：早先一个 missing byte range，可能阻挡同一 stream 后续已经到达的数据。

### 5.4 SYN, FIN, ACK, and PSH

| Event | Consumes sequence space? |
| --- | --- |
| N bytes of payload | N |
| SYN | 1 |
| FIN | 1 |
| Pure ACK without payload/SYN/FIN | 0 |

一个 segment 可以组合 control flags 与 payload，因此计算 next sequence number 时要看实际内容。

PSH 不是 message boundary；receiver 不应把“看到 PSH”当成已经得到完整 application message。Application 应基于自己的 framing 判断。

### 5.5 ACK Loss and Duplicate Data

若 ACK 丢失：

- 后续更大的 cumulative ACK 可能一起确认前面的 bytes。
- 若 sender 长时间得不到足够确认，可能重传。
- Receiver 利用 sequence space 识别重复，不把同一 range 作为新 bytes 再交给 stream reader。

这说明“network retransmission”通常不会让 application 从同一个 TCP stream 读到重复的那段 bytes。

但 **application 重连后重发 request** 是另一回事。它发送了新的 stream bytes，TCP 不知道业务上是不是同一个 operation，见 Section 15。

### 5.6 Independent Sequence Spaces

Full-duplex connection 的两个方向有各自的 sequence numbers：

```text
A → B: seq_A, acknowledged by B
B → A: seq_B, acknowledged by A
```

B 发 data 时可同时 piggyback 对 A 的 ACK。B 的 sequence number 不需要因为确认 A 的 500 bytes 而增加 500；它由 B 自己发送的 sequence-space events 决定。

## 6. Loss Detection and Retransmission

### 6.1 Sender Retains Unacknowledged Data

Sender 通常需要保留尚未被可靠累计确认的 stream data，以便在 loss 时重传。Kernel send buffer、application buffer 和实际 wire transmission 是不同层次。

概念上可以把发送序列分成：

```text
[acknowledged][sent but unacknowledged][not yet sent]
```

这些 ranges 随 ACK、新 writes 和 transmission policy 变化。Socket API 接受了 data，并不表示所有 bytes 已出现在网络上。

### 6.2 Retransmission Timeout (RTO)

Sender 在缺少足够确认时使用 retransmission timer。RTO 不应该只取某个固定 RTT，因为 RTT 会变化。

常见 estimator 思路：

```text
RTO ≈ SRTT + max(clock_granularity, 4 * RTTVAR)
```

- **SRTT:** Smoothed RTT estimate。
- **RTTVAR:** RTT variation estimate。
- **RTO:** 决定何时认为需要 timeout recovery 的 interval，不等于 RTT 本身。

RFC 6298 使用平滑估计、初始化/minimum rules，并在 timeout 后对 RTO 进行 exponential backoff。具体 OS 的实现策略和 timer bounds不应凭印象套成统一数字。[RFC 6298](https://www.rfc-editor.org/rfc/rfc6298.html)

### 6.3 Karn's Algorithm and RTT Ambiguity

假设原 segment 与 retransmission 都可能引起后来收到的 ACK：

```text
Original send ──────?
Retransmission ────?──> ACK arrives
```

若无法区分 ACK 对应哪次 transmission，用它直接测 RTT 会产生歧义。Karn-style handling 避免从这种 ambiguous retransmitted data 取得普通 RTT samples；timestamps 等机制可以帮助解除某些歧义。

这个问题不是“重传后的网络一定慢”，而是 measurement 的起点不确定。

### 6.4 Fast Retransmit: Detect a Hole Before RTO

教学例子，所有后续 segments 都到达 receiver：

```text
Sender range          Receiver reaction
[1001,1501) arrives    ACK 1501
[1501,2001) lost       -
[2001,2501) arrives    duplicate ACK 1501
[2501,3001) arrives    duplicate ACK 1501
[3001,3501) arrives    duplicate ACK 1501
```

经典 Reno-style fast retransmit 在满足相关条件的第三个 duplicate ACK 后，可以重传 presumed missing segment，不必一直等 RTO。

Missing `[1501,2001)` 到达后，若后续 ranges 都保留着，cumulative ACK 可以跳到 3501。

**Duplicate ACK 不一定证明 packet 丢失**，也可能由 reordering 等引起。Threshold 与 recovery logic 是在快速反应和避免误判之间取舍。

### 6.5 SACK

**Selective Acknowledgment** 让 receiver 在 cumulative ACK 之外报告已经收到的非连续 ranges，例如：

```text
Cumulative ACK = 1501
SACK block = [2001,3501)
```

Sender 因而更清楚哪些 holes 需要修复，尤其在一轮中有多处 loss 时。SACK 是 receiver 报告信息，不是 receiver 替 sender 执行 retransmission。

SACK 也不把 TCP 变成对 application 无序交付的 datagram protocol；应用仍读取 ordered stream。[RFC 2018](https://www.rfc-editor.org/rfc/rfc2018.html)

### 6.6 Modern Recovery and Tail Loss

若 stream 尾部丢失，可能没有足够后续 packets 产生多次 duplicate ACK。仅靠“三个 dup ACK”并不能解释所有 loss recovery。

现代 implementations 可能使用 RACK/TLP 等 timing-based loss detection/probing，减少某些场景下等待长 RTO 的成本。[RFC 8985](https://www.rfc-editor.org/rfc/rfc8985.html)

面试先讲 sequence/ACK + timeout，再讲 classic fast retransmit/SACK；不要声称所有 TCP stacks 只实现这套简化 Reno 流程。

## 7. Flow Control: Protect the Receiver

### 7.1 Why a Fast Sender Can Overwhelm a Slow Reader

Receiver 的 application 可能读取较慢，kernel receive buffer 中的数据不断积累。

```text
Sender → network → receiver TCP buffer → slow application
```

**Flow control** 让 sender 尊重 receiver 的可接收范围，避免无限把数据塞给它。

注意：receiver application 慢，可能是业务处理慢、没有及时 read、线程被阻塞等，不一定是网络带宽不足。

### 7.2 Receive Window (rwnd)

Receiver 在 ACK 等 TCP headers 中公布 receive window。Sender 用相应 ACK position 与 window 信息，知道 receiver 允许接收的 sequence range。

在简化的稳定 snapshot 中：

```text
Allowed outstanding data is limited by receiver capacity.
```

Advertised window 与可用 receive buffering 有关，但不必恰好等于某个 API 设置值减去已读 bytes；autotuning、accounting 和 OS policy 会影响实际值。

### 7.3 Zero Window and Persist Probing

如果 rwnd=0，sender 不应继续像有正常窗口一样大量发送新 data。

当 application 读取 buffer 后，receiver 可以发 window update。若这个 update 丢失，双方可能形成“receiver 以为已通知、sender 仍以为窗口为零”的停滞。

Zero-window probing/persist mechanism 帮助 sender 重新探测窗口状态。它与检测网络拥塞的普通 retransmission recovery目的不同。

Zero window 表示 receiver 的 flow-control 状态，不自动意味着 peer crashed，也不是“把 connection 永久关闭”。

### 7.4 Window Scaling

TCP header 的原始 window field 是 16 bits。为了支持更大 receive windows，可以在 handshake 中协商 window scaling。

概念公式：

```text
Effective advertised window = field_value * 2^scale
```

双方的 scale factor 可以不同；它们分别用于解释各自方向的 advertised window。SYN segments 中的 window field 不按此方式缩放。

例如 field=32768、scale=4，得到 524288 bytes，即 512 KiB。它不是扩大 MSS，而是扩大 window 的表达能力。[RFC 7323](https://www.rfc-editor.org/rfc/rfc7323.html)

### 7.5 Application Backpressure

Transport flow control 只有在 application 的读取和处理保持适当关系时，才能帮助控制端到端积压。

若 application 不断从 socket 读出数据，再放进无上限 user-space queue：

```text
TCP buffer stays small
       ↓
rwnd can stay open
       ↓
Unbounded application queue grows
```

因此，即使 TCP 有 flow control，application 仍可能内存耗尽。需要 bounded queues、限制 outstanding requests、暂停读取或其他 backpressure policy。

## 8. Congestion Control: Protect the Network

### 8.1 Different from Flow Control

Receiver 容量充足，不代表 network path 也有容量。Intermediate links、queues 和竞争流量都可能成为瓶颈。

| Mechanism | Protects | Main signal/state |
| --- | --- | --- |
| **Flow control** | Receiver capacity | Advertised rwnd |
| **Congestion control** | Network path | Sender cwnd、ACK/loss/ECN/timing 等 |
| **Application concurrency limit** | Service/business resources | Outstanding requests、queue capacity |

三个限制可能同时存在，不应只调大 socket buffer 就期待解决所有吞吐问题。

### 8.2 Congestion Window (cwnd)

**cwnd** 是 sender 内部维护的 congestion-control state，不是 receiver 在 TCP header 中公布的 rwnd。

简单教学 snapshot：

```text
Effective in-flight bound ≈ min(cwnd, rwnd)

Additional data credit ≈
    max(0, min(cwnd, rwnd) - bytes_in_flight)
```

这个公式帮助理解“双重限制”；实际 sending/recovery、window edges、pacing 和 SACK-based flight estimates 更复杂，不能当成所有 TCP code 的精确实现。

例如 cwnd=12 KiB、rwnd=20 KiB、已有8 KiB in flight，简化可新增4 KiB；如果 rwnd只有6 KiB，就不能因为cwnd较大继续新增。

### 8.3 Slow Start

Classic slow start 在 ACKs 反映 network 有进展时较快扩大 cwnd，以探测可用 capacity。

理想示意，假设从2 MSS开始、data充足且ACK行为理想：

```text
RTT 0: 2 MSS
RTT 1: 4 MSS
RTT 2: 8 MSS
RTT 3: 16 MSS
```

这是大致每 RTT 倍增的教学模型，**不是规定所有连接 initial window 都是2 MSS，也不是保证每个 RTT精确翻倍**。ACK policy、application limitation 与implementation都影响增长。

名字里的 slow 是相对一开始就向网络灌入巨大窗口，不表示一直线性缓慢增长。

### 8.4 Congestion Avoidance and Recovery

Classic Reno-style 模型中：

- `ssthresh` 用于区分快速增长与较保守的 congestion avoidance。
- Congestion avoidance 通常表现为大约每 RTT 增加一个 MSS。
- Loss signal 会触发减小发送规模等反应。
- Fast recovery 与 RTO recovery 的处理不同；不能一概说“每丢一包都完全重新开始”。

用 **AIMD** 理解经典 additive increase / multiplicative decrease 的取舍：逐渐探测容量，出现拥塞迹象时明显回退。[RFC 5681](https://www.rfc-editor.org/rfc/rfc5681.html)

### 8.5 CUBIC, BBR, and ECN

- **CUBIC:** 使用不同于 Reno 线性增长的 window evolution，适用于多种大带宽/较长RTT路径；具体算法见 [RFC 9438](https://www.rfc-editor.org/rfc/rfc9438.html)。
- **BBR family:** 基于对路径 bandwidth 与 propagation delay 等的估计进行控制；versions 和部署行为有区别，不能当成所有主机的默认算法。
- **ECN:** 在 endpoints 与 network 支持并正确协商/处理时，使用 congestion marking 等信息，不必每次都依靠 packet drop 才反馈拥塞。

面试不必背每个算法所有公式；应明确“TCP 允许不同 congestion-control algorithms”，以及 loss、delay、bandwidth estimates 等 signals 的作用。

### 8.6 ACK Clocking, Pacing, and Bufferbloat

ACKs 的返回能反映部分传输进展，帮助调节继续发送；**pacing** 进一步控制发送时间分布，避免仅按窗口可用量突然形成大 burst。

Queues 太大可能让 packets 少丢，却在里面等待很久，造成 **bufferbloat**。因此“没有 loss”不等于 latency 好；增加 buffer 不总是正确的优化。


## 9. Throughput, Latency, and Practical Performance

### 9.1 Bandwidth-delay Product (BDP)

为了让高带宽、长RTT路径持续有数据在传输，通常需要允许足够多的 bytes 同时 in flight。

```text
BDP = bandwidth * RTT
```

先统一 units：bandwidth 若是 bits/s，转换成 bytes/s 后再乘 seconds。

例子：100 Mbit/s，RTT=100 ms：

```text
100,000,000 bits/s / 8 * 0.1 s
= 1,250,000 bytes
= 1.25 MB, about 1.19 MiB
```

如果有效窗口只有256 KiB，理想的 window/RTT bound：

```text
262,144 bytes / 0.1 s = 2,621,440 bytes/s
                         ≈ 20.97 Mbit/s
```

即使 link 标称100 Mbit/s，过小窗口也可能限制单条connection throughput。实际还受loss、congestion policy、processing、headers和其他flows影响，不能把这个bound当成保证。

### 9.2 Runnable Python: BDP and a Window Bound

```python
bandwidth_mbps = 100
rtt_seconds = 0.100
window_bytes = 256 * 1024

bdp_bytes = bandwidth_mbps * 1_000_000 / 8 * rtt_seconds
window_bound_mbps = window_bytes / rtt_seconds * 8 / 1_000_000

print(f"BDP: {bdp_bytes:,.0f} bytes")
print(f"Window bound: {window_bound_mbps:.2f} Mbit/s")
assert bdp_bytes == 1_250_000
# BDP: 1,250,000 bytes
# Window bound: 20.97 Mbit/s
```

单位里的 MB 与 MiB、Mbit 与 Mbyte 不可互换。面试计算时先写清单位，再代入公式。

### 9.3 Startup vs Steady State

一个小request的完成时间可能主要由这些部分组成：

- Name resolution。
- TCP handshake。
- TLS handshake。
- Application processing。
- Network round trips。

大文件steady-state transfer则可能主要受bandwidth、window、loss和storage速度影响。

因此，只优化大流throughput，不一定能降低短请求latency。Connection reuse可以减少重复startup成本，但也需要pool limits、idle handling和失败恢复。

### 9.4 Nagle's Algorithm and Delayed ACK

**Nagle's algorithm** 的基本目标是减少大量tiny segments：在已有未确认data时，对新的小量data进行合并等待，但完整MSS等条件会影响实际发送。

**Delayed ACK** 允许receiver在一定条件下稍后发送ACK，例如等待更多data或与反向data合并，不是每收到一个segment都立即发一个独立ACK。

某些小消息request/response patterns下，两种策略可能相互等待而增加latency。具体是否发生、等待多久，要看implementation与application写入方式，不能一概说必定延迟固定毫秒数。

`TCP_NODELAY` 可关闭Nagle，适合部分latency-sensitive workloads，但：

- 不关闭TCP的其他buffering/scheduling。
- 不保证每次write独立成为一个segment。
- 不提供message boundaries。
- 不保证总throughput更好。

可以先减少无意义的碎片writes、采用合理framing，再根据capture和测量判断是否需要此选项。[Linux TCP options](https://man7.org/linux/man-pages/man7/tcp.7.html)

### 9.5 More Connections Are Not Unlimited Bandwidth

更多connections可能覆盖等待、分摊某些限制，但它们仍共享bottleneck link、server CPU和memory。也可能增加handshakes、TIME_WAIT、FD usage和tail latency。

Server thread pool与connection pool不是同一个概念：前者管理执行工作的人，后者管理可复用的通信connections。一个worker可以先后处理多个connections，一个connection也不一定永久绑定一个worker。

## 10. Connection Termination and TCP States

### 10.1 Closing Two Directions

TCP是full-duplex，两边的发送方向需要分别结束。FIN表示：

> 我在这个方向不会再发送新的stream data；请在此前的数据之后观察到结束。

它不等于“我拒绝再接收任何data”。Application可以结束自己的写方向，继续读取对方response。

### 10.2 A Typical Four-segment Close

假设A先结束发送，next seq=2001；B结束发送时next seq=5001：

```text
A: active closer                            B
ESTABLISHED                                 ESTABLISHED
    | FIN, seq=2001
    |---------------------------------------->
FIN-WAIT-1                                  CLOSE-WAIT
    |                           ACK=2002
    |<----------------------------------------
FIN-WAIT-2
    |                  B application finishes
    |                           FIN, seq=5001
    |<----------------------------------------
    | ACK=5002
    |---------------------------------------->
TIME-WAIT                                   CLOSED
    |
    | wait according to TIME_WAIT rules
    v
CLOSED
```

B在发送FIN后、收到最后ACK前处于LAST-ACK；图中仅压缩显示部分时刻。

FIN消耗一个sequence number，所以对应ACK加1。A进入FIN-WAIT-2后仍可以接收B此前或随后发送、但位于B的FIN之前的data。

### 10.3 Why Four Segments Are Not Mandatory

收到FIN后，可以先ACK，再等application完成后发送自己的FIN，因此常见四步。

如果对方当时也准备结束发送，它的ACK与FIN可以组合，形成三segments等实际形态。还存在simultaneous close与retransmissions，所以面试应说“典型四步”，而不是“TCP关闭永远恰好四个packets”。

### 10.4 Important States

| State | Meaning / diagnostic interpretation |
| --- | --- |
| **CLOSED** | 没有活跃的该connection state |
| **LISTEN** | 被动等待incoming connection |
| **SYN-SENT** | 发出SYN，等待建立 |
| **SYN-RECEIVED** | 收到SYN并回复，等待完成确认 |
| **ESTABLISHED** | 可以进行正常双向data transfer |
| **FIN-WAIT-1** | 已发送自己的FIN，等待相应ACK或peer FIN |
| **FIN-WAIT-2** | 自己的FIN已确认，继续等peer结束发送 |
| **CLOSE-WAIT** | 已收到peer FIN，等待本地application结束自己这边 |
| **LAST-ACK** | 已在被动关闭路径发出FIN，等待确认 |
| **CLOSING** | 某些simultaneous-close路径中，已收到peer FIN但自己的FIN仍待确认 |
| **TIME-WAIT** | 保留终止状态，处理最后ACK丢失与旧segments等问题 |

不同工具的state名称可能略有缩写，例如SYN-RECV。Connection state是kernel层状态，不等于application request状态。

### 10.5 Why TIME_WAIT Exists

典型active closer发送最后ACK后保留state，主要帮助：

- Peer若未收到最后ACK而重传FIN，本端仍可再次回应。
- 降低旧connection的延迟segments影响后续使用同一tuple的connection的风险。

经典模型使用 **2 × MSL (Maximum Segment Lifetime)**。MSL不是RTT，也不是一个跨平台统一的固定秒数。

**TIME_WAIT不必在client上**；谁按相应关闭路径成为active closer才是关键。Server可以主动close，simultaneous close也可能让双方进入TIME_WAIT。

大量TIME_WAIT未必是bug，但短连接高频建立/关闭可能增加资源和tuple压力。先看reuse、traffic模式和限制，不要第一步就修改kernel timers。

### 10.6 CLOSE_WAIT vs TIME_WAIT

- 大量持续的CLOSE_WAIT：Peer已经结束发送，本地application可能没有及时close或仍在完成合理工作。重点检查本地lifetime/cleanup。
- TIME_WAIT：常见协议生命周期的一部分，不能把它与“application忘记关闭”简单等同。

检查state的持续时间与连接数量，而不是只看一张瞬时截图。

### 10.7 RST, Graceful Close, and Half-open Connections

**FIN** 是按序结束一个stream方向。**RST** 表示reset/abort相关情况，通常会让application观察到连接异常；具体错误与何时暴露依OS/API。

Graceful transport close也不自动表示业务成功。A可能正常发送完一个格式错误的request，B也可能在未完成业务时正常结束connection。

**Half-closed:** 一个方向已结束，另一个方向仍可传data。

**Half-open:** 某一方丢失connection state或不可达，另一方尚未得知。它不是有意的half-close，也不一定被立即发现。

## 11. Socket APIs and the Data Path

### 11.1 Server and Client Lifecycle

```text
Server                                     Client
socket()
bind(local_address)
listen()
accept() waits                             socket()
                                            connect(server_address)
   connected socket <────────────────────── connected socket
send / recv                                send / recv
shutdown / close                           shutdown / close
```

Listening socket与accept返回的connected socket是不同objects。关闭其中一个connected socket，通常不影响listener接受其他connections。

### 11.2 Where Data Goes

```text
Sender application buffer
          ↓ send/write API
Sender kernel send buffering / TCP state
          ↓ segmentation, congestion/flow control
Network
          ↓
Receiver TCP buffering and reassembly
          ↓ recv/read API
Receiver application buffer
          ↓ parse, validate, execute
Business result
```

这条路径解释了为什么“send成功”“ACK到了”“业务成功”是不同事件。

### 11.3 send vs sendall

在一般stream socket API里：

- `send(data)` 返回本次接受的byte count，可能小于data长度。
- Application必须处理剩余bytes，不能直接丢弃。
- Python `sendall(data)` 帮你循环直到提交完这些bytes，或发生error。

成功通常意味着相应data已按API规则交给本地发送路径，不表示对端application已经read或执行。

若sendall中途失败，不能据此推断“对方完全没收到”；部分data可能已经到达。重试要由application protocol判断。[Python socket API](https://docs.python.org/3/library/socket.html)

### 11.4 recv Semantics

当请求长度大于0时，常见blocking TCP `recv(n)`：

- 最多返回n bytes，不保证填满。
- 有data时可以返回更少的bytes。
- 暂时没有data且没有EOF/error时，通常等待。
- 正常读完peer发送方向后返回empty bytes，Python中为`b""`。
- Reset、timeout等可能以exception/error报告。

`b""`通常不是“现在没消息，稍后再试”；在上述TCP调用前提下它表示stream EOF。相反，`recv(0)`返回空值不能用来证明peer关闭。

Non-blocking socket暂无可读data时，通常报告would-block，而不是把它伪装成EOF。[recv](https://man7.org/linux/man-pages/man2/recv.2.html)

### 11.5 Blocking, Non-blocking, and Readiness

**Blocking:** Caller可能等待data、buffer space或connection events。

**Non-blocking:** 暂时不能完成时返回特定状态，例如EAGAIN/EWOULDBLOCK；application随后通过readiness/event机制重试。

Readiness例如“现在有可能读而不block”，不代表已有完整application frame，也可能表示EOF/error。即使有readiness通知，robust non-blocking code仍需处理实际操作结果。

`select/poll/epoll/kqueue`等是不同平台的notification机制，不改变TCP byte-stream semantics。Edge-triggered loop通常要正确drain直到would-block，具体规则依API。

### 11.6 shutdown vs close

`shutdown(SHUT_WR)`结束socket的发送方向，在适当正常路径下把此前data之后的FIN安排出去；仍可以继续read。

`close`释放当前FD/handle reference；最后一个相关reference关闭后的底层处理，还受socket state、pending data和options等影响。不要无条件认为close永远立即发送FIN并保证所有data已处理。

POSIX中，dup等造成多个references时，close一个FD与对底层connection做shutdown也不是同一件事。[shutdown](https://man7.org/linux/man-pages/man2/shutdown.2.html)

### 11.7 Multiple Readers or Writers

多个threads从同一socket读取，bytes会由实际reader消费，不会自动复制给每个thread。很容易破坏一个parser独占stream的假设。

多个writers分多次calls发送各自的header/body，也可能使application frames交错。可以让一个connection owner统一读写，或用适当同步保护完整frame serialization。

TCP保持实际写入stream的byte order，但不知道你原本想把哪几次calls合成一个message。


## 12. Message Framing and a Complete Python Example

### 12.1 Common Framing Strategies

| Strategy | Example | Trade-off |
| --- | --- | --- |
| **Fixed size** | 每条record固定32 bytes | 简单，但不适合任意长度 |
| **Delimiter** | 每条消息以newline结束 | Payload内相同delimiter需要escaping或格式约束 |
| **Length prefix** | 先传4-byte length，再传body | 必须校验length、处理partial header/body |
| **Connection EOF** | Sender结束写方向表示body结束 | 不能自然在同方向连续发送多个独立messages |

HTTP等协议定义了自己的framing，不能随意替换。本节构建一个教学用custom protocol。

### 12.2 Define the Wire Format First

```text
One frame:
+-------------------------+----------------------+
| 4-byte unsigned length  | exactly length bytes |
| network byte order     | body                 |
+-------------------------+----------------------+
```

约定：

- Length计算 **bytes**，不是Unicode characters。
- Network byte order为big-endian。
- Maximum body size为64 KiB，这是本示例的application limit，不是TCP limit。
- Zero-length message合法。
- 在新header开始前遇到EOF，表示没有下一条frame。
- Header/body只收到一部分就EOF，属于truncated frame，不能当成正常完整message。

若要传字符串，先编码，例如UTF-8，再根据encoded bytes计算length。

### 12.3 recv_exact Must Loop

概念逻辑：

```text
while collected < requested:
    read up to remaining bytes
    if EOF before completion:
        report truncation
    append received bytes
```

它不会阻止TCP把data分段，只是application根据自己的格式把正确数量的bytes组装起来。

Declared length需要在读取/分配巨大body之前验证，否则一个极大的length field就可能让程序无限等待或过度allocation。

### 12.4 Runnable Python: Two-way Framed Exchange on Localhost

保存为 `tcp_demo.py` 后运行 `python tcp_demo.py`。Server与client在同一Python process中，通过一个真实localhost TCP connection通信；server运行在一个worker thread中。这是方便演示，并非TCP要求双方必须是threads。

```python
import socket
import struct
from concurrent.futures import ThreadPoolExecutor

MAX_FRAME = 64 * 1024

def recv_exact(sock, count, allow_clean_eof=False):
    data = bytearray()
    while len(data) < count:
        chunk = sock.recv(count - len(data))
        if not chunk:
            if allow_clean_eof and not data:
                return None
            raise EOFError("Connection ended inside a frame")
        data.extend(chunk)
    return bytes(data)

def recv_frame(sock):
    header = recv_exact(sock, 4, allow_clean_eof=True)
    if header is None:
        return None

    (length,) = struct.unpack("!I", header)
    if length > MAX_FRAME:
        raise ValueError("Frame exceeds application limit")
    return recv_exact(sock, length)

def send_frame(sock, body):
    if len(body) > MAX_FRAME:
        raise ValueError("Frame exceeds application limit")
    header = struct.pack("!I", len(body))
    sock.sendall(header + body)

def serve_one_connection(listener):
    conn, _ = listener.accept()
    with conn:
        conn.settimeout(5)
        while True:
            body = recv_frame(conn)
            if body is None:
                return
            send_frame(conn, body.upper())

def main():
    responses = []
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as listener:
        listener.settimeout(5)
        listener.bind(("127.0.0.1", 0))  # OS chooses a free local port
        listener.listen(1)

        with ThreadPoolExecutor(max_workers=1) as pool:
            server_result = pool.submit(serve_one_connection, listener)
            with socket.create_connection(
                listener.getsockname(), timeout=5
            ) as client:
                client.settimeout(5)
                for body in (b"alpha", b"beta", b""):
                    send_frame(client, body)

                client.shutdown(socket.SHUT_WR)
                while True:
                    response = recv_frame(client)
                    if response is None:
                        break
                    responses.append(response)

            server_result.result(timeout=5)

    print([body.decode("ascii") for body in responses])
    assert responses == [b"ALPHA", b"BETA", b""]

if __name__ == "__main__":
    main()

# ['ALPHA', 'BETA', '']
```

这个例子展示：

1. `bind(..., 0)`由OS选择port，避免依赖固定port恰好空闲。
2. `sendall`提交header+body；它不保证peer一次recv就拿到完整frame。
3. `recv_exact`处理partial reads。
4. `None`表示frame边界处EOF；`b""`是合法zero-length body，不能混淆。
5. Client结束写方向后仍接收responses；server见到EOF后结束并close。
6. Future用于传播server thread中的failure，避免worker error静默丢失。

每个socket timeout限制的是相应API等待行为，不等于整个request全流程的统一deadline。示例处理一条connection，不是完整production server。

Stream API的基本注意事项可参考 [Python Socket Programming HOWTO](https://docs.python.org/3/howto/sockets.html)。

### 12.5 Parser Tests Should Not Depend on Packet Boundaries

至少覆盖：

- Header按1 byte多次到达。
- 多条完整frames连续拼在同一byte stream中。
- Body被拆成任意chunks。
- Zero-length frame。
- 正常frame结束后EOF。
- Partial header/body后EOF。
- Length超过允许上限。
- Unicode strings按encoded byte length计算。

可以用一个控制 `recv` 返回chunk sizes的fake socket验证parser。真实TCP测试不能保证每次write恰好怎样被分段，因此不要把“这次localhost刚好一次recv拿全”当成正确性依据。

## 13. Timeouts, Keepalive, and Failure Detection

### 13.1 Different Timers Answer Different Questions

| Timer / mechanism | Question |
| --- | --- |
| **TCP RTO** | 未确认data是否需要重传？ |
| **Connect timeout** | 建立连接等待多久？ |
| **Socket read/write timeout** | 当前I/O按API规则可以等多久？ |
| **Application request deadline** | 整个业务操作最多允许多久？ |
| **Idle timeout** | 没有符合定义的activity多久后回收connection？ |
| **TCP keepalive** | 空闲connection的peer/network状态是否仍可探测？ |
| **Application heartbeat** | 对方application是否仍按约定响应？ |

不能用一个socket选项替代全部failure policy。

### 13.2 A Timeout Does Not Prove Non-execution

```text
Client sends request
Server receives and performs operation
Server response is lost or delayed
Client deadline expires
```

Client不知道是request没到、server没处理、response丢了，还是处理完成但很慢。Timeout表示等待没有在要求时间内获得结果，不一定表示operation没执行。

TCP connection断开同样可能留下这种business uncertainty。Retry design见Section 15。

### 13.3 TCP Keepalive vs Application Heartbeat

TCP keepalive通常是可配置的OS机制，对idle connections进行有限探测。默认是否启用、间隔和失败阈值都依平台/configuration，不能记成通用固定值。

即使peer kernel回应探测，它的application也可能卡住或不能处理正常requests。因此application heartbeat可以检查更高层的路径，但它也要有自己的timeout、load与false-positive policy。

**HTTP persistent connection / keep-alive** 与 **TCP keepalive probes** 不是同一个机制，见下一节。

### 13.4 Deadlines and Slow Drip Reads

假设每次recv都允许等5秒，而peer每4秒只发送1 byte。整个frame可能拖很久，尽管每次recv都没有超时。

需要端到端deadline时，可以：

- 用monotonic clock计算剩余budget。
- 在各阶段限制剩余时间。
- 同时限制frame大小、未完成requests和资源占用。
- Timeout后明确是close、cancel还是保留connection继续处理。

实际TLS、buffered I/O和async libraries可能有额外timeout语义，需要按API设计。

### 13.5 Firewalls, NAT, and Proxies

中间设备可能有自己的idle state和timeouts。双方socket都看似open，不代表路径中间的mapping仍存在。

Proxy还可能把client→proxy与proxy→server分成不同TCP connections。对某一段的ACK，不证明另一段传输或最终backend执行成功。

排查时先确定实际endpoints，不能把整个链路当成一条端到端TCP connection。

## 14. TCP, TLS, HTTP, and Multiplexing

### 14.1 Protocol Layers

常见HTTPS over TCP：

```text
HTTP messages
      ↓
TLS records and authenticated encryption
      ↓
TCP ordered byte stream
      ↓
IP
```

TCP本身不加密，也不验证对方就是某个domain。TLS增加相应security protocol；TCP的three-way handshake不是TLS handshake。

DNS获得address也不等于建立TCP connection，更不等于完成TLS authentication。访问网站涉及多个不同阶段。

### 14.2 Persistent Connections

HTTP persistent connection让多个requests复用transport connection，减少重复建立成本。Application需按HTTP framing判断每个message何时结束，不能把“connection还开着”当成response尚未完整。

Connection pool管理复用与并发上限。Idle connection可能被peer或中间设备关闭；从pool取到connection时仍需要处理I/O失败和安全的retry policy。

### 14.3 HTTP/2 and TCP Head-of-line Blocking

HTTP/2可以在一个connection中multiplex多个application streams，减少一些HTTP/1.x层面的排队限制。

但这些HTTP/2 bytes仍承载在同一个TCP ordered stream上：

```text
TCP bytes contain interleaved HTTP/2 stream data
                 ↓
One earlier TCP byte range is missing
                 ↓
Later bytes wait for TCP recovery before ordered delivery
```

因此不能说“HTTP/2 multiplexing消除了所有head-of-line blocking”。

### 14.4 QUIC and HTTP/3

QUIC为多个streams分别维护reliable ordered delivery。某个stream缺失data通常不会仅因为该stream的offset gap就阻止另一个独立stream交付。

但共享network capacity、congestion control、connection-level resources和application dependencies仍然可能影响多个streams。HTTP/3也不是“所有丢包都不再影响其他工作”。[QUIC streams](https://www.rfc-editor.org/rfc/rfc9000.html)、[HTTP/3](https://www.rfc-editor.org/rfc/rfc9114.html)

本节只定位transport关系，HTTP版本的详细semantics留给 `http.md`。


## 15. Retries, Idempotency, and Application-level Reliability

### 15.1 Three Different Acknowledgments

```text
1. Local send API succeeds
   → local transport path accepted the data under API semantics

2. Remote TCP ACK arrives
   → remote transport has acknowledged relevant sequence space

3. Application success response arrives
   → success according to that application's defined contract
```

第三层也要定义成功具体意味着什么：仅进入queue、完成in-memory update，还是已durably committed。TCP无法替application定义这些语义。

### 15.2 Transport Deduplication Is Not Request Deduplication

TCP retransmits同一connection中的同一byte range，receiver识别重复sequence data。

Application重新发送同一个logical request，可能使用新的sequence numbers，甚至新的connection。TCP会把它当成新的合法bytes，不知道它是不是重复扣款、重复创建order等。

因此“TCP可靠，所以retry不会重复执行”是错误的。

### 15.3 Designing Safe Retries

可以为logical operation分配stable request ID，并设计server-side deduplication：

```text
Client request: operation_id = K

Server:
    if completed result for K exists:
        return that result
    otherwise:
        coordinate ownership of K
        perform operation with the required consistency guarantees
        record result for K
        respond
```

难点不只是“建一个map”：

- 两个同ID请求同时到达怎么办？
- Operation完成、记录结果前crash怎么办？
- Result记录和business effect是否需要同一transaction？
- Deduplication记录保留多久？
- 同一个ID携带不同payload怎么办？

这些属于application/distributed consistency。不能仅在client给request加一个ID，就声称已获得exactly-once guarantee。

### 15.4 Retry Budget and Overload

瞬时故障时，所有clients立刻重试可能放大压力。常见策略包括：

- 有界attempt count或总deadline。
- Backoff与适当jitter。
- 只重试符合业务语义的operations。
- 尊重server的overload signals。
- 避免多个layers各自无限retry，导致attempt multiplication。

Backoff减少同步冲击，不自动保证任务一定成功。失败必须能被清楚地向caller报告。

### 15.5 Delivery Semantics Need an Explicit Scope

“Exactly once”必须说明是在一个connection的stream delivery、一个queue的acknowledgment，还是某个durable business effect层面。

面试回答应区分：

- Bytes传递是否有序且去重。
- Request是否执行过。
- Effect是否持久化。
- Caller是否知道结果。

这些状态在failure中可能不同步。

## 16. Troubleshooting and Measurement

### 16.1 Follow the Lifecycle

```text
Name resolution
      ↓
Routing / reachability
      ↓
TCP handshake
      ↓
Optional TLS handshake
      ↓
Application framing and processing
      ↓
Response transfer
      ↓
Close / reuse
```

先定位失败阶段，再解释原因。不要把所有“请求失败”都称为TCP丢包。

### 16.2 Common Symptoms

| Symptom | Possible causes | Useful next evidence |
| --- | --- | --- |
| **Connection refused** | 常见为目标port无人listen，或明确reject | Listener绑定、目标address/port、RST/firewall行为 |
| **Connect timeout** | Silent drop、routing/firewall、peer不可达、overload等 | SYN是否发出、回应在哪一段消失 |
| **Established but no response** | Application等待framing、server慢、deadlock、路径中断 | 两端application logs、read/write进度 |
| **High retransmissions** | Loss、reordering、capture gaps或其他network问题 | 双端capture、RTT、interface counters |
| **Zero window** | Receiver buffer压力、application不及时read | Receiver消费速度和queue积压 |
| **Many CLOSE_WAIT** | Local close/cleanup路径延迟或泄漏 | Socket ownership、异常路径、持续时间 |
| **Many TIME_WAIT** | Short-lived connection churn | Connection reuse、tuple/FD pressure |
| **Small messages intermittently slow** | RTT、scheduling、Nagle/ACK interaction等 | Timelines，不先假定固定原因 |
| **Large transfer stalls while small traffic works** | PMTU问题、buffer/window、loss或storage瓶颈 | Packet sizes、ICMP/path信息、window走势 |

单一symptom通常不是充分证据。Connection refused也不保证service本体一定坏了，可能只是连错interface或port。

### 16.3 Useful Tools

这些是供手动排查的参考commands；本笔记不要求访问任何外部host。

| Environment | Tool / example | Inspect |
| --- | --- | --- |
| Windows PowerShell | `Get-NetTCPConnection` | Local/remote endpoints、states |
| Windows | `netstat -ano` | Connection/listener与PID |
| Linux | `ss -tinp` | TCP states及可用的connection statistics |
| Linux | `ss -ltn` | Listening TCP sockets |
| Packet analysis | Wireshark、tcpdump等 | Handshake、sequence/ACK、windows、retransmissions |
| Application | Timestamped request logs、metrics | Request ID、queue wait、processing、timeouts |

权限、工具版本和OS会影响可见字段。这里只读观察，不应为试验随意修改production TCP tunables。

### 16.4 Reading a Packet Capture

建议按以下问题看：

1. Capture是在client、server、proxy还是某个中间点？
2. 是完整双向capture吗？有没有丢capture packets？
3. SYN/SYN+ACK/final ACK是否按预期出现？
4. Sequence ranges与cumulative ACK如何变化？
5. 有没有hole、duplicate ACK、SACK blocks或zero window？
6. Retransmission是紧接后续ACK触发，还是接近timer expiry？
7. 哪一方先FIN/RST？
8. Application logs与transport timeline如何对应？

Wireshark的analysis labels是根据capture推断，不是无条件的ground truth。Offloading、reordering和缺失capture都可能影响解读。

### 16.5 Backlogs, FD Limits, and Server Architecture

Connection建立速度、application accept速度和request处理速度是不同bottlenecks：

- Handshake state可能受incomplete-connection资源限制。
- 已完成connections可能等待accept。
- Accepted sockets占用FD/handles与buffers。
- Worker pool或application queue可能继续排队。

只增加listen backlog，不能修复一个永远不消费tasks的worker system。采用bounded queues并分别观察各层latency。

### 16.6 Testing Strategy

- 用localhost验证API usage和framing。
- 用controlled chunk sizes测试parser，不依赖偶然packetization。
- 验证EOF、truncation、oversized input和error propagation。
- 验证slow peer、timeout、half-close与shutdown流程。
- 在明确授权的测试环境中，才进一步使用network emulation观察loss/delay/reordering。
- 区分cold connection、warm reused connection、单connection与多connection吞吐。

Localhost成功不证明真实WAN下的performance或failure behavior；它验证的是其中一部分。

## 17. Common Interview Questions

使用方式：先口头回答，再按reference核对。详细机制只在正文展开，这里不重复完整章节。

### 17.1 Core Questions

| Question | Answer checkpoints | Reference |
| --- | --- | --- |
| **What does TCP provide?** | Connection-oriented、reliable、ordered、full-duplex byte stream | [Section 2](#2-what-does-tcp-guarantee) |
| **TCP vs UDP?** | Stream/datagram、内置机制、workload trade-offs | [2.5](#25-tcp-vs-udp-vs-quic) |
| **How is a TCP connection identified?** | 4-tuple，并说明protocol/network context | [1.3](#13-identifying-a-connection) |
| **How can many clients use one server port?** | Remote endpoints不同，listener与connected sockets不同 | [1.3](#13-identifying-a-connection) |
| **Why three-way handshake?** | 双向ISN synchronization与confirmation，旧请求问题 | [Section 4](#4-connection-establishment-the-three-way-handshake) |
| **What if the final ACK is lost?** | 双方可能暂时状态不同，SYN+ACK重传与后续ACK恢复 | [4.4](#44-what-if-a-handshake-message-is-lost) |
| **Does accept perform another handshake?** | Kernel建立与application取出connection分开 | [4.5](#45-listening-backlogs-and-accept) |
| **What do sequence numbers count?** | Bytes；SYN/FIN各占1，pure ACK不占 | [Section 5](#5-sequence-numbers-acks-and-ordered-delivery) |
| **What does ACK=N mean?** | 连续前缀到N之前，下一个期待位置N | [5.2](#52-cumulative-ack) |
| **Does TCP ACK mean the application processed the data?** | Transport reception与business completion不同 | [15.1](#151-three-different-acknowledgments) |
| **How does TCP recover from loss?** | Timeout、fast retransmit、SACK/modern recovery | [Section 6](#6-loss-detection-and-retransmission) |
| **What if an ACK is lost?** | Cumulative later ACK或retransmit；不是ACK-of-ACK链 | [5.5](#55-ack-loss-and-duplicate-data) |
| **Does duplicate ACK prove loss?** | 也可能reordering；detection有取舍 | [6.4](#64-fast-retransmit-detect-a-hole-before-rto) |
| **What is SACK?** | 报告非连续received ranges，仍ordered delivery | [6.5](#65-sack) |
| **RTT vs RTO?** | Measurement/estimate vs retransmission decision timer | [6.2](#62-retransmission-timeout-rto) |
| **Flow vs congestion control?** | Receiver capacity vs network capacity | [Sections 7–8](#7-flow-control-protect-the-receiver) |
| **rwnd vs cwnd?** | Receiver advertised field vs sender-internal congestion state | [8.2](#82-congestion-window-cwnd) |
| **What happens at zero window?** | 暂停普通新data发送，探测更新；不是自动断开 | [7.3](#73-zero-window-and-persist-probing) |
| **Why is slow start approximately exponential?** | ACK-driven growth，一个RTT内ACK数量随窗口增长 | [8.3](#83-slow-start) |
| **MSS vs MTU?** | TCP payload vs network packet size constraint | [3.3](#33-mtu-vs-mss) |
| **How does RTT limit throughput?** | In-flight window/RTT与BDP，说明简化假设 | [Section 9](#9-throughput-latency-and-practical-performance) |
| **Why does TCP close often use four segments?** | 两方向独立结束，ACK与FIN可以合并 | [10.3](#103-why-four-segments-are-not-mandatory) |
| **Why TIME_WAIT? Which side owns it?** | 最后ACK恢复、旧segments；通常active closer，不固定client | [10.5](#105-why-time_wait-exists) |
| **TIME_WAIT vs CLOSE_WAIT?** | 正常终止保留state vs 等本地application close | [10.6](#106-close_wait-vs-time_wait) |
| **FIN vs RST?** | Ordered directional EOF vs reset/abort | [10.7](#107-rst-graceful-close-and-half-open-connections) |
| **Can send/recv be partial?** | Stream API必须检查byte count并循环 | [Section 11](#11-socket-apis-and-the-data-path) |
| **How do you solve message coalescing/splitting?** | Application framing，不依赖write/recv对应 | [Section 12](#12-message-framing-and-a-complete-python-example) |
| **TCP keepalive vs HTTP keep-alive?** | Transport idle probing vs connection reuse | [13.3](#133-tcp-keepalive-vs-application-heartbeat) |
| **Does HTTP/2 eliminate TCP HOL blocking?** | App streams仍共享一个ordered TCP stream | [14.3](#143-http2-and-tcp-head-of-line-blocking) |
| **Does reliable TCP provide exactly-once execution?** | Stream dedup与request/business effect不同 | [Section 15](#15-retries-idempotency-and-application-level-reliability) |

### 17.2 Calculation and Trace Practice

先写assumptions，再作答：

| Problem | Answer |
| --- | --- |
| SYN seq=700，第一次普通data从哪里开始？ | 701 |
| Data seq=701、length=300，连续收到后ACK是多少？ | 1001 |
| FIN seq=1001，确认它的ACK是多少？ | 1002 |
| 已收到[1,501)和[1001,1501)，中间缺失，cumulative ACK是多少？ | 501 |
| 若[501,1001)补齐，且后段已保留，ACK可推进到哪里？ | 1501 |
| MTU1500，IPv4 header20，TCP header20，无其他overhead，payload上限示例？ | 1460 bytes |
| cwnd32 KiB、rwnd20 KiB、in-flight12 KiB，简化新增credit？ | 8 KiB |
| 1 Gbit/s、RTT50 ms，BDP是多少？ | 6,250,000 bytes，约5.96 MiB |
| Window field16384，scale3，effective advertised window？ | 131072 bytes，即128 KiB |

留意sequence numbers计bytes，不是packet count；BDP中的bandwidth要从bits换算为bytes。

### 17.3 Scenario: The Client Receives Half a Message

**Prompt:** The server sends a response in one call, but the client receives only part of it. Is TCP unreliable?

**Sample answer:**

> TCP provides a byte stream, not message-sized reads. A receive call can return fewer bytes than the application expects even when the connection is healthy. I would define framing, read the header and body incrementally, and distinguish normal end-of-stream from a truncated message. A successful send call also does not imply one matching receive call.

Follow-up：header本身也可能被拆开吗？Length为零与EOF怎么区分？Declared length很大时怎么办？

### 17.4 Scenario: A Request Times Out After a Successful send

**Prompt:** sendall succeeds, but the application receives no response. Can it safely retry?

**Sample answer:**

> Successful sending does not prove that the server completed the operation, and a timeout does not prove that it did not. The server may have executed the request before the response was lost. I would retry only under an appropriate application contract, such as idempotency or durable request deduplication, and use a bounded retry budget.

Follow-up：同一request ID的并发requests怎么办？TCP retransmission与application retry哪里不同？

### 17.5 Scenario: Low Throughput on a High-bandwidth Link

**Prompt:** A connection uses only a small fraction of a high-bandwidth path. What would you inspect?

**Sample answer:**

> I would compare the effective in-flight window with the bandwidth-delay product, then inspect RTT, loss recovery, receive-window limits, and whether the application supplies and consumes data fast enough. I would also check CPU, storage, and competing traffic. Increasing buffers is only useful if the evidence shows a window or buffering limit.

Follow-up：Receiver不read时发生什么？Window scale协商有何作用？Cwnd较小与rwnd较小分别说明什么？

### 17.6 Scenario: The Server Has Many CLOSE_WAIT Connections

**Prompt:** Is this a network problem or an application problem?

**Sample answer:**

> CLOSE_WAIT means the peer has closed its sending direction and the local application has not yet completed its side of the close. I would inspect socket ownership, request completion, and cleanup paths, especially exceptions and cancelled tasks. Some temporary CLOSE_WAIT is normal, but persistent accumulation often points to delayed or missing local cleanup.

Follow-up：为什么不能简单把TIME_WAIT也按同样办法处理？Half-close时server还能发送response吗？

### 17.7 Scenario: Explain a Website Request by Layer

不需要在TCP笔记里完整重复DNS/HTTP，但应能定位各阶段：

```text
Resolve name when necessary
      ↓
Choose destination address and route
      ↓
Establish transport connection
      ↓
Perform TLS when the protocol requires it
      ↓
Send framed application request
      ↓
Parse response and handle business result
      ↓
Reuse or close connection
```

如果使用HTTP/3，transport部分是QUIC，不是TCP。已有connection、DNS cache、session resumption或proxy也会改变具体流程。

面试先说明假设，例如“这里先讨论一个新建的HTTPS-over-TCP connection”，再解释步骤。

### 17.8 Common Statements to Challenge

| Statement | Missing distinction |
| --- | --- |
| “One send equals one recv.” | Byte stream vs application framing |
| “ACK means the request succeeded.” | Transport reception vs business completion |
| “Every lost ACK causes retransmission.” | Later cumulative acknowledgment可能足够 |
| “Three duplicate ACKs are the only loss signal.” | RTO与modern recovery |
| “cwnd is announced by the receiver.” | cwnd与rwnd的owner不同 |
| “Zero window means the peer is dead.” | Flow-control capacity vs failure detection |
| “TCP always closes in exactly four packets.” | Flag合并、simultaneous close、retransmission |
| “The client always enters TIME_WAIT.” | Active-close role不固定 |
| “close returning proves delivery.” | Local reference lifetime vs remote outcome |
| “TCP_NODELAY creates message boundaries.” | Packetization policy不是framing |
| “HTTP/2 removes all head-of-line blocking.” | Application multiplexing vs TCP ordering |
| “UDP cannot support reliable protocols.” | UDP本身与其上层protocol不同 |
| “64K ports means only 64K server connections.” | Full tuple与resource limits |
| “No packet loss means latency is good.” | Queueing、bufferbloat、application stalls |

### 17.9 A Strong Interview Answer Structure

1. **State the layer:** TCP、socket API，还是application protocol？
2. **State the direction:** 哪一方向的seq、ACK、window？
3. **Trace bytes or states:** 用具体range或state transition解释。
4. **Explain failure paths:** Loss、timeout、EOF、reset怎样被观察？
5. **Separate correctness from performance:** Framing正确后，再谈buffer、pacing、pooling。
6. **Qualify implementation details:** OS默认值、congestion algorithm与extensions不统一。
7. **Give evidence:** Logs、socket states、captures和metrics如何验证判断？

优先把“一次access/call到底保证了哪一层的事情”讲清楚，再扩展到复杂protocol和system-design问题。
