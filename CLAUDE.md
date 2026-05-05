# CLAUDE.md — Wiki Schema

本文件定义 LLM Wiki 工作区的静态约定。Skills（`.claude/skills/`）承载行为，此文件承载数据结构与格式规范。

## 目录结构

```
wiki/                        # Wiki 知识库根目录
├── CLAUDE.md                # 本文件 — Schema 层（LLM 只读，人类维护）
├── raw/                     # 原始来源层（不可变：LLM 只读不写，人类策展）
│   ├── assets/              # 本地图片、PDF 等附件
│   ├── gamedev/             # 游戏开发实践综合
│   │   ├── rendering/       # 渲染、Shader、URP/HDRP
│   │   ├── physics/         # 物理、碰撞检测
│   │   ├── animation/       # 动画、程序化动画
│   │   ├── networking/      # 网络同步、多人游戏
│   │   ├── ai/              # AI、行为树、NavMesh
│   │   ├── gameplay/        # 玩法系统、战斗、技能
│   │   ├── ui/              # UGUI、UI Toolkit
│   │   ├── audio/           # 音频系统
│   │   └── optimization/    # 性能优化、内存管理
│   ├── cs/                  # 计算机科学理论沉淀
│   │   ├── languages/       # C#, C++, Lua, Python
│   │   ├── design-patterns/ # GoF, 游戏编程模式
│   │   ├── architecture/    # SOLID, DDD, ECS, 软件架构
│   │   └── algorithms/      # 算法与数据结构
│   └── tools/               # 工具操作流程
│       ├── git/             # 版本控制
│       ├── ide/             # Rider, VS, VS Code
│       ├── docker/          # 容器化
│       └── ci-cd/           # 持续集成/部署
├── wiki/                    # Wiki 层（LLM 拥有：LLM 读写，人类审阅）
│   ├── index.md             # 内容导向目录（按页面类型分节，双入口之一）
│   ├── log.md               # 时间导向日志（只追加，grep 友好格式，双入口之一）
│   ├── overview.md          # 高层综合页面（跨主题整合）
│   ├── concepts/            # 概念页面 — 抽象概念、理论、方法论
│   ├── entities/            # 实体页面 — 具体事物、人物、项目、工具
│   ├── sources/             # 来源摘要 — 每篇 raw/ 来源对应一个摘要页面
│   └── comparisons/         # 对比分析 — 跨来源的矛盾、对比、权衡
├── drafts/                  # 用户打磨区（不在 Karpathy 原生规范中，扩展）
├── .claude/skills/          # Skill 定义目录
│   ├── ingest.md            # /ingest — 摄取来源到 wiki
│   ├── query.md             # /query — 查询 wiki
│   ├── lint.md              # /lint  — wiki 健康检查
│   └── refine.md            # /refine — 打磨 drafts/ 笔记
├── .qmd/                    # QMD 搜索引擎索引缓存（自动管理）
└── PLAN.md                  # 初始化总计划（历史参考）
```

## 页面类型定义

| 页面类型 | 存放目录 | frontmatter `type` | 说明 |
|----------|----------|---------------------|------|
| 来源摘要 | `wiki/sources/` | `source-summary` | 每篇 raw/ 来源对应一个摘要，URL/路径 + 要点列表 |
| 实体页面 | `wiki/entities/` | `entity` | 具体事物、人物、项目、工具 |
| 概念页面 | `wiki/concepts/` | `concept` | 跨来源的抽象概念、理论、方法论 |
| 对比分析 | `wiki/comparisons/` | `comparison` | 跨来源的矛盾、对比、权衡分析 |
| 综合概述 | `wiki/overview.md` | `overview` | 高层整合，连接各领域 |
| 索引页面 | `wiki/index.md` | `index` | 内容导向目录，双入口导航之一 |
| 日志页面 | `wiki/log.md` | `log` | 时间导向日志，双入口导航之一 |

## 页面格式规范

### Frontmatter（必填/可选）

```yaml
---
title: "页面标题"
type: concept | entity | source-summary | comparison | overview | index | log
updated: YYYY-MM-DD
# 以下为可选字段
tags: [tag1, tag2]
source: "raw/gamedev/rendering/来源文件.md"  # 来源摘要页必填
aliases: [别名1, 别名2]
---
```

### 内容规范

- **H1 (`#`)** — 页面标题，与 frontmatter `title` 一致
- **H2 (`##`)** — 主要章节
- **H3 (`###`)** — 子章节
- **`[[wikilink]]`** — 交叉引用语法，页面间的主要连接机制：
  - `[[concepts/渲染管线]]` — 子目录/页面名
  - `[[concepts/渲染管线|渲染管线]]` — 带显示别名
  - Wikilink 是双向的：被引用的页面通过反向链接感知连接

### 页面命名

- 使用中文标题或英文技术术语，简洁明了
- 来源摘要页：`来源文件名-摘要.md`

## wiki/index.md 规范

按页面类型分节，每项一行：

```markdown
## Concepts
- [[concepts/泛型|泛型]] — 参数化类型，C# 与 C++ 模板对比

## Entities
- [[entities/Unity引擎|Unity引擎]] — 游戏引擎，版本 6000.x

## Source Summaries
- [[sources/示例文章-摘要|示例文章]] — 关于 X 的综述

## Comparisons
- [[comparisons/CSharp-vs-CPP泛型|C# vs C++ 泛型]] — 泛型实现机制对比
```

## wiki/log.md 规范

只追加，不修改历史记录。Karpathy 风格 grep 友好格式：

```markdown
## [2026-05-06] ingest | 《渲染管线详解》 — 创建 3 个页面，更新 1 个页面
## [2026-05-06] lint | 检查 15 页面，发现 2 断链，1 孤立页面
## [2026-05-06] refine | drafts/网络同步笔记.md → wiki/concepts/网络同步.md
```

## 分工原则

| 角色 | 职责 |
|------|------|
| **人类** | 策展 raw/ 来源（决定放什么进来）；引导分析方向；提问与思考意义；审核 LLM 产出 |
| **LLM** | 写 wiki 页面（concepts/entities/sources/comparisons）；维护 Wikilink 交叉引用；更新 index.md 和 log.md；检查一致性与完整性；摄取来源（/ingest）；回答查询（/query）；健康检查（/lint）；打磨笔记（/refine） |

核心理念：**Obsidian 是 IDE，LLM 是程序员，Wiki 是代码库**。人类是产品经理/架构师，决定方向和验收标准。

## Skill 入口

四个核心操作通过 `.claude/skills/` 定义，以 `/` 命令调用：

| 命令 | Skill 文件 | 职责 |
|------|-----------|------|
| `/ingest` | `.claude/skills/ingest.md` | 阅读 raw/ 来源 → 讨论 → 写摘要 → 更新 wiki/ → 更新 index.md + log.md |
| `/query` | `.claude/skills/query.md` | 读 index.md 定位 → QMD 检索 → 阅读综合 → 生成答案 |
| `/lint` | `.claude/skills/lint.md` | QMD 全局扫描 → 孤页/断链/矛盾/陈旧/缺口 → 生成报告 |
| `/refine` | `.claude/skills/refine.md` | 读 drafts/ → 审查 → 讨论迭代 → 归档判定 |

## QMD 搜索引擎

QMD 是本 Wiki 的主要检索引擎，部署为 MCP 服务器（`qmd:qmd` skill）：

- **检索方式**：BM25 关键词匹配 + 向量语义搜索 + LLM 重排序
- **覆盖范围**：`wiki/` + `raw/`
- **使用场景**：
  - `/query` — 精准检索 wiki 知识，找到最相关页面
  - `/lint` — 全局扫描断链、孤立页面、交叉引用缺口
- **索引管理**：`.qmd/` 目录存储索引缓存，由 QMD 自动管理
- **降级策略**：QMD 不可用时降级为 Glob + Grep 手动遍历

## 设计原则

1. **raw/ 不可变** — LLM 绝不修改 raw/ 中的任何文件，来源完整性是 Wiki 可信度的基石
2. **Wikilink 是主要连接机制** — 不使用复杂的目录树导航，通过 `[[wikilink]]` 形成知识网络
3. **index.md + log.md 双入口** — 内容导航和时间追溯互补，覆盖不同使用场景
4. **每个页面必须有一个 frontmatter `type`** — 页面类型决定存放位置和处理方式
5. **讨论是核心步骤** — /ingest 和 /refine 必须包含人类讨论环节，LLM 提建议，人类做决策
