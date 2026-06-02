---
name: qmd
description: QMD 搜索引擎 — 对 wiki/ 和 raw/ 执行 BM25+向量语义搜索、获取文档、检查索引状态。当需要在 Wiki 中精准检索知识、按关键词/语义查找页面、获取文档全文、检查搜索索引状态、或进行任何跨页面内容搜索时使用此 skill。"在 wiki 里搜索 X"、"帮我搜一下 Y"、"全文搜索 Z"、"索引状态怎么样"等都应触发。被 /skill:query、/skill:lint、/skill:ingest、/skill:refine 等上层 skill 调用，也用于独立的搜索需求。当 QMD 未安装或索引需要重建时，使用此 skill 指导安装和配置。
user-invocable: true
---

# qmd — QMD 搜索引擎

QMD (Query Markdown Documents) 是本 Wiki 的本地搜索引擎，通过 CLI 命令行运行，也可部署为 MCP 服务器。它索引 `wiki/` 和 `raw/` 下的所有 Markdown 文件，提供三种检索模式：BM25 关键词匹配、向量语义搜索、以及混合查询（BM25 + 向量 + LLM 重排序）。

> **包名：** `@tobilu/qmd` (npm) — 最新版本用 `npm install -g @tobilu/qmd` 安装

## 安装与初始化

### 前置条件

- Node.js ≥ 22.0 或 Bun ≥ 1.0
- 首次运行 `qmd embed` 会下载约 300MB 的 GGUF 模型文件（嵌入 + 重排序 + 查询扩展），需要网络和磁盘空间
- 嵌入模型：Gemma-300M；重排序：Qwen3-Reranker-0.6B；查询扩展：QMD-1.7B

### 首次安装流程

```bash
# 1. 全局安装
npm install -g @tobilu/qmd
# 或: bun install -g @tobilu/qmd

# 2. 在 Wiki 项目根目录初始化本地索引
cd D:/Wiki
qmd init                      # 创建 .qmd/index.sqlite

# 3. 添加集合
qmd collection add wiki --name wiki    # 索引 wiki/ 目录
qmd collection add raw --name raw      # 索引 raw/ 目录（含 _archived-vault）

# 4. 添加上下文元数据（提升搜索结果质量）
qmd context add qmd://wiki/ "LLM Wiki 知识库 — concepts(概念页) entities(实体页) sources(来源摘要) comparisons(对比分析)"
qmd context add qmd://raw/ "原始来源层 — gamedev(Unity渲染/网络/优化/动画/音频/编辑器) cs(C#/C++/Lua/Python/算法/设计模式/体系结构) tools(git/ide/docker/shell/ci-cd)"

# 5. 构建向量嵌入（首次耗时 2-3 分钟，317 文件约 1335 chunks）
qmd embed

# 6. 验证
qmd status
```

### 日常更新

文件变更后重建索引：

```bash
qmd update                     # 增量更新（新增/修改/删除）
qmd embed                      # 仅需在新文件有大量新增时重新嵌入
```

## 环境兼容性

### CI 环境阻断问题

当前 oh-my-pi 编码 harness 自动设置 `CI=true` 环境变量。QMD v2.x 检测到 `CI=true` 后会禁用所有 LLM 操作（查询扩展、向量嵌入、重排序），报错：

```
Error: LLM operations are disabled in CI (set CI=true)
```

**注意：** 设置 `CI=false` 无效 — QMD 使用 `env-var` 库校验值，且编译后的 JS 显式检查该变量。

**解决方法：** 在执行 `qmd query` 或 `qmd embed` 时用 `env -u CI` 去掉该变量：

```bash
# 混合搜索（含 LLM 扩展 + 向量 + 重排序）
env -u CI qmd query "Unity 帧同步 KCP 原理" -n 5

# 构建向量嵌入
env -u CI qmd embed

# BM25 纯关键词搜索不受 CI 影响，无需 env -u CI
qmd search "async await 状态机" -n 5
```

**三种模式对 CI 的依赖：**

| 命令 | CI 阻断？ | 说明 |
|------|----------|------|
| `qmd search` | 否 | 纯 BM25，不调 LLM |
| `qmd vsearch` | 是 | 需要嵌入模型生成查询向量 |
| `qmd query` | 是 | 需要 LLM 扩展 + 向量 + 重排序 |
| `qmd embed` | 是 | 需要嵌入模型处理文档 |
| `qmd get` / `qmd multi-get` | 否 | 纯文档读取 |
| `qmd status` | 否 | 索引元信息查询 |

## 核心概念

- **docid** — 每个文档的 6 字符哈希标识符，QMD 自动生成
- **RRF (Reciprocal Rank Fusion)** — 合并多个检索源结果的算法，k=60，顶部有排名奖励
- **LLM 重排序** — 用 0.6B cross-encoder 模型对候选文档重新打分，与 RRF 加权混合
- **上下文系统** — 路径可附加元数据描述，帮助 LLM 做出更好的文档选择

## 可用工具

### `query` — 混合搜索

执行完整的 QMD 搜索管道：查询扩展 → BM25 + 向量并行检索 → RRF 融合 → LLM 重排序 → 返回排序结果。

```
参数：
  - query (string, 必填): 搜索查询，支持自然语言或关键词
  - top (number, 可选): 返回结果数，默认 10，范围 1-50
  - mode (string, 可选): "hybrid" (默认) | "lex" (仅 BM25) | "vec" (仅向量) | "hyde" (HyDE 增强)
  - filter (string, 可选): 子路径过滤，如 "wiki/concepts" 限定搜索范围
```

**使用场景：**
- `/skill:query` — 检索知识回答问题，优先用 `hybrid` 模式
- `/skill:lint` — 全局搜索某个概念的所有引用
- 探索性搜索 — 不确定用哪个路径时用 `hybrid`

**模式选择指南：**
| 场景 | 推荐模式 | 原因 |
|------|----------|------|
| 精准术语/API名/错误码 | `lex` | BM25 对精确术语匹配最佳 |
| 模糊概念/自然语言描述 | `vec` | 向量搜索捕获语义相似性 |
| 综合查询（默认） | `hybrid` | 两种检索各取所长 |
| 已知答案方向、需要验证 | `hyde` | 用假设文档增强检索 |

### `search` — 纯 BM25 关键词搜索

不调用 LLM，速度极快（<1s），适合精确术语匹配。**在 CI 环境下唯一无需 `env -u CI` 的检索命令。**

```bash
qmd search "关键词" -n 5 -c wiki
```

### `get` — 获取单个文档

通过路径或 docid 获取文档全文。

```
参数：
  - path (string, 可选): 文档相对于集合根的路径，如 "wiki/concepts/泛型.md"
  - docid (string, 可选): 文档的 6 字符哈希标识符
  - context (boolean, 可选): 是否返回路径附加的上下文元数据，默认 false
```

> path 和 docid 至少提供一个。模糊匹配：路径不存在时自动给出最接近的建议。

**与 Read 工具的区别：** `get` 返回的文档会附带 QMD 上下文元数据（路径上下文），适合 LLM 做上下文感知的文档选择。`Read` 更适合直接阅读不需要上下文辅助的页面。

### `multi_get` — 批量获取文档

一次性获取多个文档，支持三种匹配方式。

```
参数：
  - paths (string[], 可选): 文档路径数组
  - glob (string, 可选): Glob 模式匹配，如 "wiki/concepts/*.md"
  - docids (string[], 可选): docid 数组
  - context (boolean, 可选): 是否返回上下文元数据，默认 false
```

**使用场景：**
- 加载搜索结果中前 N 个页面全文
- 按子目录批量读取，如 `wiki/sources/*.md`
- docid 批量精确获取

### `status` — 索引状态检查

获取 QMD 索引健康和集合信息。

```
无参数
```

返回内容包括：
- 索引文档总数
- 向量嵌入模型信息
- 集合列表及各自文档数
- 最后索引更新时间

**使用场景：**
- `/skill:lint` — 检查 Wiki 页面是否全部索引
- `/skill:ingest` — 新页面写入后验证索引已更新
- 排查搜索无结果问题

## 搜索策略

### 通用搜索流程

1. 分析用户查询，提取关键词和意图
2. 先读取 `wiki/index.md` 确定候选目录
3. 用 `filter` 参数缩小搜索范围（如果已知方向）
4. 执行 `query`，根据查询性质选择模式
5. 用 `multi_get` 批量拉取前 5-10 个结果的全文
6. 综合生成答案，附 `[[wikilink]]` 引用

### 高精度检索（用于 /skill:lint 等）

```
query(query="关键词", top=20, mode="lex")    # 精确匹配优先
query(query="概念描述", top=15, mode="vec")   # 语义覆盖补充
```

然后合并去重两个结果集。

### 中文搜索注意事项

Wiki 内容以中文为主，BM25 对中文的分词敏感：

- **有效的中文查询：** `"状态机 async"` → 命中 CSharp异步模型（89%）
- **有效的中文查询：** `"帧同步 KCP"` → 命中游戏帧同步（93%）
- **无效：英文术语查中文内容** — `"Factory Singleton Observer"` 不会命中中文写的设计模式页面（工厂/单例/观察者），因为内容中没有这些英文词
- **建议：** 中文内容用中文关键词搜索；如果知道页面使用了中英混合术语（如 `async/await`、`KCP`、`ECS`），混合使用效果最佳

### 路径过滤技巧

- 限定概念页：`filter="wiki/concepts"`
- 限定原始来源：`filter="raw/gamedev"`
- 搜索全站（不加 filter）：寻找跨目录关联

## 索引管理

QMD 索引在文件变更后需要更新。虽然 QMD 会检测变更，但批量操作后手动触发重新索引更可靠。

**触发重新索引：**
- 通过 `/skill:ingest` skill 在写入新页面后触发：`qmd update && env -u CI qmd embed`
- 使用 `qmd update` 增量更新文档列表
- 若有大量新文件，追加运行 `env -u CI qmd embed` 构建新向量
- 索引状态可通过 `qmd status` 检查

## 故障处理

### QMD 未安装

症状：`qmd: command not found` 或 `.qmd/` 目录只有 `.gitkeep`

解决：按上文"安装与初始化"完整走一遍

### CI 环境阻断 LLM 操作

症状：`Error: LLM operations are disabled in CI (set CI=true)`

解决：`env -u CI qmd query "..."` 或改用 `qmd search`（纯 BM25，不需要 LLM）

### 搜索无结果

1. 用 `qmd status` 检查索引是否最新
2. 尝试更短的关键词或放宽 filter
3. 中文内容用中文关键词，英文内容用英文关键词 —— 不交叉
4. 切换到对侧模式：`qmd search` 无结果时试 `env -u CI qmd vsearch`，反之亦然
5. 读取 `wiki/index.md` 手动遍历

### 结果相关性低

1. 减少 `-n` 值，让 RRF 集中到更少的候选
2. 添加 `-c wiki` 或 `-c raw` 限定集合
3. 从 `qmd query` 切换到 `qmd search`（若查询是精确术语）或 `env -u CI qmd vsearch`（若查询是抽象概念）
4. 检查查询词是否在 Wiki 中有标准术语，用标准术语重新查询

### 索引过期

- `qmd status` 显示文档数与预期不符 → `qmd update` 增量更新
- 新增的 `raw/` 文件没有搜索结果 → 检查文件是否在 QMD 集合配置的 glob 范围内

## 与上层 Skill 的协作

| 上层 Skill | QMD 使用方式 |
|------------|-------------|
| `/skill:query` | `env -u CI qmd query "..."` → `qmd multi-get` → 综合回答 |
| `/skill:lint` | `qmd search "关键词"` 检查引用 + `qmd status` 检查索引覆盖 |
| `/skill:ingest` | `qmd update && env -u CI qmd embed` 重建索引，`qmd status` 验证 |
| `/skill:refine` | `qmd search "概念"` 搜索已有知识判断草案是否重复 |

## 输出规范

- 搜索命中时，按相关度排序列出结果，每项含路径和摘要
- 无结果时，报告搜索词和使用的模式，给出后续建议
- 索引状态报告用简要表格，异常项标出
