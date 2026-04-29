# 问题排查日志示例

> **文章类型**：经验清单 + 真实排查记录示例  
> **用途**：展示如何记录和结构化问题排查过程，供参考和学习  
> **最后更新**：2026-04

---

## 📋 问题排查日志格式说明

良好的问题排查日志包含以下要素：

| 字段 | 说明 |
|------|------|
| **日期时间** | 问题发现时间 |
| **问题描述** | 现象，可观察到的异常 |
| **影响范围** | 影响哪些功能/用户/设备 |
| **排查过程** | 有序的排查步骤和每步的发现 |
| **根本原因** | Root Cause，而不仅仅是直接原因 |
| **解决方案** | 短期修复 + 长期防范 |
| **验证结果** | 修复后的验证方法和结论 |
| **经验教训** | 下次如何更快发现/避免 |

---

## 📌 示例1：FreeRTOS 任务栈溢出导致随机重启

```
日期：2026-03-15
严重程度：高（充电过程中随机重启，影响用户体验）
```

### 问题描述

充电桩在充电过程中，约每 2-3 小时出现一次系统重启。重启前没有任何明显错误提示，串口日志突然中断后重新输出 BootNotification。

### 影响范围

- 所有处于充电状态的会话会异常中断
- 充电记录可能不完整（StopTransaction 未发送）
- 客户投诉：充电突然停止

### 排查过程

**步骤1：确认不是硬件问题**

```
观察：重启后硬件状态正常，无过温/过流故障记录
排除：电源问题（用示波器确认电源稳定）
排除：复位引脚（测量正常）
结论：软件复位（看门狗触发或硬件异常）
```

**步骤2：分析看门狗日志**

```
发现：独立看门狗超时触发复位
原因：某个任务长时间未喂狗（喂狗代码在 IDLE 任务中）
推论：某个高优先级任务长时间阻塞，导致 IDLE 任务无法执行
```

**步骤3：增加 FreeRTOS 任务统计**

```c
/* 在串口菜单中添加任务状态输出 */
void print_task_stats(void)
{
    char buf[512];
    vTaskList(buf);
    printf("任务状态:\r\n%s\r\n", buf);

    vTaskGetRunTimeStats(buf);
    printf("CPU 使用率:\r\n%s\r\n", buf);
}
```

```
发现：协议处理任务（PROTOCOL_RX，优先级6）显示 Stack 水位只剩 8 words！
判断：极可能发生过栈溢出，但 FreeRTOS 默认不检查
```

**步骤4：启用栈溢出检测**

```c
/* FreeRTOSConfig.h */
#define configCHECK_FOR_STACK_OVERFLOW  2  // 启用栈溢出检查

/* 实现钩子函数 */
void vApplicationStackOverflowHook(TaskHandle_t xTask, char *pcTaskName)
{
    printf("!!! 栈溢出: 任务 %s\r\n", pcTaskName);
    /* 记录到 Flash */
    save_fault_record(FAULT_STACK_OVERFLOW, pcTaskName);
    while (1);  // 停机，让看门狗重启
}
```

**步骤5：复现并确认**

```
测试：将 PROTOCOL_RX 任务栈故意缩小到确认溢出
结果：触发 vApplicationStackOverflowHook，打印 "栈溢出: PROTOCOL_RX"
确认：问题确实是该任务栈溢出
```

**步骤6：分析为什么栈溢出**

```
检查：PROTOCOL_RX 任务原来栈大小 256 words
发现：任务中调用了 ocpp_parse_json()，该函数内使用了 char json_buf[512]
计算：512 字节局部变量 + 函数调用链 + FreeRTOS 上下文 >> 256 words (1024 bytes)
```

### 根本原因

`PROTOCOL_RX` 任务栈大小（256 words = 1024 bytes）不足以容纳 `ocpp_parse_json()` 中的 512 字节局部 JSON 缓冲区 + 函数调用链的栈空间需求。

之所以之前未发现，是因为只在收到较长 JSON 报文时才触发（如 MeterValues 报文体积较大）。

### 解决方案

**短期修复：**

```c
/* 增大 PROTOCOL_RX 任务栈 */
xTaskCreate(task_protocol_rx, "PROTO_RX",
            512,   /* 从 256 改为 512 words */
            NULL, TASK_PRIO_PROTOCOL_RX, NULL);
```

**长期防范：**

```c
/* 1. 将大缓冲区从局部变量改为静态变量（移到堆上，不占栈） */
static char s_json_parse_buf[512];  /* 文件作用域静态变量 */

/* 2. 所有任务加入栈水位监控 */
void check_all_task_stacks(void)
{
    TaskStatus_t tasks[16];
    uint32_t count = uxTaskGetSystemState(tasks, 16, NULL);
    for (uint32_t i = 0; i < count; i++) {
        if (tasks[i].usStackHighWaterMark < 32) {  // 少于 32 words 告警
            log_warn("Task '%s' low stack: %u words remaining",
                     tasks[i].pcTaskName,
                     tasks[i].usStackHighWaterMark);
        }
    }
}

/* 3. 启用 configCHECK_FOR_STACK_OVERFLOW = 2（永久保留） */
```

### 验证结果

- 修改后压力测试 48 小时无重启
- 栈水位检测：PROTO_RX 任务水位最低 87 words，安全
- 部署到现场 2 周，无异常重启工单

### 经验教训

1. **从一开始就开启 `configCHECK_FOR_STACK_OVERFLOW`**，不要等出了问题才加
2. **大局部数组是栈溢出的主要来源**，超过 256 字节的缓冲区考虑改成静态变量
3. **任务栈的估算**：用工具（`uxTaskGetStackHighWaterMark`）实测，而不是靠感觉
4. **加入定期栈水位检查**，在日志中打印预警

---

## 📌 示例2：OCPP WebSocket 连接频繁断开

```
日期：2026-03-28
严重程度：中（影响远程控制，但本地充电不受影响）
```

### 问题描述

新部署的充电桩在现场网络环境下，WebSocket 连接约每 5-10 分钟断开一次，自动重连后短时间内又断开，形成循环。实验室测试正常。

### 排查过程

**步骤1：对比现场与实验室的区别**

```
现场：4G 移动网络（NAT 设备，CGNAT）
实验室：有线以太网（直连）
差异：4G 网络存在 NAT 超时机制（通常 5-10 分钟无数据则断开 TCP 连接）
```

**步骤2：抓包分析**

```
工具：在测试充电桩上用 tcpdump 抓包
发现：断开前没有 FIN/RST 报文，TCP 连接直接消失（NAT 超时）
服务端：返回了 RST，因为它认为连接还活着但客户端不响应
```

**步骤3：检查心跳配置**

```
OCPP 配置：HeartbeatInterval = 300 秒（5分钟）
NAT 超时：约 5-7 分钟（运营商不同）
问题：心跳间隔接近 NAT 超时，在边界情况下会触发
```

### 根本原因

4G CGNAT 的 TCP 连接超时约 5-7 分钟，而 OCPP 心跳间隔配置为 300 秒（5分钟），在 NAT 超时窗口边界容易触发连接被路由器丢弃。

### 解决方案

```c
/* 方案一：缩短 OCPP 心跳间隔（向 CSMS 请求更新） */
/* 建议：在移动网络环境下设置为 60-120 秒 */

/* 方案二：在 TCP 层启用 keepalive（更根本） */
int enable_tcp_keepalive(int sockfd)
{
    int optval = 1;
    int idle    = 60;   /* 60秒无数据后开始探测 */
    int interval = 10;  /* 探测间隔 10 秒 */
    int count   = 3;    /* 最多 3 次探测 */

    setsockopt(sockfd, SOL_SOCKET,  SO_KEEPALIVE,  &optval, sizeof(int));
    setsockopt(sockfd, IPPROTO_TCP, TCP_KEEPIDLE,  &idle,   sizeof(int));
    setsockopt(sockfd, IPPROTO_TCP, TCP_KEEPINTVL, &interval,sizeof(int));
    setsockopt(sockfd, IPPROTO_TCP, TCP_KEEPCNT,   &count,  sizeof(int));
    return 0;
}
```

### 验证结果

- 启用 TCP keepalive 后，同一 4G 网络环境下测试 72 小时，连接稳定
- 同时建议 CSMS 后台将该设备的心跳间隔配置为 60 秒

### 经验教训

1. **实验室测试不能替代现场网络环境测试**，尤其是 4G/NAT 场景
2. **TCP keepalive 是长连接的标配**，应该在开发时就加上
3. **OCPP 心跳间隔要考虑网络环境**，不能只用默认值

---

## 📝 排查日志空白模板

```markdown
### 问题：[简短描述]

**日期**：YYYY-MM-DD  
**严重程度**：高/中/低  

#### 问题描述
（现象，何时发生，频率）

#### 影响范围
（影响哪些功能/用户）

#### 排查过程
**步骤1：**
- 操作：
- 发现：

**步骤2：**
- 操作：
- 发现：

#### 根本原因
（Root Cause）

#### 解决方案
**短期**：
**长期**：

#### 验证结果

#### 经验教训
1. 
```
