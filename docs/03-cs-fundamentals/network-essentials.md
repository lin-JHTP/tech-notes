# 计算机网络精华笔记

> **文章类型**：知识精华 + 面试速查  
> **适用读者**：想系统复习网络协议、备战面试的开发者  
> **最后更新**：2026-04  
> **标签**：`计算机网络` `TCP/IP` `HTTP` `WebSocket` `DNS` `CS基础`

---

## 目录

1. [TCP/IP 协议栈四层模型](#1-tcpip-协议栈四层模型)
2. [TCP 详解](#2-tcp-详解)
3. [UDP vs TCP](#3-udp-vs-tcp)
4. [HTTP/HTTPS](#4-httphttps)
5. [WebSocket](#5-websocket)
6. [DNS 解析流程](#6-dns-解析流程)
7. [常用网络命令速查](#7-常用网络命令速查)
8. [高频面试题](#8-高频面试题)
9. [TODO](#9-todo)

---

## 1. TCP/IP 协议栈四层模型

```
应用层        HTTP  HTTPS  WebSocket  FTP  DNS  SMTP  MQTT  OCPP
─────────────────────────────────────────────────────────────────
传输层        TCP                         UDP
─────────────────────────────────────────────────────────────────
网络层        IP (IPv4/IPv6)   ICMP   ARP   BGP
─────────────────────────────────────────────────────────────────
链路层        Ethernet   WiFi   4G/5G   蓝牙
```

### OSI 七层 vs TCP/IP 四层对照

| TCP/IP | OSI 对应层 |
|--------|-----------|
| 应用层 | 应用层 + 表示层 + 会话层 |
| 传输层 | 传输层 |
| 网络层 | 网络层 |
| 链路层 | 数据链路层 + 物理层 |

---

## 2. TCP 详解

### 三次握手

```
客户端                           服务端
  │                               │
  │──── SYN (seq=x) ────────────→│  第1次：客户端发起连接
  │                               │
  │←─── SYN+ACK (seq=y,ack=x+1) ─│  第2次：服务端确认并同步
  │                               │
  │──── ACK (ack=y+1) ──────────→│  第3次：客户端确认
  │                               │
  │              [连接建立]        │
```

**为什么要三次握手？**

> 两次握手无法确认客户端的接收能力是否正常（第二次握手后服务端不知道客户端是否收到了 SYN+ACK）。三次握手让双方都能确认对方的收发能力正常。

### 四次挥手

```
主动关闭方                       被动关闭方
  │                               │
  │──── FIN (seq=u) ────────────→│  第1次：我不再发送数据
  │                               │  [服务端收到，进入 CLOSE_WAIT]
  │←─── ACK (ack=u+1) ───────────│  第2次：我知道了
  │                               │  [服务端处理剩余数据...]
  │←─── FIN (seq=v) ─────────────│  第3次：服务端数据也发完了
  │                               │
  │──── ACK (ack=v+1) ──────────→│  第4次：好的，我等 2MSL 后关闭
  │                               │
  │     [等待 2MSL = 2×60s]       │
  │              [连接关闭]        │
```

**TIME_WAIT 为什么等 2MSL？**

> MSL（Maximum Segment Lifetime）= 最大报文存活时间（通常 60s）。
> 等待 2MSL 的原因：① 确保最后一个 ACK 能到达对方（对方没收到会重发 FIN）；② 确保本次连接的所有网络报文都已过期消散，避免影响新连接。

### TCP 可靠性机制

| 机制 | 描述 |
|------|------|
| 序号与确认 | 每个字节有序号，接收方确认已收到 |
| 超时重传 | 超过 RTO 未收到 ACK 则重传 |
| 流量控制 | 接收方通告窗口大小（rwnd），控制发送速率 |
| 拥塞控制 | 慢启动、拥塞避免、快重传、快恢复 |
| 校验和 | 检测数据传输中的比特错误 |

### TCP 拥塞控制

```
拥塞窗口（cwnd）变化：

  cwnd
   ^
   │         /\  拥塞避免（+1/RTT）
   │        /  \
   │       / 慢启│  快重传+快恢复
   │      /  启 │→ cwnd = ssthresh/2，重新慢启动
   │     / (*2) │
   │────┼────────┼──────────────→ 时间
   0  ssthresh（拥塞发生时 = cwnd/2）
   
慢启动：cwnd 每个 RTT 翻倍（指数增长），直到 ssthresh
拥塞避免：cwnd 每个 RTT 增加 1（线性增长）
检测到丢包：ssthresh = cwnd/2，根据丢包类型决定 cwnd
```

---

## 3. UDP vs TCP

| 对比项 | TCP | UDP |
|--------|-----|-----|
| 连接 | 面向连接（三次握手） | 无连接 |
| 可靠性 | 可靠（重传、排序） | 不可靠（可能丢失、重复） |
| 顺序 | 保证有序 | 不保证顺序 |
| 速度 | 较慢（建连+拥塞控制） | 快 |
| 开销 | 头部 20 字节 | 头部 8 字节 |
| 适用场景 | HTTP/文件传输/OCPP | DNS/视频流/游戏/实时音视频 |

---

## 4. HTTP/HTTPS

### HTTP/1.1 vs HTTP/2 vs HTTP/3

| 特性 | HTTP/1.1 | HTTP/2 | HTTP/3 |
|------|----------|--------|--------|
| 传输 | TCP，文本 | TCP，二进制帧 | QUIC（UDP上层） |
| 多路复用 | 无（队头阻塞） | 有（同一连接多流） | 有（无队头阻塞） |
| 头部压缩 | 无 | HPACK | QPACK |
| 服务器推送 | 无 | 有 | 有 |
| TLS | 可选 | 事实必须 | 内置 |

### HTTPS 握手过程（TLS 1.3）

```
客户端                              服务端
  │──── ClientHello ──────────────→│  支持的算法、随机数
  │←─── ServerHello ───────────────│  选定算法、随机数、证书
  │←─── Certificate ───────────────│  服务端证书（含公钥）
  │←─── CertificateVerify ─────────│  用私钥签名证明身份
  │←─── Finished ──────────────────│
  │──── Finished ─────────────────→│  握手完成
  │                                 │
  │          [应用数据加密传输]       │
```

### 常见 HTTP 状态码

| 状态码 | 含义 |
|--------|------|
| 200 OK | 请求成功 |
| 201 Created | 资源创建成功（POST） |
| 204 No Content | 成功但无返回内容 |
| 301 Moved Permanently | 永久重定向 |
| 302 Found | 临时重定向 |
| 304 Not Modified | 缓存有效，使用缓存 |
| 400 Bad Request | 请求格式错误 |
| 401 Unauthorized | 未认证 |
| 403 Forbidden | 已认证但无权限 |
| 404 Not Found | 资源不存在 |
| 429 Too Many Requests | 限流 |
| 500 Internal Server Error | 服务端内部错误 |
| 502 Bad Gateway | 网关错误（上游服务器错误） |
| 503 Service Unavailable | 服务暂时不可用 |

---

## 5. WebSocket

WebSocket 是 HTTP 升级的全双工通信协议，是 OCPP 的传输层。

### 握手过程

```
客户端发送 HTTP 升级请求：
GET /ocpp/CP001 HTTP/1.1
Host: csms.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
Sec-WebSocket-Protocol: ocpp1.6

服务端响应：
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
Sec-WebSocket-Protocol: ocpp1.6
```

**握手后**：双方可以随时互发 WebSocket 帧，不需要请求-响应模型。

### WebSocket vs HTTP 长轮询 vs SSE

| 特性 | WebSocket | HTTP 长轮询 | Server-Sent Events |
|------|-----------|------------|-------------------|
| 方向 | 全双工 | 模拟双向（开销大） | 服务器→客户端单向 |
| 实时性 | 毫秒级 | 秒级 | 毫秒级 |
| 协议开销 | 低（帧头 2-10 字节） | 高（HTTP 头每次） | 中 |
| 适用场景 | OCPP/游戏/协作 | 兼容旧系统 | 股票行情/通知 |

---

## 6. DNS 解析流程

```
浏览器输入 www.example.com

1. 检查浏览器缓存 → 命中则返回
2. 检查操作系统缓存（/etc/hosts）→ 命中则返回
3. 查询本地 DNS 解析器（递归解析器，通常是 ISP 提供）
4. 本地 DNS → 根域名服务器（返回 .com 的 TLD 服务器地址）
5. 本地 DNS → .com TLD 服务器（返回 example.com 的权威 DNS 地址）
6. 本地 DNS → example.com 权威 DNS（返回 www.example.com 的 IP）
7. 本地 DNS 缓存结果，返回给客户端
8. 客户端建立 TCP 连接到该 IP
```

### DNS 记录类型

| 类型 | 用途 | 示例 |
|------|------|------|
| A | 域名→IPv4 | `example.com → 93.184.216.34` |
| AAAA | 域名→IPv6 | `example.com → 2606:2800:...` |
| CNAME | 别名 | `www → example.com` |
| MX | 邮件服务器 | `mail.example.com` |
| TXT | 文本信息 | SPF/DKIM 记录 |
| NS | 权威 DNS 服务器 | `ns1.example.com` |

---

## 7. 常用网络命令速查

```bash
# 查看网络接口信息
ip addr show
ifconfig  # 旧版

# 测试连通性
ping 8.8.8.8 -c 4
ping6 2001:4860:4860::8888

# 路由追踪
traceroute google.com
tracert google.com  # Windows

# DNS 查询
nslookup www.example.com
dig www.example.com
dig @8.8.8.8 www.example.com A  # 指定 DNS 服务器

# 端口扫描/监听查看
netstat -tlnp        # 查看所有监听端口
ss -tlnp             # 更现代的替代工具
nmap -p 80,443 host  # 扫描指定端口

# 抓包
tcpdump -i eth0 port 9000 -w capture.pcap
wireshark  # GUI 工具

# WebSocket 测试
wscat -c ws://server:9000/ocpp/CP001
```

---

## 8. 高频面试题

**Q：TCP 三次握手为什么不是两次？**
> A：两次握手不能确认客户端的接收能力。三次握手让双方都能验证自己的发送和接收能力都正常。另外，防止历史连接请求（网络延迟的旧 SYN）错误地建立连接。

**Q：TIME_WAIT 状态为什么存在？**
> A：① 确保最后一个 ACK 能可靠到达，对方没收到会重发 FIN，等待 2MSL 可以处理重发的 FIN；② 让本次连接的所有报文在网络上彻底消失，避免影响相同五元组的新连接。

**Q：HTTPS 的工作原理？**
> A：TLS 握手阶段通过非对称加密（证书）验证服务端身份并安全交换密钥，之后使用对称加密（AES 等）加密传输内容。非对称慢但安全，对称快。

**Q：HTTP 和 WebSocket 的区别？**
> A：HTTP 是请求-响应模型，半双工。WebSocket 是 HTTP 升级的全双工协议，建立后双方可以随时发数据，适合实时通信场景（如 OCPP 充电协议）。

**Q：什么是 TCP 粘包？如何解决？**
> A：TCP 是流协议，不保留消息边界，多个小消息可能在同一个 read 中返回。解决：① 固定长度消息；② 消息头含长度字段；③ 特殊分隔符（如 HTTP 的 `\r\n\r\n`）；④ 序列化框架（如 Protobuf）。

---

## 9. TODO

- [ ] QUIC 协议详解（HTTP/3 的传输层）
- [ ] TCP keepalive vs 应用层心跳对比
- [ ] NAT 穿透原理（P2P 通信基础）
- [ ] 网络安全基础（XSS/CSRF/SQL注入防护）
- [ ] IPv6 基础配置与过渡技术
