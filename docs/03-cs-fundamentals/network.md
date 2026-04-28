# 计算机网络

> 网络基础知识，涵盖 TCP/IP 协议栈与 HTTP，为充电桩云端通信和 AI 应用开发打基础。

## 📌 知识点清单

- [ ] OSI 七层模型 vs TCP/IP 四层模型
- [ ] IP 地址与子网划分
- [ ] TCP 三次握手 / 四次挥手
- [ ] UDP 与 TCP 对比
- [ ] HTTP / HTTPS 基础
- [ ] WebSocket（OCPP 使用的传输层）
- [ ] DNS 解析过程
- [ ] Socket 编程基础

---

## 🔑 TCP/IP 四层模型

| 层 | 代表协议 | 说明 |
|----|---------|------|
| 应用层 | HTTP, MQTT, OCPP, DNS | 面向应用的协议 |
| 传输层 | TCP, UDP | 端到端可靠/不可靠传输 |
| 网络层 | IP, ICMP | 路由与寻址 |
| 链路层 | Ethernet, WiFi | 物理链路数据传输 |

---

## 🔑 TCP 三次握手

```
客户端                      服务器
  |  ── SYN ──────────────► |   第一次：客户端发起连接
  |  ◄── SYN+ACK ────────── |   第二次：服务器确认
  |  ── ACK ──────────────► |   第三次：客户端确认
  |         连接建立         |
```

---

## 🔑 WebSocket

充电桩 OCPP 协议运行在 WebSocket 之上，WebSocket 特点：
- 基于 HTTP 握手升级（Upgrade）
- 建立后实现全双工通信
- 低延迟，适合云-桩实时交互

---

## 📝 待补充内容

- Socket 编程实例（C 语言）
- MQTT 协议（IoT 常用）
- HTTPS TLS 握手过程
- 常见网络问题排查命令（ping、traceroute、netstat）
