# UART + DMA 调试经验

> **文章类型**：经验清单 + 问题与解答  
> **适用读者**：使用 GD32/STM32 串口，需要实现高效不定长接收的开发者  
> **最后更新**：2026-04  
> **标签**：`UART` `DMA` `嵌入式` `调试` `不定长接收` `IDLE中断`

---

## 目录

1. [为什么用 DMA 接收？](#1-为什么用-dma-接收)
2. [不定长接收方案：IDLE 中断 + DMA](#2-不定长接收方案idle-中断--dma)
3. [发送端：DMA 异步发送](#3-发送端dma-异步发送)
4. [FreeRTOS 场景下的集成](#4-freertos-场景下的集成)
5. [常见问题 Q&A](#5-常见问题-qa)
6. [调试清单](#6-调试清单)
7. [TODO](#7-todo)

---

## 1. 为什么用 DMA 接收？

### 中断接收的问题

```c
/* 传统做法：每收到一个字节触发一次中断 */
void USART0_IRQHandler(void)
{
    if (usart_interrupt_flag_get(USART0, USART_INT_FLAG_RBNE) == SET) {
        rx_buf[rx_idx++] = usart_data_receive(USART0);
        // 问题：115200bps 下约每 87us 中断一次
        // 高波特率（1Mbps）下约每 10us 一次，严重占用 CPU
    }
}
```

**DMA 接收的优势：**

| 比较项 | 中断接收 | DMA 接收 |
|--------|----------|----------|
| CPU 占用 | 高（每字节一次中断） | 极低（仅帧尾中断一次） |
| 适合场景 | 低波特率、少量数据 | 高波特率、大数据量、RTOS 场景 |
| 实现复杂度 | 低 | 中等 |
| 不定长处理 | 需要协议帧判断 | 配合 IDLE 中断优雅解决 |

---

## 2. 不定长接收方案：IDLE 中断 + DMA

### 原理

- **DMA**：持续将 UART 收到的数据搬移到缓冲区，不打扰 CPU
- **IDLE 中断**：当总线连续超过一个字节时间无数据时触发，标志一帧结束
- 组合效果：**DMA 收数据，IDLE 中断通知帧接收完成**

### 完整实现（GD32F303）

```c
#include "gd32f30x.h"
#include <string.h>

#define UART_RX_BUF_SIZE  256

static uint8_t uart_rx_dma_buf[UART_RX_BUF_SIZE];
static uint8_t uart_rx_proc_buf[UART_RX_BUF_SIZE];
static uint16_t uart_rx_len = 0;
static volatile uint8_t uart_rx_done_flag = 0;

/*!
 * @brief UART0 DMA 接收初始化
 *        使用 DMA0 通道4（USART0_RX 对应）
 */
void uart0_dma_rx_init(void)
{
    dma_parameter_struct dma_init_struct;

    /* 使能时钟 */
    rcu_periph_clock_enable(RCU_GPIOA);
    rcu_periph_clock_enable(RCU_USART0);
    rcu_periph_clock_enable(RCU_DMA0);

    /* GPIO：PA9(TX) 复用推挽，PA10(RX) 浮空输入 */
    gpio_init(GPIOA, GPIO_MODE_AF_PP, GPIO_OSPEED_50MHZ, GPIO_PIN_9);
    gpio_init(GPIOA, GPIO_MODE_IN_FLOATING, GPIO_OSPEED_50MHZ, GPIO_PIN_10);

    /* USART 配置：115200 8N1 */
    usart_deinit(USART0);
    usart_baudrate_set(USART0, 115200U);
    usart_word_length_set(USART0, USART_WL_8BIT);
    usart_stop_bit_set(USART0, USART_STB_1BIT);
    usart_parity_config(USART0, USART_PM_NONE);
    usart_hardware_flow_rts_config(USART0, USART_RTS_DISABLE);
    usart_hardware_flow_cts_config(USART0, USART_CTS_DISABLE);
    usart_receive_config(USART0, USART_RECEIVE_ENABLE);
    usart_transmit_config(USART0, USART_TRANSMIT_ENABLE);

    /* 使能 USART DMA 接收请求 */
    usart_dma_receive_config(USART0, USART_DENR_ENABLE);

    /* 使能 IDLE 中断 */
    usart_interrupt_enable(USART0, USART_INT_IDLE);
    nvic_irq_enable(USART0_IRQn, 5, 0);

    /* DMA 配置 */
    dma_deinit(DMA0, DMA_CH4);
    dma_init_struct.direction = DMA_PERIPHERAL_TO_MEMORY;
    dma_init_struct.memory_addr = (uint32_t)uart_rx_dma_buf;
    dma_init_struct.memory_inc = DMA_MEMORY_INCREASE_ENABLE;
    dma_init_struct.memory_width = DMA_MEMORY_WIDTH_8BIT;
    dma_init_struct.number = UART_RX_BUF_SIZE;
    dma_init_struct.periph_addr = (uint32_t)&USART_DATA(USART0);
    dma_init_struct.periph_inc = DMA_PERIPH_INCREASE_DISABLE;
    dma_init_struct.periph_width = DMA_PERIPHERAL_WIDTH_8BIT;
    dma_init_struct.priority = DMA_PRIORITY_HIGH;
    dma_init(DMA0, DMA_CH4, &dma_init_struct);

    /* 使能循环模式（可选：如果用循环模式需要额外处理） */
    /* dma_circulation_enable(DMA0, DMA_CH4); */

    /* 启动 DMA */
    dma_channel_enable(DMA0, DMA_CH4);

    usart_enable(USART0);
}

/*!
 * @brief USART0 中断处理：仅处理 IDLE 中断
 */
void USART0_IRQHandler(void)
{
    if (usart_interrupt_flag_get(USART0, USART_INT_FLAG_IDLE) == SET) {
        /* 必须先读 SR，再读 DR 才能清除 IDLE 标志 */
        usart_interrupt_flag_clear(USART0, USART_INT_FLAG_IDLE);
        (void)USART_DATA(USART0);  // 读 DR 清标志

        /* 关闭 DMA，计算接收到的字节数 */
        dma_channel_disable(DMA0, DMA_CH4);
        uart_rx_len = UART_RX_BUF_SIZE - dma_transfer_number_get(DMA0, DMA_CH4);

        /* 将数据复制到处理缓冲区 */
        memcpy(uart_rx_proc_buf, uart_rx_dma_buf, uart_rx_len);
        uart_rx_done_flag = 1;

        /* 重新启动 DMA 接收 */
        dma_transfer_number_config(DMA0, DMA_CH4, UART_RX_BUF_SIZE);
        dma_channel_enable(DMA0, DMA_CH4);
    }
}

/*!
 * @brief 在主循环或任务中处理接收到的数据
 */
void uart_rx_process(void)
{
    if (uart_rx_done_flag) {
        uart_rx_done_flag = 0;
        /* 处理 uart_rx_proc_buf 中 uart_rx_len 字节的数据 */
        handle_received_data(uart_rx_proc_buf, uart_rx_len);
    }
}
```

---

## 3. 发送端：DMA 异步发送

```c
static volatile uint8_t uart_tx_busy = 0;
static uint8_t uart_tx_buf[256];

void uart0_dma_tx_init(void)
{
    dma_parameter_struct dma_init_struct;

    dma_deinit(DMA0, DMA_CH3);  // USART0_TX 对应 DMA0 通道3
    dma_init_struct.direction = DMA_MEMORY_TO_PERIPHERAL;
    dma_init_struct.memory_addr = (uint32_t)uart_tx_buf;
    dma_init_struct.memory_inc = DMA_MEMORY_INCREASE_ENABLE;
    dma_init_struct.memory_width = DMA_MEMORY_WIDTH_8BIT;
    dma_init_struct.number = 0;  // 发送时再配置
    dma_init_struct.periph_addr = (uint32_t)&USART_DATA(USART0);
    dma_init_struct.periph_inc = DMA_PERIPH_INCREASE_DISABLE;
    dma_init_struct.periph_width = DMA_PERIPHERAL_WIDTH_8BIT;
    dma_init_struct.priority = DMA_PRIORITY_MEDIUM;
    dma_init(DMA0, DMA_CH3, &dma_init_struct);

    /* 使能 DMA 传输完成中断 */
    dma_interrupt_enable(DMA0, DMA_CH3, DMA_INT_FTF);
    nvic_irq_enable(DMA0_Channel3_IRQn, 6, 0);

    usart_dma_transmit_config(USART0, USART_DENT_ENABLE);
}

/*!
 * @brief 异步发送数据（非阻塞）
 * @return 0: 成功启动发送, -1: 上次发送未完成
 */
int uart_send_async(const uint8_t *data, uint16_t len)
{
    if (uart_tx_busy || len == 0 || len > sizeof(uart_tx_buf)) {
        return -1;
    }

    uart_tx_busy = 1;
    memcpy(uart_tx_buf, data, len);

    dma_channel_disable(DMA0, DMA_CH3);
    dma_transfer_number_config(DMA0, DMA_CH3, len);
    dma_channel_enable(DMA0, DMA_CH3);
    return 0;
}

/* DMA 发送完成中断：清除 busy 标志 */
void DMA0_Channel3_IRQHandler(void)
{
    if (dma_interrupt_flag_get(DMA0, DMA_CH3, DMA_INT_FLAG_FTF) == SET) {
        dma_interrupt_flag_clear(DMA0, DMA_CH3, DMA_INT_FLAG_FTF);
        uart_tx_busy = 0;
    }
}
```

---

## 4. FreeRTOS 场景下的集成

在 RTOS 场景中，用队列代替全局标志位，实现线程安全的数据传递：

```c
/* ISR 中给出信号量或往队列发数据，唤醒处理任务 */
void USART0_IRQHandler(void)
{
    BaseType_t woken = pdFALSE;
    if (usart_interrupt_flag_get(USART0, USART_INT_FLAG_IDLE) == SET) {
        /* 清标志 */
        usart_interrupt_flag_clear(USART0, USART_INT_FLAG_IDLE);
        (void)USART_DATA(USART0);

        dma_channel_disable(DMA0, DMA_CH4);
        uint16_t len = UART_RX_BUF_SIZE
                       - dma_transfer_number_get(DMA0, DMA_CH4);

        /* 构造消息包并发送到队列 */
        UartMsg_t msg;
        msg.length = len;
        memcpy(msg.data, uart_rx_dma_buf, len);
        xQueueSendFromISR(uart_rx_queue, &msg, &woken);

        /* 重启 DMA */
        dma_transfer_number_config(DMA0, DMA_CH4, UART_RX_BUF_SIZE);
        dma_channel_enable(DMA0, DMA_CH4);
    }
    portYIELD_FROM_ISR(woken);
}
```

---

## 5. 常见问题 Q&A

**Q：IDLE 中断不触发？**

> A：检查是否调用了 `usart_interrupt_enable(USART0, USART_INT_IDLE)`。另外，首次上电后如果没有收到任何数据，IDLE 不会触发，这是正常的。

**Q：数据偶尔乱序或丢失？**

> A：检查以下几点：
> 1. DMA 是否使用了循环模式但未做 head/tail 指针处理
> 2. IDLE 中断优先级是否足够高
> 3. 处理函数是否在 IDLE 中断前完成，若未完成则 DMA 重启后可能覆盖

**Q：DMA 重启后第一个字节丢失？**

> A：`dma_transfer_number_config` 后立即 `dma_channel_enable`，在 USART 忙时不会立即接收，不会丢失。如果仍丢失，检查 USART IDLE 标志清除时序（必须读 DR）。

**Q：高波特率（921600+）下数据错误？**

> A：
> 1. 检查时钟分频，确保 USART 时钟频率够高
> 2. 使用逻辑分析仪抓取波形确认硬件无误
> 3. 适当降低 DMA 其他通道的优先级，保证 UART RX DMA 优先

**Q：使用循环 DMA 如何检测数据？**

> A：循环 DMA + IDLE 中断配合时，需用两个指针（write_ptr、read_ptr）跟踪当前写入位置：
> ```c
> uint16_t old_pos = read_ptr;
> uint16_t new_pos = UART_RX_BUF_SIZE - dma_transfer_number_get(...);
> if (new_pos != old_pos) {
>     /* 处理从 old_pos 到 new_pos 的数据（注意环绕） */
> }
> read_ptr = new_pos;
> ```

---

## 6. 调试清单

在实现 UART DMA 接收时，按此清单逐项验证：

- [ ] 确认使用的是正确的 DMA 通道（查芯片手册 DMA 请求映射表）
- [ ] GPIO 引脚模式配置正确（TX 复用推挽，RX 浮空/上拉输入）
- [ ] USART 波特率与对方设备一致（用逻辑分析仪验证）
- [ ] DMA 外设地址指向 `USART_DATA(USARTx)`，非 `USART_STAT0`
- [ ] IDLE 中断的清除方式正确（读 SR + 读 DR，不同芯片有差异）
- [ ] DMA 完成后正确更新传输数量并重新使能
- [ ] 在 RTOS 中使用 FromISR API
- [ ] 验证最大包长不超过 DMA 缓冲区大小

---

## 7. TODO

- [ ] 添加双缓冲 DMA 接收方案（ping-pong buffer）
- [ ] 环形 DMA + IDLE 的完整实现
- [ ] RS485 方向控制（发送前置位 DE/RE，发完清除）
- [ ] UART 协议帧解析通用框架（帧头+长度+校验）

---

*调试经验：当不确定 IDLE 中断是否触发时，在中断里先加一个 LED 翻转，用肉眼确认中断确实进了再做数据处理逻辑。*
