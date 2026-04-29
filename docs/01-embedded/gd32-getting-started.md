# GD32 入门与工程实践

> **文章类型**：标准知识文档 + 操作指南  
> **适用读者**：有 C 语言基础，想从零上手 GD32 的开发者  
> **最后更新**：2026-04  
> **标签**：`GD32` `嵌入式` `ARM Cortex-M` `HAL` `时钟配置`

---

## 目录

1. [GD32 芯片系列概览](#1-gd32-芯片系列概览)
2. [开发环境搭建](#2-开发环境搭建)
3. [工程模板结构](#3-工程模板结构)
4. [时钟系统配置](#4-时钟系统配置)
5. [GPIO 基础操作](#5-gpio-基础操作)
6. [中断系统详解](#6-中断系统详解)
7. [调试技巧与常见问题](#7-调试技巧与常见问题)
8. [工程实践清单](#8-工程实践清单)
9. [TODO 与延伸阅读](#9-todo-与延伸阅读)

---

## 1. GD32 芯片系列概览

GD32 是兆易创新（GigaDevice）推出的基于 ARM Cortex-M 内核的 32 位微控制器，与 STM32 高度兼容，是国产化替代的主流选项之一。

### 主要系列对比

| 系列 | 内核 | 主频 | 特点 | 典型型号 |
|------|------|------|------|----------|
| GD32F1xx | Cortex-M3 | 108 MHz | 入门级，Pin 兼容 STM32F1 | GD32F103 |
| GD32F3xx | Cortex-M4 + FPU | 120 MHz | 中端，含 DSP 指令 | GD32F303 |
| GD32F4xx | Cortex-M4 + FPU | 168 MHz | 高性能，对标 STM32F4 | GD32F407 |
| GD32E23x | Cortex-M23 | 72 MHz | 超低功耗，Arm TrustZone | GD32E230 |
| GD32VF1xx | RISC-V | 108 MHz | 国产 RISC-V 内核 | GD32VF103 |

!!! note "与 STM32 的兼容性"
    GD32F103 与 STM32F103 **引脚和寄存器高度兼容**，大多数 STM32 代码可以直接移植。主要差异在于：时钟启动速度、Flash 等待状态默认值不同，需要注意调整。

---

## 2. 开发环境搭建

### 方案一：Keil MDK（推荐用于工作/量产）

```bash
# 1. 安装 Keil MDK 5.x
# 2. 安装 GD32 Pack（兆易官网下载）
# 3. 安装 J-Link / CMSIS-DAP 驱动
```

**安装 GD32 芯片包步骤：**

1. 打开 Keil → Pack Installer
2. 搜索 `GigaDevice`
3. 安装对应系列的 DFP（Device Family Pack）

### 方案二：VSCode + GCC（推荐用于开发效率）

```bash
# 安装工具链
sudo apt install gcc-arm-none-eabi gdb-multiarch openocd

# 安装 VSCode 扩展
# - C/C++ (Microsoft)
# - Cortex-Debug
# - CMake Tools (可选)
```

参考：[VSCode嵌入式开发配置全攻略](../05-tools/vscode-embedded-setup.md)

### 方案三：GD32 IDE（官方 Eclipse 基础）

兆易官网提供基于 Eclipse 的 GD32 IDE，开箱即用，适合快速验证。

---

## 3. 工程模板结构

良好的工程结构是可维护性的基础：

```
project/
├── Application/          # 用户应用层
│   ├── main.c
│   ├── app_task.c        # 应用任务（RTOS场景）
│   └── app_config.h      # 全局配置
├── BSP/                  # 板级支持包
│   ├── bsp_gpio.c/.h
│   ├── bsp_uart.c/.h
│   ├── bsp_led.c/.h
│   └── bsp_key.c/.h
├── Drivers/              # 外设驱动
│   ├── GD32F3xx_Firmware/ # 官方固件库
│   └── CMSIS/
├── Middleware/           # 中间件
│   ├── FreeRTOS/         # 可选
│   └── FATFS/            # 可选
├── Utilities/            # 工具函数
│   ├── ring_buffer.c/.h
│   └── crc.c/.h
├── Makefile / *.uvprojx
└── README.md
```

!!! tip "分层原则"
    保持 **应用层 → BSP层 → 驱动层** 的单向依赖，避免底层代码调用上层业务逻辑，这样在换芯片时只需修改 BSP 和驱动层。

---

## 4. 时钟系统配置

时钟配置是嵌入式开发的第一道关，配错后所有外设都会异常。

### GD32F303 典型时钟配置（120 MHz）

```c
/*!
 * @brief 系统时钟配置：使用 HSE 8MHz，PLL 倍频到 120MHz
 *        AHB = 120MHz, APB1 = 60MHz, APB2 = 120MHz
 */
void system_clock_config(void)
{
    /* 使能 HSE，等待稳定 */
    rcu_osci_on(RCU_HXTAL);
    while (SUCCESS != rcu_osci_stab_wait(RCU_HXTAL));

    /* 设置 AHB/APB 分频 */
    rcu_ahb_clkdiv_config(RCU_AHB_CKSYS_DIV1);     // AHB = SYSCLK
    rcu_apb1_clkdiv_config(RCU_APB1_CKAHB_DIV2);   // APB1 = AHB/2 = 60MHz
    rcu_apb2_clkdiv_config(RCU_APB2_CKAHB_DIV1);   // APB2 = AHB = 120MHz

    /* 配置 Flash 等待状态（120MHz 需要 3 个等待周期）*/
    fmc_wscnt_set(WS_WSCNT_3);

    /* 配置 PLL：HSE/1 * 15 = 120MHz */
    rcu_predv0_config(RCU_PREDV0_DIV1);
    rcu_pll_config(RCU_PLLSRC_HXTAL, RCU_PLL_MUL15);

    /* 使能 PLL，等待稳定 */
    rcu_osci_on(RCU_PLL_CK);
    while (SUCCESS != rcu_osci_stab_wait(RCU_PLL_CK));

    /* 切换系统时钟到 PLL */
    rcu_system_clock_source_config(RCU_CKSYSSRC_PLL);
    while (RCU_SCSS_PLL != rcu_system_clock_source_get());
}
```

### 时钟配置常见问题

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 系统跑飞/死机 | Flash 等待周期不足 | 先设置 `fmc_wscnt_set()`，再切换时钟 |
| UART 波特率偏差 | APB 时钟算错 | 用逻辑分析仪验证，检查 APB 分频配置 |
| SysTick 不准 | SystemCoreClock 未更新 | 调用 `SystemCoreClockUpdate()` |
| 从 HSI 切到 HSE 失败 | 外部晶振未起振 | 检查晶振电路，加大启动超时 |

---

## 5. GPIO 基础操作

### 初始化流程

```c
#include "gd32f30x.h"

/*!
 * @brief LED GPIO 初始化（PC13 推挽输出）
 */
void bsp_led_init(void)
{
    /* 1. 使能 GPIO 时钟 */
    rcu_periph_clock_enable(RCU_GPIOC);

    /* 2. 配置引脚模式：推挽输出，最高速 50MHz */
    gpio_init(GPIOC, GPIO_MODE_OUT_PP, GPIO_OSPEED_50MHZ, GPIO_PIN_13);

    /* 3. 初始状态：熄灭（高电平有效外接 LED 则置高） */
    gpio_bit_set(GPIOC, GPIO_PIN_13);
}

/* LED 控制函数 */
void bsp_led_on(void)  { gpio_bit_reset(GPIOC, GPIO_PIN_13); }
void bsp_led_off(void) { gpio_bit_set(GPIOC, GPIO_PIN_13); }
void bsp_led_toggle(void) { gpio_bit_write(GPIOC, GPIO_PIN_13,
    (RESET == gpio_input_bit_get(GPIOC, GPIO_PIN_13)) ? SET : RESET); }
```

### GPIO 模式速查

| 模式宏 | 说明 | 适用场景 |
|--------|------|----------|
| `GPIO_MODE_IPU` | 输入上拉 | 按键检测 |
| `GPIO_MODE_IPD` | 输入下拉 | 外部信号检测 |
| `GPIO_MODE_IN_FLOATING` | 浮空输入 | ADC 输入、外部时钟 |
| `GPIO_MODE_OUT_PP` | 推挽输出 | LED、普通数字输出 |
| `GPIO_MODE_OUT_OD` | 开漏输出 | I2C 总线、电平转换 |
| `GPIO_MODE_AF_PP` | 复用推挽 | UART TX、SPI CLK/MOSI |
| `GPIO_MODE_AF_OD` | 复用开漏 | I2C SCL/SDA |

---

## 6. 中断系统详解

### NVIC 优先级配置

GD32 使用 4 位优先级位，采用抢占优先级 + 子优先级分组。

```c
/*!
 * @brief 中断优先级分组配置
 *        推荐：4位全用于抢占优先级（NVIC_PRIGROUP_PRE4_SUB0）
 *        抢占优先级 0-15，数值越小优先级越高
 */
void nvic_config(void)
{
    nvic_priority_group_set(NVIC_PRIGROUP_PRE4_SUB0);
}

/* UART 中断配置示例 */
void uart0_nvic_config(void)
{
    nvic_irq_enable(USART0_IRQn, 2, 0);  // 抢占优先级2，子优先级0
}
```

### 优先级分配建议（参考）

| 外设/中断 | 建议抢占优先级 | 原因 |
|-----------|--------------|------|
| SysTick（RTOS） | 最低（15） | 系统节拍，不应抢占业务 |
| 安全相关（过压/过流） | 0 或 1 | 最高优先级，立即响应 |
| CAN/UART 接收 | 2-3 | 数据不能丢失 |
| 按键 EXTI | 5-6 | 人机交互，可以稍晚响应 |
| 普通定时器 | 7-8 | 一般周期任务 |

---

## 7. 调试技巧与常见问题

### SWD 调试连接问题

```
问题：J-Link 无法连接目标板
排查步骤：
1. 检查 VCC/GND/SWDIO/SWDCLK 连接
2. 确认目标板已上电
3. 尝试降低 SWD 速率（从 4MHz 降到 1MHz）
4. 检查是否有代码关闭了 SWD 引脚复用
5. 使用 J-Link 的 "Connect under Reset" 模式
```

### 常见 HardFault 原因

| 原因 | 排查方法 |
|------|----------|
| 空指针解引用 | 检查指针初始化，GDB 查看 PC 寄存器 |
| 栈溢出 | 增大任务栈/主栈，使用栈水位检测 |
| 未对齐内存访问 | 检查结构体对齐，使用 `__packed` |
| 除零错误 | 添加除法前检查 |
| 访问越界 | 检查数组下标，添加边界断言 |

```c
/* HardFault 调试辅助：打印寄存器信息 */
void HardFault_Handler(void)
{
    __asm volatile (
        "TST LR, #4\n"
        "ITE EQ\n"
        "MRSEQ R0, MSP\n"
        "MRSNE R0, PSP\n"
        "B hard_fault_handler_c\n"
    );
}

void hard_fault_handler_c(unsigned int *stack)
{
    /* 在调试器中查看这些寄存器值 */
    volatile unsigned int r0  = stack[0];
    volatile unsigned int r1  = stack[1];
    volatile unsigned int r2  = stack[2];
    volatile unsigned int r3  = stack[3];
    volatile unsigned int r12 = stack[4];
    volatile unsigned int lr  = stack[5];  // 链接寄存器
    volatile unsigned int pc  = stack[6];  // 出错的 PC 地址
    volatile unsigned int psr = stack[7];
    (void)r0; (void)r1; (void)r2; (void)r3;
    (void)r12; (void)lr; (void)pc; (void)psr;
    while (1);  // 在这里设置断点
}
```

---

## 8. 工程实践清单

在开始一个新的 GD32 项目前，检查以下清单：

- [ ] **硬件确认**：核对原理图，确认晶振频率、引脚分配
- [ ] **时钟配置**：根据系统需求配置正确的主频，验证 APB 分频
- [ ] **Flash 等待周期**：与主频匹配，避免读取错误
- [ ] **中断优先级分组**：在 main() 最开始就设置，后续不再更改
- [ ] **看门狗**：工程初期可先关闭，稳定后再开启独立看门狗
- [ ] **SWD 引脚保护**：不要将 PA13/PA14 用于普通 GPIO
- [ ] **代码风格**：遵守 [嵌入式C编程规范](embedded-c-coding-style.md)
- [ ] **版本管理**：建立 Git 仓库，Keil 工程配置文件加入 `.gitignore`

---

## 9. TODO 与延伸阅读

### 本文待补充

- [ ] GD32F4 系列特有外设（SDIO、ETH）配置
- [ ] GD32 vs STM32 完整兼容性对比表
- [ ] 使用 CubeMX 类似工具（GD32 Configuration Tool）

### 延伸阅读

- [GD32F3xx 用户手册（官方）](https://www.gigadevice.com.cn)
- [ARM Cortex-M3 权威指南（中文版）](https://book.douban.com)
- [FreeRTOS实战笔记](freertos-practical-notes.md)
- [嵌入式C编程规范](embedded-c-coding-style.md)

---

*本文基于 GD32F303 系列编写，其他系列可能存在差异，请参考对应芯片手册。*
