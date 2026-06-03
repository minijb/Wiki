# Exec Plan: Draft Refine v2 — 深度重处理 My_Vault 笔记

> **For agentic workers:** 使用 executing-plans skill 按 Task 逐个执行。每个 Task 对应一个独立领域，由 subagent 自主完成全流程。

**Goal:** 将 `drafts/My_Vault/` 中约 400+ 篇笔记按领域分类，由 subagent 理解、重排、优化、扩展后，写入 `raw/` 并生成 `wiki/` 页面。核心区别于 v1：**不抽取摘要，而是理解后深化**——保留用户原始细节，补充上下文、代码示例、交叉引用。

**Architecture:** 7 个独立 subagent Task，每个覆盖一个不重叠的技术领域。每个 subagent 遵循精简 refine 流程：读取领域内全部 draft → 理解内容体系 → 决定文件拆分/合并策略 → 写入 raw/ → 创建/更新 wiki/ 概念页+来源摘要 → 更新交叉引用。最后串行执行全局 index.md + log.md 更新 + 新建领域 AGENTS.md 同步 + lint 验证。

**新建领域：** 无现有领域的主题，按 AGENTS.md 领域扩展规则创建新 `raw/` 子目录：

| 新领域 | 路径 | 文件数 | 判据 |
|--------|------|:--:|------|
| GPGPU/CUDA | `raw/cs/gpgpu/` | 3+ | 正交于 languages/architecture/algorithms；抽象层级为子领域而非单一技术；预计后续可扩展 OpenCL/Metal 等 |

已有领域扩展（非新建，但文件数大幅增长）：`raw/tools/ai-coding/`（Claude Code 7 篇）、`raw/tools/shell/`（Shell 脚本 5+ 篇）

**Key Decision — 跳过人工讨论环节：** refine skill 标准流程要求逐条与用户讨论。本计划中每个 subagent 自主判断，因为：(1) 笔记为用户自己总结，事实准确性高于 LLM 推测；(2) 用户明确要求"理解重排并优化扩展"；(3) 规模（400+ 文件）不允许逐条讨论。产出后用户审查整体结果。


**内容策略（所有 Task 强制执行）：**

| 原则 | 规则 |
|------|------|
| **保留优先** | 原笔记内容为主，不删减、不扭曲用户原意。所有原始要点必须在新 raw 文件中可追溯 |
| **扩展必须搜索** | 补充任何事实性内容（API 签名、版本差异、性能数据、最佳实践）前，MUST 进行网络搜索验证。禁止凭空编造"典型用法"、"常见模式" |
| **扩展可以大量** | 网络搜索验证后可大量补充：官方文档摘要、社区最佳实践、性能 benchmark、版本演进历史、替代方案对比。扩展内容无上限，以信息密度为准 |
---

## Task 1: C# / .NET 体系

**输入目录：**
- `drafts/My_Vault/02_Knowledge/01_Language/Csharp/`
- `drafts/My_Vault/02_Knowledge/01_Language/Csharp_tools/`
- `drafts/My_Vault/02_Knowledge/01_Language/注意/`
- `drafts/My_Vault/02_Knowledge/01_Language/常用工具/`
- `drafts/My_Vault/02_Knowledge/01_Language/常用接口/`
- `drafts/My_Vault/02_Knowledge/异步+多线程/`
- `drafts/My_Vault/02_Knowledge/依赖注入/`

**主题覆盖：** async/await 状态机、Task/ValueTask、多线程、CancellationToken、struct 装箱/GC、匿名函数 GC、WeakReference/IDisposable、集合接口 (IEnumerable→IList)、序列化 (XML/Stream)、Socket 网络、委托/事件/Attribute、ref/in/out、Process 进程管理、文件 I/O、DI 容器/生命周期

**输出 raw/ 文件：**
- `raw/cs/languages/csharp-async-awaiter.md` — 合并异步+多线程全部草稿，深度扩展状态机原理
- `raw/cs/languages/csharp-memory-gc.md` — 合并装箱/GC/匿名函数/Dispose，补充实战案例
- `raw/cs/languages/csharp-collections.md` — 合并集合接口草稿
- `raw/cs/languages/csharp-serialization.md` — 序列化 + 文件 I/O 合并
- `raw/cs/languages/csharp-socket-network.md` — Socket + TCP 异步
- `raw/cs/languages/csharp-delegates-attributes.md` — 委托/事件/特性
- `raw/cs/architecture/dependency-injection.md` — DI 完整内容
- `raw/cs/languages/csharp-threading.md` — 多线程专题（Thread/Task 演进）

**输出 wiki/ 页面：** 每个 raw 对应 1 concept + 1 source-summary，更新已有页面（非新建），补充交叉引用

**验收标准：**
- [ ] 所有 draft 内容被覆盖（不留遗漏）
- [ ] 每个 raw 文件比 v1 版本增加 ≥30% 内容（代码示例、上下文、对比）
- [ ] wiki concept 页包含完整代码示例和性能考量
- [ ] 双向 [[wikilink]] 完备：C# 异步 ↔ C# 并发模型 ↔ Unity 脚本架构 ↔ C++ 异步对比

---

## Task 2: C++ / Lua / Python / EmmyLua

**输入目录：**
- `drafts/My_Vault/02_Knowledge/01_Language/Lua/`
- `drafts/My_Vault/02_Knowledge/01_Language/基础知识/`（Lua 值传递/UpValue/闭包/循环）
- `drafts/My_Vault/02_Knowledge/01_Language/python/`
- `drafts/My_Vault/02_Knowledge/01_Language/emmylua 环境安装/`
- `drafts/My_Vault/02_Knowledge/CUDA/`
- `drafts/My_Vault/project/Algo/`
- `drafts/My_Vault/project/lan/Shell/`

**主题覆盖：** C++ 移动语义/STL/哈希/并发、Lua table/闭包/UpValue/元表/协程、Python 子进程/文件 IO、EmmyLua 注解系统（~25 种注解）、CUDA 基础、Shell 脚本

**输出 raw/ 文件：**
- `raw/cs/languages/cpp-core-syntax.md` — 基于用户笔记扩展
- `raw/cs/languages/cpp-concurrency.md`
- `raw/cs/languages/lua-core.md`
- `raw/cs/languages/xlua-hotfix.md`
- `raw/cs/languages/python-data-and-misc.md` — 合并 Python 文件操作/子进程
- `raw/cs/languages/emmyLua-annotations-reference.md` — 保留完整 25 种注解语法
- `raw/cs/languages/emmyLua-environment-setup.md`
- `raw/cs/gpgpu/cuda-basics.md` — **新建领域**，CUDA 3 篇草稿合并整理
- `raw/tools/shell/shell-scripting.md` — 合并 Shell 草稿
- `raw/cs/algorithms/*.md` — 算法草稿合并到现有 4 篇

**领域扩展 — `raw/cs/gpgpu/`：** CUDA 无现有合适领域，按 AGENTS.md 规则创建新领域：
- 命名：`gpgpu`（GPGPU 为行业通用术语，kebab-case 合规，单数表示学科）
- 深度：`raw/cs/gpgpu/` = 2 层 ✅
- 文件数：当前 3 篇 CUDA 笔记 + 预期后续可扩展 OpenCL/Metal/Vulkan Compute ✅
- 正交性：GPU 并行计算独立于 languages/architecture/algorithms ✅
- 抽象层级：GPGPU 为子领域，不低于单一技术（CUDA/OpenCL/Metal 均可归入）✅
- 后续 Task 8 同步更新 AGENTS.md 目录树

**验收标准：**
- [ ] EmmyLua 注解参考页保留全部 ~25 种注解，不压缩
- [ ] Lua 值传递/引用传递对比代码完整
- [ ] CUDA 基础内容被保留（不因无合适领域而丢弃）
- [ ] 算法草稿细节合并入现有 4 篇 raw，不丢失

---

## Task 3: 设计模式（22 种 GoF）

**输入目录：**
- `drafts/My_Vault/02_Knowledge/设计模式/创建模式/`（5 文件）
- `drafts/My_Vault/02_Knowledge/设计模式/结构模式/`（7 文件）
- `drafts/My_Vault/02_Knowledge/设计模式/行为模式/`（10 文件）

**重要：** v1 将 22 种模式压缩到 3 个 raw 文件，丢失了大量细节。v2 目标：**保留每种模式的独立 UML、代码示例、适用场景、与其他模式的关系**。

**输出 raw/ 文件（策略：保持 3 文件但大幅扩展）：**
- `raw/cs/design-patterns/creational-patterns.md` — 5 种：工厂方法/抽象工厂/生成器/原型/单例，每种含 UML + 完整 C# 示例 + 变体讨论
- `raw/cs/design-patterns/structural-patterns.md` — 7 种：适配器/桥接/组合/装饰/外观/享元/代理
- `raw/cs/design-patterns/behavioral-patterns.md` — 10 种：责任链/命令/迭代器/中介者/备忘录/观察者/状态/策略/模板方法/访问者

**输出 wiki/ 页面：** 现有 3 concept + 3 source-summary 原地更新，大幅扩充

**验收标准：**
- [ ] 每种模式有独立 H2 章节，含 UML 类图（mermaid）、C# 代码示例、适用场景
- [ ] 模式间关系被标注（如"策略常与工厂组合"）
- [ ] 与 [[concepts/依赖注入]] 的交叉引用

---

## Task 4: Unity 渲染与图形学

**输入目录：**
- `drafts/My_Vault/files/` 中 shader/raytracing 相关文件
- `drafts/My_Vault/06_Learning/ComputerGraphics/`
- `drafts/My_Vault/project/class/`（Games202, Learn Opengl, CS61C 等课程笔记）

**主题覆盖：** Shader 基础/高级、渲染管线 (Games101)、光线追踪 (Ray Tracing in One Weekend)、OpenGL、CS61C 体系结构

**输出 raw/ 文件：** 更新现有 5 篇
- `raw/gamedev/rendering/shader-basics.md`
- `raw/gamedev/rendering/shader-advanced.md`
- `raw/gamedev/rendering/rendering-pipeline-theory.md`
- `raw/gamedev/rendering/ray-tracing-in-one-weekend.md`
- `raw/gamedev/rendering/learn-opengl-notes.md`

**验收标准：**
- [ ] Shader 文件包含用户 `files/` 中的原子笔记细节（不丢失）
- [ ] 光线追踪 5 个章节的笔记全部被覆盖
- [ ] 课程笔记（Games202/LearnOpengl）的关键洞察被保留

---

## Task 5: Unity 引擎核心系统

**输入目录：**
- `drafts/My_Vault/02_Knowledge/Unity相关/`（全部子目录）
- `drafts/My_Vault/02_Knowledge/Dots/`
- `drafts/My_Vault/02_Knowledge/02_Framework/`

**最大 Task — 覆盖约 80+ 文件，跨 15+ 子主题：**

| 子领域 | 输入 | 输出 raw |
|--------|------|----------|
| 帧同步 | `Unity相关/帧同步/` + `KCP/` | `raw/gamedev/networking/unity-frame-sync.md` |
| Socket 网络 | `Unity相关/` (网络相关) | `raw/gamedev/networking/unity-socket-tcp-udp.md` |
| 内存/资源管理 | `Unity相关/内存管理/` | `raw/gamedev/optimization/unity-asset-management.md` |
| 性能优化 | `02_Framework/优化/` | `raw/gamedev/optimization/unity-performance.md` |
| XLua 热更新 | `Unity相关/Xlua/` | `raw/cs/languages/xlua-hotfix.md` |
| ECS/DOTS | `Dots/` (ECS + JobSystem) | `raw/gamedev/gameplay/unity-dots-ecs.md` |
| 动画/Spine | `Unity相关/Spine 动画组件/` | `raw/gamedev/animation/unity-animation-spine.md` |
| 音频 FMOD/LipSync | `Unity相关/FMod/` + `LipSync/` | `raw/gamedev/audio/unity-fmod-lipsync.md` |
| 脚本架构 | `Unity相关/Unity项目结构/` + `Unitask/` + `DOTWeen/` | `raw/gamedev/gameplay/unity-script-architecture.md` |
| API 速查 | `Unity相关/工具函数/` | `raw/gamedev/gameplay/unity-api-cheatsheet.md` |
| 平台交互 | `Unity相关/不同平台和Unity交互/` | `raw/gamedev/gameplay/unity-platform-and-logging.md` |
| 日志系统 | `Unity相关/日志系统/` | 合并入 `unity-platform-and-logging.md` |
| 编辑器扩展 | `02_Framework/编辑器拓展/` + `Unity相关/编辑器拓展/` + `Unity相关/UnPacked/` | `raw/gamedev/editor-extensions/` 下 6 个文件 |
| 程序集/package | `Unity相关/零零散散_files/程序集合package/` | 合并入相关文件 |
| luban | `Unity相关/luban/` | 新增 `raw/gamedev/gameplay/luban-config.md` |

**验收标准：**
- [ ] 帧同步 5 篇草稿（Lockstep/乐观帧/影子跟随/KCP）全部保留细节
- [ ] FMod 22KB 原始笔记的 DSP/Event/Group 细节不丢失
- [ ] LipSync 共振峰方案 27KB 笔记完整保留
- [ ] ECS JobSystem 草稿与现有 raw 合并（非覆盖）
- [ ] 编辑器扩展 6 个 raw 文件与原有内容合并
- [ ] luban 配置工具作为新 raw 文件添加

---

## Task 6: 开发工具 / 环境 / CS 基础

**输入目录：**
- `drafts/My_Vault/02_Knowledge/03_Tools/`（WSL/VSCode/AI_Tools/Tools/Env_Setup/NoteTake/Dotnet/Recourse/Static_Blog）
- `drafts/My_Vault/02_Knowledge/04_CS_Basics/Linux/`
- `drafts/My_Vault/05_StartUp/`
- `drafts/My_Vault/project/Tools/`
- `drafts/My_Vault/project/lan/Shell/`
- `drafts/My_Vault/project/awesome/`
- `drafts/My_Vault/files/` 中的 Linux/Git/VSCode/Docker/工具相关文件

**主题覆盖：** Git 操作、VSCode 配置/调试、WSL2 环境、Linux Shell (Bash/Fish/Zsh/tmux)、Docker、dotnet CLI、CMake、Dev 工具集、Neovim 配置、AI Coding 工具 (Claude Code)、Obsidian 笔记方法、静态博客

**输出 raw/ 文件（更新现有 + 新增）：**
- `raw/tools/git/git-operations-guide.md` — 合并 files/ 中全部 git 原子笔记
- `raw/tools/ide/vscode-config-debug.md`
- `raw/tools/shell/wsl-windows-dev-setup.md`
- `raw/tools/shell/linux-shell-guide.md` — 合并 Shell + Linux 分散笔记
- `raw/tools/ci-cd/cmake-guide.md`
- `raw/tools/ci-cd/dev-tools-misc.md` — 合并 Docker/dotnet/SVN/Obsidian
- `raw/tools/ai-coding/ai-coding-tools.md` — Claude Code MCP/Agents/Skills/记忆系统（新建或扩展现有）
- `raw/cs/architecture/os-fundamentals.md` — 合并 Linux 体系结构笔记

**验收标准：**
- [ ] files/ 中约 50+ Git 原子笔记被合并整理（不丢失操作细节）
- [ ] Claude Code 7 篇笔记整理为系统化文档
- [ ] Linux shell 分散笔记 (bash/zsh/fish/tmux/fzf/rg/fd/lazygit) 统一编排
- [ ] VSCode 多 Profile/tasks/launch/Vim 插件细节完整

---

## Task 7: 面试 / 职业（可选，待确认）

**输入目录：**
- `drafts/My_Vault/04_Career/`
- `drafts/My_Vault/files/` 中面试相关文件（约 30+ 篇）

**注意：** 此领域涉及个人面试经历，是否公开到 wiki 需确认。默认策略：技术性面试题（算法/C#/Unity/图形学）归入对应技术领域 raw 文件作为"常见面试问题"补充章节；个人面试实录保持归档状态。

**若执行：**
- 提取通用技术面试题 → 补充对应概念页的"面试要点"章节
- 个人经历 → 保留在 `raw/_archived-vault/interview-experiences/`

---

## Task 8: 全局收尾（串行，在全部 Task 完成后）

**前置：** Task 1-7 全部完成

**步骤：**
- [ ] **Step 8.1: 更新 wiki/index.md** — 全量重写，反映所有新增/更新页面，按 Concepts/Entities/Source Summaries/Comparisons 分节
- [ ] **Step 8.2: 更新 wiki/log.md** — 追加 v2 refine 操作日志
- [ ] **Step 8.3: 更新 AGENTS.md** — 如有新建 raw/ 领域，同步目录树
- [ ] **Step 8.4: QMD 索引重建**
  ```bash
  qmd update && env -u CI qmd embed
  ```
- [ ] **Step 8.5: Lint 检查**
  ```bash
  env -u CI qmd query "断链检查" -n 5  # 辅以手动遍历
  ```
  - 验证：断链 = 0，孤立页面 = 0

**验收标准：**
- [ ] index.md 包含所有 wiki 页面条目
- [ ] QMD status 显示文档数与预期一致
- [ ] Lint 零问题

---
## 风险与缓解

| 风险 | 概率 | 影响 | 缓解措施 |
|------|------|------|---------|
| Subagent 理解偏差导致内容错误 | 中 | 高 | 保留原内容为主；扩展部分强制网络搜索验证；用户可抽查关键文件
| Task 间交叉引用遗漏 | 中 | 中 | Task 8 lint 环节专门检查交叉引用 |
| raw/ 文件合并策略不当（过粗/过细） | 中 | 中 | 每个 Task 的 assignment 中明确文件拆分策略 |
| 新建领域 AGENTS.md 同步遗漏 | 低 | 中 | Task 8 Step 8.3 专门负责同步；/skill:lint 可检测不匹配
| 文件数量过大导致 subagent 超时 | 中 | 中 | Task 5（Unity 核心）最大，必要时拆分为 5a/5b |

## 验收标准（全局）

- [ ] 所有 `02_Knowledge/` 下 draft 内容在 raw/ 中有对应覆盖，无遗漏
- [ ] 每个 raw 文件比 v1 版本实质性增长（更多代码示例、更多上下文、更多交叉引用）
- [ ] wiki/ 概念页从"摘要"升级为"深度文档"
- [ ] `files/` 中技术性原子笔记被归入对应 raw 文件
- [ ] Lint 零问题，index.md/log.md 已更新
- [ ] QMD 索引最新
