# 📌 嵌入式开发板块 · TODO 清单

> 本文件记录嵌入式开发板块待补充的文章和改进项，欢迎认领或提交 PR。

---

## 🔥 高优先级（近期计划）

- [ ] **GD32 CAN 总线实战笔记**
  - 标准帧/扩展帧配置
  - 报文发送与接收过滤器设置
  - 实际调试中的波特率匹配问题
- [ ] **串口 DMA 不定长接收方案（完善版）**
  - 空闲中断 + DMA 接收方案
  - 双缓冲乒乓机制
  - 实际项目中的稳定性优化
- [ ] **I2C 时序详解与常见器件驱动**
  - I2C 时序标准与软件模拟
  - OLED (SSD1306) 驱动实现
  - EEPROM (AT24Cxx) 读写操作

---

## 📝 中期计划

- [ ] **低功耗模式详解（Stop/Standby/Shutdown）**
  - 各模式唤醒源对比
  - RTC 唤醒定时器配置
  - 低功耗下外设的保存与恢复
- [ ] **Bootloader 设计与实现**
  - 内存分区规划（App + Bootloader）
  - 固件校验（CRC/SHA）
  - IAP 升级流程实现
- [ ] **嵌入式单元测试入门（Unity 框架）**
  - Unity 框架集成方法
  - 模拟硬件依赖（Mock/Stub）
  - 在 CI 中自动运行测试

---

## 💡 长期方向

- [ ] 嵌入式 Linux 移植笔记（Buildroot / Yocto 入门）
- [ ] LVGL 图形库在低端 MCU 上的实践
- [ ] 嵌入式 Rust 尝鲜记录
- [ ] 多核 MCU（如 GD32H7xx）开发经验
- [ ] 嵌入式安全（Secure Boot / TrustZone 概念）

---

## ✅ 已完成

- [x] GD32入门与工程实践
- [x] FreeRTOS实战笔记
- [x] 串口与DMA调试经验（基础版）
- [x] 嵌入式C编程规范

---

> 📬 如你有好的主题建议，请在 [GitHub Issues](https://github.com/lin-JHTP/tech-notes/issues) 提出！
