# 外设驱动汇总

> GD32 平台常用外设驱动开发笔记，包括 UART、I2C、SPI、CAN 等。

## 📌 外设速查表

| 外设 | 用途 | 协议特征 |
|------|------|---------|
| UART | 串口通信、调试打印 | 异步、全双工、常见波特率 9600/115200 |
| I2C | 传感器、EEPROM 通信 | 同步、半双工、多设备共享总线 |
| SPI | Flash、显示屏、快速传感器 | 同步、全双工、速度快 |
| CAN | 充电桩内部通信、车载网络 | 差分信号、多主总线、可靠性高 |

---

## 🔑 UART 驱动要点

```c
/* GD32 UART 初始化要点 */
/* 1. 使能时钟 */
rcu_periph_clock_enable(RCU_USART0);
rcu_periph_clock_enable(RCU_GPIOA);

/* 2. 配置 GPIO 为复用功能 */
gpio_af_set(GPIOA, GPIO_AF_7, GPIO_PIN_9 | GPIO_PIN_10);

/* 3. 配置串口参数 */
usart_baudrate_set(USART0, 115200);
usart_word_length_set(USART0, USART_WL_8BIT);
usart_stop_bit_set(USART0, USART_STB_1BIT);
usart_parity_config(USART0, USART_PM_NONE);
usart_enable(USART0);
```

---

## 🔑 I2C 驱动要点

<!-- TODO: 补充 I2C 初始化与读写时序 -->

---

## 🔑 SPI 驱动要点

<!-- TODO: 补充 SPI 模式配置与 DMA 传输 -->

---

## 🔑 CAN 驱动要点

充电桩中 CAN 总线常用于控制板与功率模块之间的通信。

```c
/* CAN 滤波器配置（接收所有帧） */
can_filter_parameter_struct filter;
can_filter_struct_para_init(&filter);
filter.filter_mode = CAN_FILTERMODE_MASK;
filter.filter_bits = CAN_FILTERBITS_32BIT;
filter.filter_number = 0;
filter.filter_enable = ENABLE;
can_filter_init(&filter);
```

---

## 📝 待补充内容

- I2C 时序图与软件模拟 I2C
- SPI DMA 传输
- CAN 报文帧格式与过滤器详解
- RS485 半双工收发切换
