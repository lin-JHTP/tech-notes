# 对话机器人基本原理

> 从原理上理解 ChatGPT 这类对话系统是怎么工作的，以及如何构建一个简单的对话应用。

## 📌 学习目标

- 理解 NLP 基础概念（词向量、Transformer）
- 了解大语言模型（LLM）的工作原理
- 能够使用 API 调用 LLM 构建简单对话应用
- 了解本地部署小模型的方案

---

## 🔑 NLP 发展历程（简史）

```
1990s   词袋模型（Bag of Words）→ 不考虑词序
2013    Word2Vec → 词向量（语义相似的词距离近）
2017    Transformer → 注意力机制，革命性突破
2018    BERT → 双向理解，NLP 各任务通吃
2020    GPT-3 → 超大模型，涌现出对话能力
2022    ChatGPT → RLHF 对齐，人类友好的对话
2023+   各类开源 LLM（Llama、Mistral、Qwen 等）
```

---

## 🔑 Transformer 核心思想（直觉理解）

> "注意力就是：在处理一个词时，看看句子中其他哪些词更相关。"

例如：**"充电桩在工作时温度过高"**
- 理解"它"（如果有代词）时，需要关联到"充电桩"
- Transformer 通过注意力权重自动学会这种关联

---

## 🔑 用 API 快速构建对话应用

```python
# 使用 OpenAI API（或兼容接口）构建对话
from openai import OpenAI

client = OpenAI(api_key="your-api-key")

# 多轮对话示例
messages = [
    {"role": "system", "content": "你是一个嵌入式开发专家助手"},
]

# 用户提问
user_input = "GD32 的 CAN 总线如何初始化？"
messages.append({"role": "user", "content": user_input})

response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=messages
)

answer = response.choices[0].message.content
print(answer)
```

---

## 📝 待补充内容

- 词向量（Word2Vec）详解
- Transformer 架构图解
- RAG（检索增强生成）原理与实践
- 本地运行 Qwen / Llama 的方法
- 基于知识库的问答系统搭建
