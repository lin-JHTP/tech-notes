# 调试技巧

> 嵌入式调试是日常工作的重要部分，本文汇总常用调试手段与经验技巧。

## 📌 调试工具速览

| 工具 | 用途 | 特点 |
|------|------|------|
| JTAG / SWD | 在线调试、断点、寄存器查看 | 功能最强，需要调试器硬件 |
| 串口打印（UART） | 输出日志信息 | 最简单，几乎无侵入性 |
| 逻辑分析仪 | 抓取总线时序 | 适合调试 I2C/SPI/UART 时序问题 |
| 示波器 | 测量电压波形 | 适合查看电气信号 |
| GD-Link / J-Link | GD32 专用调试器 | 支持 GD32 全系列 |

---

## 🔑 串口打印调试

最经济实用的调试方式，几乎适用所有场景。

```c
/* 重定向 printf 到 UART（GD32） */
int fputc(int ch, FILE *f)
{
    usart_data_transmit(USART0, (uint8_t)ch);
    while (RESET == usart_flag_get(USART0, USART_FLAG_TBE));
    return ch;
}

/* 使用示例 */
printf("[DEBUG] voltage = %d mV\r\n", adc_voltage);
```

---

## 🔑 JTAG / SWD 在线调试

使用 OpenOCD + GDB 或 Keil / IAR 调试器进行断点调试。

常用操作：
- `b main` — 在 main 函数处设置断点
- `c` — 继续运行
- `p variable` — 打印变量值
- `x/10x 0x20000000` — 查看内存内容

---

## 🔑 逻辑分析仪使用技巧

<!-- TODO: 补充 Saleae Logic 2 使用说明、I2C/SPI 协议解析配置 -->

---

## 📝 待补充内容

- GD-Link 配置与 OpenOCD 使用
- Keil MDK 调试技巧
- 常见 HardFault 定位方法
- 内存泄漏排查
- 看门狗与复位原因分析
