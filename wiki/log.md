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