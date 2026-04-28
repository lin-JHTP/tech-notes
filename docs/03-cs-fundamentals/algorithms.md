# 算法

> 算法学习路线，从基础到进阶，兼顾嵌入式 C 语言和未来 Python 双语实践。

## 📌 学习路线

```
算法体系
├── 基础算法
│   ├── 排序（冒泡、快排、归并）
│   ├── 搜索（二分查找、DFS/BFS）
│   └── 双指针 / 滑动窗口
├── 动态规划（DP）
│   ├── 背包问题
│   ├── 最长公共子序列（LCS）
│   └── 最短路径
└── 图算法
    ├── Dijkstra 最短路
    └── 最小生成树（Prim / Kruskal）
```

---

## 🔑 二分查找

```c
/* 在有序数组中查找目标值，返回下标，不存在返回 -1 */
int binary_search(int *arr, int len, int target)
{
    int lo = 0, hi = len - 1;
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;   /* 避免整数溢出 */
        if (arr[mid] == target) return mid;
        else if (arr[mid] < target) lo = mid + 1;
        else hi = mid - 1;
    }
    return -1;
}
```

---

## 🔑 快速排序

```c
void quicksort(int *arr, int lo, int hi)
{
    if (lo >= hi) return;
    int pivot = arr[hi], i = lo - 1;
    for (int j = lo; j < hi; j++) {
        if (arr[j] <= pivot) {
            int tmp = arr[++i]; arr[i] = arr[j]; arr[j] = tmp;
        }
    }
    int tmp = arr[++i]; arr[i] = arr[hi]; arr[hi] = tmp;
    quicksort(arr, lo, i - 1);
    quicksort(arr, i + 1, hi);
}
```

---

## 📝 待补充内容

- 归并排序与堆排序
- 动态规划经典题目
- LeetCode 刷题记录（C / Python）
