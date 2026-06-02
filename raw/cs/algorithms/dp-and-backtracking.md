---
title: 动态规划与回溯算法
updated: 2026-06-02
tags: [cs, algorithms, dynamic-programming, backtracking, greedy, knapsack]
aliases: [DP, 动态规划, 回溯算法, 背包问题, dp-and-backtracking]
cssclasses: []
---

# 动态规划与回溯算法

## 动态规划 (Dynamic Programming)

### 核心思想

将问题分解为多个**状态**，通过**状态转移方程**从初始状态逐步推导到最终状态。`dp` 数组存储各阶段的状态值。

> [!abstract] 动态规划五部曲
> 1. 确定 `dp[i]` 的含义
> 2. 推导状态转移方程
> 3. 确定初始状态（base case）
> 4. 确定遍历顺序
> 5. （可选）举例验证

### 经典入门

#### 爬楼梯 (LeetCode 70)

状态定义：`dp[i]` 表示爬到第 `i` 阶的方法数。

$$
dp[i] = dp[i-1] + dp[i-2]
$$

初始状态：`dp[1] = 1, dp[2] = 2`

```cpp
int climbStairs(int n) {
    if (n <= 2) return n;
    int dp[46] = {0};
    dp[1] = 1; dp[2] = 2;
    for (int i = 3; i <= n; i++)
        dp[i] = dp[i - 1] + dp[i - 2];
    return dp[n];
}
```

#### 最小花费爬楼梯 (LeetCode 746)

$$
dp[i] = \min(dp[i-1] + cost[i-1],\ dp[i-2] + cost[i-2])
$$

#### 整数拆分 (LeetCode 343)

将正整数 `n` 拆分为 `k` 个正整数之和，使乘积最大化。

状态转移：对每个 `i`，枚举拆分点 `j`，取 `max(j * (i-j), j * dp[i-j])`。

```cpp
int integerBreak(int n) {
    if (n == 2) return 1;
    vector<int> dp(n + 1, 0);
    dp[2] = 1;
    for (int i = 3; i <= n; i++)
        for (int j = 1; j <= i / 2; j++)
            dp[i] = max((i - j) * j, max(dp[i], j * dp[i - j]));
    return dp[n];
}
```

#### 最大子数组和 (LeetCode 53)

`dp[i]` = 以 `nums[i]` 结尾的最大子数组和。

$$
dp[i] = \max(nums[i],\ dp[i-1] + nums[i])
$$

```cpp
int maxSubArray(vector<int>& nums) {
    vector<int> dp(nums.size(), 0);
    dp[0] = nums[0];
    int result = nums[0];
    for (int i = 1; i < nums.size(); i++) {
        dp[i] = max(nums[i], dp[i - 1] + nums[i]);
        if (dp[i] > result) result = dp[i];
    }
    return result;
}
```

#### 买卖股票的最佳时机 II (LeetCode 122)

定义两个状态：`dp[i][0]` = 第 i 天**持有**股票的最大收益，`dp[i][1]` = 第 i 天**不持有**股票的最大收益。

```cpp
int maxProfit(vector<int>& prices) {
    if (prices.size() == 0) return 0;
    vector<vector<int>> dp(prices.size(), vector<int>(2, 0));
    dp[0][0] = -prices[0];  // 持有
    dp[0][1] = 0;           // 不持有

    for (int i = 1; i < prices.size(); i++) {
        dp[i][0] = max(dp[i - 1][0], dp[i - 1][1] - prices[i]);
        dp[i][1] = max(dp[i - 1][0] + prices[i], dp[i - 1][1]);
    }
    return max(dp[prices.size() - 1][0], dp[prices.size() - 1][1]);
}
```

> [!tip] 贪心解法
> 如果允许无限次交易（本题），贪心更简单：只要今天价格比昨天高，就累加差价。

---

### 背包问题

背包问题是动态规划的经典模型，核心是"容量约束下的选择优化"。

**01 背包**：每件物品最多选一次
**完全背包**：每件物品可选无限次
**多重背包**：每件物品有数量限制 $k_i$

#### 状态定义

`dp[i][j]` = 前 `i` 件物品，容量为 `j` 时的最大价值。

#### 降维优化

由于状态转移只依赖 `dp[i-1][*]` 和 `dp[i][*]`，可以将二维 `dp` 数组压缩到一维：

- **01 背包**：内层容量**倒序**遍历 —— 确保每件物品只用一次
- **完全背包**：内层容量**正序**遍历 —— 允许同一物品多次使用
- **多重背包**：展开为 01 背包后倒序遍历

```cpp
// 01 背包一维优化
for (int i = 0; i < n; i++)
    for (int j = capacity; j >= weight[i]; j--)
        dp[j] = max(dp[j], dp[j - weight[i]] + value[i]);

// 完全背包一维优化
for (int i = 0; i < n; i++)
    for (int j = weight[i]; j <= capacity; j++)
        dp[j] = max(dp[j], dp[j - weight[i]] + value[i]);
```

#### 二进制分组优化（多重背包）

多重背包中 $O(W \sum k_i)$ 的时间复杂度在物品数量大时不可接受。使用**二进制分组**：

将每种物品的数量 $k$ 拆分为 $1, 2, 4, 8, \dots, 2^m, r$（其中前 $m+1$ 个数可组合出 $0$ 到 $2^{m+1}-1$，加上余数 $r$ 组成完整范围）。拆分后转化为 01 背包问题，复杂度降为 $O(W \sum \log k_i)$。

> 例如：600 个 A 物品 → 拆为 10 组 `[1,2,4,8,16,32,64,128,256,89]`，只需 10 次 01 背包决策。

---

### 区间 DP：最长回文子串 (LeetCode 5)

`dp[i][j]` 表示 `s[i..j]` 是否为回文串。

$$
dp[i][j] = (s[i] == s[j])\ \&\&\ (j - i < 3\ \text{或}\ dp[i+1][j-1])
$$

```cpp
string longestPalindrome(string s) {
    if (s.size() < 2) return s;
    vector<vector<bool>> dp(s.size(), vector<bool>(s.size(), true));
    int max_length = 1;
    string result = s.substr(0, 1);

    for (int i = 1; i < s.size(); i++) {
        for (int j = 0; j < i; j++) {
            dp[i][j] = s[i] == s[j] && dp[i - 1][j + 1];
            if (dp[i][j] && i - j + 1 > max_length) {
                max_length = i - j + 1;
                result = s.substr(j, i - j + 1);
            }
        }
    }
    return result;
}
```

---

## 回溯算法

### 核心框架

回溯本质是**纯暴力搜索**，所有问题都可以抽象为一棵 **N 叉决策树**。递归深度 = 树的深度，在叶子节点收集结果。

> [!important] 回溯三部曲
> 1. **递归终止条件**：何时收集结果（通常是路径长度满足要求）
> 2. **选择列表**：当前节点可选的子节点范围
> 3. **剪枝优化**：提前排除不可能的分支

### 三类问题的关系

| 类型 | 顺序 | 特点 |
|------|------|------|
| 子集 `subset` | 无关 | 所有可能的组合，包括空集 |
| 组合 `combination` | 无关 | 从 n 个中选 k 个，`{1,2}` 和 `{2,1}` 视为同一种 |
| 排列 `permutation` | 有关 | `{1,2}` 和 `{2,1}` 是两种不同结果 |

### 子集问题

两种写法：

```cpp
// 写法1：选/不选模型
void dfs(int i) {
    if (i == n) { result.push_back(path); return; }
    dfs(i + 1);                           // 不选
    path.push_back(nums[i]);
    dfs(i + 1);                           // 选
    path.pop_back();
}

// 写法2：逐步扩展模型
void dfs(int start) {
    result.push_back(path);               // 每个节点都收集
    for (int i = start; i < n; i++) {
        path.push_back(nums[i]);
        dfs(i + 1);
        path.pop_back();
    }
}
```

### 组合问题

在子集的基础上增加**数量限制**和**剪枝优化**。核心剪枝：剩余元素不足以凑够 `k` 个时提前退出。

```python
def dfs(i):
    d = k - len(path)
    if i < d:                    # 剪枝：剩余元素不够
        return
    if len(path) == k:
        ans.append(path.copy())
        return
    for j in range(i, 0, -1):   # 倒序枚举
        path.append(j)
        dfs(j - 1)
        path.pop()
```

**C++ 实现**（组合 LeetCode 77）：

```cpp
void backtracking(int startindex) {
    if (path.size() == k) {
        result.push_back(path);
        return;
    }
    // 剪枝：剩余元素必须足够
    for (int i = startindex; i <= n - (k - path.size()) + 1; i++) {
        path.push_back(i);
        backtracking(i + 1);
        path.pop_back();
    }
}
```

### 组合总和 (LeetCode 39)

元素可重复选取，回溯时 `startindex` 不变（允许重复选当前元素）。同时排序后按 `sum + candidates[i] <= target` 剪枝。

```cpp
void breakTracking(const vector<int>& candidates, int startindex) {
    if (sum == target) {
        result.push_back(path);
        return;
    }
    for (int i = startindex;
         i < candidates.size() && sum + candidates[i] <= target; i++) {
        sum += candidates[i];
        path.push_back(candidates[i]);
        breakTracking(candidates, i);  // 注意是 i，允许重复选
        sum -= candidates[i];
        path.pop_back();
    }
}
```

### 排列问题

与组合不同：**每层决策需要考虑所有未选元素**，标准做法使用 `used` 数组标记。

```cpp
void backtracking(vector<int>& nums, vector<bool>& used) {
    if (path.size() == nums.size()) {
        result.push_back(path);
        return;
    }
    for (int i = 0; i < nums.size(); i++) {
        if (used[i]) continue;
        used[i] = true;
        path.push_back(nums[i]);
        backtracking(nums, used);
        path.pop_back();
        used[i] = false;
    }
}
```

### 回溯去重策略

| 去重类型 | 含义 | 实现方式 |
|----------|------|----------|
| **树层去重** | 同一层中跳过重复元素 | 排序后使用 `used[i-1] == false` 或每层独立 `set` |
| **树枝去重** | 同一路径中允许重复使用 | `used[i-1] == true`（排列中）或调整 `startindex` |

```cpp
// 树层去重（组合总和 II）
if (i > 0 && candidates[i] == candidates[i - 1] && used[i - 1] == false)
    continue;
```

> [!tip] 回溯尾部优化
> 如果回溯操作在递归调用尾部，可以使用 `recursion(path + xxx)` 的形式传递新路径，避免显式的 `pop_back()`。

### 分割问题：分割回文串 (LeetCode 131)

回溯不仅可以选元素，还可以**切割字符串**。`startindex` 表示切割线位置：

```cpp
void backtracking(const string &s, int startindex) {
    if (startindex == s.size()) {
        result.push_back(path);
        return;
    }
    for (int i = startindex; i < s.size(); i++) {
        if (isPalindrome(s, startindex, i)) {
            path.push_back(s.substr(startindex, i - startindex + 1));
            backtracking(s, i + 1);
            path.pop_back();
        }
    }
}
```

---

## 贪心算法

贪心的核心是**局部最优推导全局最优**。

> [!note] 贪心判断准则
> 如果找不到明显的反例来证明局部最优无法推导全局最优，就可以尝试贪心。

### 跳跃游戏 (LeetCode 55)

维护当前能到达的**最远范围** `right_range`，每步更新：

```cpp
bool canJump(vector<int>& nums) {
    int right_range = 0;
    if (nums.size() == 1) return true;
    for (int i = 0; i <= right_range; i++) {
        right_range = max(right_range, i + nums[i]);
        if (right_range >= nums.size() - 1) return true;
    }
    return false;
}
```

---

## 概念辨析

| 类型 | 名称 | 是否连续 | 说明 |
|------|------|----------|------|
| 数组 | 子数组 (subarray) | **连续** | |
| 数组 | 子段 (subsegment) | **连续** | |
| 数组 | 子序列 (subsequence) | 不连续 | 相对次序不能乱 |
| 字符串 | 子串 (substring) | **连续** | |
| 字符串 | 子序列 (subsequence) | 不连续 | 相对次序不能乱 |

> 来源：[[代码随想录]] https://www.programmercarl.com/

---

## 相关 LeetCode 题目

| 题目 | 模式 | 关键点 |
|------|------|--------|
| 70 爬楼梯 | DP 入门 | 斐波那契递推 |
| 746 最小花费爬楼梯 | DP | `min` 状态转移 |
| 343 整数拆分 | DP | 枚举拆分点 |
| 53 最大子数组和 | DP | Kadane |
| 122 买卖股票 II | DP / 贪心 | 双状态 |
| 5 最长回文子串 | 区间 DP | 二维布尔 dp |
| 77 组合 | 回溯 | 剪枝 |
| 39 组合总和 | 回溯 | 可重复 + 排序剪枝 |
| 40 组合总和 II | 回溯 | 树层去重 |
| 131 分割回文串 | 回溯 | 切割模型 |
| 55 跳跃游戏 | 贪心 | 维护最远范围 |
