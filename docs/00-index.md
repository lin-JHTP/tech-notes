# 📖 知识地图 · Tech Notes

> 个人技术知识库的全局导航与学习路径索引，帮助你快速定位所需知识，规划学习路线。

---

## 🗺️ 整体知识地图

```
Tech Notes 知识库
│
├── 01 嵌入式开发
│   ├── 硬件基础（时钟/GPIO/中断/电源）
│   ├── 通信外设（UART/SPI/I2C/CAN/USB）
│   ├── 高级特性（DMA/Timer/PWM/ADC）
│   └── RTOS（FreeRTOS 任务/同步/内存）
│
├── 02 充电桩技术
│   ├── 通信协议层（OCPP 1.6 / 2.0.1 / GB/T 27930）
│   ├── 业务逻辑层（认证/会话/计量/计费）
│   ├── 安全防护层（电气安全/通信安全/审计）
│   └── 云平台层（设备接入/OTA/运营数据）
│
├── 03 CS 基础
│   ├── 操作系统（进程/线程/调度/内存/文件系统）
│   ├── 计算机网络（TCP/IP/HTTP/DNS/安全）
│   └── 数据结构与算法（线性结构/树/图/排序/动态规划）
│
├── 04 AI 学习
│   ├── 数学基础（线代/概率/微积分）
│   ├── 机器学习（监督/无监督/模型评估）
│   ├── 深度学习（神经网络/CNN/RNN/Transformer）
│   └── 边缘 AI 部署（ONNX/TFLite/MCU 推理）
│
├── 05 效率工具
│   ├── 版本控制（Git 高级操作/工作流）
│   ├── 编辑器（VSCode 配置/插件/调试）
│   ├── 终端与 Shell（Bash/Zsh/tmux/脚本）
│   └── 构建系统（Makefile/CMake/Python 脚本）
│
└── 99 每日笔记
    ├── 每日学习记录
    ├── 问题排查日志
    └── 周报与复盘
```

---

## 🚀 快速起步

### 我是嵌入式开发新手
1. 阅读 [嵌入式开发板块简介](01-embedded/index.md)
2. 跟着 [GD32入门与工程实践](01-embedded/gd32-getting-started.md) 搭建开发环境
3. 掌握基础后，学习 [串口与DMA调试经验](01-embedded/uart-dma-debug.md)
4. 进阶：[FreeRTOS实战笔记](01-embedded/freertos-practical-notes.md)

### 我想了解充电桩行业技术
1. 先看 [充电桩技术板块简介](02-charging-pile/index.md)
2. 深入 [OCPP协议深度解析](02-charging-pile/ocpp-protocol-deep-dive.md)
3. 了解安全：[充电桩安全业务设计](02-charging-pile/charging-safety-design.md)
4. 标准对比：[GB/T标准对比笔记](02-charging-pile/gbt-standard-notes.md)

### 我需要查漏补缺 CS 基础
1. 浏览 [CS基础板块简介](03-cs-fundamentals/index.md)
2. 重点复习 [计算机网络精华笔记](03-cs-fundamentals/network-essentials.md)
3. 刷题必备 [数据结构与算法精华](03-cs-fundamentals/data-structures-algorithms.md)
4. 深入原理 [操作系统核心概念速查](03-cs-fundamentals/os-core-concepts.md)

### 我想入门 AI / 机器学习
1. 看 [AI学习路线与资源导航](04-ai-learning/ai-learning-roadmap.md) 规划学习
2. 动手实践 [Python机器学习入门实践](04-ai-learning/python-ml-intro.md)
3. 进阶方向 [边缘AI与嵌入式部署笔记](04-ai-learning/edge-ai-deployment.md)

### 我想提升开发效率
1. [Git高级用法与踩坑记录](05-tools/git-advanced-tips.md)
2. [VSCode嵌入式开发配置全攻略](05-tools/vscode-embedded-setup.md)
3. [Linux常用命令与Shell技巧](05-tools/linux-shell-tips.md)

---

## 📊 内容板块一览

| 板块 | 文章数 | 核心主题 | 适合人群 |
|------|--------|----------|----------|
| [嵌入式开发](01-embedded/index.md) | 4篇 | GD32、FreeRTOS、外设驱动 | 嵌入式工程师、硬件爱好者 |
| [充电桩技术](02-charging-pile/index.md) | 3篇 | OCPP、GB/T、安全业务 | 新能源行业工程师 |
| [CS基础](03-cs-fundamentals/index.md) | 3篇 | OS、网络、算法 | 备战面试、基础薄弱者 |
| [AI学习](04-ai-learning/index.md) | 3篇 | ML、深度学习、边缘AI | AI 学习者、嵌入式AI探索者 |
| [效率工具](05-tools/index.md) | 3篇 | Git、VSCode、Linux | 所有开发者 |
| [每日笔记](99-daily/index.md) | 3篇 | 日志、排查、复盘 | 有记录习惯的工程师 |

---

## 💡 如何使用这份知识库

1. **通过知识地图定位**：本页地图帮助你快速找到目标板块
2. **按路径深入学习**：每个板块都有推荐学习顺序
3. **随时查阅参考**：文章按主题分类，可按需检索
4. **参与扩展内容**：每个板块有 TODO 列表，欢迎贡献

---

## 🔗 相关链接

- [仓库主页](https://github.com/lin-JHTP/tech-notes)
- [提交 Issue](https://github.com/lin-JHTP/tech-notes/issues)
- [网站主页](index.md)

---

!!! note "持续更新"
    本知识库持续更新中。如发现内容错误或想补充，欢迎提 Issue 或 PR。
