---
title: "AI 编程工具完全指南：Claude Code 与 PI Agent"
updated: 2026-06-02
tags: [ai-coding, claude-code, pi-agent, mcp, skills, agents, commands, hooks, memory]
type: tool
aliases: [AI编程工具, ClaudeCode教程, AI Coding Tools, MCP指南]
description: AI 编程工具完整指南：Claude Code 全面教程（安装/CLAUDE.md记忆系统/MCP配置/Agents子代理/Commands命令/Hooks钩子/Skills技能/输出风格/Model Alias/思考模式/速查表）、WSL2+Tmux进阶工作流、CCSwitch API切换、PI Agent扩展生态（子代理/搜索/看板/任务编排）、AI工具对比与选择、社区资源
---

# AI 编程工具完全指南

> [!abstract] 覆盖范围
> Claude Code CLI 完整教程（六层记忆架构、MCP 服务器配置与排错、Agents/Commands/Hooks 系统、Skills 技能开发、进阶工作流）+ PI Agent 扩展生态 + CCSwitch API 代理 + 其他 AI 编程辅助工具。

## Claude Code 学习路径

### 快速入门

| 笔记 | 说明 | 推荐阅读顺序 |
|------|------|------------|
| [[#速查表]] | 常用命令速查 | ⭐ 必读 |
| [[#记忆系统]] | 理解 AI 记忆机制 | ⭐ 必读 |
| [[#mcp-使用指南]] | MCP 服务器配置 | ⭐⭐⭐ 必装 |

### 核心功能

| 章节 | 说明 | 推荐度 |
|------|------|--------|
| [[#agents-与-commands]] | 自动化与自定义 | ⭐⭐ 进阶 |
| [[#skills-技能系统]] | 专业知识注入 | ⭐⭐⭐ 进阶 |

### 高级应用

| 章节 | 说明 | 推荐度 |
|------|------|--------|
| [[#进阶工作流]] | WSL2 + Tmux 集成 | ⭐⭐ 高阶 |

### 学习路线

- **初级路线（1-2天）**：速查表 → 记忆系统 → MCP 基础配置
- **进阶路线（3-5天）**：进阶路线 + Agents/Commands → 自定义工作流
- **精通路线（1周+）**：进阶路线 + Tmux 多会话 → MCP 深度定制 → 工作流自动化

---

## 速查表

### 会话管理

| 命令 | 功能 | 说明 |
|------|------|------|
| `/init` | 初始化项目 | 读取项目根目录的 CLAUDE.md 作为全局上下文 |
| `/resume` | 重新使用会话 | 恢复之前的对话上下文 |
| `/rewind` | 切换会话节点 | 回退到之前的对话节点继续 |
| `/clear` | 清空会话 | 完全重置当前对话 |
| `/compact` | 压缩对话上下文 | 保留核心信息，节省 Token |
| `/history` | 查看历史对话 | 选择之前的对话继续 |
| `/help` | 显示帮助 | 查看所有可用命令 |
| `/exit` | 退出 | 返回普通终端 |
| `/model` | 切换模型 | 选择不同的 Claude 模型 |
| `/edit` | 编辑记忆文件 | 修改用户或项目记忆 |
| `/mcp` | 查看 MCP 状态 | 查看已安装的 MCP 服务器 |
| `/context` | 查看 Token 占用 | 查看当前上下文 Token 使用情况 |
| `/hooks` | 查看钩子 | 查看已配置的 Hook |
| `/agents` | 查看 Agent | 查看已配置的 Agent |

### 思考模式

使用 `think` / `think hard` / `think harder` / `ultrathink` 切换思考深度：

| 模式 | 思考深度 | Token 消耗 | 适用场景 | 响应时间 |
|------|----------|------------|----------|----------|
| `think` | 基础 | 低 | 简单问题、快速回答 | 2-5秒 |
| `think hard` | 深度 | 中 | 复杂逻辑、算法设计 | 5-15秒 |
| `think harder` | 更深度 | 高 | 架构设计、难题分析 | 15-30秒 |
| `ultrathink` | 极深度 | 极高 | 最复杂问题、创新方案 | 30-60秒 |

### 上下文管理

- **需求文档** — 明确目标与约束
- **项目状态** — 当前进度与问题
- **代办清单** — 待办事项追踪

```text
# 压缩上下文
/compact

# 清理会话
/clear
```

### 快捷操作

- 直接 `Ctrl+V` 粘贴图片，自动保存到附件目录
- Output Style：`/output-style [default|explanatory|learning|new <name>]`
- Mode Alias：Windows CMD `doskey crazy = claude --dangerously-skip-permissions $*`
- 后台运行：`run python -m http.server 8080 &`

---

## 记忆系统

Claude Code 记忆系统让 AI 能够跨会话记住个人编码偏好、项目特定配置、常用工具和工作流程。

### 六层记忆架构

| 层级 | 记忆类型 | 位置 | 作用域 | 用途 |
|------|----------|------|--------|------|
| 1 | 用户记忆 | `~/.claude/CLAUDE.md` | 全局 | 个人偏好、编码风格 |
| 2 | 项目记忆 | 项目根目录 `.CLAUDE.md` | 项目 | 项目特定信息 |
| 3 | 对话记忆 | 当前会话 | 会话 | 当前任务上下文 |
| 4 | 工具记忆 | `/memory` | 会话 | 跨对话的信息 |
| 5 | MCP 记忆 | MCP 服务器 | 扩展 | 外部服务数据 |
| 6 | 记忆文件 | `~/.claude/memory/` | 可配置 | 持久化知识库 |

### 第一层：用户记忆

**文件位置**：`~/.claude/CLAUDE.md`

全局级别的个人偏好设置，影响所有项目。

```markdown
# 用户偏好

## 编码风格
- 使用 4 空格缩进
- 始终在 if 语句使用大括号
- 变量命名使用 camelCase

## 常用语言
- 主要使用 C#
- 也会使用 Lua 和 Python

## 偏好工具
- 使用 Unity 作为游戏引擎
- 使用 Obsidian 做笔记
- VSCode 作为代码编辑器

## 工作流程
- 每次修改前先阅读相关代码
- 大改动先做备份
- 完成后简要总结改动
```

编辑方式：`/edit memory` 或直接编辑文件。

### 第二层：项目记忆

**文件位置**：`<项目根目录>/.CLAUDE.md`

项目级别的配置，只影响当前项目。

```markdown
# 项目配置

## 项目名称
MyGame

## 技术栈
- Unity 2022.3 LTS
- C# 10
- Addressables 资源管理

## 代码规范
- 命名空间: CompanyName.GameName
- UI 脚本放在 Scripts/UI/
- 使用 [RequireComponent] 确保依赖

## 特殊要求
- 所有 MonoBehaviour 需要 EditorWindow 配置
- 资源路径使用 Path.Combine
```

编辑方式：`/edit project`。

### 第三层：对话记忆

仅在当前会话有效，会话结束后自动清除。

**管理技巧**：
- `/compact` — 压缩上下文，保留核心信息
- `/clear` — 清空并重新开始
- `/context` — 查看 Token 使用

**优化策略**：
1. 使用分段任务：将大任务拆分为小步骤
2. 及时压缩：Token 接近上限时使用 `/compact`
3. 总结输出：每个阶段完成后要求总结

### 第四层：MCP 记忆服务器

```bash
claude mcp add memory -s user -- npx -y @modelcontextprotocol/server-memory
```

用途：跨对话保存重要信息——项目进展记录、会议纪要、决策历史、知识沉淀。

### 第五层：MCP 外部服务

通过 Context 7 等 MCP 访问开源项目文档：

```bash
claude mcp add context7 -s user -- npx -y @context7/mcp-server
```

### 第六层：记忆文件目录

```text
~/.claude/memory/
├── projects/          # 项目相关信息
├── concepts/          # 概念和知识
├── decisions/         # 重要决策记录
└── daily/            # 日常记录
```

### 上下文管理策略

在复杂任务中保持清晰的三个标准化文档：

| 文档 | 内容 | 用途 |
|------|------|------|
| 需求文档 | 目标、功能、约束 | 明确方向 |
| 项目状态 | 当前进度、问题 | 了解现状 |
| 代办清单 | 待办事项 | 追踪进度 |

**推荐工作流**：

```text
1. 项目初始化 → 2. 读取 .CLAUDE.md（项目记忆）→ 3. 读取用户 CLAUDE.md（个人偏好）
→ 4. 执行任务（对话记忆）→ 5. 使用 MCP（外部知识）→ 6. 必要时压缩上下文 → 7. 任务完成，总结记录
```

---

## MCP 使用指南

MCP（Model Context Protocol）是 Anthropic 推出的开源通信标准，让 Claude Code 可以访问本地文件系统、连接 API 服务、操作数据库、集成开发工具、自动化任务。

### 作用域配置

| 作用域 | 配置位置 | 适用场景 | 命令标志 |
|--------|----------|----------|----------|
| Local | 当前目录 `.mcp.json` | 项目特定工具 | 默认（无标记） |
| User | `~/.claude/settings.json` | 全局常用工具 | `-s user` 或 `--scope user` |
| Project | `.mcp.json`（团队共享） | 团队共享工具 | `-s project` 或 `--scope project` |

**作用域优先级**：Local > Project > User。相同工具名的本地配置会覆盖全局配置。

### 添加 MCP 服务器

#### 命令行添加（推荐新手）

```bash
# 基本语法
claude mcp add <名称> [选项] -- <命令> [参数...]

# 添加文件系统访问
claude mcp add filesystem -s user -- npx -y @modelcontextprotocol/server-filesystem ~/Documents ~/Projects

# 添加 GitHub 集成
claude mcp add github -s user -e GITHUB_TOKEN=your_token -- npx -y @modelcontextprotocol/server-github
```

#### 配置文件（推荐高级用户）

编辑 `~/.claude/settings.json`：

```json
{
  "mcpServers": {
    "filesystem": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-filesystem", "/Users/username/Documents"],
      "env": {}
    },
    "github": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": { "GITHUB_TOKEN": "your_github_token" }
    }
  }
}
```

#### Windows 配置

在 Windows 下必须使用 `cmd /c` 前缀：

```json
{
  "mcpServers": {
    "filesystem": {
      "type": "stdio",
      "command": "cmd",
      "args": ["/c", "npx -y @modelcontextprotocol/server-filesystem C:/Users/username/Documents"],
      "env": {}
    },
    "context7": {
      "type": "stdio",
      "command": "cmd",
      "args": ["/c", "npx -y @upstash/context7-mcp --api-key=YOUR_API_KEY"],
      "env": {}
    }
  }
}
```

```bash
# 命令行添加的 Windows 写法
claude mcp add context7 -- cmd /c "npx -y @upstash/context7-mcp --api-key=YOUR_API_KEY"

claude mcp add filesystem -s user -- cmd /c "npx -y @modelcontextprotocol/server-filesystem C:/Users/username/Documents"
```

> [!warning] Windows 注意事项
> 不使用 `cmd /c` 可能遇到 `ENOENT`（找不到命令）或 Windows shell 参数解析错误。

### 10 个必备 MCP 服务器

| # | 名称 | 安装命令 | 用途 |
|---|------|----------|------|
| 1 | **文件系统** | `claude mcp add filesystem -s user -- npx -y @modelcontextprotocol/server-filesystem ~/Documents ~/Projects` | 直接读写文件，修改代码 |
| 2 | **GitHub** | `claude mcp add github -s user -e GITHUB_TOKEN=xxx -- npx -y @modelcontextprotocol/server-github` | 管理 Issues、PRs、代码审查 |
| 3 | **Sequential Thinking** | `claude mcp add thinking -s user -- npx -y @modelcontextprotocol/server-sequential-thinking` | 复杂问题分步骤思考 |
| 4 | **Context 7** | `claude mcp add context7 -s user -- npx -y @context7/mcp-server` | 访问大量开源项目文档 |
| 5 | **Brave Search** | `claude mcp add search -s user -e BRAVE_API_KEY=xxx -- npx -y @modelcontextprotocol/server-brave-search` | AI 增强的网络搜索 |
| 6 | **Puppeteer** | `claude mcp add puppeteer -s user -- npx -y @modelcontextprotocol/server-puppeteer` | 自动化网页操作、爬虫 |
| 7 | **PostgreSQL** | `claude mcp add postgres -s user -e DATABASE_URL=xxx -- npx -y @modelcontextprotocol/server-postgres` | 直接查询和操作数据库 |
| 8 | **Fetch** | `claude mcp add fetch -s user -- npx -y @kazuph/mcp-fetch` | 调用各种 REST API |
| 9 | **Slack** | `claude mcp add slack -s user -e SLACK_TOKEN=xxx -- npx -y @modelcontextprotocol/server-slack` | 发送消息、管理频道 |
| 10 | **Memory** | `claude mcp add memory -s user -- npx -y @modelcontextprotocol/server-memory` | 跨对话保存信息 |

### 其他推荐 MCP

| MCP 名称 | 用途 | 场景 |
|----------|------|------|
| `figma` | Figma 设计稿读取 | UI 开发 |
| `playwright` | 浏览器自动化 | Web 测试 |
| `linear` | 任务管理 | 项目管理 |
| `sentry` | 错误监控 | 线上问题追踪 |
| `jira` | 企业任务管理 | 团队协作 |

### MCP 管理命令

```bash
claude mcp list                              # 查看已安装的 MCP 服务器
claude mcp remove <server_name>              # 删除 MCP 服务器
claude mcp test <server_name>                # 测试 MCP 服务器
claude mcp add-from-claude-desktop           # 从 Claude Desktop 导入
/mcp                                          # 在对话中查看 MCP 状态
```

### 故障排除

**常见错误 1：工具名称验证失败**

工具名称只能包含小写字母、数字和连字符。遇到此错误时检查 MCP 配置文件中的 `name` 字段。

**常见错误 2：MCP 服务器无法启动**

1. 确认 Node.js / npm 已正确安装
2. 测试手动运行命令：`npx -y @modelcontextprotocol/server-filesystem`
3. 检查环境变量是否正确设置
4. Windows 下确保使用 `cmd /c` 前缀

**常见错误 3：作用域冲突**

如果同名工具在多个作用域配置，本地配置覆盖全局。使用 `/mcp` 查看当前加载的所有 MCP 服务器及所属作用域。

---

## Agents 与 Commands

### Commands 自定义命令

Commands 允许创建可复用的快捷命令，类似 Shell Alias。

**文件位置**：`~/.claude/commands/xxx.md`

创建文件 `~/.claude/commands/code-review.md`：

```markdown
# 快捷代码审查

请审查当前项目的主要代码文件，关注以下方面：
1. 代码质量和最佳实践
2. 潜在的安全问题
3. 性能优化建议
4. 文档完整性

完成后总结发现的问题和建议。
```

**使用方式**：在 Claude Code 中调用 `/code-review` 或 `/code-review --scope backend`。

### Agents 子代理

Agents 是 Claude Code 的自动化执行单元，可独立完成复杂任务。

**文件位置**：`~/.claude/agents/xxx.md`

创建文件 `~/.claude/agents/find-txt.md`：

```markdown
# 查找项目中的 TXT 文件

你是文件搜索专家。使用 Glob 工具在项目中搜索所有 .txt 文件。

任务：
1. 搜索项目根目录及所有子目录
2. 列出找到的所有文件路径
3. 统计文件总数

---
type: file-search
capabilities:
  - glob
  - read
---
```

**使用方式**：`/agents` 查看可用 Agent，输入 Agent 名称（如 `find-txt`）调用。

**Agent 特点**：
- 可被 Claude Code 自动调用
- 支持复杂任务分解
- 可指定工具能力

### Hook 钩子系统

Hook 允许在 Claude Code 的关键节点执行自定义操作。

| Hook 类型 | 触发时机 | 用途 |
|-----------|----------|------|
| `preTool` | 工具执行前 | 验证参数、日志记录 |
| `postTool` | 工具执行后 | 结果处理、通知 |
| `preTask` | 任务开始前 | 准备工作、环境检查 |
| `postTask` | 任务完成后 | 清理工作、结果汇总 |

**文件位置**：`~/.claude/hooks/xxx.json`

```json
{
  "hooks": {
    "preTool": [
      {
        "name": "log-tool-use",
        "description": "记录工具使用日志",
        "command": "echo '[$(date)] Tool: {tool_name}, Args: {tool_args}' >> ~/.claude/hooks/log.txt"
      }
    ],
    "postTool": [
      {
        "name": "validate-output",
        "description": "验证输出格式",
        "command": "jq empty {output_path} || echo 'Invalid JSON'"
      }
    ]
  }
}
```

**常用 Hook 示例**：

自动生成提交信息：
```json
{
  "hooks": {
    "postTask": [
      { "name": "git-commit-msg", "command": "git log -1 --pretty=%B > ~/.claude/hooks/last_commit.txt" }
    ]
  }
}
```

**Hook 调试**：
```bash
claude --debug-hooks
```

安装 jq 用于验证：`winget install jqlang.jq`（Windows）或 `brew install jq`（macOS）或 `sudo apt install jq`（Linux）。

### Mode Alias

**Windows CMD**：
```cmd
doskey crazy = claude --dangerously-skip-permissions $*
```

**macOS / Linux**：
```bash
alias cc='claude'
alias cci='claude --init'
alias cch='claude --help'
```

### Output Style 输出风格

| 风格 | 说明 |
|------|------|
| `default` | 软件工程风格（默认） |
| `explanatory` | 启发性、教育性 |
| `learning` | 学习型 |
| `new` | 创建自定义风格 |

```text
/output-style explanatory
/output-style new my-style
```

---

## Skills 技能系统

Skills（技能）让 Claude Code 获得专业领域的深度知识、遵循特定规范和模式、执行复杂工作流、记忆项目上下文。

### Plugin 与 Skill 的区别

**Plugin（插件）**：
- 官方包管理体系的最小安装单位
- 必须包含根目录 `plugin.json` 元数据配置文件
- 可封装 1 个或多个 Skill、Command、Hook、依赖配置
- 安装时以插件为单位，不能只安装插件内的某个单独 Skill

**Skill（技能）**：
- 可直接调用的最小能力单元
- 必须包含 `SKILL.md` 元数据文件
- 可选包含 `skill.json` 额外配置
- 既可独立存在（手动安装），也可被封装在 Plugin 插件内（通过插件市场安装）

### Skill 与 Agent 的对比

| 维度 | Skill (技能) | Agent (智能体) |
|------|-------------|---------------|
| **核心定位** | 知识注入与规范约束。提供专业领域的上下文、最佳实践和标准操作流程。 | 自主执行引擎。具备独立思考、规划和多步工具调用的能力。 |
| **工作方式** | 被动触发/指导。像是一本"操作手册"或"专家指南"，Claude 读了之后按照指导去工作。 | 主动执行。像是一个"专门的员工"，接受任务后自主完成文件读取、搜索、修改等一系列动作。 |
| **触发机制** | 绝大多数通过上下文**自动触发**（根据 `description` 匹配），少部分通过 `/` 命令手动触发。 | 通过工具调用（Tool call）或后台子进程生成，通常用于处理复杂的多步任务。 |
| **适用场景** | 确保代码符合特定框架规范、约束解释风格、提供特定领域背景知识。 | 深入代码库探索、执行需要多次尝试和验证的重构、自主收集整理大量信息。 |
| **资源消耗** | 仅占用上下文 Token，**执行快**。 | 会生成新的独立对话流和思考过程，**耗时较长，Token 消耗较大**。 |

> [!tip] 总结
> 当你需要 Claude **"以某种特定的方式或标准来回答问题/写代码"** 时，使用 **Skill**。
> 当你需要 Claude **"去独立完成一个需要很多步骤、包含不确定性的复杂任务"** 时，使用 **Agent**。

### Skill 目录结构

**简单架构**（最基础只需要一个文件夹 + `SKILL.md`）：

```text
~/.claude/skills/
└── my-skill/
    └── SKILL.md
```

**复杂架构**（适用于功能更丰富的技能）：

```text
~/.claude/skills/
└── my-skill/
    ├── SKILL.md              # 必须：元数据和核心指令
    ├── skill.json            # 可选：额外的元数据配置
    ├── scripts/              # 可选：可执行脚本
    ├── references/           # 可选：参考文档
    └── assets/               # 可选：资源文件
```

### SKILL.md 编写规范

```markdown
---
name: explain-code
description: Explains code with visual diagrams and analogies
---

When explaining code, always include:
1. **Start with an analogy**
2. **Draw a diagram** using ASCII art
3. **Walk through the code** step-by-step
4. **Highlight a gotcha** or common mistake
```

> [!important] 关键点
> - `name` 必须与文件夹名称一致
> - `description` 是核心——Claude 依靠这段描述来决定何时激活该 Skill

### 触发机制

**自动触发**（绝大多数情况）：
Claude Code 会实时分析当前对话内容、打开的文件类型、项目结构和技术栈来自动激活 Skill。

典型触发场景：

| 场景 | 触发机制示例 |
|------|------------|
| 打开 Go 项目并询问代码问题 | `go-review`、`golang-patterns` 自动提供 Go 最佳实践 |
| 粘贴异常栈问"这是什么问题" | 调试相关 Skills 自动分析根因 |
| 修改代码后说"帮我提交" | Git 集成 Skills 自动生成 commit message |
| 在 Spring Boot 项目中添加新功能 | `springboot-patterns`、`springboot-tdd` 提供架构指导 |

**手动触发**（少数情况）：
通过 `/` 命令手动触发——自定义命令型 Skills、明确指令型 Skills（如 `/skill-create`、`/plan`）。

### 常用官方基础插件

| 分类 | 插件名称 | 说明 | 安装命令 |
|------|---------|------|----------|
| **核心基石** | code-simplifier | 自动识别冗余代码、简化复杂逻辑 | `/plugin install code-simplifier@claude-plugins-official` |
| | skill-creator | 根据需求自动生成自定义 Skill | `/plugin install skill-creator@claude-plugins-official` |
| | code-review | 代码审查工具 | `/plugin install code-review@claude-plugins-official` |
| **开发效率** | frontend-design | 前端设计辅助 | `/plugin install frontend-design@claude-plugins-official` |
| | feature-dev | 功能开发工作流 | `/plugin install feature-dev@claude-plugins-official` |
| | pr-review-toolkit | PR 审查工具集 | `/plugin install pr-review-toolkit@claude-plugins-official` |
| **工程化** | security-guidance | 安全指导 | `/plugin install security-guidance@claude-plugins-official` |
| | claude-md-management | `CLAUDE.md` 文件管理 | `/plugin install claude-md-management@claude-plugins-official` |

### 社区高星插件

**everything-claude-code**（⭐ 强烈推荐）：
- GitHub: `affaan-m/everything-claude-code` | 星标: 72k+
- Anthropic 黑客松冠军项目，经过 10 个月实战打磨
- 包含 50+ Skills，覆盖全开发流程
- 热门命令：`/tdd`、`/plan`、`/go-review`、`/python-review`、`/security-review`
- 框架深度覆盖：Spring Boot、Django、Go、Python、Swift、React/Next.js 等

**superpowers**（推荐）：
- GitHub: `obra/superpowers` | 星标: 78k+
- Claude Code 核心贡献者 Jesse Vincent 开发
- 内置 TDD 测试驱动开发工作流、结构化调试方法论、自动代码审查

**claude-mem**（推荐）：
- GitHub: `thedotmack/claude-mem` | 星标: 34k+
- 跨会话记忆项目上下文、业务规则、历史修改
- 智能渐进式披露，按需检索相关记忆

**CCPlugins**（按需使用）：
- GitHub: `brennercruvinel/CCPlugins`
- 提供 24 个专业开发斜杠命令（安装到 `~/.claude/commands/`）
- 开发工作流：`/commit`、`/format`、`/scaffold`、`/test`、`/refactor`
- 代码质量：`/review`、`/security-scan`、`/predict-issues`
- 会话管理：`/session-start`、`/session-end`、`/undo`

> [!warning] CCPlugins 注意
> 这是 Commands（命令），不是自动触发的 Skills！必须手动输入 `/命令` 触发。

---

## 进阶工作流

### WSL2 安装与配置

在 WSL2 中安装 Claude Code，通过代理在 Windows 中使用：

```bash
# 在 WSL2 中安装 Claude Code
curl -sL https://raw.githubusercontent.com/anthropics/claude-code-cli/releases/latest/download/install.sh | sh

# 配置代理（如需要）
export http_proxy=http://localhost:7890
export https_proxy=http://localhost:7890
```

**Windows 别名**：
```cmd
doskey claude=wsl -e claude $*
```

**WSL2 优势**：原生 Linux 环境、高效文件访问（`/mnt/c/` 访问 Windows 文件）、更丰富的命令行工具。

### Tmux 终端复用

Tmux 让你在单个终端中运行多个会话，适合需要同时操作多个项目的场景。

```bash
# 安装（WSL2）
sudo apt install tmux

# 基础操作
tmux new -s mysession        # 创建新会话
tmux attach -t mysession      # 重新连接
tmux ls                       # 列出会话

# 快捷键（Prefix 默认为 Ctrl+b）
## 窗口
c                  # 创建新窗口
n / p              # 下/上一个窗口
,                  # 重命名窗口

## 分屏
%                  # 垂直分屏
"                  # 水平分屏
o                  # 切换面板
x                  # 关闭面板
```

**Tmux + Claude Code 工作流**：

```bash
# 场景：同时开发多个项目
tmux new -s work

# 窗口 1: 主项目开发
cd ~/projects/my-game && claude

# 窗口 2: 笔记整理
cd ~/notes && claude

# 窗口 3: 资料查询（无 Claude Code）
```

**Tmux 配置示例** (`~/.tmux.conf`)：

```bash
# 基础配置
set -g prefix C-a
unbind C-b
bind C-a send-prefix

# 鼠标支持
set -g mouse on

# 更好的配色
set -g default-terminal "screen-256color"

# 状态栏
set -g status-left "#[fg=green]#S #[fg=white]| "
set -g status-right "#[fg=yellow]%Y-%m-%d %H:%M"

# 窗口从 1 开始
set -g base-index 1
setw -g pane-base-index 1
```

### Windows Terminal 分屏

| 快捷键 | 功能 |
|--------|------|
| `Alt + Shift + +` | 垂直分屏 |
| `Alt + Shift + -` | 水平分屏 |
| `Alt + 方向键` | 切换面板 |

### 后台运行

```bash
# 后台运行
run python -m http.server 8080 &

# 使用 nohup 持久化
nohup python -m http.server 8080 > server.log 2>&1 &

# 查看后台任务
jobs
fg %1    # 切回前台
kill %1  # 停止
```

---

## AI 环境搭建（Claude Code + CCSwitch）

### 前置要求

- Node.js >= 18（通过 NVM 管理）
- 网络代理（国内网络环境）

### 工具清单

| 工具 | 用途 |
|------|------|
| [CCSwitch](https://github.com/farion1231/cc-switch) | 跨平台 AI API 切换/代理 |
| [Claude Code](https://claude.ai/code) | Anthropic AI 编程助手 |
| [CCSwitch CLI](https://github.com/SaladDay/cc-switch-cli) | CCSwitch 命令行版本 |
| [MiniMax](https://platform.minimax.com) | 国产 AI 模型 API 提供商 |

### 安装 Claude Code CLI

```bash
npm install -g @anthropic-ai/claude-code
claude --version
```

### 配置 CCSwitch

**方式一：桌面应用**：
1. 下载安装 CCSwitch
2. 在设置中添加 Claude Code 栏位
3. 开启代理功能
4. 配置 AI API Key（推荐 MiniMax）

**方式二：CLI**：
```bash
curl -fsSL https://github.com/SaladDay/cc-switch-cli/releases/latest/download/install.sh | bash
ccswitch status
```

### 常见问题

| 问题 | 解决方案 |
|------|----------|
| 安装慢 | 使用淘宝镜像 `npm config set registry https://registry.npmmirror.com` |
| 代理不生效 | 检查 CCSwitch 代理开关和系统代理设置 |
| API Key 无效 | 确认 MiniMax 账户余额充足 |

---

## 其他 AI 工具

### DeepWiki + 沉浸式翻译

**DeepWiki**：快速翻译 GitHub 代码并使用 AI 进行 Wiki 化。

使用方式：将 `www.github.com/xxx/xxx` 替换为 `www.deepwiki.com/xxx/xxx` 访问。如果页面未被检索，填入 email 后约 10 分钟完成。

**沉浸式翻译**：优秀的翻译软件，配合 DeepWiki 使用效果极佳。

**zread.ai**：可替代 DeepWiki，同时自带中文支持。

### AI 工具推荐

> [!note] 简单建议
> 使用 `CLAUDE.md` 可以添加默认提示词，让 AI 更好地理解项目上下文。

---

## PI Agent 扩展生态

**PI** 是由 [@mariozechner](https://github.com/mariozechner) 创建的 AI 编码代理（非 Claude Code），通过 `npm install -g @mariozechner/pi-coding-agent` 安装。

### pi-subagents — 子代理委派

**安装**：`pi install npm:pi-subagents`

将任务委派给专业子代理，支持链式执行、并行执行、后台运行。每个子代理由 Markdown 文件（YAML frontmatter）定义。

**代理存放位置**（优先级从高到低）：
- 项目级：`.pi/agents/{name}.md`
- 用户级：`~/.pi/agent/agents/{name}.md`
- 内置：`~/.pi/agent/extensions/subagent/agents/`

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
# System prompt
You are a fast codebase scout. Find relevant files and report.
```

**常用命令**：

```bash
/run scout "找到所有认证相关的文件"
/chain scout "分析项目结构" planner "基于结构制定计划" worker
/parallel "scout:搜索API定义" "scout:搜索数据库模型" "scout:搜索前端路由"
/run worker "重构认证模块" --bg
/chain-file my-pipeline
```

### @ollama/pi-web-search — 网页搜索

**安装**：`pi install npm:@ollama/pi-web-search`

为 PI 代理添加 Web 搜索和网页内容抓取工具，安装后自动获得搜索互联网和读取网页内容的能力。

### @plannotator/pi-extension — 交互式计划审查

**安装**：`pi install npm:@plannotator/pi-extension`（需要 PI >= 0.53.0）

在浏览器中提供可视化的计划审查 UI。包含状态机管理（idle → planning → executing → idle），在计划阶段阻止破坏性命令。

**命令**：
```text
pi --plan                    # 以计划模式启动
/plannotator                 # 切换计划模式（或 Ctrl+Alt+P）
/plannotator-review          # 审查当前 git 变更
/plannotator-annotate <file> # 标注任意 Markdown 文件
/plannotator-last            # 标注代理最近一次回复
/plannotator-status          # 查看当前阶段、计划文件、进度
```

### taskplane — 多代理任务编排

**安装**：`pi install npm:taskplane`（需要 Node.js >= 22）

多代理 AI 编码编排系统，通过 Git worktree 隔离并行执行。

**四种代理角色**：
- **Supervisor（主管）**：监控批次进度，处理失败
- **Worker（工人）**：在持久化上下文中执行任务步骤
- **Reviewer（审查者）**：跨模型质量门禁
- **Merger（合并者）**：自动合并到编排分支

```bash
taskplane init        # 创建 .pi/ 配置
taskplane doctor      # 验证安装
taskplane dashboard   # 启动 Web 仪表盘（localhost:8099）
```

---

## 相关资源

- [Claude Code 官方文档](https://docs.claude.ai/claude-code)
- [Claude Code GitHub](https://github.com/anthropics/claude-code-cli)
- [Claude Code Workflow](https://catlog22.github.io/Claude-Code-Workflow/zh/)
- [Claude Code 上下文管理](https://claudecn.com/docs/claude-code/workflows/context-management/)
- [Claude Code 六层记忆架构](https://zhuanlan.zhihu.com/p/2012328012918596090)
- [WSL2 + Claude Code 配置教程](https://zhuanlan.zhihu.com/p/2009054711144288460)
- [B 站上下文管理教程](https://www.bilibili.com/video/BV1rnBKB2EME)
- [CCSwitch](https://github.com/farion1231/cc-switch)
- [CCSwitch CLI](https://github.com/SaladDay/cc-switch-cli)
- [PI Agent](https://github.com/mariozechner/pi-coding-agent)
