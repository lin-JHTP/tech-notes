# 充电桩安全业务设计

> **文章类型**：标准知识文档 + 业务设计笔记  
> **适用读者**：充电桩固件开发者、安全业务设计工程师  
> **最后更新**：2026-04  
> **标签**：`充电桩` `安全` `过压保护` `绝缘检测` `故障处理` `电气安全`

---

## 目录

1. [充电安全的层次模型](#1-充电安全的层次模型)
2. [电气安全检测项目](#2-电气安全检测项目)
3. [典型故障码与处理策略](#3-典型故障码与处理策略)
4. [充电前安全检查流程](#4-充电前安全检查流程)
5. [充电过程监控](#5-充电过程监控)
6. [急停与安全停机](#6-急停与安全停机)
7. [国标安全要求速查](#7-国标安全要求速查)
8. [软件设计要点](#8-软件设计要点)
9. [TODO](#9-todo)

---

## 1. 充电安全的层次模型

充电桩安全防护是**多层次**的，软件安全只是最后一道防线：

```
第一层：硬件保护
  ├── 熔断器（过流保护，不可恢复）
  ├── 断路器（过流保护，可复位）
  ├── 防雷器 / 浪涌保护（SPD）
  └── 硬件过压保护电路

第二层：固件/控制器保护
  ├── 过压 / 欠压检测（软件采样+硬件比较器）
  ├── 过流检测（电流互感器采样）
  ├── 过温检测（NTC 热敏电阻）
  ├── 绝缘检测（漏电检测模块）
  └── 接地检测

第三层：协议层保护
  ├── OCPP 故障上报（StatusNotification Faulted）
  ├── 车桩通信异常检测（CP 信号监控）
  └── 通信超时保护

第四层：运营平台监控
  ├── 远程状态监控
  ├── 异常告警推送
  └── 远程停机指令
```

---

## 2. 电气安全检测项目

### 2.1 过压/欠压检测

| 检测项 | 典型阈值（交流 220V 场景） | 动作 |
|--------|--------------------------|------|
| 过压检测（一级） | > 264V（AC 220V+20%） | 告警，继续充电 |
| 过压检测（二级） | > 276V | 立即停充，上报故障 |
| 欠压检测 | < 176V（AC 220V-20%） | 告警，继续充电 |
| 欠压停充 | < 165V | 停充，上报故障 |
| 相序检测（三相） | 缺相/错相 | 禁止充电 |

```c
/* 过压检测示例（周期性检测，每 100ms 执行一次） */
typedef enum {
    VOLTAGE_STATUS_NORMAL  = 0,
    VOLTAGE_STATUS_WARNING = 1,   // 告警，继续充电
    VOLTAGE_STATUS_FAULT   = 2,   // 停充
} VoltageStatus_e;

VoltageStatus_e check_ac_voltage(float voltage_v)
{
    if (voltage_v > 276.0f || voltage_v < 165.0f) {
        return VOLTAGE_STATUS_FAULT;
    }
    if (voltage_v > 264.0f || voltage_v < 176.0f) {
        return VOLTAGE_STATUS_WARNING;
    }
    return VOLTAGE_STATUS_NORMAL;
}
```

### 2.2 过流检测

```c
/* 过流保护：软件实现分级检测 */
#define CURRENT_RATED_A      32.0f   // 额定电流
#define CURRENT_WARN_A       (CURRENT_RATED_A * 1.1f)   // 110% 告警
#define CURRENT_FAULT_A      (CURRENT_RATED_A * 1.25f)  // 125% 停充

void check_output_current(float current_a)
{
    if (current_a > CURRENT_FAULT_A) {
        trigger_emergency_stop(FAULT_OVERCURRENT);
        report_fault_to_csms("OverCurrentFailure");
    } else if (current_a > CURRENT_WARN_A) {
        log_warning("Output current %.1fA exceeds rated", current_a);
    }
}
```

### 2.3 绝缘检测

绝缘检测是直流充电桩的必检项，检测充电线路对地的绝缘电阻是否正常。

| 检测阶段 | 要求（GB/T 18487.1） |
|---------|---------------------|
| 充电前 | 绝缘电阻 ≥ 1MΩ（500V 直流测量） |
| 充电中 | 绝缘电阻 ≥ 100kΩ（持续监测） |
| 绝缘故障阈值 | < 100kΩ 时立即停充 |

```c
/* 绝缘检测状态机 */
typedef enum {
    INSULATION_IDLE,
    INSULATION_TESTING,   // 预充电前检测
    INSULATION_OK,        // 检测通过
    INSULATION_FAIL,      // 检测失败
    INSULATION_MONITORING // 充电中持续监测
} InsulationState_e;

static InsulationState_e s_insulation_state = INSULATION_IDLE;

/*!
 * @brief 绝缘检测结果处理
 * @param resistance_kohm 测量到的绝缘电阻（kΩ）
 */
void handle_insulation_result(float resistance_kohm)
{
    if (s_insulation_state == INSULATION_TESTING) {
        if (resistance_kohm >= 1000.0f) {  // ≥ 1MΩ
            s_insulation_state = INSULATION_OK;
            log_info("Insulation OK: %.0f kOhm", resistance_kohm);
        } else {
            s_insulation_state = INSULATION_FAIL;
            trigger_emergency_stop(FAULT_INSULATION_FAILURE);
        }
    } else if (s_insulation_state == INSULATION_MONITORING) {
        if (resistance_kohm < 100.0f) {  // < 100kΩ
            trigger_emergency_stop(FAULT_INSULATION_FAILURE);
        }
    }
}
```

### 2.4 CP 信号检测（SAE J1772 / IEC 61851）

CP（Control Pilot）信号是交流充电的车桩通信基础：

| CP 电压 | 含义 |
|---------|------|
| +12V（无 PWM） | 准备就绪，无车 |
| +9V（有 PWM） | 车已连接，等待充电 |
| +6V（有 PWM） | 车已连接，请求充电 |
| +3V（有 PWM） | 车已连接，通风请求 |
| 0V | 故障 |
| -12V | 故障 |

```c
/* PWM 占空比对应充电电流（IEC 61851-1） */
float cp_duty_to_current_a(float duty_percent)
{
    if (duty_percent >= 8.0f && duty_percent <= 85.0f) {
        return duty_percent * 0.6f;         // 8%~85%: 电流 = 占空比 × 0.6
    } else if (duty_percent > 85.0f && duty_percent <= 96.0f) {
        return (duty_percent - 64.0f) * 2.5f; // 85%~96%: 高电流模式
    }
    return 0.0f;  // 超范围
}
```

---

## 3. 典型故障码与处理策略

OCPP 定义了标准故障码（ErrorCode），实现时需要正确映射：

| OCPP ErrorCode | 含义 | 典型触发条件 | 处理策略 |
|----------------|------|------------|----------|
| `GroundFailure` | 接地故障 | 接地线检测失败 | 立即停充，报修 |
| `HighTemperature` | 过温 | 温度传感器超阈值 | 降功率 → 超阈值停充 |
| `InternalError` | 内部错误 | 软件异常、通信失败 | 尝试复位，上报 |
| `LocalListConflict` | 本地列表冲突 | 白名单版本不一致 | 刷新白名单 |
| `NoError` | 无故障 | 正常状态 | - |
| `OtherError` | 其他错误 | 自定义故障 | 配合 vendorErrorCode |
| `OverCurrentFailure` | 过流 | 电流超额定值 | 立即停充 |
| `OverVoltage` | 过压 | 电压超额定范围 | 立即停充 |
| `PowerMeterFailure` | 电表故障 | 电表通信失败 | 停充（计量异常不允许充电） |
| `PowerSwitchFailure` | 开关故障 | 接触器/继电器异常 | 立即停充，报修 |
| `ReaderFailure` | 读卡器故障 | RFID 读卡器无响应 | 仅影响刷卡功能 |
| `ResetFailure` | 复位失败 | 复位命令执行失败 | 硬件复位 |
| `UnderVoltage` | 欠压 | 电压低于允许范围 | 停充 |
| `WeakSignal` | 信号弱 | 网络信号差（4G等） | 告警，不停充 |

---

## 4. 充电前安全检查流程

```
用户插枪
    │
    ▼
1. CP 信号检测（检测到车辆连接）
    │ 异常 → 故障停机
    ▼
2. 接地检测
    │ 失败 → GroundFailure，禁止充电
    ▼
3. 授权验证（刷卡/APP/白名单）
    │ 失败 → 保持 Preparing，超时释放
    ▼
4. 绝缘检测（直流桩必须）
    │ 失败 → 绝缘故障，禁止充电
    ▼
5. 电表初始化读数
    │ 失败 → 禁止充电（计量合规要求）
    ▼
6. 发送 StartTransaction → CSMS
    │ 失败（离线）→ 本地记录，继续充电（按配置）
    ▼
7. 闭合接触器，开始充电
    │
    ▼
充电中监控（见第5节）
```

---

## 5. 充电过程监控

充电中需要定期采样并检查以下参数：

```c
/* 充电监控任务（每 500ms 执行） */
void charging_monitor_task(void *pvParameters)
{
    ChargingMonitorData_t data;

    for (;;) {
        /* 采样所有监控参数 */
        data.voltage_v     = adc_read_voltage();
        data.current_a     = adc_read_current();
        data.temperature_c = adc_read_temperature();
        data.insulation_kohm = read_insulation_resistance();
        data.cp_voltage_v  = adc_read_cp_voltage();

        /* 逐项检查 */
        check_voltage_range(data.voltage_v);
        check_current_range(data.current_a);
        check_temperature(data.temperature_c);
        check_insulation_monitoring(data.insulation_kohm);
        check_cp_signal(data.cp_voltage_v);

        /* 定期上报 MeterValues（按 OCPP 配置的间隔） */
        if (should_report_meter_values()) {
            ocpp_send_meter_values(&data);
        }

        vTaskDelay(pdMS_TO_TICKS(500));
    }
}

/* 过温处理（分级策略） */
void check_temperature(float temp_c)
{
    if (temp_c > 85.0f) {
        /* 超过 85°C：立即停充 */
        trigger_emergency_stop(FAULT_HIGH_TEMPERATURE);
    } else if (temp_c > 75.0f) {
        /* 超过 75°C：降额充电（降低输出功率 50%）*/
        reduce_charging_power(50);
        report_warning_to_csms("HighTemperature", temp_c);
    } else if (temp_c > 65.0f) {
        /* 超过 65°C：告警，不降功率 */
        log_warning("Temperature warning: %.1f°C", temp_c);
    }
    /* 低于 65°C：恢复正常功率（如有降额） */
    else if (temp_c < 60.0f) {
        restore_charging_power();
    }
}
```

---

## 6. 急停与安全停机

急停必须是**硬件级别**的快速响应，软件处理是辅助：

```c
/*!
 * @brief 触发急停（安全停机）
 *        硬件联锁优先，然后软件处理后续流程
 * @param fault_code 故障原因
 */
void trigger_emergency_stop(FaultCode_e fault_code)
{
    /* 第一步：立即断开主回路（硬件继电器/接触器） */
    /* 此操作应尽快完成，不依赖 RTOS 调度 */
    GPIO_RELAY_OFF();   // 直接操作 GPIO，不走 BSP 层

    /* 第二步：记录故障信息 */
    fault_record.code      = fault_code;
    fault_record.timestamp = get_rtc_timestamp();
    fault_record.voltage   = last_voltage_reading;
    fault_record.current   = last_current_reading;

    /* 第三步：更新状态机 */
    set_charger_state(CHARGER_STATE_FAULT);

    /* 第四步：上报 OCPP StatusNotification */
    /* 注意：此操作在 RTOS 任务中完成，ISR 中只置位标志 */
    ocpp_report_fault(fault_code_to_ocpp_error(fault_code));

    /* 第五步：本地指示（LED/蜂鸣器） */
    bsp_buzzer_on(BUZZER_FAULT_PATTERN);
    bsp_led_set_fault_pattern();
}

/* 安全恢复：只有明确的恢复条件满足后才能复位 */
bool can_clear_fault(FaultCode_e fault_code)
{
    switch (fault_code) {
    case FAULT_OVERVOLTAGE:
        /* 电压恢复正常范围后 3 秒内无新故障才可清除 */
        return (get_current_voltage() < VOLTAGE_FAULT_THRESHOLD)
               && (fault_duration_sec() > 3);
    case FAULT_INSULATION_FAILURE:
        /* 绝缘故障需要人工复位（物理按键或后台指令） */
        return is_manual_reset_triggered();
    case FAULT_HIGH_TEMPERATURE:
        /* 温度降至安全范围以下 */
        return (get_temperature() < TEMPERATURE_RECOVER_THRESHOLD);
    default:
        return false;
    }
}
```

---

## 7. 国标安全要求速查

| 标准 | 内容 | 关键条款 |
|------|------|----------|
| GB/T 18487.1-2023 | 电动汽车传导充电系统 总则 | 绝缘要求、接地、保护联结 |
| GB/T 20234.1-2023 | 交流充电接口 | 插头插座物理尺寸、CP 信号要求 |
| GB/T 20234.2-2015 | 直流充电接口 | 物理接口、CC/CC2 信号 |
| GB/T 20234.3-2015 | 无线充电接口 | 无线充电功率等级 |
| GB/T 27930-2023 | 直流充电通信协议 | BMS-EVSE 通信（CANopen-like） |
| NB/T 33001-2022 | 充电桩技术规范 | 效率、功率因数、谐波 |

---

## 8. 软件设计要点

### 故障数据持久化

```c
/* 故障记录需写入非易失存储（Flash/EEPROM），重启后可读取 */
typedef struct __attribute__((packed)) {
    uint32_t  fault_code;
    uint32_t  timestamp;
    float     voltage_at_fault;
    float     current_at_fault;
    float     temperature_at_fault;
    uint32_t  crc32;              // 数据完整性校验
} FaultRecord_t;

#define MAX_FAULT_RECORDS  100    // 循环存储最近 100 条

void save_fault_record(const FaultRecord_t *record);
int  read_fault_records(FaultRecord_t *out_buf, int max_count);
```

### 安全相关任务的优先级原则

- 安全监控任务优先级：**系统最高**（高于通信、HMI、日志）
- 故障处理：能用硬件中断或比较器触发的，绝不依赖软件轮询
- 看门狗：独立看门狗（IWDG）必须开启，确保软件死锁时硬件自恢复

---

## 9. TODO

- [ ] 补充直流桩 BMS 通信（GB/T 27930）安全业务详解
- [ ] 充电桩接触器粘连检测方案
- [ ] 充电桩 OTA 升级的安全性设计（签名校验、回滚机制）
- [ ] 充电桩防盗电业务逻辑（异常用电检测）
- [ ] 通信安全：TLS 证书管理与 OCPP Security Profile 实现

---

*安全设计黄金原则：**失效安全（Fail-Safe）**——任何组件失效时，系统应进入最安全的状态（停止充电），而非试图继续运行。*
