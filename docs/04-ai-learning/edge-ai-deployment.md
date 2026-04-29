# 边缘 AI 与嵌入式部署笔记

> **文章类型**：实践指南 + 技术笔记  
> **适用读者**：想在 MCU/嵌入式 Linux 上部署 AI 模型的工程师  
> **最后更新**：2026-04  
> **标签**：`边缘AI` `TFLite` `ONNX` `嵌入式部署` `模型压缩` `MCU推理`

---

## 目录

1. [边缘 AI 概述](#1-边缘-ai-概述)
2. [模型压缩技术](#2-模型压缩技术)
3. [TensorFlow Lite 部署流程](#3-tensorflow-lite-部署流程)
4. [ONNX Runtime 部署（嵌入式 Linux）](#4-onnx-runtime-部署嵌入式-linux)
5. [TFLite Micro（MCU 无 OS 部署）](#5-tflite-microμc-无-os-部署)
6. [性能优化技巧](#6-性能优化技巧)
7. [实战：充电桩异常检测 MCU 部署](#7-实战充电桩异常检测-mcu-部署)
8. [TODO](#8-todo)

---

## 1. 边缘 AI 概述

### 云端推理 vs 边缘推理

| 对比项 | 云端推理 | 边缘推理 |
|--------|----------|----------|
| 延迟 | 高（网络 RTT） | 低（本地执行） |
| 带宽 | 需要上传数据 | 无需持续联网 |
| 隐私 | 数据离开设备 | 数据留在本地 |
| 算力 | 无限（可扩展） | 受限（MCU/NPU） |
| 功耗 | 不关心 | 极其重要 |
| 成本 | 按量付费 | 一次性硬件成本 |

### 嵌入式 AI 硬件档次

```
算力等级（TOPS/W）
         低              ←→                  高
┌─────────────────────────────────────────────────┐
│ MCU            │ 嵌入式 Linux  │ 边缘 AI 芯片   │
│ (Cortex-M)     │ (Cortex-A)    │ (NPU)          │
│ STM32H7        │ 树莓派 4      │ 瑞芯微 RK3588  │
│ GD32F4         │ Jetson Nano   │ 地平线 RDK X5  │
│ 支持: TFLite   │ 支持: ONNX/  │ 支持: 专用SDK  │
│ Micro, CMSIS-  │ TFLite/PyTch  │ MNN/RKNN-Toolkit│
│ NN             │ +ARM NN       │                │
└─────────────────────────────────────────────────┘
典型应用：        │               │               
关键词/触发词     │ 人脸识别      │ 实时目标检测   
简单异常检测      │ 语音唤醒      │ 视频分析       
传感器数据分类    │ 复杂分类      │ 多任务并行推理  
```

---

## 2. 模型压缩技术

### 量化（Quantization）— 最常用

将模型权重/激活值从 float32 降低精度：

| 量化类型 | 原精度 | 目标精度 | 大小压缩 | 精度损失 |
|---------|--------|---------|---------|---------|
| FP32 → FP16 | 32位浮点 | 16位浮点 | ×0.5 | 极小 |
| FP32 → INT8 | 32位浮点 | 8位整数 | ×0.25 | 小 |
| FP32 → INT4 | 32位浮点 | 4位整数 | ×0.125 | 中等 |
| 二值化 | 32位浮点 | 1位 | ×0.03 | 较大 |

```python
# TFLite INT8 量化示例
import tensorflow as tf
import numpy as np

def representative_dataset():
    """代表性数据集，用于校准量化参数（取100个真实样本）"""
    for i in range(100):
        yield [calibration_data[i:i+1].astype(np.float32)]

converter = tf.lite.TFLiteConverter.from_saved_model('my_model')

# 全整数量化（模型+输入/输出都量化为 INT8）
converter.optimizations = [tf.lite.Optimize.DEFAULT]
converter.representative_dataset = representative_dataset
converter.target_spec.supported_ops = [tf.lite.OpsSet.TFLITE_BUILTINS_INT8]
converter.inference_input_type = tf.int8
converter.inference_output_type = tf.int8

tflite_model = converter.convert()
with open('model_int8.tflite', 'wb') as f:
    f.write(tflite_model)

print(f"FP32 模型大小: {os.path.getsize('my_model') // 1024} KB")
print(f"INT8 模型大小: {len(tflite_model) // 1024} KB")
```

### 剪枝（Pruning）

去除不重要的神经网络连接（权重置零）：

```python
import tensorflow_model_optimization as tfmot

# 全局非结构化剪枝（50% 权重置零）
pruning_params = {
    'pruning_schedule': tfmot.sparsity.keras.PolynomialDecay(
        initial_sparsity=0.0,
        final_sparsity=0.5,  # 目标稀疏度 50%
        begin_step=0,
        end_step=1000
    )
}
model_for_pruning = tfmot.sparsity.keras.prune_low_magnitude(
    model, **pruning_params
)
```

### 知识蒸馏（Knowledge Distillation）

用大模型（教师）指导小模型（学生）训练，小模型学习大模型的"软标签"：

```
教师模型（大，精度高）
    │ 输出软标签（概率分布）
    ▼
学生模型（小，部署用）
    └── 同时学习：真实标签 + 教师软标签
```

---

## 3. TensorFlow Lite 部署流程

### 完整流程

```
1. 训练 TF/Keras 模型
    ↓
2. 转换为 TFLite 格式（.tflite）
    ↓
3. 可选量化（INT8/FP16）
    ↓
4. 在目标平台运行推理
   ├── ARM Linux：TFLite C++ API 或 Python
   └── MCU：TFLite Micro
```

### Python 推理验证

```python
import numpy as np
import tensorflow as tf

# 加载模型
interpreter = tf.lite.Interpreter(model_path='model_int8.tflite')
interpreter.allocate_tensors()

# 获取输入/输出信息
input_details  = interpreter.get_input_details()
output_details = interpreter.get_output_details()

print("输入信息:")
print(f"  形状: {input_details[0]['shape']}")
print(f"  类型: {input_details[0]['dtype']}")
print(f"  量化参数: {input_details[0]['quantization']}")

# 准备输入数据（INT8 量化模型）
input_scale, input_zero_point = input_details[0]['quantization']
input_float = sensor_data.astype(np.float32)
input_int8 = (input_float / input_scale + input_zero_point).astype(np.int8)

# 设置输入并运行推理
interpreter.set_tensor(input_details[0]['index'], input_int8)
interpreter.invoke()

# 获取输出并反量化
output_int8 = interpreter.get_tensor(output_details[0]['index'])
output_scale, output_zero_point = output_details[0]['quantization']
output_float = (output_int8.astype(np.float32) - output_zero_point) * output_scale

predicted_class = np.argmax(output_float)
print(f"预测结果: {predicted_class}，置信度: {output_float[0, predicted_class]:.2f}")
```

---

## 4. ONNX Runtime 部署（嵌入式 Linux）

ONNX 是通用的模型交换格式，支持 PyTorch/TF 等主流框架导出。

### 模型导出（PyTorch → ONNX）

```python
import torch
import onnx

model.eval()
dummy_input = torch.zeros(1, 11)  # batch_size=1, features=11

torch.onnx.export(
    model,
    dummy_input,
    'fault_classifier.onnx',
    opset_version=13,
    input_names=['input'],
    output_names=['output'],
    dynamic_axes={'input': {0: 'batch_size'}}  # 动态 batch
)

# 验证 ONNX 模型
onnx_model = onnx.load('fault_classifier.onnx')
onnx.checker.check_model(onnx_model)
print("ONNX 模型验证通过")
```

### 在嵌入式 Linux（ARM）上运行

```bash
# 在 ARM Linux 上安装 ONNX Runtime
pip install onnxruntime  # CPU 版本（支持 ARM）
# 或下载预编译包：https://github.com/microsoft/onnxruntime/releases
```

```python
import onnxruntime as ort
import numpy as np

# 创建推理会话
sess_options = ort.SessionOptions()
sess_options.intra_op_num_threads = 2  # 限制线程数（嵌入式场景）
session = ort.InferenceSession('fault_classifier.onnx', sess_options)

input_name = session.get_inputs()[0].name
output_name = session.get_outputs()[0].name

# 运行推理
input_data = np.array([[220.0, 25.0, 45.0, ...]], dtype=np.float32)
result = session.run([output_name], {input_name: input_data})
prediction = np.argmax(result[0])
```

---

## 5. TFLite Micro（MCU 无 OS 部署）

TFLite Micro（TFLM）可以在没有操作系统的 MCU 上运行（如 GD32/STM32）。

### 支持的运算（常见子集）

- 全连接（Dense）/ 卷积（Conv2D/DepthwiseConv2D）
- ReLU / Sigmoid / Softmax
- Pooling / Reshape / Concatenate
- **不支持**：某些动态形状操作、复杂 Transformer 算子

### 集成到 GD32 项目

```c
/* tflite_micro_inference.cc（C++ 文件）*/
#include "tensorflow/lite/micro/all_ops_resolver.h"
#include "tensorflow/lite/micro/micro_interpreter.h"
#include "tensorflow/lite/schema/schema_generated.h"

/* 模型数据（通过 xxd -i model.tflite > model_data.h 生成）*/
#include "fault_classifier_model_data.h"

/* 分配给 TFLM 的内存（根据模型大小调整）*/
constexpr int kTensorArenaSize = 32 * 1024;  /* 32 KB */
static uint8_t tensor_arena[kTensorArenaSize];

static tflite::MicroInterpreter *interpreter = nullptr;

extern "C" void tflm_init(void)
{
    const tflite::Model *model =
        tflite::GetModel(g_fault_classifier_model_data);

    static tflite::AllOpsResolver resolver;
    static tflite::MicroInterpreter static_interpreter(
        model, resolver, tensor_arena, kTensorArenaSize);
    interpreter = &static_interpreter;

    TfLiteStatus status = interpreter->AllocateTensors();
    if (status != kTfLiteOk) {
        /* 内存不足，需要增大 tensor_arena */
        Error_Handler();
    }
}

extern "C" int tflm_predict_fault(const float *features, int num_features)
{
    /* 填充输入张量 */
    TfLiteTensor *input = interpreter->input(0);
    for (int i = 0; i < num_features; i++) {
        /* INT8 量化：需要先量化 */
        input->data.int8[i] = (int8_t)(features[i] / input->params.scale
                                       + input->params.zero_point);
    }

    /* 运行推理 */
    TfLiteStatus invoke_status = interpreter->Invoke();
    if (invoke_status != kTfLiteOk) {
        return -1;
    }

    /* 获取输出 */
    TfLiteTensor *output = interpreter->output(0);
    int max_idx = 0;
    int8_t max_val = output->data.int8[0];
    for (int i = 1; i < output->dims->data[1]; i++) {
        if (output->data.int8[i] > max_val) {
            max_val = output->data.int8[i];
            max_idx = i;
        }
    }
    return max_idx;
}
```

---

## 6. 性能优化技巧

| 优化手段 | 效果 | 适用平台 |
|---------|------|---------|
| INT8 量化 | 速度提升 2-4×，大小缩减 4× | 所有平台 |
| CMSIS-NN | 利用 ARM 专用指令，速度提升 5-10× | Cortex-M4/M7 |
| 批量推理 | 提高 GPU/NPU 利用率 | 有加速器的平台 |
| 算子融合 | 减少内存访问 | 编译器自动优化 |
| 模型剪枝 | 减少计算量 | 所有平台 |
| 使用 NPU | 专用加速，100× 以上提升 | 有 NPU 的 SoC |

---

## 7. 实战：充电桩异常检测 MCU 部署

### 目标

在 GD32F407 上部署一个小型异常检测模型，实时检测充电过程异常。

### 约束

- RAM：320 KB（tensor_arena ≤ 64 KB）
- Flash：1 MB（模型 ≤ 100 KB）
- 推理延迟：< 100ms（每次采样后推理）

### 模型设计（轻量 MLP）

```python
import tensorflow as tf

model = tf.keras.Sequential([
    tf.keras.layers.Input(shape=(8,)),      # 8 个特征
    tf.keras.layers.Dense(32, activation='relu'),
    tf.keras.layers.Dense(16, activation='relu'),
    tf.keras.layers.Dense(4, activation='softmax')  # 4 个故障类别
])

# 参数量：8×32 + 32×16 + 16×4 = 256+512+64 = 832 参数
# INT8 量化后大小 ≈ 832 字节 + 模型开销 ≈ 4KB
```

### 部署验证流程

```
1. 在 PC 上训练并验证（准确率 > 90%）
2. 转换为 TFLite INT8
3. 用 TFLite Python 验证量化后精度损失可接受（< 2%）
4. 生成 C 数组文件（xxd -i）
5. 集成到 GD32 工程，在 RAM/Flash 约束内运行
6. 逻辑分析仪测量实际推理时间
7. 与人工阈值方法做对比验证
```

---

## 8. TODO

- [ ] CMSIS-NN 在 GD32F4 上的使能与性能对比
- [ ] 模型在线更新机制（OTA 推送新模型）
- [ ] 联邦学习入门（多个充电桩协同训练，不共享数据）
- [ ] 瑞芯微 RK3588 + RKNN-Toolkit2 部署实践
- [ ] Transformer 小模型在嵌入式 Linux 上的部署

---

*关键原则：**永远先在 PC/服务器上验证模型精度，再考虑优化和部署**。不要在还不知道模型有没有用的时候就开始做硬件适配。*
