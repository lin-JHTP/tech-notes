# Python 机器学习入门实践

> **文章类型**：实践指南 + 代码示例  
> **适用读者**：有 C/Python 基础，想用机器学习解决实际问题的工程师  
> **最后更新**：2026-04  
> **标签**：`Python` `机器学习` `Sklearn` `NumPy` `Pandas` `数据分析`

---

## 目录

1. [环境搭建](#1-环境搭建)
2. [NumPy 核心操作速查](#2-numpy-核心操作速查)
3. [Pandas 数据处理速查](#3-pandas-数据处理速查)
4. [实战：充电桩故障分类器](#4-实战充电桩故障分类器)
5. [模型评估与选择](#5-模型评估与选择)
6. [常见问题 Q&A](#6-常见问题-qa)
7. [TODO](#7-todo)

---

## 1. 环境搭建

### 推荐：用 conda 管理环境

```bash
# 安装 Miniconda（轻量版 Anaconda）
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash Miniconda3-latest-Linux-x86_64.sh

# 创建专用 Python 环境（避免污染全局）
conda create -n ml python=3.11
conda activate ml

# 安装核心包
pip install numpy pandas matplotlib scikit-learn jupyter
pip install torch torchvision  # 深度学习框架
pip install xgboost lightgbm   # 梯度提升模型

# 验证安装
python -c "import sklearn; print(sklearn.__version__)"
```

### 或者直接用 Google Colab（无需本地配置）

```
访问 colab.research.google.com
新建笔记本，所有常用包已预装
提供免费 GPU（T4）
```

---

## 2. NumPy 核心操作速查

```python
import numpy as np

# ── 创建数组 ──────────────────────────────────────────────
a = np.array([1, 2, 3, 4, 5])          # 一维数组
b = np.array([[1, 2], [3, 4]])          # 二维数组（矩阵）
c = np.zeros((3, 4))                    # 3×4 全零矩阵
d = np.ones((2, 3))                     # 2×3 全一矩阵
e = np.arange(0, 10, 0.5)              # [0, 0.5, 1.0, ..., 9.5]
f = np.linspace(0, 1, 100)             # 0到1均匀分布100个点
r = np.random.randn(100, 5)            # 100×5 标准正态分布随机数

# ── 数组属性 ──────────────────────────────────────────────
print(r.shape)          # (100, 5)
print(r.dtype)          # float64
print(r.ndim)           # 2（维度数）
print(r.size)           # 500（总元素数）

# ── 基础运算（逐元素）────────────────────────────────────
a + 1           # [2, 3, 4, 5, 6]
a * 2           # [2, 4, 6, 8, 10]
a ** 2          # [1, 4, 9, 16, 25]
np.sqrt(a)      # [1.0, 1.41, 1.73, 2.0, 2.24]
np.exp(a)       # e 的幂次

# ── 矩阵运算 ─────────────────────────────────────────────
A = np.array([[1, 2], [3, 4]])
B = np.array([[5, 6], [7, 8]])
A @ B           # 矩阵乘法（等价于 np.matmul(A, B)）
A.T             # 转置
np.linalg.inv(A)   # 逆矩阵
np.linalg.det(A)   # 行列式

# ── 统计函数 ─────────────────────────────────────────────
r.mean(axis=0)  # 按列求均值（每个特征的均值）
r.std(axis=0)   # 按列求标准差
r.min(), r.max()
np.percentile(r, 75)  # 75% 分位数

# ── 索引与切片 ───────────────────────────────────────────
a[0]            # 第一个元素
a[-1]           # 最后一个元素
a[1:4]          # [2, 3, 4]（左闭右开）
b[0, :]         # 第一行
b[:, 1]         # 第二列

# 布尔索引（非常常用！）
mask = a > 2
a[mask]         # [3, 4, 5]，等价于 a[a > 2]
```

---

## 3. Pandas 数据处理速查

```python
import pandas as pd

# ── 读取数据 ─────────────────────────────────────────────
df = pd.read_csv('charging_log.csv')
df = pd.read_excel('report.xlsx', sheet_name='Sheet1')
df = pd.read_json('data.json')

# ── 数据探索 ─────────────────────────────────────────────
df.head(10)         # 前10行
df.tail(5)          # 后5行
df.shape            # (行数, 列数)
df.columns          # 列名
df.dtypes           # 每列数据类型
df.describe()       # 数值列的统计摘要（均值/标准差/分位数）
df.info()           # 各列信息和非空数量
df.isnull().sum()   # 每列空值数量

# ── 选择数据 ─────────────────────────────────────────────
df['voltage']           # 选择单列（返回 Series）
df[['voltage', 'current']]  # 选择多列（返回 DataFrame）
df.loc[0:5, 'voltage']  # 按标签索引
df.iloc[0:5, 2]         # 按位置索引
df[df['status'] == 'Fault']  # 条件筛选
df.query("voltage > 220 and current < 32")  # 字符串条件

# ── 数据清洗 ─────────────────────────────────────────────
df.dropna()                         # 删除含 NaN 的行
df.fillna(df.mean())                # 用均值填充 NaN
df.drop_duplicates()                # 删除重复行
df['timestamp'] = pd.to_datetime(df['timestamp'])  # 类型转换
df['voltage'] = df['voltage'].astype(float)

# ── 数据转换 ─────────────────────────────────────────────
df['power_w'] = df['voltage'] * df['current']      # 新增计算列
df.rename(columns={'v': 'voltage', 'i': 'current'})
df.sort_values('timestamp', ascending=True)
df.groupby('connector_id')['energy_kwh'].sum()     # 分组聚合

# ── 保存数据 ─────────────────────────────────────────────
df.to_csv('processed.csv', index=False)
df.to_excel('report.xlsx', index=False)
```

---

## 4. 实战：充电桩故障分类器

### 4.1 数据说明

假设我们有充电桩运行数据，需要分类识别以下故障类型：

| 标签 | 故障类型 |
|------|----------|
| 0 | 正常 |
| 1 | 过温 |
| 2 | 过流 |
| 3 | 过压 |
| 4 | 绝缘故障 |

### 4.2 特征工程

```python
import pandas as pd
import numpy as np
from sklearn.model_selection import train_test_split
from sklearn.preprocessing import StandardScaler
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import classification_report, confusion_matrix
import matplotlib.pyplot as plt
import seaborn as sns

# 加载数据
df = pd.read_csv('charger_data.csv')

# ── 特征工程 ────────────────────────────────────────────
# 计算派生特征
df['power_w']         = df['voltage_v'] * df['current_a']
df['temp_delta']      = df['temp_c'] - df['ambient_temp_c']
df['current_ratio']   = df['current_a'] / df['rated_current_a']
df['voltage_deviation'] = abs(df['voltage_v'] - 220.0) / 220.0

# 时序特征（滑动窗口统计）
df['current_mean_5m'] = df['current_a'].rolling(window=10, min_periods=1).mean()
df['current_std_5m']  = df['current_a'].rolling(window=10, min_periods=1).std().fillna(0)

# 选择特征列
feature_cols = [
    'voltage_v', 'current_a', 'temp_c', 'power_w',
    'temp_delta', 'current_ratio', 'voltage_deviation',
    'current_mean_5m', 'current_std_5m',
    'insulation_kohm', 'cp_voltage_v'
]
target_col = 'fault_label'

X = df[feature_cols].values
y = df[target_col].values

print(f"数据集大小: {X.shape}")
print(f"各类别分布:\n{pd.Series(y).value_counts().sort_index()}")
```

### 4.3 模型训练

```python
# ── 数据分割 ────────────────────────────────────────────
X_train, X_test, y_train, y_test = train_test_split(
    X, y,
    test_size=0.2,
    random_state=42,
    stratify=y        # 保持各类别比例
)

# ── 特征标准化 ──────────────────────────────────────────
scaler = StandardScaler()
X_train_scaled = scaler.fit_transform(X_train)
X_test_scaled  = scaler.transform(X_test)  # 只 transform，不 fit！

# ── 训练随机森林 ─────────────────────────────────────────
model = RandomForestClassifier(
    n_estimators=200,
    max_depth=10,
    min_samples_leaf=5,
    class_weight='balanced',  # 处理类别不平衡
    random_state=42,
    n_jobs=-1  # 使用所有 CPU 核心
)
model.fit(X_train_scaled, y_train)
print(f"训练集准确率: {model.score(X_train_scaled, y_train):.4f}")
print(f"测试集准确率: {model.score(X_test_scaled, y_test):.4f}")
```

### 4.4 评估与可视化

```python
# ── 分类报告 ────────────────────────────────────────────
y_pred = model.predict(X_test_scaled)
labels = ['正常', '过温', '过流', '过压', '绝缘故障']
print(classification_report(y_test, y_pred, target_names=labels))

# ── 混淆矩阵 ────────────────────────────────────────────
cm = confusion_matrix(y_test, y_pred)
plt.figure(figsize=(8, 6))
sns.heatmap(cm, annot=True, fmt='d', cmap='Blues',
            xticklabels=labels, yticklabels=labels)
plt.title('混淆矩阵')
plt.ylabel('真实标签')
plt.xlabel('预测标签')
plt.tight_layout()
plt.savefig('confusion_matrix.png', dpi=150)

# ── 特征重要性 ───────────────────────────────────────────
importances = pd.Series(model.feature_importances_, index=feature_cols)
importances.sort_values().plot(kind='barh', figsize=(8, 6))
plt.title('特征重要性')
plt.tight_layout()
plt.savefig('feature_importance.png', dpi=150)

# ── 保存模型 ─────────────────────────────────────────────
import joblib
joblib.dump(model, 'fault_classifier.pkl')
joblib.dump(scaler, 'scaler.pkl')
print("模型已保存")
```

### 4.5 在生产中使用模型

```python
# 加载模型并进行在线预测
model  = joblib.load('fault_classifier.pkl')
scaler = joblib.load('scaler.pkl')

def predict_fault(sensor_data: dict) -> str:
    """
    输入：传感器数据字典
    输出：故障类型字符串
    """
    features = [
        sensor_data['voltage_v'],
        sensor_data['current_a'],
        # ... 其他特征
    ]
    X = np.array(features).reshape(1, -1)
    X_scaled = scaler.transform(X)
    label_id = model.predict(X_scaled)[0]
    proba = model.predict_proba(X_scaled)[0]

    labels = ['正常', '过温', '过流', '过压', '绝缘故障']
    confidence = proba[label_id]
    return f"{labels[label_id]} (置信度: {confidence:.1%})"
```

---

## 5. 模型评估与选择

### 评估指标速查

| 指标 | 公式 | 适用场景 |
|------|------|----------|
| 准确率（Accuracy） | 正确数 / 总数 | 类别均衡时 |
| 精确率（Precision） | TP / (TP+FP) | 误报代价高时（如安全告警） |
| 召回率（Recall） | TP / (TP+FN) | 漏报代价高时（如故障检测） |
| F1 分数 | 2 × P×R/(P+R) | 类别不均衡的综合指标 |
| AUC-ROC | ROC曲线下面积 | 二分类综合评估 |

**充电桩故障检测场景建议**：优先优化**召回率**（不漏报），可以接受一定的误报率。

### 交叉验证

```python
from sklearn.model_selection import cross_val_score

# 5折交叉验证（防止过拟合）
scores = cross_val_score(model, X_scaled, y,
                          cv=5, scoring='f1_weighted')
print(f"5折交叉验证 F1: {scores.mean():.4f} ± {scores.std():.4f}")
```

---

## 6. 常见问题 Q&A

**Q：数据量太少怎么办？**
> A：① 数据增强（加入合理的噪声）；② 迁移学习；③ 用更简单的模型（避免过拟合）；④ 收集更多数据（根本方法）。

**Q：类别不平衡怎么处理？**
> A：① `class_weight='balanced'`；② 过采样（SMOTE）；③ 欠采样；④ 调整决策阈值。

**Q：模型在训练集好但测试集差？**
> A：过拟合。方法：增加数据量、减少模型复杂度、增加正则化、使用交叉验证。

**Q：特征太多怎么办？**
> A：① 随机森林的特征重要性筛选；② PCA 降维；③ 相关性分析去除高度相关特征。

---

## 7. TODO

- [ ] 时序异常检测（Isolation Forest 在充电桩场景的应用）
- [ ] 模型导出为 ONNX 格式的详细流程
- [ ] 在树莓派/Jetson 上部署 sklearn 模型
- [ ] 超参数调优（GridSearchCV/Optuna）实践
- [ ] AutoML 工具（auto-sklearn）入门体验
