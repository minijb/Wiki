---
title: 算法面试通用方法论
updated: 2026-06-02
tags: [cs, algorithms, interview, patterns, methodology]
aliases: [面试算法, 算法面试方法论, interview-patterns]
cssclasses: []
---

# 算法面试通用方法论

> 系统化整理算法面试中的核心模式、经典问题与解题策略，覆盖图论、高级数据结构、数学推导等面试常见领域。

---

## 图论算法

### Prim 最小生成树

核心数据结构：`minDist` 数组 + `isInTree` 标记。

```
循环：
  1. 从 minDist 中选取未入树的最小距离节点 cur
  2. 标记 isInTree[cur] = true
  3. 更新 cur 所有邻居的 minDist（若更短且未入树）
```

> 类似 Dijkstra，区别在于 Prim 选择离**生成树**最近的节点，Dijkstra 选择离**起点**最近的节点。

### Kruskal 最小生成树

核心数据结构：**并查集** + 边列表（按权重排序）。

```
1. 将所有边按权重升序排序
2. 遍历每条边：
   若两个端点不在同一集合 → 加入生成树，合并两个集合
   若已在同一集合 → 跳过（避免成环）
```

### 并查集 (Disjoint Set Union, DSU)

```cpp
int n = 10001;
vector<int> father(n, -1);

void init() {
    for (int i = 0; i < n; i++) father[i] = i;
}

int find(int u) {
    return u == father[u] ? u : father[u] = find(father[u]);  // 路径压缩
}

void uni(int x, int y) {
    int fx = find(x), fy = find(y);
    if (fx == fy) return;
    father[fx] = fy;
}
```

> [!tip] 路径压缩
> `father[u] = find(father[u])` 将查找路径上所有节点直接指向根，均摊复杂度接近 $O(1)$。

### 拓扑排序

用于 **DAG（有向无环图）的线性排序**，判断图中是否有环。

**步骤**：
1. 找到所有入度为 0 的节点（起始节点）
2. 反复从图中移除入度为 0 的节点及其出边
3. 更新受影响节点的入度

**环的判定**：若图中还有节点但没有入度为 0 的节点 → **一定有环**。

---

## A* 路径搜索算法

在静态路网中求解最短路径的高效启发式搜索算法。评估函数：

$$
F = G + H
$$

- **G**：从起点到当前位置的实际步数
- **H**：预估从当前位置到终点的步数（启发函数）

### 优化策略

#### 1. OpenList 优化

5 种核心操作：添加、删除、获取长度、判断存在、排序 → 使用**优先队列**（二叉堆）实现。

#### 2. GetNeighbour 优化

- 邻居顺序随机化而非固定方向
- 扩大搜索眼界
- **JPS 跳点搜索算法**：只支持斜向移动场景，通过跳过冗余节点减少邻居数量

#### 3. 启发函数优化

$$
F = G + H + C
$$

- 引入拐弯惩罚（减少拐弯次数）
- 可穿透障碍物：给障碍物添加通行代价而非完全禁止

#### 4. Map 矩阵优化

- 控制地图维度
- **HPA 分层优化**：小地图拼接成大地图，先在高层规划路径，再细化到低层

---

## 红黑树

### 基本概念

- 叶子节点（NIL）不存储数据，**叶子节点和根节点一定是黑色**
- 每个存储数据的节点都有两个子节点（可能为 NIL）
- 中序遍历为升序 → 属于二叉搜索树

> [!important] 核心约束
> 1. **所有路径**：根到叶子的黑节点数量相同
> 2. 红色节点不能连续（红节点的父和子必为黑）

### 插入操作

| 情况 | 父节点颜色 | 叔节点颜色 | 处理方式 |
|------|-----------|-----------|----------|
| 1 | 黑 | — | 直接插入红节点 |
| 2 | 红 | 红 | 父、叔、爷爷颜色反转 |
| 3 | 红 | 黑 | 爷爷-父-当前在直线上 → **旋转** + 换色 |
| 4 | 红 | 黑 | 不在直线上 → 先旋转到直线 → 按情况 3 处理 |

> 关键：父亲和叔叔**必须一起变换**，否则根到叶子的黑节点数量不一致。

### 删除操作

| 情况 | 处理方式 |
|------|----------|
| 单个红节点 | 直接删除 |
| 含一个子节点 | 可推理：当前节点为黑，子节点为红。红替换黑，删除红 |
| 含两个子节点 | 找左子树最大节点替换，转为删除该替换节点的子问题 |
| 单个黑节点 + 兄弟为黑 | 见下方兄弟黑子情况 |
| 单个黑节点 + 兄弟为红 | 父兄旋转换色，转为兄弟黑处理 |

**兄弟为黑子情况**：

| 子情况 | 处理 |
|--------|------|
| 对侄为红 | 父兄交替旋转 + 换色 |
| 顺侄为红 | 兄侄交替旋转 + 换色 → 变为对侄红 |
| 双侄黑 | 兄变红，父变黑；若父原本为黑，递归向上处理 |

---

## 经典算法问题（按模式分类）

### 链表操作

#### 反转链表 (LeetCode 206)

```cpp
ListNode* reverseList(ListNode* head) {
    if (head == nullptr || head->next == nullptr) return head;
    ListNode *cur = head->next, *pre = head;
    pre->next = nullptr;
    while (cur != nullptr) {
        ListNode* nextNode = cur->next;
        cur->next = pre;
        pre = cur;
        cur = nextNode;
    }
    return pre;
}
```

#### 删除链表倒数第 N 个节点 (LeetCode 19) —— 快慢指针

```cpp
ListNode* removeNthFromEnd(ListNode* head, int n) {
    ListNode *dummy = new ListNode(0, head);
    ListNode *first = head, *second = dummy;
    for (int i = 0; i < n; ++i) first = first->next;
    while (first) { first = first->next; second = second->next; }
    second->next = second->next->next;
    return dummy->next;
}
```

#### 两两交换链表节点 (LeetCode 24)

```cpp
ListNode* swapPairs(ListNode* head) {
    if (!head || !head->next) return head;
    ListNode* res = new ListNode(0, head);
    ListNode* node = res;
    while (node->next && node->next->next) {
        ListNode* cur = node->next;
        ListNode* cur_n = node->next->next;
        node->next = cur_n;
        cur_n->next = cur;
        cur->next = node->next->next;
        node = cur;
    }
    return res->next;
}
```

### 双指针 / 滑动窗口

#### 盛最多雨水 (LeetCode 11)

```cpp
int maxArea(vector<int>& height) {
    if (height.size() == 1) return 0;
    int l = 0, r = height.size() - 1, result = 0;
    while (l < r) {
        int h = min(height[l], height[r]);
        result = max(result, (r - l) * h);
        if (height[l] <= height[r]) l++; else r--;
    }
    return result;
}
```

#### 无重复字符最长子串 (LeetCode 3)

滑动窗口 + 哈希表记录字符最后出现位置：

```cpp
int lengthOfLongestSubstring(string s) {
    unordered_map<char, int> m;
    int max_len = 0, i = 0, j = 0;
    for (; i < s.size(); i++) {
        if (m.find(s[i]) != m.end()) {
            int target_index = m[s[i]];
            for (; j <= target_index; j++) m.erase(s[j]);
        }
        m[s[i]] = i;
        max_len = max(max_len, i - j + 1);
    }
    return max_len;
}
```

#### 长度最小的子数组 (LeetCode 209)

```cpp
int minSubArrayLen(int target, vector<int>& nums) {
    int current = 0, min_len = INT_MAX;
    int i = 0, j = 0;
    for (; i < nums.size(); i++) {
        current += nums[i];
        while (current >= target) {
            min_len = min(min_len, i - j + 1);
            current -= nums[j++];
        }
    }
    return min_len == INT_MAX ? 0 : min_len;
}
```

### 数组变换

#### 轮转数组 (LeetCode 189)

```cpp
void rotate(vector<int>& nums, int k) {
    int n = nums.size();
    vector<int> tmp(n);
    for (int i = 0; i < n; i++) tmp[(i + k) % n] = nums[i];
    nums.assign(tmp.begin(), tmp.end());
}
```

#### 旋转矩阵 (LeetCode 48)

先转置，再左右交换：

```cpp
void rotate(vector<vector<int>>& matrix) {
    int n = matrix.size();
    for (int i = 0; i < n; i++)           // 转置
        for (int j = i + 1; j < n; j++)
            swap(matrix[i][j], matrix[j][i]);
    for (int i = 0; i < n; i++)           // 左右翻转
        for (int j = 0; j < n / 2; j++)
            swap(matrix[i][j], matrix[i][n - 1 - j]);
}
```

#### 颜色分类 (LeetCode 75) —— 荷兰国旗问题

三指针划分 0/1/2：

```cpp
void sortColors(vector<int>& nums) {
    int p2 = nums.size() - 1, p0 = 0;
    for (int i = 0; i <= p2; i++) {
        while (i <= p2 && nums[i] == 2) swap(nums[p2--], nums[i]);
        if (nums[i] == 0) swap(nums[i], nums[p0++]);
    }
}
```

### 哈希妙用

#### 两数之和 (LeetCode 1)

```cpp
vector<int> twoSum(vector<int>& nums, int target) {
    unordered_map<int, int> m;
    for (int i = 0; i < nums.size(); i++) {
        if (m.find(target - nums[i]) != m.end())
            return {m[target - nums[i]], i};
        m[nums[i]] = i;
    }
    return {};
}
```

#### 构造最长回文串 (LeetCode 409)

```cpp
int longestPalindrome(string s) {
    unordered_map<char, int> char_map;
    for (char c : s) char_map[c]++;
    int result = 0;
    for (auto p : char_map) {
        result += p.second / 2 * 2;
        if (result % 2 == 0 && p.second % 2 == 1) result++;
    }
    return result;
}
```

#### 只出现一次的数字 (LeetCode 136)

```cpp
int singleNumber(vector<int>& nums) {
    int x = 0;
    for (int num : nums) x ^= num;  // a ^ a = 0, a ^ 0 = a
    return x;
}
```

### 栈

#### 有效括号 (LeetCode 20)

```cpp
bool isValid(string s) {
    if (s.size() == 1) return false;
    stack<char> st;
    for (char c : s) {
        if (c == '(' || c == '[' || c == '{')
            st.push(c);
        else {
            if (st.empty()) return false;
            char top = st.top();
            if ((top == '(' && c != ')') ||
                (top == '[' && c != ']') ||
                (top == '{' && c != '}')) return false;
            st.pop();
        }
    }
    return st.empty();
}
```

#### 比较版本号 (LeetCode 165)

```cpp
int compareVersion(string v1, string v2) {
    int i = 0, j = 0, n = v1.size(), m = v2.size();
    while (i < n || j < m) {
        long long r1 = 0, r2 = 0;
        for (; i < n && v1[i] != '.'; i++) r1 = r1 * 10 + v1[i] - '0';
        for (; j < m && v2[j] != '.'; j++) r2 = r2 * 10 + v2[j] - '0';
        i++; j++;
        if (r1 < r2) return -1;
        if (r1 > r2) return 1;
    }
    return 0;
}
```

> [!warning] 溢出注意
> 用 `long long` 防止十进制累加溢出。如 `(long)nums[i] + nums[j]`，先转换为 `long` 再相加。

---

## 面试实录

### 两个队列实现一个栈

经典数据结构转换题：使用两个队列，`push` 时总是将元素放入非空队列（或任意一个）；`pop` 时将非空队列的元素逐个移到另一个队列，只剩最后一个时弹出。

---

## 游戏开发中的数学几何

### 判断目标在左侧还是右侧

```csharp
Vector3 direction = target.position - transform.position;
var cross = Vector3.Cross(transform.forward, direction);
if (cross.y < 0) Debug.Log("Left");
else Debug.Log("Right");
```

### 判断点是否在三角形内部

1. 用三角形三条边分别与点做叉积得到 `cross1, cross2, cross3`
2. 三点积同号 → 在内部：
   `Vector3.Dot(cross1, cross2) >= 0 && Vector3.Dot(cross1, cross3) >= 0`

### 2D 点逆时针旋转

点 $P(x, y)$ 绕原点逆时针旋转角 $\alpha$：

$$
X = x \cos\alpha - y \sin\alpha
$$

$$
Y = x \sin\alpha + y \cos\alpha
$$

### 判断面向还是背向

自己的 `forward` 与自己到 target 的方向做点乘：

- `dot > 0` → 面向
- `dot < 0` → 背向

### 反射向量

已知入射向量 $I$ 和法线 $N$（单位向量），反射向量 $R$：

$$
R = I - 2(I \cdot N)N
$$

### 判断两个矩形相交及求相交区域

给定两个矩形 A 和 B：

$$
|x_b^{mid} - x_a^{mid}| \leq \frac{W_a + W_b}{2}
$$

$$
|y_b^{mid} - y_a^{mid}| \leq \frac{H_a + H_b}{2}
$$

同时满足上述两式则相交。

相交矩形坐标：

```
Xc1 = max(Xa1, Xb1)    Yc1 = max(Ya1, Yb1)
Xc2 = min(Xa2, Xb2)    Yc2 = min(Ya2, Yb2)
```

`Xc1 <= Xc2 && Yc1 <= Yc2` 时相交区域有效。

### 圆环上随机三点组成锐角三角形的概率

结论：圆心落在三角形内接圆内时，该三角形为锐角三角形。概率为 $1/4$。

---

## 面试准备建议

> [!tip] 面试策略
> - 先确认题目边界条件（输入范围、特殊情况）
> - 口述思路后再开始编码，让面试官跟上节奏
> - 写完代码后主动跑测试用例
> - 分析时间复杂度与空间复杂度
> - 讨论可能的优化方向

### 常见题型优先级

| 优先级 | 类别 | 原因 |
|--------|------|------|
| 高 | 数组 / 字符串 | 基础中的基础，几乎所有面试都有 |
| 高 | 双指针 / 滑动窗口 | 高频经典，容易写出 bug |
| 高 | 二叉树 / BST | 递归思维的综合考察 |
| 中 | 动态规划 | 门槛较高但必会几道经典题 |
| 中 | 回溯 | 组合/排列模板化后稳定性高 |
| 中 | 栈 / 队列 | 括号匹配、单调栈等 |
| 低 | 图论 (Prim/Kruskal/DSU) | 少数公司考察 |
| 低 | 高级数据结构 (红黑树/线段树) | 主要考察理解而非手写 |
