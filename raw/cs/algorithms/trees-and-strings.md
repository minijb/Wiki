---
title: "树结构与字符串算法"
type: source
updated: 2026-06-02
tags: [cs, algorithms, binary-tree, segment-tree, string-hash, prefix-sum, stl]
aliases: [二叉树, 线段树, 字符串哈希, 前缀和]
---

# 树结构与字符串算法

## 二叉树基础

### 二叉树的分类

| 类型 | 定义 | 节点数 / 性质 | 应用 |
|------|------|---------------|------|
| **满二叉树** | 每层节点都达到最大值 | $2^k - 1$ | |
| **完全二叉树** | 除最后一层外其余层满，最后一层节点左对齐 | | 堆 (heap) |
| **二叉搜索树 (BST)** | 左子树 < 根 < 右子树 | 中序遍历有序 | 查找 / 排序 |
| **平衡二叉搜索树 (AVL)** | 左右子树高度差 ≤ 1 | $O(\log n)$ 操作 | `map` `set` 底层 |

#### 二叉搜索树 (BST)

> [!important] BST 性质
> - 左子树所有节点 < 根节点
> - 右子树所有节点 > 根节点
> - 左右子树各自也是 BST
> - **中序遍历得到升序序列**
> - 最左节点最小，最右节点最大

#### 在 BST 中使用双指针

利用中序遍历的有序性，可以用 `pre` 指针记录前驱节点：

```cpp
TreeNode* pre = nullptr;
void recursion(TreeNode* current) {
    // 左 ...
    // 中：处理逻辑
    if (pre != nullptr) {
        // 需要用到 pre 的时候
    }
    pre = current;
    // 右 ...
}
```

### 存储方式

**链式存储**：每个节点持有 `left` 和 `right` 指针（最常用）。

**顺序存储**（数组表示，根在索引 0）：

- 左孩子：`i * 2 + 1`
- 右孩子：`i * 2 + 2`
- 父节点：`(i - 1) / 2`

### 遍历方式

**深度优先搜索 (DFS)** —— 通常用递归实现，也可用栈迭代：

- **前序遍历**：根 → 左 → 右
- **中序遍历**：左 → 根 → 右（BST 得到有序序列）
- **后序遍历**：左 → 右 → 根

**广度优先搜索 (BFS)** —— 层次遍历，用队列实现。

### 经典树问题

#### 二叉树的最大深度 (LeetCode 104)

```cpp
int maxDepth(TreeNode* root) {
    if (root == nullptr) return 0;
    int left = maxDepth(root->left);
    int right = maxDepth(root->right);
    return max(left, right) + 1;
}
```

#### 二叉树最近公共祖先 (LeetCode 236)

后序遍历，自底向上查找：

```cpp
TreeNode* lowestCommonAncestor(TreeNode* root, TreeNode* p, TreeNode* q) {
    if (root == nullptr || root == p || root == q) return root;
    TreeNode* left = lowestCommonAncestor(root->left, p, q);
    TreeNode* right = lowestCommonAncestor(root->right, p, q);
    if (left && right) return root;   // 左右各一个 → 当前是 LCA
    return left ? left : right;       // 都在一侧 → 传递上去
}
```

#### 二叉搜索树第 K 小元素 (LeetCode 230)

中序遍历到第 k 个即可停止：

```cpp
int kthSmallest(TreeNode* root, int k) {
    stack<TreeNode*> stack;
    while (root || stack.size() > 0) {
        while (root) { stack.push(root); root = root->left; }
        root = stack.top(); stack.pop();
        if (--k == 0) return root->val;
        root = root->right;
    }
    return -1;
}
```

#### 有序数组转平衡 BST (LeetCode 108)

每次取中点作为根，递归构建左右子树：

```cpp
TreeNode* sortedArrayToBST(vector<int>& nums, int low, int high) {
    if (low >= high) return nullptr;
    int mid = (low + high) / 2;
    TreeNode* root = new TreeNode(nums[mid]);
    root->left = sortedArrayToBST(nums, low, mid);
    root->right = sortedArrayToBST(nums, mid + 1, high);
    return root;
}
```

#### 求根节点到叶节点数字之和 (LeetCode 129)

```cpp
int dfs(TreeNode* root, int prevSum) {
    if (root == nullptr) return 0;
    int sum = prevSum * 10 + root->val;
    if (!root->left && !root->right) return sum;
    return dfs(root->left, sum) + dfs(root->right, sum);
}
int sumNumbers(TreeNode* root) { return dfs(root, 0); }
```

---

## 完全二叉树的特殊性质

### 性质 1：递归结构

左右子树必为以下两种情况之一：

- 左子树深度 == 右子树深度 → 左子树为**满二叉树**，右子树为完全二叉树
- 左子树深度 > 右子树深度 → 左子树为完全二叉树，右子树为**满二叉树**

> 不断递归下去，最终所有子树都变为满二叉树（只剩一个节点时）。

### 性质 2：与位运算结合

利用完全二叉树的编号计算节点是否存在（[[LeetCode 222 完全二叉树的节点个数]]）：

```cpp
bool exist(TreeNode* root, int level, int k) {
    int bits = 1 << (level - 1);
    TreeNode* node = root;
    while (node != nullptr && bits > 0) {
        if (!(bits & k))
            node = node->left;
        else
            node = node->right;
        bits >>= 1;
    }
    return node != nullptr;
}
```

- `level`：树的深度
- `k`：第 k 个节点（从 1 开始编号）
- `bits`：从最高位开始逐位判断路径方向（0 → 左，1 → 右）

---

## 线段树

线段树是一种用于**区间查询和区间修改**的数据结构。每个节点代表一个区间，支持在 $O(\log n)$ 时间内完成区间的聚合查询（求和、最大值等）和更新。

> 相关题目：[[LeetCode 2286 以组为单位订音乐会的门票]]

---

## 字符串算法

### 字符串哈希

#### 原理

将字符串映射为一个整数，实现 $O(1)$ 字符串判等（期望意义上）。

长度为 $l$ 的字符串 $s$ 的哈希值：

$$
f(s) = \sum_{i=1}^{l} s[i] \cdot b^{l-i} \pmod m
$$

其中 $b$ 为基数（通常取素数 131 或 233），$m$ 为模数。

#### 单哈希实现

```cpp
using ll = long long;
constexpr int M = 1e9 + 7;
constexpr int B = 233;

int get_hash(const string& s) {
    int res = 0;
    for (int i = 0; i < s.size(); ++i)
        res = ((ll)res * B + s[i]) % M;
    return res;
}

bool cmp(const string& s, const string& t) {
    return get_hash(s) == get_hash(t);
}
```

#### 双哈希实现（降低碰撞概率）

```cpp
using ull = unsigned long long;
ull base = 131;
ull mod1 = 212370440130137957, mod2 = 1e9 + 7;

ull get_hash1(string s) {
    ull ans = 0;
    for (int i = 0; i < s.size(); i++)
        ans = (ans * base + (ull)s[i]) % mod1;
    return ans;
}

ull get_hash2(string s) {
    ull ans = 0;
    for (int i = 0; i < s.size(); i++)
        ans = (ans * base + (ull)s[i]) % mod2;
    return ans;
}

bool cmp(const string& s, const string& t) {
    return get_hash1(s) == get_hash1(t) && get_hash2(s) == get_hash2(t);
}
```

#### 应用场景

- **字符串匹配**：$O(n)$ 预处理前缀哈希，$O(1)$ 查询任意子串哈希
- **最长回文子串**：结合二分查找
- **最长公共前缀**：二分 + 哈希判等

#### 最长公共前缀 (LeetCode 14)

```cpp
string longestCommonPrefix(vector<string>& strs) {
    if (!strs.size()) return "";
    int length = strs[0].size();
    int count = strs.size();
    for (int i = 0; i < length; i++) {
        char c = strs[0][i];
        for (int j = 1; j < count; j++) {
            if (i == strs[j].size() || strs[j][i] != c)
                return strs[0].substr(0, i);
        }
    }
    return strs[0];
}
```

---

## 前缀和与差分数组

### 前缀和

快速计算任意区间和（可扩展到乘法、异或等可加性操作）：

```
sum[1, k] + sum[k+1, N] = sum[1, N]
sum[l, r] = sum[1, r] - sum[1, l-1]
```

> [!tip] 典型应用
> 连续子数组求和、二维区域求和、和为 K 的子数组。

### 差分数组

定义：$d_i = a_i - a_{i-1}$

**性质**：
1. 通过求前缀和还原原数组：$\sum_{j=1}^{i} d_j = a_i$
2. **区间修改 $O(1)$**：对区间 $[l, r]$ 加 $v$，只需 `d[l] += v`, `d[r+1] -= v`（静态场景：先修改再询问）

> [!important] 适用条件
> 静态数组 + 先完成全部区间修改 + 最后一次性查询。动态修改需线段树或树状数组。

---

## C++ STL 常用接口速查

### `unordered_set`

```cpp
unordered_set<T> u_set;
u_set.insert(t);               // 插入
auto it = u_set.find(t);       // 查找
if (it != u_set.end()) { }     // 存在性判断
u_set.erase(t);                // 删除
```

### `stack` 和 `queue`

```cpp
// stack
stack<int> q;
q.push(x);  q.top();  q.pop();  q.size();  q.empty();

// queue
queue<int> q;
q.push(x);  q.pop();  q.front();  q.back();  q.size();  q.empty();

// 指定底层容器
stack<int, vector<int>> third;
queue<int, list<int>> third;
```

栈的底层实现可以是 `vector`、`deque`、`list`（默认 `deque`）。

### 迭代器构造（避免反复扩容）

```cpp
// 通过迭代器直接构造，复杂度 O(N)（而非逐个 push_back 的 O(N·amortized)）
return vector<int>(result_temp.begin(), result_temp.end());
```

### `priority_queue`

```cpp
// 大顶堆（默认）
priority_queue<int> pq;

// 小顶堆
priority_queue<int, vector<int>, greater<int>> min_pq;

// 自定义比较
auto cmp = [](pair<int, int> a, pair<int, int> b) { return a.second > b.second; };
priority_queue<pair<int,int>, vector<pair<int,int>>, decltype(cmp)> pq(cmp);
```

---

## 相关 LeetCode 题目

| 题目 | 模式 |
|------|------|
| 104 二叉树最大深度 | DFS |
| 236 二叉树最近公共祖先 | 后序遍历 |
| 230 BST 第 K 小 | 中序遍历 |
| 108 有序数组转 BST | 分治 + 二分 |
| 222 完全二叉树节点数 | 位运算 + 二分 |
| 2286 音乐会门票 | 线段树 |
| 14 最长公共前缀 | 逐位比较 |
| 3 无重复字符最长子串 | 滑动窗口 |
| 5 最长回文子串 | 字符串 DP / 哈希 |

---

## 树的遍历进阶

### 迭代式中序遍历

```cpp
vector<int> inorderTraversal(TreeNode* root) {
    vector<int> res;
    stack<TreeNode*> st;
    TreeNode* cur = root;
    while (cur || !st.empty()) {
        while (cur) { st.push(cur); cur = cur->left; }
        cur = st.top(); st.pop();
        res.push_back(cur->val);
        cur = cur->right;
    }
    return res;
}
```

### 迭代式前序遍历

```cpp
vector<int> preorderTraversal(TreeNode* root) {
    if (!root) return {};
    vector<int> res;
    stack<TreeNode*> st;
    st.push(root);
    while (!st.empty()) {
        TreeNode* cur = st.top(); st.pop();
        res.push_back(cur->val);
        if (cur->right) st.push(cur->right);  // 先压右
        if (cur->left) st.push(cur->left);    // 后压左
    }
    return res;
}
```

### 迭代式后序遍历

前序"根-左-右"调整为"根-右-左"，最后反转结果：

```cpp
vector<int> postorderTraversal(TreeNode* root) {
    if (!root) return {};
    vector<int> res;
    stack<TreeNode*> st;
    st.push(root);
    while (!st.empty()) {
        TreeNode* cur = st.top(); st.pop();
        res.push_back(cur->val);
        if (cur->left) st.push(cur->left);
        if (cur->right) st.push(cur->right);
    }
    reverse(res.begin(), res.end());
    return res;
}
```

### BFS 层次遍历

```cpp
vector<vector<int>> levelOrder(TreeNode* root) {
    if (!root) return {};
    vector<vector<int>> res;
    queue<TreeNode*> q;
    q.push(root);
    while (!q.empty()) {
        int size = q.size();
        vector<int> level;
        for (int i = 0; i < size; i++) {
            TreeNode* cur = q.front(); q.pop();
            level.push_back(cur->val);
            if (cur->left) q.push(cur->left);
            if (cur->right) q.push(cur->right);
        }
        res.push_back(level);
    }
    return res;
}
```

---

## KMP 字符串匹配

KMP 的核心是利用**部分匹配表 `next`** 来避免主串指针回溯，实现 $O(n+m)$ 的单模式匹配。

### 构建 next 数组

`next[i]` 表示 `pattern[0..i]` 中最长相等前后缀的长度。

```cpp
vector<int> buildNext(const string& p) {
    int m = p.size();
    vector<int> next(m, 0);
    for (int i = 1, j = 0; i < m; i++) {
        while (j > 0 && p[i] != p[j]) j = next[j - 1];
        if (p[i] == p[j]) j++;
        next[i] = j;
    }
    return next;
}
```

### 匹配过程

```cpp
int kmp(const string& s, const string& p) {
    auto next = buildNext(p);
    for (int i = 0, j = 0; i < s.size(); i++) {
        while (j > 0 && s[i] != p[j]) j = next[j - 1];
        if (s[i] == p[j]) j++;
        if (j == p.size()) return i - j + 1;  // 匹配成功
    }
    return -1;
}
```

---

## 二维前缀和

扩展到二维矩阵，$O(1)$ 查询任意子矩阵和：

$$
\text{sum}[x_1,y_1,x_2,y_2] = \text{pre}[x_2][y_2] - \text{pre}[x_1-1][y_2] - \text{pre}[x_2][y_1-1] + \text{pre}[x_1-1][y_1-1]
$$

其中 `pre[i][j]` 表示 $(0,0)$ 到 $(i,j)$ 的矩形区域和。

```cpp
// 构建二维前缀和
vector<vector<int>> build2DPre(const vector<vector<int>>& mat) {
    int r = mat.size(), c = mat[0].size();
    vector<vector<int>> pre(r + 1, vector<int>(c + 1, 0));
    for (int i = 1; i <= r; i++)
        for (int j = 1; j <= c; j++)
            pre[i][j] = pre[i-1][j] + pre[i][j-1] - pre[i-1][j-1] + mat[i-1][j-1];
    return pre;
}

// 查询 (r1,c1) 到 (r2,c2) 的子矩阵和
int query(const auto& pre, int r1, int c1, int r2, int c2) {
    return pre[r2+1][c2+1] - pre[r1][c2+1] - pre[r2+1][c1] + pre[r1][c1];
}
```

---

## 数据结构接口补充

### `map` / `unordered_map`

```cpp
unordered_map<K, V> m;
m[key] = value;              // 插入或覆盖
m.find(key);                 // 返回迭代器，找不到返回 m.end()
m.erase(key);                // 删除
for (auto& [k, v] : m) { }  // 结构绑定遍历
```

### `vector` 常用操作

```cpp
vector<int> v;
v.push_back(x); v.pop_back();
v.size(); v.empty(); v.clear();
v.front(); v.back();
v.insert(v.begin() + i, x);  // 在位置 i 插入
v.erase(v.begin() + i);      // 删除位置 i
sort(v.begin(), v.end());    // 排序
```

### `string` 常用操作

```cpp
string s;
s.substr(pos, len);   // 子串
s.find(t);            // 查找子串位置
s.append(t);          // 追加
s.compare(t);         // 比较
to_string(42);        // 数字转字符串
stoi("42");           // 字符串转 int
```

---

## 补充 LeetCode 题目

| 题目 | 模式 | 关键点 |
|------|------|--------|
| 94 二叉树中序遍历 | 迭代 | 栈模拟 |
| 102 二叉树层次遍历 | BFS | 队列 + 层级计数 |
| 226 翻转二叉树 | DFS | 左右子树交换 |
| 98 验证 BST | 中序遍历 | 比较前驱 |
| 110 平衡二叉树 | 后序遍历 | 高度差 ≤ 1 |
| 543 二叉树直径 | DFS | 全局最大路径 |
| 28 实现 `strStr()` | KMP / 哈希 | 子串匹配 |
| 304 二维区域和检索 | 二维前缀和 | 矩阵不可变 |
