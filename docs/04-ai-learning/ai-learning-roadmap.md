# AI 学习路线与资源导航

> **文章类型**：学习路线图 + 资源推荐  
> **适用读者**：有一定编程基础（Python/C），想系统学习 AI/机器学习的工程师  
> **最后更新**：2026-04  
> **标签**：`AI` `机器学习` `深度学习` `学习路线` `资源推荐`

---

## 目录

1. [为什么嵌入式工程师要学 AI？](#1-为什么嵌入式工程师要学-ai)
2. [总体学习路线](#2-总体学习路线)
3. [第一阶段：数学基础](#3-第一阶段数学基础)
4. [第二阶段：Python 与数据处理](#4-第二阶段python-与数据处理)
5. [第三阶段：机器学习基础](#5-第三阶段机器学习基础)
6. [第四阶段：深度学习入门](#6-第四阶段深度学习入门)
7. [第五阶段：边缘 AI 部署](#7-第五阶段边缘-ai-部署)
8. [资源清单](#8-资源清单)
9. [TODO](#9-todo)

---

## 1. 为什么嵌入式工程师要学 AI？

AI 正在深入嵌入式领域，具体体现在：

| 应用场景 | AI 技术 | 价值 |
|---------|---------|------|
| 充电桩故障预测 | 时序异常检测 | 提前发现硬件劣化，减少宕机 |
| 充电电流优化 | 强化学习/预测控制 | 延长电池寿命，提高效率 |
| 语音交互充电桩 | 端侧语音识别（TFLite） | 提升用户体验 |
| 边缘图像识别 | CNN 量化部署 | 车牌识别、异常行为检测 |
| 工业设备状态监测 | 振动/电流特征分析 | 预测性维护 |

**嵌入式工程师的 AI 优势**：
- 理解底层硬件，知道模型推理的实际约束
- 能做端到端优化（从传感器采集到推理输出）
- 懂实时性要求，能合理评估边缘部署可行性

---

## 2. 总体学习路线

```
阶段一：数学基础（2-4 周）
    线性代数 → 概率统计 → 微积分基础

阶段二：Python 工具链（2-3 周）
    Python 语法 → NumPy → Pandas → Matplotlib

阶段三：机器学习基础（4-6 周）
    监督学习 → 无监督学习 → 模型评估 → Sklearn 实践

阶段四：深度学习（8-12 周）
    神经网络 → CNN → RNN/LSTM → Transformer 基础
    框架：PyTorch（推荐）或 TensorFlow

阶段五：边缘 AI 部署（4-6 周）
    模型压缩 → TFLite/ONNX → MCU 推理 → 性能优化
```

**学习原则：**

1. **边学边做**：每个阶段都要有动手项目，不能只看课程
2. **从应用出发**：先跑通充电桩故障检测这个具体项目，再回头补理论
3. **不要追求全面**：先把一个方向做深，而不是每个都浅尝
4. **记录学习笔记**：写下来才是真正学会了

---

## 3. 第一阶段：数学基础

### 线性代数（重点）

AI 最核心的数学，每天接触：

| 概念 | AI 中的用途 |
|------|------------|
| 矩阵乘法 | 神经网络前向传播 |
| 特征值/特征向量 | PCA 降维 |
| 行列式 | 理解矩阵奇异性 |
| 向量范数 | 正则化、相似度 |

**学习建议**：用 3Blue1Brown 的《线性代数的本质》视频系列入门（B站有中文版）。

### 概率与统计（重点）

| 概念 | AI 中的用途 |
|------|------------|
| 条件概率/贝叶斯 | 贝叶斯分类器、概率图模型 |
| 期望与方差 | 损失函数设计 |
| 正态分布 | 权重初始化、噪声建模 |
| 最大似然估计 | 模型参数学习 |

### 微积分基础（了解即可）

- 梯度（偏导数）：理解反向传播的本质
- 链式法则：理解多层神经网络的梯度计算

---

## 4. 第二阶段：Python 与数据处理

### 核心库速查

```python
import numpy as np          # 数值计算（向量/矩阵运算）
import pandas as pd         # 数据处理（DataFrame）
import matplotlib.pyplot as plt  # 数据可视化
from sklearn.xxx import xxx  # 机器学习算法
import torch                 # 深度学习框架
```

### 嵌入式数据处理示例

```python
# 场景：分析充电桩 CAN 日志文件，提取充电电流时序
import pandas as pd
import matplotlib.pyplot as plt

# 读取 CSV 格式的 CAN 日志
df = pd.read_csv('charge_log_20260401.csv',
                 names=['timestamp', 'can_id', 'data'],
                 sep=',')

# 筛选 BCS 报文（电池充电状态，CAN ID = 0x1882D6F4）
bcs_df = df[df['can_id'] == '0x1882D6F4'].copy()
bcs_df['timestamp'] = pd.to_datetime(bcs_df['timestamp'], unit='s')

# 提取充电电流字段（BCS 报文 Byte 2-3，单位 0.1A）
def extract_current(data_hex):
    data = bytes.fromhex(data_hex.replace(' ', ''))
    raw = int.from_bytes(data[2:4], 'little')
    return raw * 0.1  # 转换为 A

bcs_df['current_a'] = bcs_df['data'].apply(extract_current)

# 绘制充电电流曲线
plt.figure(figsize=(12, 4))
plt.plot(bcs_df['timestamp'], bcs_df['current_a'])
plt.xlabel('时间')
plt.ylabel('充电电流 (A)')
plt.title('充电电流时序曲线')
plt.grid(True)
plt.tight_layout()
plt.savefig('charging_current.png', dpi=150)
plt.show()
```

---

## 5. 第三阶段：机器学习基础

### 常见算法速查

| 算法 | 类型 | 适用场景 | Python 库 |
|------|------|----------|-----------|
| 线性回归 | 监督/回归 | 预测连续值（充电时间预测） | sklearn |
| 逻辑回归 | 监督/分类 | 二分类（故障/正常） | sklearn |
| 决策树/随机森林 | 监督 | 可解释性强的分类 | sklearn |
| SVM | 监督 | 小样本高维数据 | sklearn |
| K-Means | 无监督 | 聚类（用户行为分群） | sklearn |
| K近邻（KNN） | 监督 | 简单基线模型 | sklearn |
| XGBoost | 监督 | 结构化数据竞赛利器 | xgboost |

### 标准 ML 工作流

```python
from sklearn.model_selection import train_test_split
from sklearn.preprocessing import StandardScaler
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import classification_report

# 1. 加载数据
X, y = load_charger_fault_data()

# 2. 数据分割
X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42, stratify=y)

# 3. 特征缩放
scaler = StandardScaler()
X_train_scaled = scaler.fit_transform(X_train)
X_test_scaled  = scaler.transform(X_test)  # 注意：只用 transform，不 fit！

# 4. 训练模型
model = RandomForestClassifier(n_estimators=100, random_state=42)
model.fit(X_train_scaled, y_train)

# 5. 评估
y_pred = model.predict(X_test_scaled)
print(classification_report(y_test, y_pred,
                             target_names=['正常', '过温故障', '过流故障']))

# 6. 保存模型
import joblib
joblib.dump(model, 'fault_classifier.pkl')
joblib.dump(scaler, 'scaler.pkl')
```

---

## 6. 第四阶段：深度学习入门

### 为什么用 PyTorch？

- 动态图，调试方便（嵌入式工程师调试习惯）
- 社区最活跃，最新研究都先出 PyTorch 版本
- TorchScript 可以导出用于嵌入式部署

### 简单 MLP 分类器示例

```python
import torch
import torch.nn as nn
import torch.optim as optim

class FaultClassifier(nn.Module):
    def __init__(self, input_dim, num_classes):
        super().__init__()
        self.net = nn.Sequential(
            nn.Linear(input_dim, 128),
            nn.ReLU(),
            nn.Dropout(0.3),
            nn.Linear(128, 64),
            nn.ReLU(),
            nn.Linear(64, num_classes)
        )

    def forward(self, x):
        return self.net(x)

# 训练循环
model = FaultClassifier(input_dim=20, num_classes=5)
optimizer = optim.Adam(model.parameters(), lr=1e-3)
criterion = nn.CrossEntropyLoss()

for epoch in range(100):
    model.train()
    for X_batch, y_batch in train_loader:
        optimizer.zero_grad()
        output = model(X_batch)
        loss = criterion(output, y_batch)
        loss.backward()
        optimizer.step()
```

---

## 7. 第五阶段：边缘 AI 部署

参考：[边缘AI与嵌入式部署笔记](edge-ai-deployment.md)

**核心路线**：

```
PyTorch/TensorFlow 模型
    │
    ▼ 模型转换
ONNX 格式（通用中间格式）
    │
    ├── ONNX Runtime（Linux ARM 部署）
    └── TFLite（MCU/手机部署）
            │
            └── TFLite Micro（无 OS 的 MCU）
```

---

## 8. 资源清单

### 免费课程（强烈推荐）

| 资源 | 内容 | 链接 |
|------|------|------|
| 吴恩达 ML 专项课程 | 机器学习入门，中文字幕 | Coursera（可免费旁听） |
| 动手学深度学习（d2l.ai） | 深度学习中文教材，含代码 | d2l.ai 免费 |
| 3Blue1Brown 神经网络 | 可视化直觉讲解 | YouTube/B站 |
| fast.ai | 实用深度学习，自顶向下 | fast.ai 免费 |
| CS231n（Stanford） | 计算机视觉深度学习 | 官网/B站 |

### 必读书籍

| 书名 | 适合阶段 | 评价 |
|------|----------|------|
| 《动手学深度学习》（李沐） | 入门-中级 | 代码+理论兼备，强烈推荐 |
| 《机器学习》（周志华/西瓜书） | 系统学习 | 理论严谨，适合系统复习 |
| 《深度学习》（花书）| 深入研究 | 经典但难度大，先看西瓜书 |
| 《TinyML》 | 边缘 AI 部署 | 嵌入式 AI 必读 |

### 实践平台

- **Kaggle**：免费 GPU、丰富数据集和竞赛
- **Colab**：Google 免费 GPU/TPU
- **魔搭社区**（国内）：丰富的中文模型和数据集

---

## 9. TODO

- [ ] 建立充电桩故障检测实战项目笔记
- [ ] PyTorch 快速入门实践笔记
- [ ] Transformer/注意力机制原理可视化笔记
- [ ] 大模型 API 调用与 Prompt 工程入门
- [ ] 推荐系统基础（充电桩场景：个性化充电建议）
