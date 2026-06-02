---
title: 排序与搜索算法
updated: 2026-06-02
tags: [cs, algorithms, sorting, searching, monotonic-stack, priority-queue]
aliases: [排序算法, 二分查找, 单调栈, 优先队列, sort-and-search]
cssclasses: []
---

# 排序与搜索算法

## 排序算法概览

| 算法 | 平均时间 | 最优时间 | 最差时间 | 空间 | 稳定性 |
|------|----------|----------|----------|------|--------|
| 冒泡排序 | $O(n^2)$ | $O(n)$ | $O(n^2)$ | $O(1)$ | 稳定 |
| 插入排序 | $O(n^2)$ | $O(n)$ | $O(n^2)$ | $O(1)$ | 稳定 |
| 选择排序 | $O(n^2)$ | $O(n^2)$ | $O(n^2)$ | $O(1)$ | 不稳定 |
| 希尔排序 | $O(n^{1.3})$ | $O(n)$ | $O(n^2)$ | $O(1)$ | 不稳定 |
| 堆排序 | $O(n\log n)$ | $O(n\log n)$ | $O(n\log n)$ | $O(1)$ | 不稳定 |
| 快速排序 | $O(n\log n)$ | $O(n\log n)$ | $O(n^2)$ | $O(\log n)$ | 不稳定 |
| 归并排序 | $O(n\log n)$ | $O(n\log n)$ | $O(n\log n)$ | $O(n)$ | 稳定 |

### 简单排序

#### 冒泡排序

相邻元素两两比较，每轮将最大元素"冒泡"到末尾。引入 `isSwitch` 标志位提前终止。

```cpp
void Bubble_sort(vector<int> &nums) {
    for (int j = nums.size() - 1; j >= 1; j--) {
        bool isSwitch = false;
        for (int i = 0; i <= j - 1; i++) {
            if (nums[i] > nums[i + 1]) {
                swap(nums[i], nums[i + 1]);
                isSwitch = true;
            }
        }
        if (!isSwitch) return;  // 本轮无交换，已经有序
    }
}
```

#### 插入排序

从左到右逐个元素插入到前面已排序部分的正确位置。适合小规模或近乎有序的数据。

```cpp
void Insert_sort(vector<int> &nums) {
    for (int i = 1; i < nums.size(); i++) {
        for (int j = i; j > 0 && nums[j] < nums[j - 1]; j--) {
            swap(nums[j], nums[j - 1]);
        }
    }
}
```

#### 选择排序

每轮选择未排序部分的最小元素，放到已排序部分的末尾。

```cpp
void Select_sort(vector<int> &nums) {
    for (int i = 0; i < nums.size(); i++) {
        int min = i;
        for (int j = i + 1; j < nums.size(); j++) {
            if (nums[min] > nums[j]) min = j;
        }
        swap(nums[i], nums[min]);
    }
}
```

### 希尔排序

插入排序的改进版：通过递减增量序列 `h` 对元素进行分组插入排序，逐步缩小增量直至 `h=1`。

```cpp
void Shell_sort(vector<int> &nums) {
    int N = nums.size();
    int h = 1;
    while (h < N / 3) h = 3 * h + 1;  // Knuth 增量序列

    while (h >= 1) {
        for (int i = h; i < N; i++) {
            for (int j = i; j >= h && nums[j] < nums[j - h]; j -= h) {
                swap(nums[j], nums[j - h]);
            }
        }
        h = h / 3;
    }
}
```

### 堆排序

利用 [[#堆 heap 与优先队列]] 的堆结构进行排序，分为两步：

1. **建堆**：从最后一个非叶子节点 `n/2 - 1` 开始向下调整
2. **排序**：反复将堆顶（最大值）与末尾交换，再对剩余元素重新堆化

```cpp
void heapify(vector<int> &arr, int len, int i) {
    int largest = i;
    int lson = i * 2 + 1;
    int rson = i * 2 + 2;

    if (lson < len && arr[largest] < arr[lson]) largest = lson;
    if (rson < len && arr[largest] < arr[rson]) largest = rson;

    if (largest != i) {
        swap(arr[largest], arr[i]);
        heapify(arr, len, largest);
    }
}

void heap_sort(vector<int> &arr, int len) {
    // 建堆
    for (int i = len / 2 - 1; i >= 0; i--) {
        heapify(arr, len, i);
    }
    // 排序
    for (int i = len - 1; i > 0; i--) {
        swap(arr[i], arr[0]);
        heapify(arr, i, 0);
    }
}
```

> [!tip] 堆化复杂度
> 建堆是 $O(n)$，排序是 $O(n\log n)$。堆排序是不稳定排序。

### 快速排序

分区 + 递归。选取 `pivot`，将小于 pivot 的放左边，大于的放右边，递归处理左右子数组。

```cpp
int part(vector<int> &nums, const int &low, const int &high) {
    int i = low, j = high, pivot = nums[low];
    while (i < j) {
        while (i < j && nums[j] > pivot) j--;
        if (i < j) swap(nums[i++], nums[j]);
        while (i < j && nums[i] <= pivot) i++;
        if (i < j) swap(nums[i], nums[j--]);
    }
    return i;
}

void fast_sort(vector<int> &nums, int low, int high) {
    if (low < high) {
        int mid = part(nums, low, high);
        fast_sort(nums, low, mid - 1);
        fast_sort(nums, mid + 1, high);
    }
}
```

> [!warning] 最坏情况
> 有序或逆序数组退化为 $O(n^2)$。随机选取 pivot 或三数取中可以避免。

### 归并排序

分治思想的典范：递归二分数组至单元素，然后合并两个有序子数组。稳定排序，需要 $O(n)$ 额外空间。

```cpp
void _mergeSort(vector<int> &nums, int left, int right, vector<int> &temp) {
    if (left >= right) return;
    int mid = (left + right) / 2;
    _mergeSort(nums, left, mid, temp);
    _mergeSort(nums, mid + 1, right, temp);

    int left_b = left, left_e = mid;
    int right_b = mid + 1, right_e = right;
    int temp_index = left;

    while (left_b <= left_e && right_b <= right_e) {
        if (nums[left_b] < nums[right_b])
            temp[temp_index++] = nums[left_b++];
        else
            temp[temp_index++] = nums[right_b++];
    }
    while (left_b <= left_e) temp[temp_index++] = nums[left_b++];
    while (right_b <= right_e) temp[temp_index++] = nums[right_b++];

    for (int i = left; i <= right; i++) nums[i] = temp[i];
}

void merge_sort(vector<int> &nums) {
    vector<int> temp(nums.size());
    _mergeSort(nums, 0, nums.size() - 1, temp);
}
```

---

## 二分查找

二分查找的核心是**确定搜索区间**和**循环不变量**。

### 两种区间写法

**左闭右闭** `[left, right]`：`while (left <= right)`，因为 `left == right` 时区间还有效。

**左闭右开** `[left, right)`：`while (left < right)`，因为 `left == right` 时区间为空。

```cpp
// 左闭右闭版本 -- 查找 target 的插入位置（第一个 >= target）
int left = 0, right = nums.size() - 1;
while (left <= right) {
    int mid = left + (right - left) / 2;
    if (nums[mid] < target)
        left = mid + 1;
    else
        right = mid - 1;
}
return left;  // 第一个 >= target 的位置

// 左闭右开版本
int left = 0, right = nums.size();
while (left < right) {
    int mid = left + (right - left) / 2;
    if (nums[mid] < target)
        left = mid + 1;
    else
        right = mid;
}
return left;  // 第一个 >= target 的位置
```

### 查找条件转换

默认可得到 **第一个 `>= x`** 的位置，其他条件通过转换得到：

```
>= x  → 直接二分
>  x  → 查找 >= (x + 1)
<  x  → (查找 >= x 的位置) - 1
<= x  → (查找 >  x 的位置) - 1
```

### 标准库函数

```cpp
lower_bound(begin, end, val)  // 第一个 >= val
upper_bound(begin, end, val)  // 第一个 >  val
binary_search(begin, end, val)  // 是否存在
```

### 二分搜索变体

**值域二分**：当数组本身无序，但可通过计数判断时，在值域上做二分。例如 [[LeetCode 287 寻找重复数]]：统计 `<= mid` 的元素个数，若 `cnt > mid` 说明重复数在左半。

**二维矩阵搜索**：先二分确定行，再二分确定列（或整体视为一维数组二分）：

```cpp
bool searchMatrix(vector<vector<int>>& matrix, int target) {
    // 用 upper_bound 定位目标行
    auto row = upper_bound(matrix.begin(), matrix.end(), target,
        [](const int b, const vector<int> &a) { return b < a[0]; });
    if (row == matrix.begin()) return false;
    --row;
    return binary_search(row->begin(), row->end(), target);
}
```

---

## 单调栈

单调栈维护栈内元素的**单调性**（递增或递减），常用于解决 **"下一个更大/更小元素"** 类型问题。

- **单调递增栈**（栈底大 → 栈顶小）：找下一个**较大**元素
- **单调递减栈**（栈底小 → 栈顶大）：找下一个**较小**元素

### 基础模板：下一个更大元素（LeetCode 739 每日温度）

```cpp
vector<int> dailyTemperatures(vector<int>& temperatures) {
    if (temperatures.size() == 1) return {0};
    stack<int> s;
    vector<int> result(temperatures.size(), 0);
    s.push(0);

    for (int i = 1; i < temperatures.size(); i++) {
        while (!s.empty() && temperatures[i] > temperatures[s.top()]) {
            int index = s.top(); s.pop();
            result[index] = i - index;
        }
        s.push(i);
    }
    return result;
}
```

### 高级用法：确定最长区间

单调栈也可以用来找以某个元素为最小值/最大值的**最长连续区间**。在栈中保留索引，利用栈的单调性确定左右边界。

> 相关题目：[[LeetCode 962 最大宽度坡]]、[[LeetCode 1124 表现良好的最长时间段]]

---

## 单调队列

维护滑动窗口内的最大值/最小值。使用 `deque`，保持队列单调递减（求最大值）或单调递增（求最小值）。

```cpp
class Myqueue {
private:
    deque<int> dq;
public:
    void Push(const int &num) {
        // 保持单调递减：弹出所有小于 num 的队尾元素
        while (!dq.empty() && dq.back() < num) dq.pop_back();
        dq.push_back(num);
    }
    void Pop(int num) {
        // 仅在要弹出的元素等于队首时才弹出（它可能已经被后来的更大元素顶掉了）
        if (!dq.empty() && num == dq.front()) dq.pop_front();
    }
    int front() { return dq.front(); }
};
```

> [!tip] 应用场景
> 滑动窗口最大值、带限制的最短子数组等。单调队列的核心是"老且小"的元素没有存在的必要。

---

## 堆 (Heap) 与优先队列

### 堆的定义

堆是一棵**完全二叉树**，满足：

- 大顶堆：父节点 ≥ 子节点
- 小顶堆：父节点 ≤ 子节点

### 核心操作

**插入 (insert)**：
1. 在堆末尾新建节点并赋值
2. 与父节点比较并交换（上浮），直到满足堆性质

**弹出 (poll)**：
1. 移除根节点
2. 将最后一个元素放到根位置
3. 从根开始向下比较交换（下沉），直到满足堆性质

### 优先队列的典型应用

对频率进行排序时，用 `map` 统计频率后，不需要对所有 key 排序——只维护一个大小为 k 的优先队列：

> **LeetCode 347 前 K 个高频元素**：用小顶堆维护频率最高的 k 个元素，堆大小始终保持 k，最终堆中即为答案。时间复杂度 $O(n\log k)$，优于全局排序的 $O(n\log n)$。

### 代码实现：通过迭代器构建

```cpp
// 从已有容器构建，避免逐个 push_back（复杂度 O(N) vs O(N log N)）
return vector<int>(result_temp.begin(), result_temp.end());
```

---

## 相关 LeetCode 题目

| 题目 | 模式 |
|------|------|
| 34 在排序数组中查找元素的第一个和最后一个位置 | 二分查找边界 |
| 74 搜索二维矩阵 | 两次二分 |
| 287 寻找重复数 | 值域二分 |
| 739 每日温度 | 单调栈 |
| 239 滑动窗口最大值 | 单调队列 |
| 347 前 K 个高频元素 | 优先队列 |
