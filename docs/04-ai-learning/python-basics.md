# Python 基础

> 专为 C 语言背景的嵌入式工程师设计的 Python 入门路线，对比 C 语言讲解，快速上手。

!!! tip "给 C 程序员的提示"
    Python 比 C 灵活很多：不用声明变量类型、不用手动管理内存、缩进代替大括号。
    放心大胆地用，Python 会帮你处理很多细节。

---

## 📌 学习路线（针对 AI 方向）

```
阶段 1：Python 语法基础（2 周）
├── 变量与基本数据类型
├── 控制流（if/for/while）
├── 函数定义与调用
├── 列表、字典、元组
└── 文件读写

阶段 2：常用库入门（2 周）
├── NumPy（数值计算，AI 必备）
├── Pandas（数据处理）
└── Matplotlib（数据可视化）

阶段 3：AI 框架入门（4 周）
├── scikit-learn（传统机器学习）
└── PyTorch（深度学习）
```

---

## 🔑 C vs Python 语法对比

=== "C 语言"
    ```c
    /* 变量声明必须指定类型 */
    int x = 10;
    float pi = 3.14;
    char name[] = "hello";

    /* for 循环 */
    for (int i = 0; i < 5; i++) {
        printf("%d\n", i);
    }
    ```

=== "Python"
    ```python
    # 变量类型自动推断
    x = 10
    pi = 3.14
    name = "hello"

    # for 循环更简洁
    for i in range(5):
        print(i)
    ```

---

## 🔑 Python 列表（类比 C 数组）

```python
# Python 列表比 C 数组强大很多
nums = [1, 2, 3, 4, 5]

# 追加元素（C 中需要手动管理内存）
nums.append(6)

# 切片（C 中需要手动计算偏移）
print(nums[1:3])   # 输出：[2, 3]

# 列表推导式（一行代码生成新列表）
squares = [x**2 for x in range(10)]
```

---

## 🔑 第一个 Python 程序：读取传感器数据文件

```python
# 读取 CSV 格式的传感器日志，计算平均值
import csv

values = []
with open("sensor_log.csv", "r") as f:
    reader = csv.reader(f)
    for row in reader:
        values.append(float(row[1]))  # 第二列为数值

avg = sum(values) / len(values)
print(f"平均值：{avg:.2f}")
```

---

## 📝 待补充内容

- 函数进阶（lambda、装饰器）
- 类与面向对象（OOP）
- 异常处理（try/except）
- NumPy 入门
- 推荐学习资源（中文）
