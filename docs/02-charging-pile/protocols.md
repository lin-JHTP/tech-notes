# 协议规范

> 充电桩领域涉及的主要通信协议：GB/T 国标、OCPP 云端协议等。

## 📌 协议速览

| 协议 | 全称 | 用途 |
|------|------|------|
| GB/T 18487 | 电动汽车传导充电系统 | 交流充电接口标准（国标） |
| GB/T 27930 | 非车载直流充电机通信协议 | 直流充电 CAN 通信协议 |
| OCPP 1.6 | Open Charge Point Protocol | 充电桩与云平台通信（JSON over WebSocket） |
| OCPP 2.0.1 | — | OCPP 最新版本，支持双向能源管理 |

---

## 🔑 GB/T 18487（交流充电）

交流慢充桩遵循 GB/T 18487 标准，通过 CP（Control Pilot）信号进行充电过程控制。

### CP 信号状态

| CP 电压 | 状态 | 含义 |
|---------|------|------|
| +12V DC | A | 无车连接 |
| +9V PWM | B | 车辆已连接，未就绪 |
| +6V PWM | C | 车辆就绪，请求充电 |
| +3V PWM | D | 需要通风（特殊情况） |
| 0V | E/F | 错误状态 |

CP 信号的 PWM 占空比表示最大可用电流：
- 占空比 10%～85% → 电流 = 占空比 × 0.6A
- 例：占空比 25% → 最大电流 15A

---

## 🔑 OCPP 1.6 简介

OCPP（Open Charge Point Protocol）是充电桩与充电运营平台（CSMS）之间的通信协议。

### 核心消息类型

| 消息 | 方向 | 说明 |
|------|------|------|
| BootNotification | 桩→云 | 上电启动注册 |
| Heartbeat | 桩→云 | 心跳保活（默认每 5 分钟） |
| StatusNotification | 桩→云 | 状态变化上报 |
| StartTransaction | 桩→云 | 充电开始通知 |
| StopTransaction | 桩→云 | 充电结束通知（含电量数据） |
| RemoteStartTransaction | 云→桩 | 远程启动充电 |

---

## 📝 待补充内容

- GB/T 接口机械结构说明
- OCPP 报文格式详解（JSON 示例）
- 鉴权（Authorization）流程
- 计费策略与电量计量精度要求
