# LLM Wiki 工作区初始化 — 总领计划

## 概述

基于 Karpathy 的 LLM Wiki 理念，在 `F:\Project_AI\wiki\` 创建三层架构的 Wiki 工作区。核心比喻：**Obsidian 是 IDE，LLM 是程序员，Wiki 是代码库**。

## 架构总览

```
┌─────────────────────────────────────────────┐
│  Schema 层：CLAUDE.md（根目录）              │
│  定义 Wiki 约定、页面类型、工作流入口          │
├─────────────────────────────────────────────┤
│  Skills 层（行为）：.claude/skills/          │
│  /ingest  /query  /lint  /refine            │
├─────────────────────────────────────────────┤
│  Wiki 层（LLM 拥有）：wiki/                  │
│  ├── index.md / log.md（双入口导航）          │
│  ├── concepts/（概念页面）                    │
│  ├── entities/（实体页面）                    │
│  ├── sources/（来源摘要）                     │
│  ├── comparisons/（对比分析）                 │
│  └── overview.md（高层综合）                  │
├─────────────────────────────────────────────┤
│  Drafts 层（打磨区）：drafts/（扩展）          │
├─────────────────────────────────────────────┤
│  Raw 层（不可变）：raw/                       │
│  gamedev / cs / tools + assets/             │
├─────────────────────────────────────────────┤
│  搜索引擎：QMD（BM25 + 向量 + LLM 重排序）     │
└─────────────────────────────────────────────┘
```

## 子计划清单

| 序号 | 计划 | 说明 | 依赖 |
|------|------|------|------|
| 01 | [创建目录结构](plan/01-创建目录结构.md) | 搭建目录骨架：raw/（17子领域+assets）、wiki/（4子目录）、drafts/、.claude/skills/、.qmd/ | 无 |
| 02 | [创建 Skill 文件](plan/02-创建Skill文件.md) | 定义 /ingest /query /lint /refine 四个核心技能 | 01 |
| 03 | [创建 CLAUDE 配置](plan/03-创建CLAUDE配置.md) | 根目录 CLAUDE.md：目录约定、页面类型（concepts/entities/sources/comparisons）、frontmatter、分工原则 | 01 |
| 04 | [创建索引和日志](plan/04-创建索引和日志.md) | 在 wiki/ 内创建 index.md + log.md（Karpathy 规范：属于 wiki 的一部分） | 01 |
| 05 | [配置 QMD 搜索引擎](plan/05-配置QMD搜索引擎.md) | 配置 QMD 作为主要检索后端，集成到 query/lint | 01, 02 |
| 06 | [归档学习笔记](plan/06-归档学习笔记.md) | 移入 raw/ → 首次 /ingest → 打通全流程 | 01, 02, 03, 04, 05 |

## 执行顺序

```
01 创建目录结构
 ├── 02 创建 Skill 文件
 ├── 03 创建 CLAUDE 配置
 ├── 04 创建索引和日志
 └── 05 配置 QMD 搜索引擎
      └── 06 归档学习笔记（全流程验证）
```

01 必须先执行；02-05 可在 01 完成后并行执行；06 依赖所有前置步骤完成后执行，作为端到端验证。

## 与 Karpathy 规范的对照

| 元素 | Karpathy 规范 | 本项目实现 |
|------|--------------|-----------|
| 来源层 | `raw/` | `raw/` + 三大支柱子目录 |
| Wiki 层 | `wiki/`（LLM 拥有） | `wiki/` + 子目录分类 |
| 索引/日志 | wiki/ 内部 | `wiki/index.md` + `wiki/log.md` |
| Schema | `CLAUDE.md`（根目录） | 根目录 `CLAUDE.md` |
| 操作 | ingest/query/lint | Skill 文件实现 |
| 打磨区 | 无 | `drafts/` + `/refine`（扩展） |
| 搜索 | 可选 | QMD BM25+向量+LLM重排序 |

## 关键设计决策

- **三层架构**：`raw/` → `wiki/` → `CLAUDE.md`（Karpathy 核心）
- **wiki/ 按页面类型分子目录**：`concepts/`、`entities/`、`sources/`（来源摘要）、`comparisons/`（对应 Karpathy 定义的页面类型）
- **wiki/index.md + wiki/log.md 属于 wiki/**：Karpathy 视 index 和 log 为 Wiki 的一部分，不是外部文件
- **Skills = 行为，CLAUDE.md = 数据**：操作流程用 Skill 定义，静态约定由根目录 CLAUDE.md 描述
- **raw/ 三大支柱**：`gamedev/`（实践综合）、`cs/`（理论沉淀）、`tools/`（操作流程）
- **drafts/ 独立打磨**：非 Karpathy 原生，作为 `/refine` 的打磨区
- **QMD 为主，index 为辅**：搜索引擎处理精准检索，索引提供浏览导航
- **每个子计划自带验证步骤**：确保每步可独立验证，问题可快速定位

## 最终验证清单

全流程完成后执行：

1. [ ] 目录结构完整（含 `raw/assets/`、`wiki/concepts/`、`wiki/entities/`、`wiki/sources/`、`wiki/comparisons/`、`.qmd/`）
2. [ ] 四个 Skill 文件存在且可被 `/skill-name` 调用
3. [ ] 根目录 `CLAUDE.md` 完整描述约定（含 wiki/ 子目录分类和页面类型定义）并引用 Skill
4. [ ] QMD MCP 服务器正常响应，`qmd:qmd` skill 就绪
5. [ ] `wiki/index.md` 和 `wiki/log.md` 有初始内容
6. [ ] `学习笔记.md` 已归档至 `raw/`
7. [ ] `/ingest` 全流程跑通（raw来源→讨论→wiki/sources/摘要→wiki/concepts/→wiki/index.md→wiki/log.md）
8. [ ] `/query` 检索返回正确结果（含 QMD 检索 + index 定位）
9. [ ] `/lint` 健康检查正常执行
10. [ ] `/refine` 打磨流程可用
