---
title: "PI Agent 扩展插件"
status: archived
created: 2026-05-09
archived-to: raw/tools/ai-coding/PI-Agent-扩展插件.md
tags: [pi-agent, ai-coding, extensions, plugins]
---

# PI Agent 扩展插件

**PI** 是由 [@mariozechner](https://github.com/mariozechner) 创建的 AI 编码代理（非 Claude Code），可通过 `npm install -g @mariozechner/pi-coding-agent` 安装。其扩展系统允许从 npm 或 git 安装插件，每个插件提供额外的工具、命令或 UI。

插件管理命令：`pi install npm:<package-name>`

---

## 1. pi-subagents — 子代理委派

**安装**：`pi install npm:pi-subagents`

**功能**：将任务委派给专业子代理，支持链式执行、并行执行、后台运行。每个子代理由 Markdown 文件（YAML frontmatter）定义，可指定模型、思考级别、可用工具。

**内置子代理**：`scout`（代码侦察）、`planner`（规划）、`worker`（执行）、`reviewer`（审查）、`context-builder`（上下文构建）、`researcher`（研究）、`delegate`（委派）等。

**代理存放位置**（优先级从高到低）：

| 作用域 | 路径 |
|--------|------ |
| 项目级 | `.pi/agents/{name}.md`（向上搜索目录树） |
| 用户级 | `~/.pi/agent/agents/{name}.md` |
| 内置 | `~/.pi/agent/extensions/subagent/agents/` |

**代理定义格式**：

```yaml
---
name: scout
description: Fast codebase recon
tools: read, grep, glob, bash
model: claude-haiku-4-5
thinking: high
skill: safe-bash
output: context.md
---
# System prompt — Markdown body 即为系统提示词
You are a fast codebase scout. Find relevant files and report.
```

**常用命令**：

```bash
# 单个子代理执行
/run scout "找到所有认证相关的文件"

# 链式执行 — 多个子代理按顺序执行，前一个输出作为后一个输入
/chain scout "分析项目结构" planner "基于结构制定计划" worker

# 并行执行 — 多个子代理同时运行
/parallel "scout:搜索API定义" "scout:搜索数据库模型" "scout:搜索前端路由"

# 后台运行
/run worker "重构认证模块" --bg

# 打开 TUI 管理器（浏览、编辑、创建、删除代理和链）
Ctrl+Shift+A

# 链文件 — 在 .chain.md 中定义可复用的多步骤管道
/chain-file my-pipeline
```

---

## 2. @ollama/pi-web-search — 网页搜索

**安装**：`pi install npm:@ollama/pi-web-search`

**功能**：为 Pi 代理添加 Web 搜索和网页内容抓取工具。安装后代理自动获得搜索互联网和读取网页内容的能力，无需额外配置。

**使用**：安装即用。当代理需要查找最新信息、文档或外部资料时，会自动调用搜索和抓取工具。

```
# 安装后直接在对话中使用，代理会自动搜索
> 查一下 React 19 的新特性
（代理自动触发 web_search 工具）
```

**备选**：社区版 `pi-search`（`pi install npm:pi-search`），支持 9 种搜索后端（DuckDuckGo、Tavily、Brave、Exa 等），带自动回退和速率限制。

---

## 3. @plannotator/pi-extension — 交互式计划审查

**安装**：`pi install npm:@plannotator/pi-extension`

**要求**：Pi >= 0.53.0

**功能**：在浏览器中提供可视化的计划审查 UI。代理制定计划后，你可以在浏览器中标注、批注、批准或拒绝。包含状态机管理（idle → planning → executing → idle），在计划阶段阻止破坏性命令。

**配置层级**（优先级从高到低）：
1. `<cwd>/.pi/plannotator.json` — 项目级覆盖
2. `~/.pi/agent/plannotator.json` — 全局用户配置
3. 内置 `plannotator.json` — 包自带默认配置

可配置项：模型、思考级别、可用工具、状态标签、各阶段系统提示词。

**命令**：

```
# 以计划模式启动 PI
pi --plan

# 切换计划模式
/plannotator                # 或 Ctrl+Alt+P

# 审查当前 git 变更
/plannotator-review

# 标注任意 Markdown 文件
/plannotator-annotate <file>

# 标注代理最近一次回复
/plannotator-last

# 查看当前阶段、计划文件、进度
/plannotator-status
```

**典型工作流**：
1. `pi --plan` 启动，代理探索代码库
2. 代理编写计划（Markdown 清单），调用 `plannotator_submit_plan`
3. 浏览器自动打开 Plannotator UI
4. 你在 UI 中标注问题、批准或拒绝
5. 批准后进入执行阶段，代理按计划实施

---

## 4. @juicesharp/rpiv-ask-user-question — 结构化问卷

**安装**：`pi install npm:@juicesharp/rpiv-ask-user-question`

**来源**：[rpiv-mono monorepo](https://github.com/juicesharp/rpiv-mono/tree/main/packages/rpiv-ask-user-question)

**功能**：当代理遇到模棱两可的决策时，不会自由猜测，而是以带类型选项的结构化问卷向你提问。选项可以是单选、多选、文本输入等固定类型，避免自由文本回复的歧义。

**使用**：安装即用。当代理需要确认或选择时自动触发问卷：

```yaml
# 代理会在类似以下场景触发问卷
# - 选择技术方案："用 JWT 还是 Session 认证？"
#   options: [{label: "JWT", description: "无状态，适合分布式"}, {label: "Session", description: "有状态，撤销方便"}]
# - 确认破坏性操作："确定要删除 user 表吗？"
#   type: confirm
# - 输入参数："数据库连接字符串是什么？"
#   type: text
```

**依赖**：`@earendil-works/pi-coding-agent`（使用 Earendil 分支的 Pi 代理）。

---

## 5. @juicesharp/rpiv-todo — 持久化任务列表

**安装**：`pi install npm:@juicesharp/rpiv-todo`

**来源**：[rpiv-mono monorepo](https://github.com/juicesharp/rpiv-mono/tree/main/packages/rpiv-todo)

**功能**：为代理提供持久化的 Todo 列表覆盖层。任务列表在 `/reload` 和对话压缩后依然存活。配合 **pi-kanban**（`pi install npm:pi-kanban`）可将 Todo 渲染为看板列。

**使用**：

```
# 代理在对话中自动维护 Todo 列表
# 任务状态在 TUI 中以覆盖层形式实时展示

# 启动看板 Web 界面（需先安装 pi-kanban）
/kanban start           # 启动看板服务
/kanban open            # 在浏览器中打开 http://localhost:8099
/kanban status          # 查看服务状态
/kanban stop            # 停止服务
```

**配合 pi-kanban 的完整效果**：见下方 pi-kanban 章节。

---

## 6. pi-kanban — Web 看板仪表盘

**安装**：`pi install npm:pi-kanban`

**功能**：Pi 代理的 Web 仪表盘，提供会话浏览、任务看板、子代理可视化。自带完整交互式 Demo，可在接入真实 Pi 会话前探索功能。

**依赖**：需配合 `pi-subagents` 和 `@juicesharp/rpiv-todo` 才能发挥全部功能。

**全部命令**：

```
/kanban start           # 启动仪表盘服务
/kanban stop            # 停止服务
/kanban restart         # 重启服务
/kanban status          # 查看运行状态
/kanban open            # 在浏览器中打开（默认 http://localhost:8099）
/kanban web             # 打开 Web UI
/kanban app             # 启动应用视图
/kanban pin             # 固定仪表盘
/kanban sticky-pin      # 粘性固定
/kanban unpin           # 取消固定
/kanban preview         # 预览仪表盘
/kanban link            # 获取可分享链接
```

**典型启动流程**：

```bash
# 1. 安装 pi-kanban 及其配套扩展
pi install npm:pi-kanban
pi install npm:pi-subagents
pi install npm:@juicesharp/rpiv-todo

# 2. 在 Pi 会话中启动仪表盘
/kanban start

# 3. 打开浏览器
/kanban open
```

**仪表盘功能区域**：
| 区域 | 说明 |
|------|------|
| Sessions | 浏览、查看所有 Pi 编码会话 |
| Kanban 看板 | Todo → In Progress → Done 列，数据来自 rpiv-todo |
| 子代理视图 | 每个父会话下嵌套展示 pi-subagents 层级 |
| 存储管理 | 管理会话数据和存储空间 |
| 主题系统 | 内置多套主题，支持自定义主题 |

---

## 7. taskplane — 多代理任务编排

**安装**：`pi install npm:taskplane`

**要求**：Node.js >= 22, Pi, Git

**来源**：[GitHub](https://github.com/HenryLach/taskplane)

**功能**：多代理 AI 编码编排系统，将想法拆解为结构化任务，通过 Git worktree 隔离并行执行。四种代理角色协作完成复杂编码任务。

**四种代理角色**：

| 角色 | 职责 |
|------|------|
| Supervisor（主管） | 监控批次进度，处理失败，可自主调用编排命令 |
| Worker（工人） | 在持久化上下文中执行任务步骤 |
| Reviewer（审查者） | 跨模型质量门禁（建议使用不同于 Worker 的模型） |
| Merger（合并者） | 自动合并到编排分支 `orch` |

**初始化与使用**：

```bash
# 在项目中初始化
cd my-project
taskplane init        # 创建 .pi/ 配置、代理提示词、示例任务
taskplane doctor      # 验证安装

# 启动 Web 仪表盘（http://localhost:8099，SSE 实时流）
taskplane dashboard
```

**Pi 会话内命令**：

```
/ orch                      # 通用入口 — 自动检测状态，引导流程
/ orch-plan all             # 预览执行计划（批次、泳道、依赖）
/ orch all                  # 并行执行所有待处理任务
/ orch taskplane-tasks/EXAMPLE-001-hello-world/PROMPT.md  # 单任务隔离执行
/ orch-status               # 查看批次进度
/ orch-pause                # 暂停批次
/ orch-resume               # 恢复暂停批次
/ orch-abort                # 中止批次
/ orch-deps                 # 显示依赖图
/ orch-sessions             # 列出活动 Worker 会话
/ orch-integrate            # 将完成批次合并到工作分支
```

**任务结构**：每个任务目录包含 `PROMPT.md`（任务使命、步骤、约束）和 `STATUS.md`（进度跟踪）。Worker 按步骤执行，每步边界自动 commit，即使崩溃也不丢失工作。

---

## 8. pi-gsd — GSD 结构化开发框架

**安装**：`pi install npm:pi-gsd`（暂时不安装）

**功能**：「Get Shit Done」方法论的非官方移植（v1.30.0 → pi 平台）。提供 6 阶段开发工作流 + 57 条 `/gsd-*` 命令 + WXP 预处理引擎。所有状态存储在 `.planning/` 目录中，提交到 git，跨会话存活。

**模型配置档**：

| Profile | 说明 |
|---------|------|
| `quality` | 全代理使用最强模型（Opus/Pro） |
| `balanced` | 默认档（Sonnet/Flash 级别） |
| `budget` | 每个代理使用最便宜的可用模型 |
| `inherit` | 跟随当前会话模型 |

**6 阶段工作流**：

```
/gsd-new-project                        # 初始化项目
  └─► /gsd-discuss-phase <N>            # 讨论阶段 — 澄清需求
        └─► /gsd-plan-phase <N>         # 计划阶段 — 制定方案
              └─► /gsd-execute-phase <N> # 执行阶段 — 实施编码
                    └─► /gsd-verify-work <N>  # 验证阶段 — 检查成果
                          └─► /gsd-validate-phase <N>  # 确认阶段 — 最终验收
                                └─► /gsd-complete-milestone  # 完成里程碑

# 切换模型配置档
/gsd-set-profile balanced
```

**CLI 工具**：`pi-gsd-tools` 提供状态查询、路线图分析、健康检查等命令。

```bash
pi-gsd-tools state json                # 导出 STATE.md 为 JSON
pi-gsd-tools roadmap analyze --raw     # 分析 ROADMAP.md
pi-gsd-tools validate health --repair  # 检查并自动修复 .planning/
pi-gsd-tools stats json                # 项目统计数据
```

**WXP 预处理引擎**：v2.0 引入的 XML 预处理引擎，在 LLM 看到消息之前执行 shell 命令、条件判断、数组迭代、文件读写等操作，避免 LLM 用 bash 来回执行。

---

## 安装汇总

```sh
pi install npm:pi-subagents
pi install npm:@ollama/pi-web-search
pi install npm:@plannotator/pi-extension
pi install npm:@juicesharp/rpiv-ask-user-question
pi install npm:@juicesharp/rpiv-todo
pi install npm:taskplane
pi install npm:pi-gsd               # 暂时不安装
pi install npm:pi-kanban            # 配合 rpiv-todo 使用（可选）
```
