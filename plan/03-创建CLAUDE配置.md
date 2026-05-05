# 03 — 创建 CLAUDE.md（Schema 层）

## 目标

在项目**根目录**创建 `CLAUDE.md`（Karpathy 规范：Schema 文件在根目录），定义 Wiki 的静态约定。

## 需定义的内容

- **目录结构约定**：
  - `raw/` — 不可变来源层，LLM 只读不写
  - `wiki/` 子目录分类及其用途：
    - `wiki/concepts/` — 概念页面（抽象概念/理论/方法论）
    - `wiki/entities/` — 实体页面（具体事物/人物/项目）
    - `wiki/sources/` — 来源摘要（每篇 raw 来源对应一个摘要）
    - `wiki/comparisons/` — 对比分析（跨来源的矛盾/对比）
    - `wiki/overview.md` — 高层综合页面
  - `wiki/index.md` — 内容导向目录（Wiki 页面索引）
  - `wiki/log.md` — 时间导向日志（只追加，Karpathy 可 grep 格式）
  - `drafts/` — 用户打磨区（扩展）
- **页面格式规范**：
  - frontmatter 必填字段：`title`, `type`（concept/entity/source-summary/comparison/overview）, `updated`
  - frontmatter 可选字段：`tags`, `source`（来源文件路径）, `aliases`
  - 标题层级：H1 为页面标题，H2/H3 为章节
  - 交叉引用语法 `[[wikilink]]`（Wikilink 是主要连接机制）
- **页面类型定义**（对应 Karpathy 的 page types）：
  - Summary pages → `wiki/sources/` — 每来源一篇摘要
  - Entity pages → `wiki/entities/` — 具体事物
  - Concept pages → `wiki/concepts/` — 跨来源的概念/主题
  - Comparison pages → `wiki/comparisons/` — 矛盾/对比
  - Synthesis/Overview → `wiki/overview.md` — 高层整合
  - Index page → `wiki/index.md` — 内容导向目录
  - Log page → `wiki/log.md` — 时间导向日志
- **wiki/index.md 规范**：按页面类型分节，每项 `[[路径|标题]] — 一行摘要`
- **wiki/log.md 规范**：`## [YYYY-MM-DD] 操作类型 | 摘要` 格式，grep 友好
- **分工原则**（Karpathy 核心理念）：
  - 人类：策展来源、引导分析、提问、思考意义
  - LLM：写 wiki、维护交叉引用、更新索引、检查一致性
- **Skill 入口引用**：指向 `.claude/skills/` 下四个 Skill 文件
- **QMD 集成说明**：搜索引擎在 `/query` 和 `/lint` 中的角色

## 验证

1. 确认文件存在于项目根目录 `CLAUDE.md`（不是 `.claude/CLAUDE.md`）
2. 确认包含完整的 wiki/ 子目录分类描述和页面类型定义
3. 确认定义了 frontmatter 规范（至少 title, type, updated）
4. 确认包含 `[[wikilink]]` 交叉引用语法说明
5. 确认分工原则中明确区分人类和 LLM 职责
6. 确认包含四个 Skill 的引用和 QMD 集成说明
