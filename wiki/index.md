---
title: "Wiki 索引"
type: index
updated: 2026-06-02
---

# Wiki 索引

按页面类型组织的知识库目录。

## Concepts

### C# / .NET
- [[concepts/CSharp异步模型|CSharp 异步模型]] — async/await 状态机、Awaiter 模式、Task/ValueTask、SyncContext、死锁
- [[concepts/CSharp内存GC|CSharp 内存 GC]] — struct 装箱、匿名函数 GC 陷阱、WeakReference、IDisposable
- [[concepts/CSharp集合框架|CSharp 集合框架]] — IEnumerable→IList 层次、List 扩容、Span、ArrayPool
- [[concepts/CSharp序列化IO|CSharp 序列化 IO]] — XML 序列化、Stream 分层、字节处理、文件 I/O
- [[concepts/CSharp网络Socket|CSharp 网络 Socket]] — TCP 异步接收、BeginReceive、自定义协议
- [[concepts/CSharp委托特性|CSharp 委托特性]] — delegate/event/Action/Func、Attribute 反射、ref/in/out
- [[concepts/依赖注入|依赖注入]] — DI 容器原理、生命周期管理、与 Strategy/Factory 关系
- [[concepts/CSharp并发模型|C# 并发模型]] — Thread→Task→async/await→Channel 演进与线程安全
- [[concepts/CSharp文件IO|C# 文件 I/O]] — File/FileStream 分层架构、真正异步 I/O、缓冲区优化
- [[concepts/CSharp进程管理|C# 进程管理]] — Process 类、stdout/stderr 并发读取防死锁、协作式取消
- [[concepts/CSharp值类型性能|C# 值类型与 GC]] — 装箱消除、readonly/ref struct、零分配优化策略

### C++ / CMake
- [[concepts/C++核心语法|C++ 核心语法]] — 移动语义、STL 容器、哈希自定义、值类别
- [[concepts/C++并发与异步|C++ 并发与异步]] — std::thread/future/async、原子操作、协程
- [[concepts/现代CMake构建|现代 CMake 构建]] — target-based 设计、变量作用域、vcpkg 集成

### Lua / EmmyLua
- [[concepts/Lua核心特性|Lua 核心特性]] — table 实现、闭包/UpValue、元表、协程、值传递
- [[concepts/XLua热补丁|XLua 热补丁]] — C#↔Lua 互调、IL 注入原理、配置与性能
- [[concepts/EmmyLua注解系统|EmmyLua 注解系统]] — 通过注释为动态类型 Lua 添加静态类型标注

### 数据结构与算法
- [[concepts/算法-排序与搜索|排序与搜索]] — 排序全家桶、二分查找、单调栈/队列、优先队列
- [[concepts/算法-动态规划与回溯|动态规划与回溯]] — DP 经典题型、回溯模板、贪心、代码随想录
- [[concepts/算法-树与字符串|树与字符串]] — 二叉树、线段树、字符串哈希、KMP、前缀和/差分
- [[concepts/算法-面试通用方法论|算法面试通用方法论]] — 图论、链表/双指针/滑动窗口/栈/哈希、数学几何

### 设计模式
- [[concepts/设计模式-创建型|创建型模式]] — 工厂方法/抽象工厂/生成器/原型/单例 — 含 UML + C# 示例
- [[concepts/设计模式-结构型|结构型模式]] — 适配器/桥接/组合/装饰/外观/享元/代理
- [[concepts/设计模式-行为型|行为型模式]] — 责任链/命令/迭代器/中介者/备忘录/观察者/状态/策略/模板方法/访问者

### 计算机体系结构 / OS
- [[concepts/CSAPP程序结构与机器级表示|CSAPP 程序与机器]] — C 程序结构、数据表示、机器级编程
- [[concepts/CSAPP系统级主题|CSAPP 系统]] — 存储/链接/异常控制流/虚拟内存/优化
- [[concepts/操作系统基础|操作系统基础]] — 进程/线程/调度/内存管理/文件系统

### Python
- [[concepts/Python数据处理与杂项|Python 数据处理与杂项]] — 性能检测、PIL/OpenCV/Matplotlib、MySQL、JS
- [[concepts/Python子进程管理|Python 子进程管理]] — subprocess 接口分层、安全模型与管道死锁
- [[concepts/Python文件IO模型|Python 文件 I/O 模型]] — pathlib/os/shutil 分层架构与设计哲学

### Unity 渲染与图形学
- [[concepts/Unity Shader基础|Unity Shader 基础]] — 渲染管线、表面着色器、顶点/片元着色器、纹理、光照
- [[concepts/Shader高级特性|Shader 高级特性]] — 透明/曲面细分/几何着色器、复杂光源、shader 优化
- [[concepts/渲染管线理论|渲染管线理论]] — Games101 全套：线性代数/变换/光栅化/着色/几何/光线追踪/材质/动画
- [[concepts/光线追踪入门|光线追踪入门]] — Ray Tracing in One Weekend：光线/球体/材质/景深/蒙特卡洛
- [[concepts/OpenGL学习笔记|OpenGL 学习笔记]] — 环境搭建、Shader 编程、图形管线

### Unity 引擎
- [[concepts/Unity性能优化|Unity 性能优化]] — Draw Call/Batch、字符串/GC、UI 优化、Profiler 热点分析
- [[concepts/Unity资源管理|Unity 资源管理]] — AssetBundle、动态加载、内存管理、引用计数
- [[concepts/Unity-Socket网络编程|Unity Socket 网络]] — TCP/UDP 异步、自定义协议、连接管理
- [[concepts/游戏帧同步|游戏帧同步]] — 帧同步原理、KCP、锁步/状态同步对比
- [[concepts/Unity脚本架构|Unity 脚本架构]] — UniTask/DOTWeen/项目结构/游戏研发流程
- [[concepts/Unity DOTS 与 ECS|Unity DOTS 与 ECS]] — ECS 架构、JobSystem、Burst 编译器
- [[concepts/Unity FMOD 与 LipSync|Unity FMOD 与 LipSync]] — FMOD Studio 集成、共振峰口型同步
- [[concepts/Unity动画与Spine|Unity 动画与 Spine]] — Animator 系统、Spine 2D 动画组件、动画系统演进
- [[concepts/Unity编辑器特性速查|Unity 编辑器特性]] — 内置 Attribute 速查表，属性/方法/类三分类
- [[concepts/Unity自定义PropertyDrawer|Unity 自定义 PropertyDrawer]] — PropertyAttribute + PropertyDrawer 三步实现
- [[concepts/Unity自定义Inspector|Unity 自定义 Inspector]] — CustomEditor、SerializedProperty、布局与数组处理
- [[concepts/Unity编辑器窗口|Unity 编辑器窗口]] — ScriptableWizard / EditorWindow / PopupWindowContent
- [[concepts/Unity Gizmos 调试|Unity Gizmos 调试]] — Gizmos / DrawGizmo / Handles 对比与最佳实践
- [[concepts/Unity编辑器全局设置|Unity 编辑器全局设置]] — Editor 文件夹、MenuItem、ContextMenu、Selection
- [[concepts/Unity常用API速查|Unity 常用 API 速查]] — UI 组件/序列化/Input/Transform/常见模式的速查表
- [[concepts/Unity平台交互与日志|Unity 平台交互与日志]] — Android/iOS 互调、日志系统、平台适配

### 开发工具
- [[concepts/Git操作完全指南|Git 操作完全指南]] — 分支/合并/变基/储藏/子模块/标签/撤销/bisect
- [[concepts/VSCode配置与调试|VSCode 配置与调试]] — 多 Profile、tasks/launch、Vim/Neovim、LeetCode 插件
- [[concepts/Linux Shell环境|Linux Shell 环境]] — Bash/Fish/Zsh、tmux、Neovim、lazygit/fzf/rg/fd
- [[concepts/WSL2与Windows开发环境|WSL2 与 Windows 开发环境]] — WSL2 配置/迁移/.wslconfig、Scoop、AI 环境
- [[concepts/Dev工具集|Dev 工具集]] — Docker、Protobuf3、dotnet CLI、SVN、Obsidian、xmake

## Entities

- [[entities/EmmyLua|EmmyLua]] — Lua 静态类型注解系统与语言服务器
- [[entities/PI-Agent|PI Agent]] — AI 编码代理 CLI，npm 扩展生态系统

## Source Summaries

### C# / .NET
- [[sources/csharp-async-awaiter-摘要|CSharp 异步 Awaiter]] — async/await 状态机、Awaiter 接口、调度控制
- [[sources/csharp-memory-gc-摘要|CSharp 内存 GC]] — struct 装箱、匿名函数 GC、WeakReference、IDisposable
- [[sources/csharp-collections-摘要|CSharp 集合]] — IList 层次、List 扩容、Span、ArrayPool
- [[sources/csharp-serialization-摘要|CSharp 序列化 IO]] — XML 序列化、Stream、字节处理、文件 I/O
- [[sources/csharp-socket-network-摘要|CSharp Socket 网络]] — TCP BeginReceive、自定义协议
- [[sources/csharp-delegates-attributes-摘要|CSharp 委托特性]] — delegate/event/Action、Attribute 反射、ref/in/out
- [[sources/dependency-injection-摘要|依赖注入]] — DI 容器、生命周期、Strategy/Factory 关系
- [[sources/csharp-file-io-摘要|C# 文件 I/O]] — File/FileStream 分层 API、异步 I/O
- [[sources/csharp-process-摘要|C# Process]] — 进程管理、stdout/stderr 防死锁
- [[sources/csharp-struct-boxing-gc-摘要|C# Struct 装箱]] — 装箱触发、零分配优化
- [[sources/csharp-threading-摘要|C# 多线程]] — 并发模型演进、线程安全

### C++ / CMake
- [[sources/cpp-core-syntax-摘要|C++ 核心语法]] — 移动语义、STL、哈希、值类别
- [[sources/cpp-concurrency-摘要|C++ 并发]] — thread/future/async/协程
- [[sources/cmake-guide-摘要|CMake 构建指南]] — 现代 CMake、vcpkg 集成

### Lua
- [[sources/lua-core-摘要|Lua 核心]] — table、闭包/UpValue、元表、协程
- [[sources/xlua-hotfix-摘要|XLua 热补丁]] — C#↔Lua 互调、IL 注入、热更新
- [[sources/emmyLua-environment-setup-摘要|EmmyLua 环境安装]] — Rider/VS Code 搭建
- [[sources/emmyLua-annotations-reference-摘要|EmmyLua 注解参考]] — 约 25 种注解完整语法

### 算法
- [[sources/算法-排序与搜索-摘要|排序与搜索]] — 排序、二分、单调栈、优先队列
- [[sources/算法-动态规划与回溯-摘要|DP 与回溯]]
- [[sources/算法-树与字符串-摘要|树与字符串]]
- [[sources/算法-面试方法论-摘要|面试通用方法论]]

### 设计模式
- [[sources/design-patterns-creational-摘要|创建型模式]]
- [[sources/design-patterns-structural-摘要|结构型模式]]
- [[sources/design-patterns-behavioral-摘要|行为型模式]]

### CS 体系结构
- [[sources/csapp-program-structure-摘要|CSAPP 程序与机器]]
- [[sources/csapp-systems-摘要|CSAPP 系统]]
- [[sources/os-fundamentals-摘要|操作系统基础]]

### Python
- [[sources/Python-运行命令行-摘要|Python 运行命令行]]
- [[sources/Python-文件操作-摘要|Python 文件操作]]
- [[sources/python-data-misc-摘要|Python 数据处理]]

### Unity 渲染/图形学
- [[sources/shader-basics-摘要|Shader 基础]]
- [[sources/shader-advanced-摘要|Shader 高级]]
- [[sources/rendering-pipeline-theory-摘要|渲染管线理论]]
- [[sources/ray-tracing-in-one-weekend-摘要|光线追踪入门]]
- [[sources/learn-opengl-notes-摘要|OpenGL 学习笔记]]

### Unity 引擎
- [[sources/Unity编辑器全局-摘要|Unity 编辑器全局]]
- [[sources/Unity编辑器特性速查-摘要|Unity 编辑器特性速查]]
- [[sources/Unity自定义PropertyDrawer-摘要|Unity 自定义 PropertyDrawer]]
- [[sources/Unity自定义Inspector-摘要|Unity 自定义 Inspector]]
- [[sources/Unity编辑器窗口-摘要|Unity 编辑器窗口]]
- [[sources/Unity Gizmos调试-摘要|Unity Gizmos 调试]]
- [[sources/Unity性能优化-摘要|Unity 性能优化]]
- [[sources/Unity资源管理-摘要|Unity 资源管理]]
- [[sources/Unity-Socket网络编程-摘要|Unity Socket 网络]]
- [[sources/游戏帧同步-摘要|游戏帧同步]]
- [[sources/Unity脚本架构-摘要|Unity 脚本架构]]
- [[sources/Unity-DOTS-ECS-摘要|Unity DOTS 与 ECS]]
- [[sources/Unity-FMOD-LipSync-摘要|Unity FMOD 与 LipSync]]
- [[sources/Unity动画与Spine-摘要|Unity 动画与 Spine]]
- [[sources/Unity常用API速查表-摘要|Unity 常用 API 速查]]
- [[sources/Unity平台交互与日志-摘要|Unity 平台交互与日志]]

### 开发工具
- [[sources/git-operations-guide-摘要|Git 操作指南]]
- [[sources/vscode-config-debug-摘要|VSCode 配置与调试]]
- [[sources/linux-shell-guide-摘要|Linux Shell 环境]]
- [[sources/wsl-windows-dev-setup-摘要|WSL2 Windows 环境]]
- [[sources/dev-tools-misc-摘要|Dev 工具集]]
- [[sources/PI-Agent-扩展插件-摘要|PI Agent 扩展插件]]

## Comparisons
- [[comparisons/CSharp-vs-CPP-async|C# vs C++ 异步模型]] — async/await (TAP) vs std::future/coroutine 对比
- [[comparisons/frame-sync-vs-state-sync|帧同步 vs 状态同步]] — Lockstep 与状态同步在网络游戏中的权衡

