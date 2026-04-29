# 数据结构与算法精华

> **文章类型**：知识精华 + 代码实现  
> **适用读者**：C 语言开发者，想复习数据结构与算法基础的工程师  
> **最后更新**：2026-04  
> **标签**：`数据结构` `算法` `C语言` `面试` `CS基础`

---

## 目录

1. [时间空间复杂度速查](#1-时间空间复杂度速查)
2. [线性结构](#2-线性结构)
3. [树结构](#3-树结构)
4. [哈希表](#4-哈希表)
5. [常用排序算法](#5-常用排序算法)
6. [常用查找算法](#6-常用查找算法)
7. [嵌入式场景应用](#7-嵌入式场景应用)
8. [TODO](#8-todo)

---

## 1. 时间空间复杂度速查

### 常见复杂度排序

```
O(1) < O(log n) < O(n) < O(n log n) < O(n²) < O(2ⁿ) < O(n!)
```

### 常见操作复杂度

| 数据结构 | 访问 | 搜索 | 插入 | 删除 |
|---------|------|------|------|------|
| 数组 | O(1) | O(n) | O(n) | O(n) |
| 链表 | O(n) | O(n) | O(1) | O(1) |
| 栈/队列 | O(n) | O(n) | O(1) | O(1) |
| 哈希表 | - | O(1)avg | O(1)avg | O(1)avg |
| 二叉搜索树 | O(log n) | O(log n) | O(log n) | O(log n) |
| 堆 | O(1)最大/小 | O(n) | O(log n) | O(log n) |

---

## 2. 线性结构

### 环形缓冲区（Ring Buffer）— 嵌入式高频使用

```c
/* 无锁单生产者单消费者环形缓冲区 */
#define RING_BUF_SIZE   256U    /* 必须是 2 的幂，方便位与运算 */
#define RING_BUF_MASK   (RING_BUF_SIZE - 1)

typedef struct {
    volatile uint32_t head;             /* 写指针（生产者更新） */
    volatile uint32_t tail;             /* 读指针（消费者更新） */
    uint8_t           buf[RING_BUF_SIZE];
} RingBuffer_t;

/* 写入一个字节（生产者调用，可在 ISR 中）*/
int ring_buf_push(RingBuffer_t *rb, uint8_t byte)
{
    uint32_t head = rb->head;
    uint32_t next_head = (head + 1) & RING_BUF_MASK;

    if (next_head == rb->tail) {
        return -1;  /* 缓冲区满 */
    }
    rb->buf[head] = byte;
    __DSB();                    /* 数据同步屏障，确保数据先写入 */
    rb->head = next_head;       /* 再更新 head */
    return 0;
}

/* 读取一个字节（消费者调用）*/
int ring_buf_pop(RingBuffer_t *rb, uint8_t *out)
{
    if (rb->tail == rb->head) {
        return -1;  /* 缓冲区空 */
    }
    *out = rb->buf[rb->tail];
    __DSB();
    rb->tail = (rb->tail + 1) & RING_BUF_MASK;
    return 0;
}

/* 查询缓冲区中可读字节数 */
uint32_t ring_buf_available(const RingBuffer_t *rb)
{
    return (rb->head - rb->tail) & RING_BUF_MASK;
}
```

### 链表（带哨兵节点）

```c
/* 双向链表，带哨兵头节点，避免空链表特殊处理 */
typedef struct ListNode {
    int            data;
    struct ListNode *prev;
    struct ListNode *next;
} ListNode_t;

typedef struct {
    ListNode_t sentinel;    /* 哨兵节点：sentinel.next = 头, sentinel.prev = 尾 */
    int        count;
} LinkedList_t;

void list_init(LinkedList_t *list)
{
    list->sentinel.next = &list->sentinel;
    list->sentinel.prev = &list->sentinel;
    list->count = 0;
}

/* 在链表尾部插入 */
void list_push_back(LinkedList_t *list, ListNode_t *node)
{
    ListNode_t *tail = list->sentinel.prev;
    node->next = &list->sentinel;
    node->prev = tail;
    tail->next = node;
    list->sentinel.prev = node;
    list->count++;
}

/* 删除节点（O(1)，不需要知道链表头） */
void list_remove(LinkedList_t *list, ListNode_t *node)
{
    node->prev->next = node->next;
    node->next->prev = node->prev;
    list->count--;
}
```

### 栈的应用：括号匹配

```c
/* 检查括号是否匹配 */
int is_brackets_valid(const char *s)
{
    char stack[1024];
    int top = 0;

    while (*s) {
        if (*s == '(' || *s == '[' || *s == '{') {
            if (top >= (int)sizeof(stack)) return 0;  /* 溢出 */
            stack[top++] = *s;
        } else if (*s == ')' || *s == ']' || *s == '}') {
            if (top == 0) return 0;  /* 空栈遇到右括号 */
            char open = stack[--top];
            if ((*s == ')' && open != '(') ||
                (*s == ']' && open != '[') ||
                (*s == '}' && open != '{')) {
                return 0;
            }
        }
        s++;
    }
    return top == 0;  /* 栈空则匹配 */
}
```

---

## 3. 树结构

### 二叉搜索树（BST）

```c
typedef struct BSTNode {
    int            key;
    struct BSTNode *left;
    struct BSTNode *right;
} BSTNode_t;

/* 插入 */
BSTNode_t *bst_insert(BSTNode_t *root, int key)
{
    if (root == NULL) {
        BSTNode_t *node = malloc(sizeof(BSTNode_t));
        node->key   = key;
        node->left  = node->right = NULL;
        return node;
    }
    if (key < root->key) {
        root->left  = bst_insert(root->left, key);
    } else if (key > root->key) {
        root->right = bst_insert(root->right, key);
    }
    return root;  /* key 相等则不重复插入 */
}

/* 中序遍历（输出有序序列）*/
void bst_inorder(const BSTNode_t *root)
{
    if (root == NULL) return;
    bst_inorder(root->left);
    printf("%d ", root->key);
    bst_inorder(root->right);
}
```

### 最小堆（优先队列）

```c
/* 数组实现最小堆 */
#define HEAP_MAX_SIZE  128

typedef struct {
    int data[HEAP_MAX_SIZE];
    int size;
} MinHeap_t;

static void heap_swap(MinHeap_t *h, int i, int j)
{
    int tmp = h->data[i];
    h->data[i] = h->data[j];
    h->data[j] = tmp;
}

/* 上浮（插入后修复堆） */
static void heap_shift_up(MinHeap_t *h, int idx)
{
    while (idx > 0) {
        int parent = (idx - 1) / 2;
        if (h->data[idx] < h->data[parent]) {
            heap_swap(h, idx, parent);
            idx = parent;
        } else {
            break;
        }
    }
}

/* 下沉（删除顶后修复堆） */
static void heap_shift_down(MinHeap_t *h, int idx)
{
    while (2 * idx + 1 < h->size) {
        int child = 2 * idx + 1;  /* 左子节点 */
        /* 取较小的子节点 */
        if (child + 1 < h->size && h->data[child + 1] < h->data[child]) {
            child++;
        }
        if (h->data[child] < h->data[idx]) {
            heap_swap(h, idx, child);
            idx = child;
        } else {
            break;
        }
    }
}

void heap_push(MinHeap_t *h, int val)
{
    if (h->size >= HEAP_MAX_SIZE) return;  /* 堆满 */
    h->data[h->size++] = val;
    heap_shift_up(h, h->size - 1);
}

int heap_pop(MinHeap_t *h)
{
    int top = h->data[0];
    h->data[0] = h->data[--h->size];
    heap_shift_down(h, 0);
    return top;
}
```

---

## 4. 哈希表

### 哈希表实现要点

```c
/* 简单开放地址法哈希表（线性探测） */
#define HT_SIZE     64
#define HT_EMPTY    -1

typedef struct {
    int key;
    int value;
} HTEntry_t;

typedef struct {
    HTEntry_t entries[HT_SIZE];
} HashMap_t;

void hashmap_init(HashMap_t *hm)
{
    for (int i = 0; i < HT_SIZE; i++) {
        hm->entries[i].key = HT_EMPTY;
    }
}

static int hash(int key)
{
    return ((unsigned int)key * 2654435761U) % HT_SIZE;  /* Knuth 乘法散列 */
}

void hashmap_put(HashMap_t *hm, int key, int value)
{
    int idx = hash(key);
    while (hm->entries[idx].key != HT_EMPTY &&
           hm->entries[idx].key != key) {
        idx = (idx + 1) % HT_SIZE;  /* 线性探测 */
    }
    hm->entries[idx].key   = key;
    hm->entries[idx].value = value;
}

int hashmap_get(const HashMap_t *hm, int key, int *out_value)
{
    int idx = hash(key);
    while (hm->entries[idx].key != HT_EMPTY) {
        if (hm->entries[idx].key == key) {
            *out_value = hm->entries[idx].value;
            return 1;  /* 找到 */
        }
        idx = (idx + 1) % HT_SIZE;
    }
    return 0;  /* 未找到 */
}
```

---

## 5. 常用排序算法

### 快速排序（平均 O(n log n)）

```c
static void quick_sort_recursive(int *arr, int low, int high)
{
    if (low >= high) return;

    /* 三数取中法选 pivot，减少最坏情况概率 */
    int mid = low + (high - low) / 2;
    if (arr[low] > arr[mid]) { int t=arr[low]; arr[low]=arr[mid]; arr[mid]=t; }
    if (arr[low] > arr[high]){ int t=arr[low]; arr[low]=arr[high]; arr[high]=t; }
    if (arr[mid] > arr[high]){ int t=arr[mid]; arr[mid]=arr[high]; arr[high]=t; }

    int pivot = arr[mid];
    int i = low - 1, j = high + 1;

    for (;;) {
        do { i++; } while (arr[i] < pivot);
        do { j--; } while (arr[j] > pivot);
        if (i >= j) break;
        int tmp = arr[i]; arr[i] = arr[j]; arr[j] = tmp;
    }

    quick_sort_recursive(arr, low, j);
    quick_sort_recursive(arr, j + 1, high);
}

void quick_sort(int *arr, int n)
{
    if (n > 1) quick_sort_recursive(arr, 0, n - 1);
}
```

### 排序算法对比

| 算法 | 平均时间 | 最坏时间 | 空间 | 稳定 | 适用场景 |
|------|----------|----------|------|------|----------|
| 冒泡排序 | O(n²) | O(n²) | O(1) | ✅ | 教学演示 |
| 选择排序 | O(n²) | O(n²) | O(1) | ❌ | 小数组 |
| 插入排序 | O(n²) | O(n²) | O(1) | ✅ | 近乎有序的小数组 |
| 归并排序 | O(n log n) | O(n log n) | O(n) | ✅ | 链表排序、外排序 |
| 快速排序 | O(n log n) | O(n²) | O(log n) | ❌ | 通用（最常用） |
| 堆排序 | O(n log n) | O(n log n) | O(1) | ❌ | 内存受限场景 |
| 计数排序 | O(n+k) | O(n+k) | O(k) | ✅ | 整数、范围小 |

---

## 6. 常用查找算法

### 二分查找

```c
/*!
 * @brief 在有序数组中二分查找
 * @return 找到时返回下标，未找到返回 -1
 */
int binary_search(const int *arr, int n, int target)
{
    int lo = 0, hi = n - 1;

    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;  /* 防止 (lo+hi) 溢出 */
        if (arr[mid] == target) {
            return mid;
        } else if (arr[mid] < target) {
            lo = mid + 1;
        } else {
            hi = mid - 1;
        }
    }
    return -1;
}

/* 查找第一个 >= target 的位置（左边界） */
int lower_bound(const int *arr, int n, int target)
{
    int lo = 0, hi = n;
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        if (arr[mid] < target) lo = mid + 1;
        else                   hi = mid;
    }
    return lo;  /* lo == hi，即第一个 >= target 的位置 */
}
```

---

## 7. 嵌入式场景应用

### 场景：命令行解析（哈希表查命令）

在充电桩固件中，常需要解析串口调试命令：

```c
/* 用哈希表存命令函数指针，O(1) 查找 */
typedef void (*CmdHandler_t)(const char *args);

typedef struct {
    const char   *name;
    CmdHandler_t  handler;
} CmdEntry_t;

static const CmdEntry_t cmd_table[] = {
    { "status",    cmd_get_status  },
    { "reset",     cmd_reset       },
    { "setaddr",   cmd_set_addr    },
    { "readreg",   cmd_read_reg    },
};

void dispatch_command(const char *cmd_str)
{
    for (int i = 0; i < ARRAY_SIZE(cmd_table); i++) {
        if (strcmp(cmd_table[i].name, cmd_str) == 0) {
            cmd_table[i].handler(/* args */);
            return;
        }
    }
    printf("Unknown command: %s\r\n", cmd_str);
}
```

### 场景：滑动窗口均值滤波（ADC 去噪）

```c
/* 对 ADC 采样值做滑动窗口均值滤波 */
#define FILTER_WINDOW  8

typedef struct {
    float    samples[FILTER_WINDOW];
    uint32_t idx;
    uint32_t count;
    float    sum;
} MovingAvgFilter_t;

float filter_update(MovingAvgFilter_t *f, float new_sample)
{
    /* 减去将被覆盖的旧值 */
    if (f->count >= FILTER_WINDOW) {
        f->sum -= f->samples[f->idx];
    } else {
        f->count++;
    }

    f->samples[f->idx] = new_sample;
    f->sum += new_sample;
    f->idx = (f->idx + 1) % FILTER_WINDOW;

    return f->sum / f->count;
}
```

---

## 8. TODO

- [ ] 图的基本算法（BFS/DFS/Dijkstra）C 语言实现
- [ ] 字典树（Trie）在命令匹配中的应用
- [ ] 跳表（Skip List）原理与 Redis 应用
- [ ] 动态规划经典题目（背包/最长公共子序列）
- [ ] 红黑树原理与 Linux 内核中的应用
