# FreeRTOS 实战笔记

> **文章类型**：知识精华 + 经验清单  
> **适用读者**：熟悉裸机嵌入式开发，想引入 RTOS 的开发者  
> **最后更新**：2026-04  
> **标签**：`FreeRTOS` `RTOS` `任务管理` `队列` `信号量` `嵌入式`

---

## 目录

1. [为什么需要 RTOS？](#1-为什么需要-rtos)
2. [FreeRTOS 核心概念速览](#2-freertos-核心概念速览)
3. [任务管理实战](#3-任务管理实战)
4. [队列：任务间通信利器](#4-队列任务间通信利器)
5. [信号量与互斥锁](#5-信号量与互斥锁)
6. [软件定时器](#6-软件定时器)
7. [内存管理策略选择](#7-内存管理策略选择)
8. [常见陷阱与最佳实践](#8-常见陷阱与最佳实践)
9. [移植到 GD32 的注意点](#9-移植到-gd32-的注意点)
10. [TODO 与延伸阅读](#10-todo-与延伸阅读)

---

## 1. 为什么需要 RTOS？

### 裸机的局限性

在裸机（超级循环）模式下：

```c
/* 典型裸机主循环 */
while (1) {
    task_a();   // 假设需要 50ms
    task_b();   // 假设需要 10ms
    task_c();   // 假设需要 5ms
    // task_c 的最坏响应延迟 = 50+10+5 = 65ms，难以保证实时性
}
```

**痛点：**
- 响应延迟不可控，无法保证实时性
- 任务间优先级无法动态调整
- 阻塞等待（如等待 UART 数据）会卡死整个程序
- 代码随功能增加而变得难以维护

### RTOS 带来了什么

- **多任务并发**：每个功能模块独立运行，互不干扰
- **优先级抢占**：高优先级任务可立即抢占低优先级任务
- **标准同步原语**：队列、信号量、互斥锁规范任务间通信
- **延时精确**：`vTaskDelay()` 让出 CPU，不浪费资源

---

## 2. FreeRTOS 核心概念速览

```
FreeRTOS
├── 调度器（Scheduler）
│   ├── 抢占式（preemptive，默认）
│   └── 协作式（cooperative）
├── 任务（Task）          ← 最基本执行单元
├── 同步原语
│   ├── 队列（Queue）     ← 任务间传递数据
│   ├── 信号量（Semaphore）← 计数/同步事件
│   ├── 互斥锁（Mutex）   ← 保护共享资源
│   └── 事件组（Event Group）← 多事件等待
├── 软件定时器（Software Timer）
└── 内存管理（heap_1 ~ heap_5）
```

### 关键配置文件：FreeRTOSConfig.h

```c
/* 关键配置项说明 */
#define configCPU_CLOCK_HZ          (120000000UL)  // 主频，影响 SysTick
#define configTICK_RATE_HZ          (1000)          // RTOS 节拍率（1ms/tick）
#define configMAX_PRIORITIES        (8)             // 最大优先级数，按需设置
#define configMINIMAL_STACK_SIZE    (128)           // 最小任务栈（单位：word）
#define configTOTAL_HEAP_SIZE       (10 * 1024)     // 总堆大小
#define configUSE_PREEMPTION        (1)             // 1=抢占式，0=协作式
#define configUSE_MUTEXES           (1)             // 使能互斥锁
#define configUSE_TIMERS            (1)             // 使能软件定时器
#define configSUPPORT_DYNAMIC_ALLOCATION (1)        // 动态内存分配
```

---

## 3. 任务管理实战

### 创建任务

```c
/* 任务函数签名固定：返回 void，参数为 void* */
void task_led(void *pvParameters)
{
    /* 任务初始化代码（只执行一次） */
    bsp_led_init();

    /* 任务主循环（必须是无限循环！） */
    for (;;) {
        bsp_led_toggle();
        vTaskDelay(pdMS_TO_TICKS(500));  // 延时 500ms，让出 CPU
    }
    /* 注意：任务函数不能返回！如果确实需要退出，调用 vTaskDelete(NULL) */
}

void task_uart_recv(void *pvParameters)
{
    char buf[64];
    for (;;) {
        /* 阻塞等待队列数据，超时 100ms */
        if (xQueueReceive(uart_rx_queue, buf, pdMS_TO_TICKS(100)) == pdTRUE) {
            process_uart_data(buf);
        }
    }
}

/* 在 main() 中创建任务 */
int main(void)
{
    system_init();

    /* xTaskCreate 参数：函数、名称、栈大小(word)、参数、优先级、句柄 */
    xTaskCreate(task_led,       "LED",      128, NULL, 2, NULL);
    xTaskCreate(task_uart_recv, "UART_RX",  256, NULL, 5, NULL);

    /* 启动调度器（之后不会返回） */
    vTaskStartScheduler();

    /* 正常情况下不会到达这里 */
    while (1);
}
```

### 任务状态机

```
          xTaskCreate()
               │
               ▼
           [就绪态 Ready]
          ↗           ↘
[运行态 Running]   调度器未选中
          ↘           ↗
           [阻塞态 Blocked]  ← vTaskDelay/xQueueReceive 等待
               │
           [挂起态 Suspended] ← vTaskSuspend/vTaskResume
```

### 栈大小估算技巧

- 基础任务（无大数组）：128-256 words（512-1024 bytes）
- 有 printf/sprintf 的任务：+256 words
- 使用了较深函数调用链：适当增加
- **验证方法**：`uxTaskGetStackHighWaterMark()` 查看最小剩余栈

```c
/* 在任务中周期性打印栈水位（开发调试用） */
UBaseType_t watermark = uxTaskGetStackHighWaterMark(NULL);
printf("Task '%s' stack watermark: %u words\r\n",
       pcTaskGetName(NULL), (unsigned)watermark);
```

---

## 4. 队列：任务间通信利器

队列是 FreeRTOS 中最常用的通信机制，支持从 ISR 发送数据到任务。

### 队列使用模式

```c
/* 定义队列句柄（全局） */
QueueHandle_t uart_rx_queue;
QueueHandle_t sensor_data_queue;

/* 数据结构：建议使用结构体，包含类型和内容 */
typedef struct {
    uint8_t  type;       // 消息类型
    uint16_t length;     // 数据长度
    uint8_t  data[64];   // 数据内容
} MsgPacket_t;

void app_init(void)
{
    /* 创建队列：最多存 10 个 MsgPacket_t */
    uart_rx_queue = xQueueCreate(10, sizeof(MsgPacket_t));
    if (uart_rx_queue == NULL) {
        /* 内存不足，处理错误 */
        Error_Handler();
    }
}

/* ISR 中发送队列（使用 FromISR 版本！） */
void USART0_IRQHandler(void)
{
    MsgPacket_t pkt;
    BaseType_t higher_priority_woken = pdFALSE;

    if (/* 接收完成条件 */) {
        pkt.type = MSG_TYPE_UART;
        pkt.length = received_len;
        memcpy(pkt.data, rx_buf, received_len);

        xQueueSendFromISR(uart_rx_queue, &pkt, &higher_priority_woken);
    }

    /* 如果唤醒了更高优先级任务，请求调度 */
    portYIELD_FROM_ISR(higher_priority_woken);
}

/* 任务中接收队列 */
void task_protocol(void *pvParameters)
{
    MsgPacket_t pkt;
    for (;;) {
        /* portMAX_DELAY 表示永久等待 */
        if (xQueueReceive(uart_rx_queue, &pkt, portMAX_DELAY) == pdTRUE) {
            handle_message(&pkt);
        }
    }
}
```

---

## 5. 信号量与互斥锁

### 二值信号量：事件同步

```c
SemaphoreHandle_t button_sem;

void app_init(void)
{
    button_sem = xSemaphoreCreateBinary();
}

/* ISR：按键按下，给出信号量 */
void EXTI0_IRQHandler(void)
{
    BaseType_t woken = pdFALSE;
    if (exti_interrupt_flag_get(EXTI_0) == SET) {
        xSemaphoreGiveFromISR(button_sem, &woken);
        exti_interrupt_flag_clear(EXTI_0);
    }
    portYIELD_FROM_ISR(woken);
}

/* 任务：等待按键事件 */
void task_button(void *pvParameters)
{
    for (;;) {
        if (xSemaphoreTake(button_sem, portMAX_DELAY) == pdTRUE) {
            handle_button_press();
        }
    }
}
```

### 互斥锁：保护共享资源

```c
/* 场景：多个任务共享一个 SPI 总线 */
MutexHandle_t spi_mutex;

void app_init(void)
{
    spi_mutex = xSemaphoreCreateMutex();
}

void task_spi_read(void *pvParameters)
{
    for (;;) {
        /* 获取互斥锁，超时 100ms */
        if (xSemaphoreTake(spi_mutex, pdMS_TO_TICKS(100)) == pdTRUE) {
            /* ---- 临界区开始 ---- */
            spi_select_device(DEVICE_A);
            spi_transfer(read_buf, write_buf, len);
            spi_deselect_device(DEVICE_A);
            /* ---- 临界区结束 ---- */
            xSemaphoreGive(spi_mutex);
        } else {
            /* 获取超时，记录错误 */
            log_error("SPI mutex timeout");
        }
        vTaskDelay(pdMS_TO_TICKS(10));
    }
}
```

!!! warning "互斥锁 vs 二值信号量"
    互斥锁有**优先级继承**机制，可避免优先级反转，共享资源保护**必须用互斥锁**，不要用二值信号量。

---

## 6. 软件定时器

```c
TimerHandle_t heartbeat_timer;

/* 定时器回调函数（在 Timer Service 任务中执行） */
void heartbeat_timer_callback(TimerHandle_t xTimer)
{
    /* 注意：回调函数中不能调用会阻塞的 API */
    send_heartbeat_packet();
}

void app_init(void)
{
    /* 创建周期定时器：每 30 秒触发一次 */
    heartbeat_timer = xTimerCreate(
        "Heartbeat",                    // 定时器名称
        pdMS_TO_TICKS(30000),           // 周期 30s
        pdTRUE,                          // pdTRUE=周期，pdFALSE=单次
        (void *)0,                       // 定时器 ID
        heartbeat_timer_callback         // 回调函数
    );

    if (heartbeat_timer != NULL) {
        xTimerStart(heartbeat_timer, 0);
    }
}
```

---

## 7. 内存管理策略选择

| 方案 | 特点 | 适用场景 |
|------|------|----------|
| heap_1 | 只分配不释放，无碎片 | 所有对象在初始化时创建，之后不删除 |
| heap_2 | 可释放，可能有碎片 | 任务数量固定，大小固定 |
| heap_3 | 封装标准 malloc/free | 有标准库时使用 |
| heap_4 | 合并相邻空闲块，碎片少 | **推荐**：动态创建/删除任务 |
| heap_5 | 支持不连续内存区域 | 片外 RAM 和片内 RAM 混用 |

**推荐使用 heap_4**，并配合 `configUSE_MALLOC_FAILED_HOOK` 监控内存分配失败。

---

## 8. 常见陷阱与最佳实践

### ❌ 常见错误

| 错误 | 后果 | 正确做法 |
|------|------|----------|
| 在 ISR 中调用非 `FromISR` API | 系统崩溃 | 始终用 `xQueueSendFromISR` 等 |
| 任务函数 `return` 退出 | 未定义行为 | 调用 `vTaskDelete(NULL)` |
| 共享资源不加保护 | 数据竞争 | 使用互斥锁 |
| SysTick 优先级设置错误 | 调度器异常 | 优先级设为最低（数值最大） |
| 栈太小 | 栈溢出，随机崩溃 | 先设大，再用水位检测优化 |
| 在 Timer 回调中阻塞 | Timer 任务死锁 | 回调中只用非阻塞 API |

### ✅ 最佳实践

1. **初始化顺序**：先创建所有任务和同步对象，最后 `vTaskStartScheduler()`
2. **任务命名**：有意义的名称方便调试（`pcTaskGetName`）
3. **统一优先级定义**：用宏集中管理所有任务优先级
4. **加入 Hook 函数**：实现 `vApplicationMallocFailedHook`、`vApplicationStackOverflowHook`
5. **调试时开启运行时统计**：`configGENERATE_RUN_TIME_STATS=1` 查看 CPU 占用

```c
/* 推荐的优先级分配宏 */
#define TASK_PRIO_SAFETY_MONITOR   7   // 最高：安全监控
#define TASK_PRIO_PROTOCOL_RX      6   // 通信接收
#define TASK_PRIO_CONTROL_LOOP     5   // 控制逻辑
#define TASK_PRIO_DATA_PROCESS     4   // 数据处理
#define TASK_PRIO_HMI              3   // 人机交互
#define TASK_PRIO_LOG              2   // 日志
#define TASK_PRIO_IDLE_EXT         1   // 扩展空闲任务
/* tskIDLE_PRIORITY = 0（FreeRTOS 空闲任务） */
```

---

## 9. 移植到 GD32 的注意点

### SysTick 冲突

GD32 HAL 库默认使用 SysTick 作为延时基准，FreeRTOS 也使用 SysTick。解决方案：

```c
/* FreeRTOS port 已接管 SysTick，HAL 延时改用其他定时器 */
/* 或者重写 HAL_GetTick()，返回 xTaskGetTickCount() 转换的 ms 值 */
uint32_t HAL_GetTick(void)
{
    return (uint32_t)(xTaskGetTickCount() * portTICK_PERIOD_MS);
}
```

### 中断优先级配置

```c
/* FreeRTOS 要求：所有调用 FromISR API 的中断，其优先级数值
   必须 >= configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY */
#define configLIBRARY_MAX_SYSCALL_INTERRUPT_PRIORITY  5

/* 安全相关中断（不调用 FreeRTOS API）可以设置更高优先级（数值更小） */
nvic_irq_enable(SAFETY_IRQn, 2, 0);        // 优先级 2，可以
nvic_irq_enable(USART0_IRQn, 5, 0);        // 优先级 5，调用 FromISR API
nvic_irq_enable(USART0_IRQn, 4, 0);        // ❌ 错误！4 < 5，不能调用 FreeRTOS API
```

---

## 10. TODO 与延伸阅读

### 本文待补充

- [ ] 事件组（EventGroup）使用场景与实战
- [ ] 流缓冲区（StreamBuffer）和消息缓冲区（MessageBuffer）
- [ ] FreeRTOS+TCP 网络栈配置
- [ ] 任务通知（Task Notification）替代简单信号量

### 延伸阅读

- [FreeRTOS 官方文档](https://www.freertos.org/Documentation/RTOS_book.html)
- [串口与DMA调试经验](uart-dma-debug.md)
- [GD32 移植 FreeRTOS 详细步骤](gd32-getting-started.md)

---

*经验提示：引入 RTOS 后，调试难度会上升。建议先在裸机上验证各外设驱动，再移植到 RTOS 框架中。*
