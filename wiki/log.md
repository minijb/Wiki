---
title: "Wiki 日志"
type: log
updated: 2026-05-11
---

# Wiki 日志

按时间倒序记录所有 Wiki 操作。只追加，不修改历史。

## [2026-05-11] lint | 检查 32 页面，发现 5 交叉引用缺口、1 领域结构问题

## [2026-05-11] refine+ingest | Unity 编辑器扩展系列（6 drafts → 6 raw + 12 wiki pages）
- refine: 打磨 6 个 Unity 编辑器草稿（全局设置、特性速查、PropertyDrawer、Inspector、窗口、Gizmos）
- 新建领域: `raw/gamedev/editor-extensions/`
- raw 归档: 6 个文件至 editor-extensions/
- wiki 新建 6 来源摘要: [[sources/Unity编辑器全局-摘要]], [[sources/Unity编辑器特性速查-摘要]], [[sources/Unity自定义PropertyDrawer-摘要]], [[sources/Unity自定义Inspector-摘要]], [[sources/Unity编辑器窗口-摘要]], [[sources/Unity Gizmos调试-摘要]]
- wiki 新建 6 概念页: [[concepts/Unity编辑器全局设置]], [[concepts/Unity编辑器特性速查]], [[concepts/Unity自定义PropertyDrawer]], [[concepts/Unity自定义Inspector]], [[concepts/Unity编辑器窗口]], [[concepts/Unity Gizmos 调试]]
- 更新 index.md（12 条目）

## [2026-05-07] refine+ingest | Lua EmmyLua 系列笔记（4 drafts → 2 raw + 4 wiki pages）
- refine: 合并 drafts/00-03 为 2 个 raw 文件，补充网上资料，修正 frontmatter/分类/示例
- raw 归档: `raw/cs/languages/emmyLua-environment-setup.md`, `raw/cs/languages/emmyLua-annotations-reference.md`
- wiki 新建: [[sources/emmyLua-environment-setup-摘要]], [[sources/emmyLua-annotations-reference-摘要]], [[entities/EmmyLua]], [[concepts/EmmyLua注解系统]]
- 更新 index.md（4 条目），首次写入 wiki 页面

## [2026-05-07] refine+ingest | Python 笔记（2 drafts → 2 raw + 4 wiki pages）
- refine: 重写 drafts/Python-运行命令行.md（补全参数、shlex/Popen方法/异常处理/死锁/环境隔离）
- refine: 重写 drafts/Python-文件操作.md（补全 pathlib 完整章节、os.path 工具函数、压缩/安全/原子写入）
- raw 归档: `raw/cs/languages/Python-运行命令行.md`, `raw/cs/languages/Python-文件操作.md`
- wiki 新建: [[sources/Python-运行命令行-摘要]], [[sources/Python-文件操作-摘要]], [[concepts/Python子进程管理]], [[concepts/Python文件IO模型]]
- 更新 index.md（4 条目）

## [2026-05-09] refine+ingest | PI Agent 扩展插件 — 创建 3 个页面
- refine: 扩展 drafts/PI Plugin 安装.md（网络搜索 8 个插件的详细用法，新增 pi-kanban 章节）
- raw 归档: `raw/tools/ai-coding/PI-Agent-扩展插件.md`（新建领域 `raw/tools/ai-coding/`）
- wiki 新建: [[sources/PI-Agent-扩展插件-摘要]], [[entities/PI-Agent]]
- 更新 index.md（2 条目）

## [2026-05-10] ingest | C# 系列笔记（4 drafts → 4 raw + 8 wiki pages）
- 移动 4 个 C# 草稿至 `raw/cs/languages/`：csharp-file-io, csharp-process, csharp-struct-boxing-gc, csharp-threading
- wiki 新建 4 来源摘要：[[sources/csharp-file-io-摘要]], [[sources/csharp-process-摘要]], [[sources/csharp-struct-boxing-gc-摘要]], [[sources/csharp-threading-摘要]]
- wiki 新建 4 概念页：[[concepts/CSharp文件IO]], [[concepts/CSharp进程管理]], [[concepts/CSharp值类型性能]], [[concepts/CSharp并发模型]]
- 更新 Python 概念页双向链接：[[concepts/Python文件IO模型]], [[concepts/Python子进程管理]]
- 更新 index.md（8 条目）

## [2026-05-07] lint | 检查 10 页面，发现 1 问题（已修复）

## [2026-06-02] My_Vault 迁移 | 11 Tasks 全部完成 — ~45 raw + ~90 wiki pages
- 迁移范围: `drafts/My_Vault/` 全部技术笔记 → `raw/` 对应领域
- Task 1 (C#): 7 raw + 7 source summaries + 7 concepts — 异步/GC/集合/序列化/Socket/委托/DI
- Task 2 (C++): 3 raw + 3 source summaries + 3 concepts — 核心语法/并发/CMake
- Task 3 (Lua): 2 raw + 2 source summaries + 2 concepts — Lua 核心/xlua 热补丁
- Task 4 (算法): 4 raw + 4 source summaries + 4 concepts — 排序搜索/DP回溯/树字符串/面试方法论
- Task 5 (设计模式): 3 raw + 3 source summaries + 3 concepts — 创建型/结构型/行为型(22种模式)
- Task 6 (体系结构): 3 raw + 3 source summaries + 3 concepts — CSAPP/OS 基础
- Task 7 (渲染): 5 raw + 5 source summaries + 5 concepts — Shader/管线/光线追踪/OpenGL
- Task 8 (Unity 综合): 8 raw + 8 source summaries + 8 concepts — 优化/资源/网络/帧同步/架构/ECS/音频/动画
- Task 9 (Unity 碎片): 2 raw + 2 source summaries + 2 concepts — API 速查/平台交互
- Task 10 (工具): 5 raw + 5 source summaries + 5 concepts — Git/VSCode/Shell/WSL/Dev 工具集
- Task 11 (Python 杂项): 1 raw + 1 concept — 数据处理/MySQL/JS
- 新建 raw/ 目录: `raw/tools/shell/` (需 AGENTS.md 同步)
- 不迁移内容归档: `raw/_archived-vault/` (8 子目录)
- 更新 index.md: 全新全量写入

## [2026-06-02] lint | 检查 116 页面，发现问题 5 项
- 孤立页面: 0 — 全部页面已在 index.md 注册
- 断链: 0 — 扫描全部 wikilink 有效
- 陈旧页面: 0 — 所有页面 updated 在 90 天内
- 源链完整性: 0 断 — 所有 source summary → raw 文件存在
- 交叉引用缺口: 7 对概念缺失双向链接
- 空目录: 5 个预置空目录（assets/gamedev-ai-physics-ui/tools-docker）
- 比较页: wiki/comparisons/ 为空
- 领域结构: AGENTS.md 已同步，命名合规，深度未超 2 层
- 草稿残留: drafts/My_Vault/ 仍保留原文件（cp 非 mv，待清理）

## [2026-06-02] lint-fix | 处理 5 项 lint 问题全部完成
- 交叉引用: 补充 7 对概念双向 [[wikilink]]（Unity性能↔C#GC、异步↔脚本架构、DI↔创建型模式 等）
- 比较页: 新建 [[comparisons/CSharp-vs-CPP-async]] + [[comparisons/frame-sync-vs-state-sync]]
- 空目录: 5 个预置空目录添加 .gitkeep + 用途说明
- 草稿清理: `rm -rf drafts/My_Vault/`
- 更新 index.md（comparisons 条目）