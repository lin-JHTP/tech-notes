# 数据结构

> 计算机科学基础：数据结构是算法的基石，也是面试必考内容。

## 📌 学习路线

```
基础数据结构
├── 线性结构
│   ├── 数组（Array）
│   ├── 链表（Linked List）
│   ├── 栈（Stack）
│   └── 队列（Queue）
├── 树形结构
│   ├── 二叉树（Binary Tree）
│   ├── 二叉搜索树（BST）
│   ├── 堆（Heap）
│   └── 平衡树（AVL / 红黑树）
└── 散列结构
    └── 哈希表（Hash Table）
```

---

## 🔑 数组 vs 链表

| 特性 | 数组 | 链表 |
|------|------|------|
| 内存 | 连续 | 不连续 |
| 访问 | O(1) 随机访问 | O(n) 顺序访问 |
| 插入/删除 | O(n) | O(1)（已知位置） |
| 适用场景 | 频繁读，少写 | 频繁插入删除 |

---

## 🔑 栈（Stack）

后进先出（LIFO）。嵌入式中的函数调用栈就是典型的栈结构。

```c
/* C 语言简单栈实现 */
#define STACK_SIZE 32

typedef struct {
    int  data[STACK_SIZE];
    int  top;
} Stack;

void stack_push(Stack *s, int val) { s->data[++s->top] = val; }
int  stack_pop(Stack *s)           { return s->data[s->top--]; }
```

---

## 📝 待补充内容

- 链表操作（反转、合并）
- 二叉树遍历（前/中/后序、层序）
- 堆的应用（优先队列）
- 哈希表冲突处理
