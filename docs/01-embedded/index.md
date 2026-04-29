# 嵌入式开发板块

> 本板块记录嵌入式系统开发中的核心知识与实践经验，覆盖 GD32/STM32 微控制器、C 语言嵌入式编程规范、FreeRTOS 实时操作系统以及常用外设驱动。

---

## 📚 文章列表

| 文章 | 描述 | 难度 |
|------|------|------|
| [GD32入门与工程实践](gd32-getting-started.md) | 从零搭建 GD32 开发环境，含时钟/GPIO/中断配置详解 | ⭐⭐ |
| [FreeRTOS实战笔记](freertos-practical-notes.md) | 任务管理、队列、信号量与常见陷阱总结 | ⭐⭐⭐ |
| [串口与DMA调试经验](uart-dma-debug.md) | UART+DMA不定长接收方案与实际调试记录 | ⭐⭐⭐ |
| [嵌入式C编程规范](embedded-c-coding-style.md) | 适用于裸机和RTOS项目的C编码规范清单 | ⭐⭐ |

---

## 🗺️ 知识脉络

```
嵌入式开发
├── 硬件基础
│   ├── 时钟体系 (RCC)
│   ├── GPIO 配置与复用
│   ├── 中断系统 (NVIC/EXTI)
│   └── 电源管理
├── 通信外设
│   ├── UART / USART
│   ├── SPI / I2C
│   ├── CAN
│   └── USB
├── 高级特性
│   ├── DMA
│   ├── Timer / PWM
│   └── ADC / DAC
└── RTOS
    ├── 任务管理
    ├── 同步与通信
    └── 内存管理
```

---

## 🎯 学习路径建议

1. **裸机基础**：先掌握时钟、GPIO、中断，能跑通 Blinky
2. **外设驱动**：逐一攻克 UART、SPI、I2C、CAN
3. **DMA加速**：学会用 DMA 解放 CPU，处理高速数据流
4. **引入RTOS**：在熟悉裸机后移植 FreeRTOS，实现多任务
5. **工程实践**：建立标准的工程模板、调试方法和版本管理习惯

---

## 📌 TODO

- [ ] 补充 GD32 CAN 总线实战笔记
- [ ] 添加低功耗模式（Stop/Standby）详解
- [ ] 编写 Bootloader 设计与实现笔记
- [ ] I2C 时序与常见器件（OLED/EEPROM）驱动
- [ ] 嵌入式单元测试（Unity 框架）入门
