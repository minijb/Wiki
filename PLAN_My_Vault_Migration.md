# Exec Plan: My_Vault → Wiki 知识库迁移

> **For agentic workers:** 使用 `executing-plans` skill 按 Task 逐个执行。
> 每个 Task 使用独立 subagent，通过 `/skill:refine` → `/skill:ingest` 流程完成。

**Goal:** 将 `drafts/My_Vault/` 中约 500+ 条笔记按程序员知识体系分类，经 `refine` 打磨后归档到 `raw/`，再经 `ingest` 创建 wiki 页面（sources/concepts/entities/comparisons），更新 `wiki/index.md` 和 `wiki/log.md`。面试笔记中的技术内容提取后分散到各领域 Task；个人面试经历和废弃内容（私有工作代码、个人待办、模板、配置）移入 `raw/_archived-vault/` 按类别归档。

**Architecture:** 按技术领域分 11 个 Task（取消独立面试 Task），每个 Task 对应一个独立 subagent。统一工作流：盘点该领域所有笔记（含该领域面试笔记）→ 对每个碎片文件显式分类（分配到具体 raw 文件，不设通用碎片 catch-all）→ `/skill:refine` 打磨（深化内容、合并碎片、去重）→ 归档到 `raw/` → `/skill:ingest` 创建 wiki 页面。不迁移内容移入 `raw/_archived-vault/<category>/`。

**Tech Stack:** 纯 Markdown 处理；现有 skills（refine, ingest, obsidian-md, lint）；QMD 检索

---

## 领域 → raw/ 目录映射

| 领域 | raw/ 目标 | AGENTS.md 已有？ |
|------|----------|-----------------|
| C# 语言 | `raw/cs/languages/` | ✅ |
| C++ 语言 | `raw/cs/languages/` | ✅ |
| Lua/EmmyLua | `raw/cs/languages/` | ✅ |
| 数据结构与算法 | `raw/cs/algorithms/` | ✅ |
| 设计模式 | `raw/cs/design-patterns/` | ✅ |
| 计算机体系结构/OS | `raw/cs/architecture/` | ✅ |
| Unity 渲染/Shader | `raw/gamedev/rendering/` | ✅ |
| Unity 性能优化 | `raw/gamedev/optimization/` | ✅ |
| Unity 网络/帧同步 | `raw/gamedev/networking/` | ✅ |
| Unity 脚本/架构 | `raw/gamedev/gameplay/` | ✅ |
| Unity 动画/音频 | `raw/gamedev/animation/` + `raw/gamedev/audio/` | ✅ |
| 图形学 | `raw/gamedev/rendering/` | ✅ |
| Git | `raw/tools/git/` | ✅ |
| VSCode/IDE | `raw/tools/ide/` | ✅ |
| Linux/WSL/Shell | `raw/tools/shell/` | 🆕 需新增目录 + 更新 AGENTS.md |
| Docker | `raw/tools/docker/` | ✅ |
| CMake/Build | `raw/tools/ci-cd/` | ✅ |
| Python | `raw/cs/languages/` | ✅ |
| 不迁移存档 | `raw/_archived-vault/` | 🆕 需创建，按类别分子目录 |

---

## Task 1: C# 语言深度

> **领域:** C# 语法、异步模型、内存管理、集合、文件/进程/网络 I/O
> **raw/ 目标:** `raw/cs/languages/`
> **产出预估:** 8-10 raw 文件 → 8-10 source summaries + 5-8 concept pages

**源笔记清单（含面试笔记）:**
- `drafts/My_Vault/02_Knowledge/01_Language/Csharp/Csharp_Struct 如何减少装箱造成的GC.md` (1.2KB)
- `drafts/My_Vault/02_Knowledge/01_Language/Csharp_tools/Thread/` — 子目录内容
- `drafts/My_Vault/02_Knowledge/01_Language/Csharp_tools/Process/` — 子目录内容
- `drafts/My_Vault/02_Knowledge/01_Language/注意/00_CSharp_匿名函数GC问题.md` (1.3KB)
- `drafts/My_Vault/02_Knowledge/01_Language/常用接口/Tech_Csharp_常用接口_集合接口.md` (2.4KB)
- `drafts/My_Vault/02_Knowledge/01_Language/常用工具/00_CSharp_进程.md` (617B)
- `drafts/My_Vault/02_Knowledge/01_Language/常用工具/01_CSharp_文件处理.md` (283B)
- `drafts/My_Vault/02_Knowledge/异步+多线程/` — 全部 7 文件
- `drafts/My_Vault/02_Knowledge/依赖注入/00_依赖注入_背景.md` (16.4KB)
- `drafts/My_Vault/files/C Sharp 异步以及Awaiter.md` (10.4KB)
- `drafts/My_Vault/files/C sharp xml序列化_反序列化.md` (9.7KB)
- `drafts/My_Vault/files/C sharp xml.md` (6.1KB)
- `drafts/My_Vault/files/C sharp byte.md` (9.1KB)
- `drafts/My_Vault/files/C Sharp stream 1.md` (4.4KB)
- `drafts/My_Vault/files/C sharp socket recieve and beginReceive.md` (3.2KB)
- `drafts/My_Vault/files/C Sharp List优化 -- table.md` (3.2KB)
- `drafts/My_Vault/files/C sharpe Event, Delegates, Action.md` (1.9KB)
- `drafts/My_Vault/files/C Sharp Attribute.md` (1.2KB)
- `drafts/My_Vault/files/C sharp stopWatch.md` (769B)
- `drafts/My_Vault/files/C Sharp WeakReference.md` (413B)
- `drafts/My_Vault/files/C sharp List扩容.md` (605B)
- `drafts/My_Vault/files/C Sharp IDispose.md` (579B)
- `drafts/My_Vault/files/C Sharp TimeSpan.md` (580B)
- `drafts/My_Vault/files/C sharp 中 in， out 和 ref 关键字.md` (557B)
- `drafts/My_Vault/files/C_Sharp_IList_IReadOnlyList.md` (536B)
- `drafts/My_Vault/project/lan/C sharpe/` — 子目录内容
- `drafts/My_Vault/project/C sharp list.md` (568B) — 索引文件
- **面试笔记（提取技术内容后合并入对应 raw）：**
  - `drafts/My_Vault/files/面试 - csharp.md` (2.4KB) → I/O 与序列化 raw
  - `drafts/My_Vault/files/面试 - 多线程 task.md` (8.3KB) → 异步模型 raw
  - `drafts/My_Vault/files/面试实录 -- Csharp.md` (7.2KB) → 分散到各子主题 raw
  - `drafts/My_Vault/files/面试准备.md` (12.9KB) — 提取 C# 相关部分

**碎片文件（<500B，需显式分类到目标 raw）:**
| 碎片 | 大小 | → 目标 raw |
|------|------|-----------|
| `C sharp 库.md` | 71B | → `csharp-serialization.md` |
| `C sharp 重写list.md` | 191B | → `csharp-collections.md` |
| `C sharp uri.md` | 184B | → `csharp-socket-network.md` |
| `C sharp dotnet命令简单总结.md` | 114B | → `csharp-serialization.md` |
| `C Sharp 文件处理.md` | 107B | → `csharp-serialization.md` |
| `C sharp 中的 线程及同步异步.md` | 124B | → `csharp-async-awaiter.md` |
| `C Sharp stream 2.md` | 120B | → `csharp-serialization.md` |

**Steps:**
- [ ] **Step 1: 盘点与碎片分类** — 读取全部源文件，按子主题归类（异步模型、内存/GC、集合、I/O、XML/序列化、委托/事件、依赖注入）。每个碎片文件必须标注目标 raw。标记与已有 wiki（[[concepts/CSharp并发模型]]、[[concepts/CSharp文件IO]]、[[concepts/CSharp进程管理]]、[[concepts/CSharp值类型性能]]）的重复/互补关系。
  - 验证：输出分类清单（每个子主题下：源文件路径 + 大小 + 一句话摘要 + 碎片去向），确认无遗漏

- [ ] **Step 2: `/skill:refine` — C# 异步模型** — 打磨：`C Sharp 异步以及Awaiter.md` + `02_Knowledge/异步+多线程/` 全部 + `面试 - 多线程 task.md` + `C sharp 中的 线程及同步异步.md`
  - 目标 raw: `raw/cs/languages/csharp-async-awaiter.md`
  - 深化要求：补充 async/await 状态机原理、Task/ValueTask 对比、同步上下文、常见死锁场景
  - 面试精华提取：将面试 Q&A 中的易错点融入正文
  - 验证：raw ≥ 10KB，含完整代码示例

- [ ] **Step 3: `/skill:refine` — C# 内存与 GC** — 打磨：`Csharp_Struct 如何减少装箱造成的GC.md` + `00_CSharp_匿名函数GC问题.md` + `C Sharp WeakReference.md` + `C Sharp IDispose.md` + `面试 - Unity GC.md`（提取 C# 相关部分）
  - 目标 raw: `raw/cs/languages/csharp-memory-gc.md`
  - 注意：已有 [[concepts/CSharp值类型性能]]，去重并交叉引用
  - 验证：覆盖 struct/class 分配差异、闭包GC陷阱、WeakReference/IDisposable 模式

- [ ] **Step 4: `/skill:refine` — C# 集合与泛型** — 打磨：`Tech_Csharp_常用接口_集合接口.md` + `C Sharp List优化 -- table.md` + `C sharp List扩容.md` + `C_Sharp_IList_IReadOnlyList.md` + `C sharp 重写list.md`
  - 目标 raw: `raw/cs/languages/csharp-collections.md`
  - 验证：覆盖 IEnumerable→IList→IReadOnlyList 层次、List 内部数组扩容、自定义集合

- [ ] **Step 5: `/skill:refine` — C# I/O、序列化与网络** — 注意已有 [[concepts/CSharp文件IO]] 和 [[concepts/CSharp进程管理]]。合并新内容 + 面试笔记
  - 拆分两个 raw：
    - `raw/cs/languages/csharp-serialization.md` — XML序列化、字节处理、文件 I/O 补充
    - `raw/cs/languages/csharp-socket-network.md` — Socket 网络编程
  - 碎片吸附：`C sharp 库.md`、`C sharp dotnet命令简单总结.md`、`C Sharp 文件处理.md`、`C Sharp stream 2.md` → 附在序列化 raw
  - `C sharp uri.md` → 附在网络 raw
  - 验证：每个 ≥ 8KB

- [ ] **Step 6: `/skill:refine` — C# 委托、事件、Attribute、ref/in/out** — 打磨：`C sharpe Event, Delegates, Action.md` + `C Sharp Attribute.md` + `C sharp 中 in， out 和 ref 关键字.md` + `面试实录 -- Csharp.md`（提取相关部分）
  - 目标 raw: `raw/cs/languages/csharp-delegates-attributes.md`
  - 验证：覆盖 delegate/event/Action/Func、自定义 Attribute/反射读取、ref/in/out 内存语义

- [ ] **Step 7: `/skill:refine` — C# 依赖注入** — 打磨：`02_Knowledge/依赖注入/00_依赖注入_背景.md`
  - 目标 raw: `raw/cs/architecture/dependency-injection.md`
  - 深化：DI 容器原理、生命周期（Transient/Scoped/Singleton）、与 Strategy/Factory 模式关系
  - 验证：≥ 10KB

- [ ] **Step 8: `/skill:ingest` — 批量摄取** — 对 Step 2-7 产出的全部 raw 执行 `/skill:ingest`
  - 每个 raw → `wiki/sources/` 摘要；`wiki/concepts/` 概念页
  - 新建概念页：C#异步模型、C#集合架构、C#序列化、C#委托与事件、C#依赖注入、C# Socket 网络 等
  - 与已有 concept pages 建立双向 [[wikilink]]
  - 验证：每个 raw 有 source summary；每个 concept page 有 ≥3 个 wikilink

- [ ] **Step 9: 更新 index.md + log.md**
  - 验证：`search` 确认 index.md 新增条目数 = 产出页面数；log.md 末尾追加本次记录

---

## Task 2: C++ 语言与 CMake

> **领域:** C++ 语法、STL、移动语义、多线程/异步、CMake 构建
> **raw/ 目标:** `raw/cs/languages/` + `raw/tools/ci-cd/`
> **产出预估:** 4-5 raw 文件 → 4-5 source summaries + 3-4 concept pages

**源笔记清单（含面试笔记）:**
- `drafts/My_Vault/files/cpp 移动.md` (2.2KB)
- `drafts/My_Vault/files/cpp thread - 1.md` (6.0KB)
- `drafts/My_Vault/files/c++ 异步编程.md` (19.4KB)
- `drafts/My_Vault/files/cpp hash.md` (2.2KB)
- `drafts/My_Vault/files/cpp priority_queue.md` (1.1KB)
- `drafts/My_Vault/files/stl_基础.md` (1.9KB)
- `drafts/My_Vault/files/cpp 程序设计原理和实践(1).md` (1.7KB)
- `drafts/My_Vault/files/CMake Tutorial.md` (4.8KB)
- `drafts/My_Vault/files/CMake Professtional-1 Introduction.md` (4.6KB)
- `drafts/My_Vault/files/CMake Professtional-2 Variables.md` (2.4KB)
- `drafts/My_Vault/project/Tools/CMake.md` (5.0KB)
- `drafts/My_Vault/project/cpp List.md` (329B) — 索引
- `drafts/My_Vault/project/cpp 多线程.md` (203B)
- **面试笔记（提取技术内容后合并）：**
  - `drafts/My_Vault/files/面试 - c++.md` (6.7KB) → 核心语法 raw
  - `drafts/My_Vault/files/面试实录 -- c++.md` (6.9KB) → 核心语法 raw

**碎片分类:**
| 碎片 | 大小 | → 目标 raw |
|------|------|-----------|
| `cpp tuple.md` | 124B | → `cpp-core-syntax.md` |
| `cpp tips.md` | 147B | → `cpp-core-syntax.md` |
| `cpp 奇奇怪怪的API.md` | 181B | → `cpp-core-syntax.md` |
| `cpp_stl.md` | 183B | → `cpp-core-syntax.md` |
| `cpp 性能检测.md` | 219B | → `cpp-concurrency.md` |
| `vcpkg.md` | 167B | → `cmake-guide.md` |

**Steps:**
- [ ] **Step 1: 盘点与碎片分类** — 归类：C++ 核心语法、C++ 多线程/异步、CMake 构建。每个碎片标注目标 raw。
  - 验证：输出分类清单 + 碎片去向表

- [ ] **Step 2: `/skill:refine` — C++ 核心语法** — 打磨：`cpp 移动.md` + `cpp hash.md` + `cpp priority_queue.md` + `stl_基础.md` + `cpp 程序设计原理和实践(1).md` + 全部碎片 + `面试 - c++.md` + `面试实录 -- c++.md`
  - 目标 raw: `raw/cs/languages/cpp-core-syntax.md`
  - 深化：移动语义完整示例、哈希自定义、STL 容器选择指南
  - 验证：≥ 10KB

- [ ] **Step 3: `/skill:refine` — C++ 多线程与异步** — 打磨：`cpp thread - 1.md` + `c++ 异步编程.md` + `cpp 多线程.md` + `cpp 性能检测.md`
  - 目标 raw: `raw/cs/languages/cpp-concurrency.md`
  - 验证：≥ 10KB，覆盖 std::thread/future/async

- [ ] **Step 4: `/skill:refine` — CMake 构建系统** — 打磨：`CMake Tutorial.md` + `CMake Professtional-1.md` + `CMake Professtional-2.md` + `project/Tools/CMake.md` + `vcpkg.md`
  - 目标 raw: `raw/tools/ci-cd/cmake-guide.md`
  - 深化：target-based 现代 CMake 最佳实践
  - 验证：≥ 8KB

- [ ] **Step 5: `/skill:ingest`** → wiki

- [ ] **Step 6: 更新 index.md + log.md**

---

## Task 3: Lua 与 EmmyLua 注解

> **领域:** Lua 语法、闭包/UpValue、xlua 热更新、EmmyLua 类型注解
> **raw/ 目标:** `raw/cs/languages/`
> **产出预估:** 2-3 raw 文件 → 2-3 source summaries + 1-2 concept pages

**源笔记清单（含面试笔记）:**
- `drafts/My_Vault/02_Knowledge/01_Language/基础知识/00_Lua_资源.md` (192B)
- `drafts/My_Vault/02_Knowledge/01_Language/基础知识/01_Lua_循环.md` (2.1KB)
- `drafts/My_Vault/02_Knowledge/01_Language/基础知识/02_Lua_值传递和引用传递.md` (2.0KB)
- `drafts/My_Vault/02_Knowledge/01_Language/基础知识/03_Lua_ UpValue 和 闭包.md` (592B)
- `drafts/My_Vault/02_Knowledge/Unity相关/Xlua/` — 全部 5 文件
- `drafts/My_Vault/files/xlua 速通.md` (4.5KB)
- `drafts/My_Vault/files/Unity - lua 1 basic.md` (7.5KB)
- `drafts/My_Vault/files/Unity - lua 2 xlua基础.md` (7.7KB)
- `drafts/My_Vault/files/Unity - lua 3 lua调用c.md` (8.1KB)
- `drafts/My_Vault/files/Unity - lua 4 hotfix.md` (4.4KB)
- **面试笔记（提取技术内容后合并）：**
  - `drafts/My_Vault/files/面试 - lua.md` (16.3KB) → Lua 语法核心 raw
  - `drafts/My_Vault/files/面试实录 -- lua.md` (831B) → Lua 语法核心 raw
- **EmmyLua（wiki 已有 [[entities/EmmyLua]] + [[concepts/EmmyLua注解系统]]）：**
  - `drafts/My_Vault/02_Knowledge/01_Language/emmylua 环境安装/` — 4 文件（比对 wiki 覆盖度）

**碎片分类:**
| 碎片 | 大小 | → 目标 raw |
|------|------|-----------|
| `00_Lua_资源.md` | 192B | → `lua-core.md` 作为参考链接 |

**Steps:**
- [ ] **Step 1: 盘点与碎片分类** — Lua 语法基础 vs xlua/Unity 集成 vs EmmyLua 注解。比对 EmmyLua 已有 wiki 覆盖度。
  - 验证：输出分类清单 + EmmyLua 覆盖度对比

- [ ] **Step 2: `/skill:refine` — Lua 语法核心** — 打磨：`基础知识/` 全部 + `面试 - lua.md`（提取技术干货，去 Q&A 格式）+ `面试实录 -- lua.md`
  - 目标 raw: `raw/cs/languages/lua-core.md`
  - 深化：table 实现原理、闭包与 UpValue、元表/metatable、协程
  - 验证：≥ 8KB

- [ ] **Step 3: `/skill:refine` — xlua 与 Unity 热更新** — 打磨：`Xlua/` 全部 + `xlua 速通.md` + `Unity - lua 1-4.md`
  - 目标 raw: `raw/cs/languages/xlua-hotfix.md`
  - 验证：覆盖 C#↔Lua 互调、热补丁原理、配置与性能

- [ ] **Step 4: 检查 EmmyLua wiki 覆盖** — 若 `emmylua 环境安装/` 有未覆盖的注解类型 → 更新已有 raw 并重新 ingest；若已覆盖 → 跳过
  - 验证：确认无知识遗漏

- [ ] **Step 5: `/skill:ingest`** → wiki

- [ ] **Step 6: 更新 index.md + log.md**

---

## Task 4: 数据结构与算法

> **领域:** 算法基础、数据结构、LeetCode 题型模板
> **raw/ 目标:** `raw/cs/algorithms/`
> **产出预估:** 4-5 raw 文件 → 4-5 source summaries + 3-4 concept pages

**源笔记清单（含面试笔记）:**
- `drafts/My_Vault/files/algo 线段树.md` (192B)
- `drafts/My_Vault/files/algo_字符串.md` (1.2KB)
- `drafts/My_Vault/files/algo 完全二叉树.md` (978B)
- `drafts/My_Vault/files/algo 单调栈队列.md` (450B)
- `drafts/My_Vault/files/algo 回溯.md` (1.2KB)
- `drafts/My_Vault/files/algo 单调栈.md` (307B)
- `drafts/My_Vault/files/algo 前缀和，差分和.md` (604B)
- `drafts/My_Vault/files/algo 优先队列.md` (354B)
- `drafts/My_Vault/files/algo 二分查找.md` (513B)
- `drafts/My_Vault/files/algo c++ 常用接口.md` (1.3KB)
- `drafts/My_Vault/files/DataStruct 二叉树.md` (962B)
- `drafts/My_Vault/files/DataStruct heap and priority queue.md` (498B)
- `drafts/My_Vault/files/代码随想录.md` (318B)
- `drafts/My_Vault/files/代码随想录--思路.md` (2.2KB)
- `drafts/My_Vault/project/Algo/动态规划.md` (1.7KB)
- `drafts/My_Vault/project/Algo/algo 常用技巧.md` (266B)
- `drafts/My_Vault/project/algo List.md` (326B) — 索引
- **面试笔记（提取技术内容后合并）：**
  - `drafts/My_Vault/files/面试 - 算法.md` (26.3KB) → 提取通用方法论到 interview-patterns raw
  - `drafts/My_Vault/files/面试 - 排序算法.md` (3.8KB) → 排序与搜索 raw
  - `drafts/My_Vault/files/面试 - 回溯.md` (8.8KB) → DP 与回溯 raw
  - `drafts/My_Vault/files/面试 - 动态规划.md` (1.3KB) → DP 与回溯 raw
  - `drafts/My_Vault/files/面试 - 数学.md` (3.7KB) → 杂项 raw
  - `drafts/My_Vault/files/面试实录 -- 算法.md` (324B) → 碎片随附

**碎片分类:**
| 碎片 | 大小 | → 目标 raw |
|------|------|-----------|
| `algo 线段树.md` | 192B | → `trees-and-strings.md` |
| `algo 单调栈.md` | 307B | → `sort-and-search.md` |
| `algo 单调栈队列.md` | 450B | → `sort-and-search.md` |
| `algo 优先队列.md` | 354B | → `sort-and-search.md` |
| `代码随想录.md` | 318B | → `dp-and-backtracking.md` |
| `algo 常用技巧.md` | 266B | → `trees-and-strings.md` |
| `algo List.md` | 326B | 丢弃（仅索引） |

**Steps:**
- [ ] **Step 1: 盘点与碎片分类** — 按主题归类：排序/搜索、DP/回溯、树/字符串、综合。每个碎片标注目标 raw。
  - 验证：输出归类清单 + 碎片去向表

- [ ] **Step 2: `/skill:refine` — 排序与搜索** — 合并：`面试 - 排序算法.md` + `algo 二分查找.md` + `algo 单调栈队列.md` + `algo 单调栈.md` + `algo 优先队列.md` + `DataStruct heap and priority queue.md`
  - 目标 raw: `raw/cs/algorithms/sort-and-search.md`
  - 验证：≥ 8KB

- [ ] **Step 3: `/skill:refine` — 动态规划与回溯** — 合并：`面试 - 动态规划.md` + `面试 - 回溯.md` + `algo 回溯.md` + `project/Algo/动态规划.md` + `代码随想录.md` + `代码随想录--思路.md`
  - 目标 raw: `raw/cs/algorithms/dp-and-backtracking.md`
  - 验证：≥ 10KB

- [ ] **Step 4: `/skill:refine` — 树、字符串与其他** — 合并：`DataStruct 二叉树.md` + `algo 完全二叉树.md` + `algo 线段树.md` + `algo_字符串.md` + `algo 前缀和，差分和.md` + `algo c++ 常用接口.md` + `面试 - 数学.md` + `algo 常用技巧.md`
  - 目标 raw: `raw/cs/algorithms/trees-and-strings.md`
  - 验证：≥ 8KB

- [ ] **Step 5: `/skill:refine` — 算法面试通用方法论** — 从 `面试 - 算法.md` (26.3KB) 提取通用解题框架、复杂度分析套路、易错点，去 Q&A 格式
  - 目标 raw: `raw/cs/algorithms/interview-patterns.md`
  - 验证：≥ 8KB

- [ ] **Step 6: `/skill:ingest`** → wiki

- [ ] **Step 7: 更新 index.md + log.md**

---

## Task 5: 设计模式

> **领域:** GoF 23 种设计模式 + 游戏编程模式
> **raw/ 目标:** `raw/cs/design-patterns/`
> **产出预估:** 3 raw 文件 → 3 source summaries + 3 concept pages

**源笔记清单（含面试笔记）:**
- `drafts/My_Vault/02_Knowledge/设计模式/行为模式/` — 10 文件
- `drafts/My_Vault/02_Knowledge/设计模式/结构模式/` — 7 文件
- `drafts/My_Vault/02_Knowledge/设计模式/创建模式/` — 5 文件
- `drafts/My_Vault/files/创建模式.md` (2.6KB)
- **面试笔记（提取技术内容后合并）：**
  - `drafts/My_Vault/files/面试 - 设计模式.md` (2.5KB) → 行为模式 raw

**碎片分类:**
| 碎片 | 大小 | → 目标 raw |
|------|------|-----------|
| `行为模式.md` | 180B | → `behavioral-patterns.md` 作为概览 |
| `结构模式.md` | 180B | → `structural-patterns.md` 作为概览 |
| `project/设计模式.md` | 405B | 丢弃（索引） |

**Steps:**
- [ ] **Step 1: 盘点** — 22 个模式文件 vs GoF 23 种对照，确认缺失模式（如解释器模式）
  - 验证：输出已有 vs GoF 23 对照表

- [ ] **Step 2: `/skill:refine` — 创建模式** — 合并打磨：`创建模式/` 全部 5 文件 + `创建模式.md`
  - 目标 raw: `raw/cs/design-patterns/creational-patterns.md`
  - 验证：覆盖 5 种模式，每种含 UML + C#/Unity 示例

- [ ] **Step 3: `/skill:refine` — 结构模式** — 合并打磨：`结构模式/` 全部 7 文件 + `结构模式.md`
  - 目标 raw: `raw/cs/design-patterns/structural-patterns.md`
  - 验证：覆盖 7 种模式

- [ ] **Step 4: `/skill:refine` — 行为模式** — 合并打磨：`行为模式/` 全部 10 文件 + `行为模式.md` + `面试 - 设计模式.md`
  - 目标 raw: `raw/cs/design-patterns/behavioral-patterns.md`
  - 验证：覆盖 ≥10 种模式

- [ ] **Step 5: `/skill:ingest`** → wiki；与 [[concepts/CSharp依赖注入]]（如已创建）建立链接

- [ ] **Step 6: 更新 index.md + log.md**

---

## Task 6: 计算机体系结构与操作系统

> **领域:** CSAPP、OS、虚拟内存、链接、异常控制流
> **raw/ 目标:** `raw/cs/architecture/`
> **产出预估:** 3-4 raw 文件 → 3-4 source summaries + 2-3 concept pages

**源笔记清单（含面试笔记）:**
- `drafts/My_Vault/files/CSAPP-1 C programme and System.md` (11.2KB)
- `drafts/My_Vault/files/CSAPP-2 Bit Byte and Integer.md` (5.3KB)
- `drafts/My_Vault/files/CSAPP-3 float.md` (789B)
- `drafts/My_Vault/files/CSAPP-4 machine-level.md` (9.2KB)
- `drafts/My_Vault/files/CSAPP-5 处理器设计.md` (8.5KB)
- `drafts/My_Vault/files/CSAPP-6 程序优化.md` (2.4KB)
- `drafts/My_Vault/files/CSAPP-7 存储结构.md` (4.2KB)
- `drafts/My_Vault/files/CSAPP-8 链接.md` (7.6KB)
- `drafts/My_Vault/files/CSAPP-9 异常控制流.md` (8.1KB)
- `drafts/My_Vault/files/CSAPP-10 虚拟内存.md` (9.0KB)
- `drafts/My_Vault/files/CSAPP - debug.md` (4.5KB)
- `drafts/My_Vault/files/CSAPP-Assert.md` (1.2KB)
- `drafts/My_Vault/files/Operating System(nanjin)-0 assert.md` (1.4KB)
- `drafts/My_Vault/files/Operating System(nanjin)-1 简单介绍.md` (5.0KB)
- `drafts/My_Vault/files/Operating System(nanjin)-2 硬件和数学视角.md` (1021B)
- `drafts/My_Vault/files/Operating System(nanjin) Pro-0 实验须知.md` (2.3KB)
- `drafts/My_Vault/files/OS-tep cpu.md` (4.7KB)
- `drafts/My_Vault/files/C 内存分配.md` (1.3KB)
- `drafts/My_Vault/files/linux--虚拟内存.md` (1.1KB)
- `drafts/My_Vault/files/CS61C-1 basic_C.md` (486B)
- `drafts/My_Vault/project/class/` — 课程索引（CSAPP, MIT6.S081, CS61C 等）
- **面试笔记：**
  - `drafts/My_Vault/files/面试 - 操作系统.md` (10.7KB) → OS 核心概念 raw

**碎片分类:**
| 碎片 | 大小 | → 目标 raw |
|------|------|-----------|
| `CSAPP Lab2 - bomb.md` | 106B | → `csapp-systems.md` |
| `S081 - L1.md` | 171B | → `os-fundamentals.md` |
| `OS-tep asset.md` | 290B | → `os-fundamentals.md` |

**Steps:**
- [ ] **Step 1: 盘点与碎片分类** — CSAPP 系列（13文件）、OS 杂项、CS61C、课程索引。碎片标注目标 raw。
  - 验证：输出分类清单 + 碎片去向表

- [ ] **Step 2: `/skill:refine` — CSAPP 核心笔记** — 按章节拆分 2 个 raw：
  - `raw/cs/architecture/csapp-program-structure.md` (Ch 1-4: C程序结构、数据表示、机器级)
  - `raw/cs/architecture/csapp-systems.md` (Ch 5-10: 优化、存储、链接、异常控制流、虚拟内存)
  - 验证：每个 ≥ 10KB

- [ ] **Step 3: `/skill:refine` — OS 核心概念** — 合并：OS nanjing 系列 + OS-tep + linux虚拟内存 + C内存分配 + S081 + CS61C + `面试 - 操作系统.md`
  - 目标 raw: `raw/cs/architecture/os-fundamentals.md`
  - 验证：≥ 10KB

- [ ] **Step 4: `/skill:ingest`** → wiki

- [ ] **Step 5: 更新 index.md + log.md**

---

## Task 7: Unity 渲染与图形学

> **领域:** Shader 编程、渲染管线、光照模型、光线追踪、Games101/202/104 课程
> **raw/ 目标:** `raw/gamedev/rendering/`
> **产出预估:** 5-6 raw 文件 → 5-6 source summaries + 4-5 concept pages

**源笔记清单（含面试笔记）:**
- `drafts/My_Vault/files/Shader1 -- 基础.md` (36.1KB)
- `drafts/My_Vault/files/Shader2 -- 透明 和 一些进阶.md` (7.3KB)
- `drafts/My_Vault/files/Shader3 -- 表面着色器+曲面细分.md` (3.6KB)
- `drafts/My_Vault/files/Shader4 -- 几何着色器.md` (7.3KB)
- `drafts/My_Vault/files/shader_入门精要--1_shader_基础.md` (5.5KB)
- `drafts/My_Vault/files/shader_入门精要--2_标准光照模型.md` (10.7KB)
- `drafts/My_Vault/files/shader 入门精要--3 纹理.md` (10.0KB)
- `drafts/My_Vault/files/shader 入门精要--4 透明.md` (6.5KB)
- `drafts/My_Vault/files/shader 入门精要--5 复杂光源.md` (10.8KB)
- `drafts/My_Vault/files/unity - shader.md` (11.6KB)
- `drafts/My_Vault/files/unity 四元数.md` (526B)
- `drafts/My_Vault/files/LearnOpengl 1 - windows.md` (9.7KB)
- `drafts/My_Vault/files/LearnOpengl 2 - Shader.md` (2.4KB)
- `drafts/My_Vault/files/ray tracing in one weekend - 1.md` (9.4KB)
- `drafts/My_Vault/files/ray tracing in one weekend - 2.md` (15.7KB)
- `drafts/My_Vault/files/ray tracing in one weekend - 3.md` (12.4KB)
- `drafts/My_Vault/files/ray tracing in one weekend - 4.md` (8.4KB)
- `drafts/My_Vault/files/ray tracing in one weekend - 5.md` (21.3KB)
- `drafts/My_Vault/files/Games101-1~11` — 12 文件 + 环境搭建/Work/AABB
- `drafts/My_Vault/files/Games104 - 0~1` — 2 文件
- `drafts/My_Vault/files/Games202 -1 Basic.md` (984B)
- `drafts/My_Vault/06_Learning/ComputerGraphics/学习大纲.md` (11.5KB)
- `drafts/My_Vault/03_Resources/Code_Snippets/forward_render.md` (4.2KB)
- `drafts/My_Vault/03_Resources/Code_Snippets/Shadow shader.md` (4.1KB)
- `drafts/My_Vault/project/work/一些优化/shader 优化.md` (1.0KB)
- **面试笔记：**
  - `drafts/My_Vault/files/面试实录 -- 图形学.md` (10.2KB) → 渲染管线理论 raw

**碎片分类:**
| 碎片 | 大小 | → 目标 raw |
|------|------|-----------|
| `Games101 -- AABB.md` | 246B | → `rendering-pipeline-theory.md` |
| `Games101 环境搭建.md` | 621B | → `rendering-pipeline-theory.md` |
| `Games101Work-3.md` | 328B | → `rendering-pipeline-theory.md` |
| `Games104 - 0 assert.md` | 485B | → `rendering-pipeline-theory.md` |
| `Games104 - 1 分层.md` | 271B | → `rendering-pipeline-theory.md` |
| `Learn Opengl Asset.md` | 117B | → `learn-opengl-notes.md` |
| `unity 四元数.md` | 526B | → `shader-basics.md`（数学基础） |
| `Games202.md` (project/class/) | 189B | → `rendering-pipeline-theory.md` |
| `Learn Opengl.md` (project/class/) | 263B | → `learn-opengl-notes.md` |
| `ray tracing in one weekend.md` (project/Games/) | 404B | → `ray-tracing-in-one-weekend.md` |
| `Shader List.md` | 312B | 丢弃（索引） |

**Steps:**
- [ ] **Step 1: 盘点与碎片分类** — Shader 编程、渲染管线理论（Games101/104/202）、光线追踪、OpenGL。每个碎片标注目标 raw。
  - 验证：输出分类清单 + 碎片去向表

- [ ] **Step 2: `/skill:refine` — Shader 编程** — 合并：Shader1-4 + shader 入门精要 1-5 + unity - shader.md + code snippets + shader 优化
  - 拆为 2 个 raw：`raw/gamedev/rendering/shader-basics.md` + `raw/gamedev/rendering/shader-advanced.md`
  - 验证：每个 ≥ 12KB

- [ ] **Step 3: `/skill:refine` — 渲染管线理论** — 合并 Games101 全部 + Games104 + Games202 + 学习大纲 + `面试实录 -- 图形学.md`
  - 目标 raw: `raw/gamedev/rendering/rendering-pipeline-theory.md`
  - 验证：≥ 20KB

- [ ] **Step 4: `/skill:refine` — 光线追踪** — 合并 ray tracing in one weekend 1-5
  - 目标 raw: `raw/gamedev/rendering/ray-tracing-in-one-weekend.md`
  - 验证：≥ 15KB

- [ ] **Step 5: `/skill:refine` — OpenGL 学习** — 合并 LearnOpengl 1-2
  - 目标 raw: `raw/gamedev/rendering/learn-opengl-notes.md`
  - 验证：≥ 8KB

- [ ] **Step 6: `/skill:ingest`** → wiki

- [ ] **Step 7: 更新 index.md + log.md**

---

## Task 8: Unity 引擎综合

> **领域:** Unity 性能优化、网络/帧同步、脚本架构、资源管理、DOTS/ECS、音频/动画
> **raw/ 目标:** `raw/gamedev/optimization/`, `raw/gamedev/networking/`, `raw/gamedev/gameplay/`, `raw/gamedev/animation/`, `raw/gamedev/audio/`
> **产出预估:** 7-8 raw 文件 → 7-8 source summaries + 5-7 concept pages

**注意:** 编辑器扩展系列已在 wiki（6 concept pages），跳过 `02_Knowledge/02_Framework/编辑器拓展/` 和 `02_Knowledge/Unity相关/编辑器拓展/`。

**源笔记清单（含面试笔记）:**

**性能优化:**
- `drafts/My_Vault/02_Knowledge/02_Framework/优化/00_Unity_优化_语言层面.md` (740B)
- `drafts/My_Vault/02_Knowledge/02_Framework/优化/01_Unity_优化_字符串优化.md` (1.7KB)
- `drafts/My_Vault/files/unity的批处理.md` (2.5KB)
- `drafts/My_Vault/files/Unity Draw Call.md` (632B)
- `drafts/My_Vault/files/unity UI优化.md` (355B)
- `drafts/My_Vault/files/Unity - Profile使用热点地位.md` (1.4KB)
- `drafts/My_Vault/project/work/一些优化/优化方法.md` (4.7KB)

**网络/帧同步:**
- `drafts/My_Vault/02_Knowledge/Unity相关/帧同步/` — 5 文件
- `drafts/My_Vault/files/Unity Socket - 1.md` (17.1KB)
- `drafts/My_Vault/files/Unity Socket - 2.md` (1.6KB)
- `drafts/My_Vault/files/Socket-1.md` (6.2KB)
- `drafts/My_Vault/files/Socket-2 TCP 服务器.md` (7.5KB)
- `drafts/My_Vault/files/Socket-3 UDP 服务器.md` (1.7KB)
- `drafts/My_Vault/02_Knowledge/Unity相关/KCP/` — 2 文件
- `drafts/My_Vault/files/unity - 帧同步资料.md` (542B)
- **面试笔记：**
  - `drafts/My_Vault/files/面试 - 帧同步.md` (5.1KB) → 帧同步 raw

**脚本架构/Gameplay:**
- `drafts/My_Vault/02_Knowledge/Unity相关/Unitask/` — 子目录
- `drafts/My_Vault/02_Knowledge/Unity相关/DOTWeen/` — 2 文件
- `drafts/My_Vault/files/Unity - uniTask.md` (9.9KB)
- `drafts/My_Vault/files/Game 架构.md` (2.0KB)
- `drafts/My_Vault/02_Knowledge/Unity相关/Unity项目结构/00_概述.md` (4.3KB)
- `drafts/My_Vault/02_Knowledge/Unity相关/luban/` — 1 文件
- `drafts/My_Vault/04_Career/00_游戏研发流程与团队组成.md` (5.5KB)

**资源管理:**
- `drafts/My_Vault/02_Knowledge/Unity相关/内存管理/` — 4 文件
- `drafts/My_Vault/files/Unity - AssetBundle.md` (9.3KB)
- `drafts/My_Vault/files/unity 动态资源加载.md` (3.6KB)
- **面试笔记：**
  - `drafts/My_Vault/files/面试 - AssetBundle-1~6` — 6 文件 → AssetBundle raw
  - `drafts/My_Vault/files/面试 - Unity GC.md` (2.2KB) → 性能和资源 raw 各提取相关部分

**DOTS/ECS:**
- `drafts/My_Vault/02_Knowledge/Dots/` — ECS, JobSystem
- `drafts/My_Vault/02_Knowledge/02_Framework/Dots/` — JobSystem
- `drafts/My_Vault/files/Unity ECS and DOTS - 1.md` (207B)
- `drafts/My_Vault/files/Untiy - ECS 1.md` (973B)

**音频:**
- `drafts/My_Vault/02_Knowledge/Unity相关/FMod/` — 3 文件
- `drafts/My_Vault/02_Knowledge/Unity相关/LipSync/00_LipSync_共振峰方案.md` (27.6KB)

**动画:**
- `drafts/My_Vault/02_Knowledge/Unity相关/Spine 动画组件/` — 子目录
- **面试笔记：**
  - `drafts/My_Vault/files/面试 - unity 动画.md` (5.7KB) → 动画 raw

**碎片分类:**
| 碎片 | 大小 | → 目标 raw |
|------|------|-----------|
| `unity 音频资源优化.md` | 99B | → `unity-performance.md` |
| `unity 资源策略优化.md` | 278B | → `unity-asset-management.md` |
| `Mirror1.md` | 84B | → `unity-frame-sync.md` |
| `unity - 帧同步资料.md` | 542B | → `unity-frame-sync.md` |
| `game development -- flow.md` | 475B | → `unity-script-architecture.md` |
| `Game 背包系统.md` | 53B | → `unity-script-architecture.md` |
| `Unity ECS and DOTS - 1.md` | 207B | → `unity-dots-ecs.md` |
| `unity animation.md` | 552B | → `unity-animation-spine.md` |
| `Socket list.md` | 92B | 丢弃（索引） |

**Steps:**
- [ ] **Step 1: 盘点与碎片分类** — 按 7 个子领域分类。标记已有 wiki 覆盖的编辑器扩展。每个碎片标注目标 raw。从 `面试 - unity.md` (13.5KB) 和 `面试 - unity基础.md` (22.8KB) 中提取技术干货分散到各子领域 raw。
  - 验证：输出完整分类清单 + 碎片去向表

- [ ] **Step 2: `/skill:refine` — 性能优化** — 合并性能笔记 + `面试 - unity基础.md`(提取性能部分) + `面试 - Unity GC.md`(提取 GC 部分)
  - 目标 raw: `raw/gamedev/optimization/unity-performance.md`
  - 验证：≥ 10KB，覆盖 Draw Call/Batch、字符串/GC、UI优化、Profiler

- [ ] **Step 3: `/skill:refine` — 网络与帧同步** — 拆为 2 个 raw：
  - `raw/gamedev/networking/unity-socket-tcp-udp.md` — Socket 编程
  - `raw/gamedev/networking/unity-frame-sync.md` — 帧同步 + KCP + `面试 - 帧同步.md`
  - 验证：每个 ≥ 8KB

- [ ] **Step 4: `/skill:refine` — 脚本架构与 Gameplay** — 合并：UniTask/DOTWeen/架构/项目结构/游戏研发流程 + `面试 - unity基础.md`(提取架构部分)
  - 目标 raw: `raw/gamedev/gameplay/unity-script-architecture.md`
  - 验证：≥ 10KB

- [ ] **Step 5: `/skill:refine` — 资源管理与 AssetBundle** — 合并：内存管理 + AssetBundle + 动态加载 + 面试 AssetBundle 1-6
  - 目标 raw: `raw/gamedev/optimization/unity-asset-management.md`
  - 验证：≥ 15KB

- [ ] **Step 6: `/skill:refine` — DOTS/ECS** — 合并 ECS + JobSystem
  - 目标 raw: `raw/gamedev/gameplay/unity-dots-ecs.md`
  - 验证：≥ 6KB

- [ ] **Step 7: `/skill:refine` — 音频 (FMod/LipSync)** — 合并音频笔记
  - 目标 raw: `raw/gamedev/audio/unity-fmod-lipsync.md`
  - 验证：≥ 10KB

- [ ] **Step 8: `/skill:refine` — 动画 (Animation/Spine)** — 合并动画笔记 + `面试 - unity 动画.md`
  - 目标 raw: `raw/gamedev/animation/unity-animation-spine.md`
  - 验证：≥ 6KB

- [ ] **Step 9: `/skill:ingest`** → wiki

- [ ] **Step 10: 更新 index.md + log.md**

---

## Task 9: Unity 杂项与碎片

> **领域:** Unity 单一 API 技巧、小 tips、UI 组件用法、平台交互、日志
> **raw/ 目标:** `raw/gamedev/gameplay/`
> **产出预估:** 2 raw 文件 → 2 source summaries + 1-2 concept pages

**源笔记清单（含面试笔记）:**
- `drafts/My_Vault/files/unity 判断点是否在UI范围内.md` (1.6KB) → API 速查
- `drafts/My_Vault/files/unity transform and gameobj.md` (520B) → API 速查
- `drafts/My_Vault/files/Unity Scriptable object.md` (939B) → API 速查
- `drafts/My_Vault/files/Unity SerializeField and Serializable.md` (799B) → API 速查
- `drafts/My_Vault/files/unity - autohook 自动绑定.md` (1.7KB) → API 速查
- `drafts/My_Vault/files/Unity - 单例.md` (466B) → API 速查
- `drafts/My_Vault/files/Unity - struct 栈实验.md` (1.2KB) → API 速查
- `drafts/My_Vault/files/Unity - transformer.md` (1.2KB) → API 速查
- `drafts/My_Vault/files/Unity - api - SerializedProperty.md` (826B) → API 速查
- `drafts/My_Vault/files/Unity - Canvas Scaler.md` (589B) → API 速查
- `drafts/My_Vault/files/Unity - Content Size Fitter.md` (899B) → API 速查
- `drafts/My_Vault/files/Unity - 闭包+匿名函数造成的GC.md` (882B) → API 速查
- `drafts/My_Vault/files/Unity API - InputSystem.md` (612B) → API 速查
- `drafts/My_Vault/files/Untiy Input System.md` (1.3KB) → API 速查
- `drafts/My_Vault/files/unity拓展1 简介.md` (5.0KB) → API 速查
- `drafts/My_Vault/files/Unity高级编程-1 简介及C Sharp.md` (725B) → API 速查
- `drafts/My_Vault/02_Knowledge/Unity相关/工具函数/` — 子目录 → API 速查
- `drafts/My_Vault/02_Knowledge/Unity相关/不同平台和Unity交互/` — 子目录 → 平台交互 raw
- `drafts/My_Vault/02_Knowledge/Unity相关/日志系统/` — 2 文件 → 平台交互 raw
- `drafts/My_Vault/project/work/使用 Android, IOS 方法回调.md` (2.2KB) → 平台交互 raw
- `drafts/My_Vault/03_Resources/CheatSheets/Unity/` — 子目录 → API 速查
- `drafts/My_Vault/03_Resources/Res_Unity.md` (852B) → API 速查
- `drafts/My_Vault/03_Resources/遇到的坑/Unity/00_KEN_unity_实例化材质球问题.md` (665B) → API 速查
- **面试笔记：**
  - `drafts/My_Vault/files/面试 - UGUI.md` (4.8KB) → API 速查
  - `drafts/My_Vault/files/面试实录 -- unity.md` (6.3KB) → 分散到 API 速查和平台交互

**碎片分类（≤200B 的极小文件）:**
| 碎片 | 大小 | → 目标 raw |
|------|------|-----------|
| `unity 制作font.md` | 111B | → `unity-api-cheatsheet.md` |
| `unity plugin 自动复制项目.md` | 129B | → `unity-api-cheatsheet.md` |
| `unity select editor.md` | 149B | → `unity-api-cheatsheet.md` |
| `Unity UI 基础知识.md` | 291B | → `unity-api-cheatsheet.md` |
| `unity - 编辑器扩展1.md` | 223B | → `unity-api-cheatsheet.md` |
| `unity - 小tips.md` | 524B | → `unity-api-cheatsheet.md` |
| `Unity - 导入资源自动设置.md` | 75B | → `unity-api-cheatsheet.md` |
| `Unity - 代码打开资源+获取代码的结构.md` | 400B | → `unity-api-cheatsheet.md` |
| `Unity - time.md` | 288B | → `unity-api-cheatsheet.md` |
| `Unity - PropertyAttribute, PropertyDrawer.md` | 344B | → `unity-api-cheatsheet.md` |
| `Unity - Nolita.md` | 264B | → `unity-api-cheatsheet.md` |
| `Unity A star.md` | 124B | → `unity-api-cheatsheet.md` |
| `Unity API.md` | 168B | → `unity-api-cheatsheet.md` |
| `Unity EventSystem.md` | 54B | → `unity-api-cheatsheet.md` |
| `Unity json.md` | 297B | → `unity-api-cheatsheet.md` |
| `面试 - UI 使用和优化.md` | 228B | → `unity-api-cheatsheet.md` |
| `Unity list.md` (project/) | 2.4KB | 丢弃（索引） |

**Steps:**
- [ ] **Step 1: 盘点与碎片分类** — 按两组归类：UI 组件/序列化/Input/常用 API → API 速查；平台交互/日志 → 平台交互。每个碎片标注目标 raw。
  - 验证：输出分组清单 + 碎片去向表

- [ ] **Step 2: `/skill:refine` — Unity 常用 API 速查表** — 合并全部 API/UI/序列化/Input 碎片 + `面试 - UGUI.md` + `面试实录 -- unity.md`(提取 API 部分)
  - 目标 raw: `raw/gamedev/gameplay/unity-api-cheatsheet.md`
  - 内容：按类别（UI组件/序列化/Input/Transform/常见模式）组织的小型速查表
  - 验证：≥ 10KB

- [ ] **Step 3: `/skill:refine` — Unity 平台交互与日志** — 合并平台交互 + 日志系统 + `面试实录 -- unity.md`(提取平台部分)
  - 目标 raw: `raw/gamedev/gameplay/unity-platform-and-logging.md`
  - 验证：≥ 6KB

- [ ] **Step 4: `/skill:ingest`** → wiki

- [ ] **Step 5: 更新 index.md + log.md**

---

## Task 10: 开发工具

> **领域:** Git、VSCode、Linux/WSL/Shell、Docker、Protobuf/SVN/其他
> **raw/ 目标:** `raw/tools/git/`, `raw/tools/ide/`, `raw/tools/shell/`, `raw/tools/docker/`, `raw/tools/ci-cd/`
> **本次需新建:** `raw/tools/shell/`（+ 同步更新 AGENTS.md）
> **产出预估:** 5-6 raw 文件 → 5-6 source summaries + 3-4 concept pages

**源笔记清单:**

**Git:**
- `drafts/My_Vault/files/git 白名单.md` (351B)
- `drafts/My_Vault/files/git 初始化.md` (1000B)
- `drafts/My_Vault/files/git 分支操作.md` (380B)
- `drafts/My_Vault/files/git 分支管理.md` (1.0KB)
- `drafts/My_Vault/files/git stash.md` (968B)
- `drafts/My_Vault/files/git submodule.md` (1.4KB)
- `drafts/My_Vault/files/git tag.md` (752B)
- `drafts/My_Vault/files/git proxy.md` (300B)
- `drafts/My_Vault/files/git rebase.md` (1.3KB)
- `drafts/My_Vault/files/git merge.md` (1.7KB)
- `drafts/My_Vault/files/git cherry-pick.md` (901B)
- `drafts/My_Vault/files/git commit and log.md` (489B)
- `drafts/My_Vault/files/git delete and revert.md` (1.3KB)
- `drafts/My_Vault/files/git branch managment.md` (1.0KB) — 与分支管理重复
- `drafts/My_Vault/project/Tools/git overview.md` (622B)

**VSCode:**
- `drafts/My_Vault/files/vscode 多配置文件.md` (575B)
- `drafts/My_Vault/files/vscode 在cpp调试中查看泛型.md` (416B)
- `drafts/My_Vault/files/vscode vim.md` (2.5KB)
- `drafts/My_Vault/files/vscode tasks.md` (4.8KB)
- `drafts/My_Vault/files/vscode launch.json.md` (5.0KB)
- `drafts/My_Vault/files/vscode leetcode 插件.md` (1.2KB)
- `drafts/My_Vault/02_Knowledge/03_Tools/VSCode/00_Vscode_Neovim.md` (2.2KB)
- `drafts/My_Vault/01_Projects/Vscode/` — 2 文件

**Linux/Shell/WSL → `raw/tools/shell/`（🆕）:**
- `drafts/My_Vault/files/linux-ssh.md` (63B)
- `drafts/My_Vault/files/linux-neovim 安装.md` (427B)
- `drafts/My_Vault/files/linux-proxy.md` (1.6KB)
- `drafts/My_Vault/files/linux-python 环境问题.md` (217B)
- `drafts/My_Vault/files/linux-lazygit.md` (420B)
- `drafts/My_Vault/files/linux-mysql.md` (273B)
- `drafts/My_Vault/files/linux-fishshell.md` (4.7KB)
- `drafts/My_Vault/files/linux - zsh.md` (282B)
- `drafts/My_Vault/files/linux 安装.md` (185B)
- `drafts/My_Vault/files/linux - fzf.md` (48B)
- `drafts/My_Vault/files/shell -- 永久变量.md` (273B)
- `drafts/My_Vault/files/Bash tutorial1.md` (7.5KB)
- `drafts/My_Vault/files/终端--多路复用.md` (201B)
- `drafts/My_Vault/files/wsl 环境配置.md` (46B)
- `drafts/My_Vault/files/wsl 常用命令.md` (125B)
- `drafts/My_Vault/files/windows 常用软件.md` (487B)
- `drafts/My_Vault/files/windows scoop.md` (2.3KB)
- `drafts/My_Vault/files/arch -- pacman.md` (183B)
- `drafts/My_Vault/files/alacritty.md` (184B)
- `drafts/My_Vault/files/Ubuntu StartUp.md` (265B)
- `drafts/My_Vault/02_Knowledge/03_Tools/WSL/00_Knowledge_Tool_WSL_速览表.md` (1.8KB)
- `drafts/My_Vault/02_Knowledge/04_CS_Basics/Linux/Linux简单笔记.md` (1.5KB)
- `drafts/My_Vault/05_StartUp/StartUp_Windows_wsl2.md` (3.0KB)
- `drafts/My_Vault/05_StartUp/StartUp_AI_Environment.md` (1.5KB)
- `drafts/My_Vault/05_StartUp/StartUp_常用工具_NVM.md` (1.5KB)
- `drafts/My_Vault/project/bash List.md` (204B) — 索引
- `drafts/My_Vault/project/neovim List.md` (571B) — 索引
- `drafts/My_Vault/project/awesome/nvim_plugin.md` (730B)
- `drafts/My_Vault/project/awesome/zsh.md` (157B)
- `drafts/My_Vault/Archive/Window_Powershell_升级.md` (814B)
- `drafts/My_Vault/Archive/Window_Scoop 安装、更新、使用.md` (3.7KB)

**Docker:**
- `drafts/My_Vault/files/Docker 1 基本概念.md` (257B)
- `drafts/My_Vault/project/Tools/Docker.md` (182B)

**其他工具:**
- `drafts/My_Vault/files/protobuf 教程.md` (9.5KB)
- `drafts/My_Vault/files/protobuf 安装.md` (473B)
- `drafts/My_Vault/02_Knowledge/03_Tools/Dotnet/00_Tools_dotnet_常用命令.md` (296B)
- `drafts/My_Vault/02_Knowledge/03_Tools/Env_Setup/常用软件安装.md` (989B)
- `drafts/My_Vault/02_Knowledge/03_Tools/NoteTake/` — 5 文件
- `drafts/My_Vault/01_Projects/SVN/00_svn_基础知识.md` (740B)

**碎片分类:**
| 碎片 | 大小 | → 目标 raw |
|------|------|-----------|
| `git 白名单.md` | 351B | → `git-operations-guide.md` |
| `git 相对引用.md` | 169B | → `git-operations-guide.md` |
| `git proxy.md` | 300B | → `git-operations-guide.md` |
| `git List.md` (project/) | 121B | 丢弃（索引） |
| `vscode variable substitution.md` | 141B | → `vscode-config-debug.md` |
| `vscode setting.md` | 142B | → `vscode-config-debug.md` |
| `vscode assert.md` | 95B | → `vscode-config-debug.md` |
| `linux-ssh.md` | 63B | → 目标 raw 注：几乎所有 Shell 碎片都 <500B，合并到 2 个 raw 中具体分类由 subagent 在 Step 1 产出分类表 |
| `linux - fzf.md` | 48B | → `linux-shell-guide.md` |
| `wsl 环境配置.md` | 46B | → `wsl-windows-setup.md` |
| `wsl 常用命令.md` | 125B | → `wsl-windows-setup.md` |
| `wsl 速记.md` (00_Inbox) | 42B | → `wsl-windows-setup.md` |
| `Docker 1 基本概念.md` | 257B | → `dev-tools-misc.md` |
| `project/Tools/Docker.md` | 182B | → `dev-tools-misc.md` |
| `xmake.md` | 167B | → `dev-tools-misc.md` |
| `donet 安装.md` | 100B | → `dev-tools-misc.md` |

**Steps:**
- [ ] **Step 0: 创建目录** — 在 Step 1 前执行：
  ```bash
  mkdir -p raw/tools/shell
  ```
  - 提醒：执行后需用户手动将 `raw/tools/shell/` 添加到 `AGENTS.md` 目录树（`raw/tools/shell/ # Shell、终端、WSL 环境`）

- [ ] **Step 1: 盘点与碎片分类** — 按 Git/VSCode/Shell/Docker/其他 五组。Shell 组需进一步拆为 Linux 基础 + WSL/Windows 环境两子组。每个碎片标注目标 raw。
  - 验证：输出五组分类清单 + 碎片去向表

- [ ] **Step 2: `/skill:refine` — Git 操作指南** — 合并 15 个 Git 文件，去重（`git branch managment.md` 与 `git 分支管理.md`）
  - 目标 raw: `raw/tools/git/git-operations-guide.md`
  - 验证：≥ 12KB，覆盖分支/合并/变基/储藏/子模块/标签/撤销

- [ ] **Step 3: `/skill:refine` — VSCode 配置与调试** — 合并 10 个 VSCode 文件
  - 目标 raw: `raw/tools/ide/vscode-config-debug.md`
  - 验证：≥ 8KB

- [ ] **Step 4: `/skill:refine` — Linux/Shell/WSL 环境** — 合并约 35 个文件，拆 2 个 raw：
  - `raw/tools/shell/linux-shell-guide.md` — fish/zsh/bash/shell 变量/多路复用/包管理/lazygit/fzf 等
  - `raw/tools/shell/wsl-windows-dev-setup.md` — WSL2 配置、Windows Scoop、常用软件、AI 环境、NVM
  - 验证：每个 ≥ 10KB

- [ ] **Step 5: `/skill:refine` — 其他工具** — 合并：Docker + protobuf + SVN + 笔记工具 + dotnet 命令 + 环境配置 + 静态博客
  - 目标 raw: `raw/tools/ci-cd/dev-tools-misc.md`
  - 验证：≥ 8KB

- [ ] **Step 6: `/skill:ingest`** → wiki

- [ ] **Step 7: 更新 index.md + log.md**

---

## Task 11: Python 与杂项语言

> **领域:** Python 基础、OpenCV/Matplotlib、MySQL、JavaScript
> **raw/ 目标:** `raw/cs/languages/`
> **产出预估:** 1-2 raw 文件 → 1-2 source summaries + 1 concept page

**源笔记清单:**
- `drafts/My_Vault/02_Knowledge/01_Language/python/00_Python_运行命令行.md` (2.0KB) — wiki 已有 [[concepts/Python子进程管理]]，检查重复
- `drafts/My_Vault/02_Knowledge/01_Language/python/01_Python_文件的增删改查.md` (767B) — wiki 已有 [[concepts/Python文件IO模型]]，检查重复
- `drafts/My_Vault/files/python 性能检测.md` (226B)
- `drafts/My_Vault/files/python常见图片格式-读取方法-相互转换.md` (1.5KB)
- `drafts/My_Vault/files/opencv简单教程.md` (455B)
- `drafts/My_Vault/files/matplotlib.md` (1.3KB)
- `drafts/My_Vault/files/linux-mysql.md` (273B)
- `drafts/My_Vault/project/Mysql.md` (1.6KB)
- `drafts/My_Vault/files/mordenjs1 basic.md` (2.5KB)
- `drafts/My_Vault/project/lan/Shell/` — 子目录（已在 Task 10 覆盖）

**Steps:**
- [ ] **Step 1: 盘点** — wiki 已有 [[concepts/Python子进程管理]] + [[concepts/Python文件IO模型]]，检查重复。碎片少直接合并。
  - 验证：输出清单 + 重复标记

- [ ] **Step 2: `/skill:refine` — Python 扩展 & 杂项** — 打磨：性能检测 + 图片处理 + OpenCV + Matplotlib + MySQL + JavaScript
  - 目标 raw: `raw/cs/languages/python-data-and-misc.md`
  - 关联已有 Python concept pages
  - 验证：≥ 6KB

- [ ] **Step 3: `/skill:ingest`** → wiki

- [ ] **Step 4: 更新 index.md + log.md**

---

## 不迁移内容 → `raw/_archived-vault/`

以下内容不进入 wiki 知识库，移至 `raw/_archived-vault/` 按类别归档：

### Step 0: 创建存档目录结构

```bash
mkdir -p raw/_archived-vault/{work-projects,personal-projects,personal-notes,templates-config,interview-experiences,excalidraw,pdf-attachments,obsidian-config}
```

| 子目录 | 内容 | 来源路径 |
|--------|------|---------|
| `work-projects/` | 工作项目私有代码 | `project/work/` 全部（UniNode/painting/UGCScriptable/SpaceTree/component 等） |
| `personal-projects/` | 个人游戏项目 | `01_Projects/战旗/`, `01_Projects/三国杀/` |
| `personal-notes/` | 个人待办/目标/备忘录 | `00_Inbox/备忘录.md`, `TODO.md`, `need to do.md`, `Now.md`, `AI 思路.md`, `wiki.md`, `学习资源清单.md`；`04_Career/目标.md` |
| `templates-config/` | Obsidian 模板 | `88_Templates/` 全部 |
| `interview-experiences/` | 个人面试经历（非技术知识） | `files/米哈游突破 -- 面试.md`, `files/游卡 -- 面试.md`, `files/面试 - 经验.md`, `files/面试 - 项目.md`, `files/面试 - 别人的.md`, `files/面试现场问题.md`, `04_Career/面试实录.md`, `project/面试.md` |
| `excalidraw/` | Excalidraw 图（Obsidian 插件格式） | `Excalidraw/` 全部，`99_Assets/` 中的 `.excalidraw.md` |
| `pdf-attachments/` | PDF 书籍 | `03_Resources/PDF_Books/` 全部 |
| `obsidian-config/` | Obsidian 配置 | `.obsidian/`, `.makemd/` |

**操作约束:**
- 移动（非复制）文件到 `raw/_archived-vault/`，保持 `drafts/My_Vault/` 清洁
- 空文件 `Claude Code Agents 与 Commands.md` (0B) 直接删除
- `03_Resources/Bookmarks/` 浏览器书签导出 → 移入 `personal-notes/`
- `Archive/` 已归档的历史笔记 → 移入 `personal-notes/`
- `project/` 中的 List/索引文件全部丢弃

**验证：** `drafts/My_Vault/` 中仅保留尚未处理的 Task 对应的源文件，其余已移入 `raw/` 或 `raw/_archived-vault/`

---

## 新增 raw/ 目录清单

执行过程中需创建的目录：

| 目录 | 用途 | 需更新 AGENTS.md？ |
|------|------|-------------------|
| `raw/tools/shell/` | Shell、终端、WSL 环境 | ✅ 添加 `shell/ # Shell、终端、WSL 环境` 到 tools 子树 |
| `raw/_archived-vault/` | 不迁移内容存档 | ❌ 内部目录，不纳入知识库索引 |

---

## 风险与缓解

| 风险 | 概率 | 影响 | 缓解措施 |
|------|------|------|---------|
| refine 时发现大量 notes 内容过空无法深化 | 高 | 中 | 合并为速查表而非丢弃；标注"待补充" |
| 面试笔记与领域笔记大量重复 | 高 | 低 | 提取独特视角后合并，重复内容去重 |
| 碎片笔记太多导致 Task 膨胀 | 中 | 中 | 每个 Task Step 1 必须产出碎片分类表；<200B 必须合并到目标 raw |
| 已有 wiki 页面与新内容矛盾 | 低 | 高 | 每个 Task Step 1 先搜索 `wiki/concepts/` 已有页面 |
| subagent 执行 refine 时漏掉文件 | 中 | 高 | 每个 Task Step 1 输出完整文件清单；Step final 前验证 |
| `raw/tools/shell/` 创建后用户未更新 AGENTS.md | 中 | 低 | Task 10 Step 0 中显式提醒；lint 会在最后检测 |

## 验收标准

- [ ] `drafts/My_Vault/` 中有知识价值的笔记已分类 → `raw/` 对应领域
- [ ] 每个 raw 文件 ≥ 5KB（合并后）且内容经过 deepen
- [ ] 每个 raw 在 `wiki/sources/` 有对应来源摘要
- [ ] 每个 raw 的重要概念在 `wiki/concepts/` 或 `wiki/entities/` 有页面
- [ ] 所有 wiki 页面间有 `[[wikilink]]` 双向引用
- [ ] `wiki/index.md` 和 `wiki/log.md` 已更新
- [ ] `/skill:lint` 通过（无断链、无孤立页面、AGENTS.md 一致）
- [ ] 碎片笔记全部显式分类到目标 raw，无通用 catch-all
- [ ] 不迁移内容全部移入 `raw/_archived-vault/<category>/`
- [ ] `raw/tools/shell/` 已创建，AGENTS.md 已更新
- [ ] `project/work/` 私有代码未被迁移
- [ ] 面试笔记技术内容已分散到各领域 raw，个人经历已归档
