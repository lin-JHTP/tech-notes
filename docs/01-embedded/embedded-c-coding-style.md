# 嵌入式 C 编程规范

> **文章类型**：经验清单 + 规范文档  
> **适用读者**：嵌入式 C 语言开发者  
> **最后更新**：2026-04  
> **标签**：`C语言` `编程规范` `嵌入式` `代码风格` `可维护性`

---

## 目录

1. [命名规范](#1-命名规范)
2. [文件与模块组织](#2-文件与模块组织)
3. [类型定义与使用](#3-类型定义与使用)
4. [函数设计原则](#4-函数设计原则)
5. [错误处理模式](#5-错误处理模式)
6. [宏与常量](#6-宏与常量)
7. [注释规范](#7-注释规范)
8. [中断与并发安全](#8-中断与并发安全)
9. [规范速查清单](#9-规范速查清单)
10. [TODO](#10-todo)

---

## 1. 命名规范

### 基本原则

- 使用**蛇形命名（snake_case）**，与 C 标准库风格一致
- 名称应**自注释**，无需猜测含义
- 全局变量/函数加**模块前缀**，避免命名冲突

### 命名对照表

| 类别 | 命名规则 | 示例 |
|------|----------|------|
| 全局变量 | `g_模块_名称` | `g_uart_rx_count` |
| 静态（文件作用域）变量 | `s_名称` | `s_led_state` |
| 局部变量 | 简洁小写 | `len`, `timeout_ms` |
| 函数（公共） | `模块_动词_名词()` | `uart_send_data()` |
| 函数（私有/static） | `动词_名词()` | `calculate_checksum()` |
| 宏常量 | 全大写下划线 | `MAX_RETRY_COUNT` |
| 类型定义 | 后缀 `_t` | `UartConfig_t` |
| 枚举值 | 模块前缀全大写 | `LED_STATE_ON` |

```c
/* ✅ 好的命名 */
typedef struct {
    uint32_t baud_rate;
    uint8_t  data_bits;
    uint8_t  stop_bits;
} UartConfig_t;

typedef enum {
    CHARGER_STATE_IDLE    = 0,
    CHARGER_STATE_READY   = 1,
    CHARGER_STATE_CHARGING = 2,
    CHARGER_STATE_FAULT   = 3,
} ChargerState_e;

static uint32_t s_charge_session_id = 0;

void charger_start_session(uint32_t connector_id);

/* ❌ 不好的命名 */
int x;           // 无意义
void f(int a);   // 缩写过度
uint8_t flag2;   // 含义不明确
```

---

## 2. 文件与模块组织

### 头文件（.h）模板

```c
/*!
 * @file    bsp_uart.h
 * @brief   UART BSP 接口声明
 * @version 1.0.0
 * @date    2026-04-01
 */

#ifndef BSP_UART_H    /* 防重复包含守卫 */
#define BSP_UART_H

#include <stdint.h>
#include <stdbool.h>

/* ==================== 宏定义 ==================== */
#define UART_TX_BUF_SIZE   256U
#define UART_RX_BUF_SIZE   256U

/* ==================== 类型定义 ==================== */
typedef struct {
    uint32_t baud_rate;
    uint8_t  data_bits;
} UartConfig_t;

typedef void (*UartRxCallback_t)(const uint8_t *data, uint16_t len);

/* ==================== 公共函数声明 ==================== */
void    bsp_uart_init(const UartConfig_t *config);
int32_t bsp_uart_send(const uint8_t *data, uint16_t len);
void    bsp_uart_set_rx_callback(UartRxCallback_t cb);

#endif /* BSP_UART_H */
```

### 源文件（.c）模板

```c
/*!
 * @file    bsp_uart.c
 * @brief   UART BSP 实现
 */

/* ---- 本文件的头文件最先包含 ---- */
#include "bsp_uart.h"

/* ---- 标准库头文件 ---- */
#include <string.h>

/* ---- 第三方/平台头文件 ---- */
#include "gd32f30x.h"

/* ---- 私有宏 ---- */
#define USART_PERIPH    USART0

/* ---- 文件作用域变量 ---- */
static UartRxCallback_t s_rx_callback = NULL;
static uint8_t s_rx_buf[UART_RX_BUF_SIZE];

/* ---- 私有函数声明 ---- */
static void uart_gpio_init(void);
static void uart_dma_init(void);

/* ==================== 公共函数实现 ==================== */

void bsp_uart_init(const UartConfig_t *config)
{
    /* 参数检查 */
    if (config == NULL) {
        return;
    }
    uart_gpio_init();
    uart_dma_init();
    /* ... */
}

/* ==================== 私有函数实现 ==================== */

static void uart_gpio_init(void)
{
    /* ... */
}
```

---

## 3. 类型定义与使用

### 始终使用固定宽度类型

```c
/* ✅ 推荐 */
#include <stdint.h>
#include <stdbool.h>

uint8_t  byte_val;      // 明确 8 位无符号
uint16_t word_val;      // 明确 16 位无符号
uint32_t dword_val;     // 明确 32 位无符号
int32_t  signed_val;    // 明确 32 位有符号
bool     flag;          // 布尔类型

/* ❌ 避免 */
int   val;    // 在不同平台宽度不同
char  flag;   // 语义不清（有符号？字符？）
long  count;  // 在32位和64位系统上不同
```

### 结构体对齐与打包

```c
/* 注意：未指定 __packed 时，编译器会填充对齐 */
typedef struct {
    uint8_t  type;      // 偏移 0，占 1 字节
    /* 编译器可能在此插入 1 字节填充 */
    uint16_t length;    // 偏移 2，占 2 字节
    uint32_t timestamp; // 偏移 4，占 4 字节
} FrameHeader_t;  /* sizeof = 8，非 7 */

/* 协议帧结构体：需要精确内存布局时使用 __packed */
typedef struct __attribute__((packed)) {
    uint8_t  sof;       // 帧头
    uint16_t length;    // 数据长度
    uint8_t  type;      // 类型
    uint8_t  data[0];   // 变长数据（柔性数组）
} ProtocolFrame_t;
```

---

## 4. 函数设计原则

### 单一职责

每个函数只做一件事，函数体不超过 50 行（特殊情况除外）。

### 返回值规范

```c
/* 统一返回值定义 */
typedef enum {
    ERR_OK           =  0,   // 成功
    ERR_INVALID_PARAM = -1,  // 无效参数
    ERR_TIMEOUT      = -2,   // 超时
    ERR_NO_MEMORY    = -3,   // 内存不足
    ERR_BUSY         = -4,   // 设备忙
    ERR_NOT_INIT     = -5,   // 未初始化
} ErrCode_e;

/*!
 * @brief 发送 UART 数据
 * @param[in] data 数据指针（非 NULL）
 * @param[in] len  数据长度（1~256 字节）
 * @return ERR_OK 成功；负值 错误码
 */
int32_t bsp_uart_send(const uint8_t *data, uint16_t len)
{
    /* 参数检查放在函数最开始 */
    if (data == NULL || len == 0 || len > UART_TX_BUF_SIZE) {
        return ERR_INVALID_PARAM;
    }
    if (s_tx_busy) {
        return ERR_BUSY;
    }
    /* ... */
    return ERR_OK;
}
```

### 避免深层嵌套（Early Return）

```c
/* ❌ 深层嵌套，难以阅读 */
int process_data(uint8_t *buf, uint16_t len)
{
    if (buf != NULL) {
        if (len > 0) {
            if (len <= MAX_LEN) {
                /* 实际处理 */
                return 0;
            }
        }
    }
    return -1;
}

/* ✅ Early Return，层级浅，清晰 */
int process_data(uint8_t *buf, uint16_t len)
{
    if (buf == NULL)    return ERR_INVALID_PARAM;
    if (len == 0)       return ERR_INVALID_PARAM;
    if (len > MAX_LEN)  return ERR_INVALID_PARAM;

    /* 实际处理 */
    return ERR_OK;
}
```

---

## 5. 错误处理模式

### 断言（Assert）用于调试

```c
#include <assert.h>

/* 开发阶段：使用 assert 捕获不应发生的错误 */
void uart_send_packet(Packet_t *pkt)
{
    assert(pkt != NULL);          // 指针不为空
    assert(pkt->len <= MAX_PKT);  // 长度合法
    /* ... */
}

/* 生产固件：assert 通常被禁用（NDEBUG），需要用运行时检查替代 */
```

### GOTO 模式（资源清理）

```c
int init_module(void)
{
    int ret = ERR_OK;

    buffer = malloc(BUF_SIZE);
    if (buffer == NULL) { ret = ERR_NO_MEMORY; goto cleanup; }

    handle = open_device();
    if (handle < 0) { ret = ERR_DEVICE; goto cleanup; }

    if (register_callback() != 0) { ret = ERR_REGISTER; goto cleanup; }

    return ERR_OK;

cleanup:
    if (handle >= 0) close_device(handle);
    if (buffer != NULL) { free(buffer); buffer = NULL; }
    return ret;
}
```

---

## 6. 宏与常量

```c
/* ✅ 常量优先用 const 或 enum，有类型检查 */
static const uint32_t BAUD_RATE_DEFAULT = 115200U;
typedef enum { LED_OFF = 0, LED_ON = 1 } LedState_e;

/* 函数式宏：操作数加括号，整体加括号 */
#define MIN(a, b)       (((a) < (b)) ? (a) : (b))
#define ARRAY_SIZE(arr) (sizeof(arr) / sizeof((arr)[0]))
#define BIT(n)          (1UL << (n))

/* 防止宏副作用：不要在宏中使用自增/自减 */
/* ❌ 危险：MIN(x++, y) 会让 x 递增两次 */
/* ✅ 用内联函数代替有副作用风险的宏 */
static inline uint32_t min_u32(uint32_t a, uint32_t b)
{
    return (a < b) ? a : b;
}
```

---

## 7. 注释规范

```c
/*!
 * @brief   简短说明函数用途（一行）
 *
 * 详细说明（可选）：解释算法、限制条件、副作用等。
 *
 * @param[in]  src    源缓冲区指针，不能为 NULL
 * @param[out] dst    目标缓冲区指针，不能为 NULL
 * @param[in]  len    要复制的字节数（1~MAX_FRAME_LEN）
 * @return     实际复制的字节数；负值表示错误
 *
 * @note  调用前必须确保 dst 有足够的空间
 * @warning 该函数不是线程安全的，需在临界区内调用
 */
int32_t frame_copy(uint8_t *dst, const uint8_t *src, uint16_t len);
```

**内联注释原则：**

- 注释解释**为什么**，而不是**是什么**（代码本身说明是什么）
- 过期的注释比没有注释更糟糕，及时更新
- 用 `TODO:` / `FIXME:` / `HACK:` 标记待办和问题

```c
/* 按照 OCPP 1.6 规范 §4.1，心跳间隔不能小于 10 秒 */
if (heartbeat_interval < 10) {
    heartbeat_interval = 10;
}

/* TODO: 实现指数退避，避免网络恢复时洪泛 */
retry_connect();

/* FIXME: 在低功耗模式下此计时不准确，需要使用 RTC 替代 */
vTaskDelay(pdMS_TO_TICKS(1000));
```

---

## 8. 中断与并发安全

```c
/* 场景：中断和主任务共享变量 */
static volatile uint32_t g_tick_count = 0;  // volatile 防止优化

void SysTick_Handler(void)
{
    g_tick_count++;  // 原子操作（32位MCU对齐变量），无需额外保护
}

/* 非原子操作：必须加临界区 */
typedef struct {
    uint16_t head;
    uint16_t tail;
    uint8_t  buf[256];
} RingBuf_t;

static RingBuf_t s_ring_buf;

/* 在中断中写入：必须禁用中断或使用原子操作 */
void push_byte_from_isr(uint8_t byte)
{
    uint16_t next_head = (s_ring_buf.head + 1) % sizeof(s_ring_buf.buf);
    if (next_head != s_ring_buf.tail) {  // 非满
        s_ring_buf.buf[s_ring_buf.head] = byte;
        s_ring_buf.head = next_head;      // 写入后再更新 head（内存屏障）
        __DSB();  // 数据同步屏障
    }
}
```

---

## 9. 规范速查清单

开始代码审查时，检查以下项目：

**命名**
- [ ] 全局变量/函数有模块前缀
- [ ] 使用固定宽度类型（`uint8_t` 而非 `char`）
- [ ] 枚举值有模块前缀

**函数**
- [ ] 参数验证在函数开头
- [ ] 单一职责，函数不超过 50 行
- [ ] 返回值有意义，错误用负值
- [ ] 函数有 Doxygen 注释

**内存**
- [ ] 无动态内存分配（或有安全的分配失败处理）
- [ ] 数组访问有边界检查
- [ ] 指针使用前检查非 NULL

**并发**
- [ ] ISR 中只用 `volatile` 变量或 `FromISR` API
- [ ] 共享资源有互斥保护
- [ ] `volatile` 用于 ISR 共享变量

---

## 10. TODO

- [ ] 添加 MISRA-C 关键规则对照表
- [ ] 代码静态分析工具配置（Cppcheck、PC-lint）
- [ ] 单元测试框架（Unity）集成到嵌入式项目
- [ ] Git commit message 规范（与代码规范配套）
