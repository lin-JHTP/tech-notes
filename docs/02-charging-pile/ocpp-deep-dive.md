# OCPP 深入解析

> ⚠️ **未来内容待完善** — 本文档计划深入解析 OCPP 1.6 和 OCPP 2.0.1 协议的报文格式、状态机、安全机制等内容。

## 📌 规划内容

- [ ] OCPP 1.6 完整报文格式（CALL / CALLRESULT / CALLERROR）
- [ ] WebSocket 连接管理与重连策略
- [ ] 鉴权流程（Local Authorization / Online Authorization）
- [ ] 事务（Transaction）完整生命周期
- [ ] 固件升级（FirmwareStatusNotification）
- [ ] 远程配置（ChangeConfiguration）
- [ ] OCPP 2.0.1 与 1.6 的主要差异
- [ ] OCPP 安全扩展（TLS / 双向认证）
- [ ] 常见 OCPP 集成问题与解决方案

---

## 💡 为什么要学 OCPP？

1. 充电桩接入运营平台的标准协议，几乎所有大型运营商都要求支持
2. 涉及网络编程、JSON 解析、状态机设计等多个技术点
3. OCPP 2.0.1 引入了更多安全机制，未来方向

---

> 📝 **TODO**：结合实际接入经验，逐步完善本文档。
