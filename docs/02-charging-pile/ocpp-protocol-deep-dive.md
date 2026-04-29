# OCPP 协议深度解析

> **文章类型**：协议笔记 + 知识文档  
> **适用读者**：充电桩固件/后台开发者，希望深入理解 OCPP 协议的工程师  
> **最后更新**：2026-04  
> **标签**：`OCPP` `充电桩` `WebSocket` `JSON` `协议栈`

---

## 目录

1. [OCPP 是什么？](#1-ocpp-是什么)
2. [通信架构](#2-通信架构)
3. [消息格式详解](#3-消息格式详解)
4. [Core Profile 核心报文](#4-core-profile-核心报文)
5. [充电会话状态机](#5-充电会话状态机)
6. [错误处理与异常场景](#6-错误处理与异常场景)
7. [OCPP 1.6 vs 2.0.1 主要差异](#7-ocpp-16-vs-201-主要差异)
8. [实战调试技巧](#8-实战调试技巧)
9. [TODO](#9-todo)

---

## 1. OCPP 是什么？

**OCPP**（Open Charge Point Protocol，开放充电点协议）是充电桩（CP，Charge Point）与充电管理系统（CSMS，Charging Station Management System，原称 Central System）之间的通信协议。

- **设计目标**：实现不同厂商的充电桩与后台系统的互操作
- **协议形式**：应用层协议，传输层为 WebSocket（OCPP 1.6）或 HTTP/WebSocket（OCPP 2.0.1）
- **消息编码**：JSON（OCPP 1.6 JSON，主流）或 SOAP（OCPP 1.6 SOAP，逐渐淘汰）
- **标准组织**：OCA（Open Charge Alliance）

### 版本演进

| 版本 | 发布年份 | 主要特点 |
|------|----------|----------|
| OCPP 1.2 | 2010 | 首个公开版本，SOAP |
| OCPP 1.5 | 2012 | 局部优化 |
| OCPP 1.6 | 2015 | 增加 JSON，Smart Charging，Remote Trigger |
| OCPP 2.0 | 2018 | 设备模型重构，安全增强 |
| OCPP 2.0.1 | 2020 | 2.0 勘误版，当前最新主流版本 |

---

## 2. 通信架构

```
充电桩 (Charge Point / Charging Station)
        │
        │  WebSocket (ws:// 或 wss://)
        │  URL 格式：ws://server:port/ocpp/{charger_id}
        │
        ▼
充电管理系统 (CSMS / Central System)
```

### 连接特点

- **长连接**：WebSocket 建立后保持，双向通信
- **充电桩发起**：连接由充电桩主动建立并维持（不是服务器推送）
- **心跳保活**：充电桩定期发送 `Heartbeat`，CSMS 响应时间戳
- **断线重连**：充电桩需实现自动重连（建议指数退避）

---

## 3. 消息格式详解

OCPP 1.6 JSON 使用三种消息类型：

### CALL（请求，MessageTypeId = 2）

```json
[2, "唯一消息ID", "动作名称", {请求载荷}]

示例：充电桩发送心跳请求
[2, "19223201", "Heartbeat", {}]
```

### CALLRESULT（成功响应，MessageTypeId = 3）

```json
[3, "唯一消息ID", {响应载荷}]

示例：CSMS 回应心跳
[3, "19223201", {"currentTime": "2026-04-01T08:00:00.000Z"}]
```

### CALLERROR（错误响应，MessageTypeId = 4）

```json
[4, "唯一消息ID", "错误码", "错误描述", {错误详情}]

示例：
[4, "19223201", "NotImplemented", "Action is not implemented", {}]
```

### 消息 ID 规范

```
- 每条 CALL 消息必须有唯一的 messageId
- messageId 由发起方生成（充电桩或 CSMS）
- CALLRESULT 和 CALLERROR 使用对应 CALL 的 messageId
- 推荐格式：UUID v4 或递增整数转字符串
- 同一时间最多只允许一条未回复的 CALL（等待 CALLRESULT）
```

### 标准错误码

| 错误码 | 含义 |
|--------|------|
| `NotImplemented` | 动作未实现 |
| `NotSupported` | 动作不支持 |
| `InternalError` | 内部错误 |
| `ProtocolError` | 协议违规 |
| `SecurityError` | 安全违规 |
| `FormationViolation` | 消息格式错误 |
| `PropertyConstraintViolation` | 属性约束违规 |
| `OccurrenceConstraintViolation` | 出现次数约束违规 |
| `TypeConstraintViolation` | 类型约束违规 |
| `GenericError` | 其他错误 |

---

## 4. Core Profile 核心报文

### BootNotification（充电桩上线通知）

**场景**：充电桩上电或重启后，向 CSMS 发送设备信息并请求注册。

```json
// 充电桩 → CSMS（CALL）
[2, "001", "BootNotification", {
  "chargePointVendor": "MyVendor",
  "chargePointModel":  "AC-7kW-V1",
  "chargePointSerialNumber": "SN20240001",
  "firmwareVersion": "1.2.3"
}]

// CSMS → 充电桩（CALLRESULT）
[3, "001", {
  "currentTime": "2026-04-01T08:00:00.000Z",
  "interval": 300,                // 心跳间隔（秒）
  "status": "Accepted"            // Accepted | Pending | Rejected
}]
```

**状态流程**：

- `Accepted`：充电桩可以正常工作
- `Pending`：需要等待进一步配置，持续重试
- `Rejected`：拒绝接入，充电桩应停止工作

### Heartbeat（心跳）

```json
// 充电桩 → CSMS
[2, "002", "Heartbeat", {}]

// CSMS → 充电桩（时间同步）
[3, "002", {"currentTime": "2026-04-01T08:05:00.000Z"}]
```

### StatusNotification（状态上报）

充电枪/充电口状态变化时上报：

```json
[2, "003", "StatusNotification", {
  "connectorId": 1,
  "errorCode": "NoError",
  "status": "Available"
}]
```

**Connector 状态枚举**：

| 状态 | 含义 |
|------|------|
| `Available` | 空闲，可以充电 |
| `Preparing` | 已插枪，等待授权 |
| `Charging` | 正在充电 |
| `SuspendedEVSE` | 充电桩侧暂停 |
| `SuspendedEV` | 车辆侧暂停（BMS限流等） |
| `Finishing` | 充电结束，拔枪前 |
| `Reserved` | 已被预约 |
| `Unavailable` | 不可用（维护/故障） |
| `Faulted` | 故障 |

### Authorize（授权）

```json
// 充电桩 → CSMS（刷卡后请求授权）
[2, "004", "Authorize", {
  "idTag": "RFID_1234567890AB"
}]

// CSMS → 充电桩
[3, "004", {
  "idTagInfo": {
    "status": "Accepted",         // Accepted | Blocked | Expired | Invalid
    "expiryDate": "2027-01-01T00:00:00.000Z",
    "parentIdTag": "PARENT_001"   // 可选：父卡
  }
}]
```

### StartTransaction / StopTransaction

```json
// 启动充电事务
[2, "005", "StartTransaction", {
  "connectorId": 1,
  "idTag": "RFID_1234567890AB",
  "meterStart": 1000,             // 开始时电表读数（Wh）
  "timestamp": "2026-04-01T09:00:00.000Z"
}]

[3, "005", {
  "idTagInfo": {"status": "Accepted"},
  "transactionId": 56789          // CSMS 分配的事务 ID
}]

// 停止充电事务
[2, "006", "StopTransaction", {
  "transactionId": 56789,
  "meterStop": 8500,              // 结束时电表读数（Wh）
  "timestamp": "2026-04-01T10:30:00.000Z",
  "reason": "Local",              // Local | Remote | EVDisconnected | HardReset 等
  "idTag": "RFID_1234567890AB"    // 可选
}]
```

### MeterValues（计量数据）

```json
[2, "007", "MeterValues", {
  "connectorId": 1,
  "transactionId": 56789,
  "meterValue": [{
    "timestamp": "2026-04-01T09:15:00.000Z",
    "sampledValue": [
      {"value": "3500", "measurand": "Power.Active.Import", "unit": "W"},
      {"value": "220.3", "measurand": "Voltage", "unit": "V"},
      {"value": "15.9", "measurand": "Current.Import", "unit": "A"},
      {"value": "2400", "measurand": "Energy.Active.Import.Register", "unit": "Wh"}
    ]
  }]
}]
```

---

## 5. 充电会话状态机

```
上电/重启
    │
    ▼
[BootNotification] ──Rejected──→ 停止工作
    │Accepted
    ▼
[Available] ◄──────────────────────────────┐
    │                                       │
    │ 插枪 (EVConnected)                    │ 拔枪/完成
    ▼                                       │
[Preparing]                                 │
    │                                       │
    │ 授权成功 (Authorize/RemoteStart)      │
    ▼                                       │
[Charging] ──── 用户/远程停止 ──→ [Finishing]
    │                                       │
    │ 车辆暂停 (BMS)                        │
    ▼                                       │
[SuspendedEV] ── 恢复 ──→ [Charging]       │
    │                                       │
    │ 充电桩暂停                            │
    ▼                                       │
[SuspendedEVSE] ── 恢复 ──→ [Charging]     │
                                            │
[Faulted] ◄── 故障 ── （任意状态）         │
    │ 故障恢复                              │
    └──────────────────────────────────────┘
```

---

## 6. 错误处理与异常场景

### 常见异常处理策略

| 场景 | 处理方式 |
|------|----------|
| WebSocket 断线 | 指数退避重连（1s→2s→4s→...→最大60s） |
| BootNotification Pending | 按 `interval` 字段重发，不执行充电 |
| 授权超时（无网络） | 本地白名单 / 离线授权策略（OfflineThreshold） |
| StopTransaction 发送失败 | 本地存储，恢复网络后上传（TransactionMessageRetryInterval） |
| 收到未知 Action | 回复 `[4, id, "NotImplemented", "", {}]` |
| 消息格式错误 | 回复 `[4, id, "FormationViolation", "", {}]` |

### 离线充电策略

```
网络断开时：
1. 查找本地白名单（LocalAuthList）
2. 本地白名单命中 → 允许充电
3. 未命中 → 根据 AllowOfflineTxForUnknownId 配置决定
4. 充电记录本地缓冲（MeterValues、TransactionData）
5. 网络恢复后批量上传
```

---

## 7. OCPP 1.6 vs 2.0.1 主要差异

| 特性 | OCPP 1.6 | OCPP 2.0.1 |
|------|----------|------------|
| 设备模型 | 简单（ConnectorId） | 完整设备模型（EVSE/Connector 分层） |
| 安全 | 基础（TLS可选） | 内建安全（证书管理、签名） |
| 智能充电 | Profiles 方式 | 更灵活的 ChargingSchedule |
| 事务模型 | TransactionId | 更完整的 Transaction 生命周期 |
| 设备管理 | ChangeConfiguration | 完整的 Variables/Components 体系 |
| 显示消息 | 无 | SetDisplayMessage 支持 |
| 预约 | Reserve Profile | 内建预约功能 |

---

## 8. 实战调试技巧

### 抓包分析

```bash
# 使用 Wireshark 抓取 WebSocket 报文
# 过滤条件：tcp.port == 9000 （或充电桩使用的端口）
# 右键 WebSocket 帧 → Follow WebSocket Stream 查看完整会话

# 命令行工具 wscat 模拟 CSMS 进行测试
npm install -g wscat
wscat -l 9000  # 监听 9000 端口，模拟 CSMS
```

### 日志格式建议

```c
/* 建议在充电桩端记录完整的 OCPP 报文日志 */
void ocpp_log_message(const char *direction, const char *msg)
{
    /* direction: "TX" 或 "RX" */
    printf("[OCPP][%s][%s] %s\r\n",
           get_timestamp_str(),
           direction,
           msg);
}
```

### 常见实现错误

1. **messageId 不唯一**：导致响应匹配混乱
2. **并发 CALL**：在等待响应时又发出第二条 CALL，违反协议
3. **未处理 RemoteStart/RemoteStop**：导致无法远程控制
4. **时间戳格式错误**：OCPP 要求 ISO 8601 格式，含毫秒和 Z 后缀
5. **ConnectorId=0**：表示整个充电桩，不代表具体某个口

---

## 9. TODO

- [ ] OCPP 2.0.1 Device Model 专题（Component/Variable 体系）
- [ ] Smart Charging Profile 实战配置
- [ ] OCPP 安全白皮书（SecurityProfile 1/2/3）实现
- [ ] 本地白名单（LocalAuthList）管理实现
- [ ] WebSocket 断线重连的指数退避代码示例

---

*参考资料：[OCA 官方规范](https://www.openchargealliance.org/) | OCPP 1.6 JSON 规范（免费下载）*
