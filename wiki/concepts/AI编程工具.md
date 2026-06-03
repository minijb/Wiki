---
title: "AI 编程工具"
type: concept
updated: 2026-06-02
tags: [ai-coding, claude-code, pi-agent, mcp, skills, agents, hooks, memory]
aliases: [AI编程工具, ClaudeCode教程, AI Coding, MCP配置]
---

# AI 编程工具

AI 辅助编程工具完整知识体系：Claude Code CLI 核心能力、PI Agent 扩展生态、CCSwitch API 代理与 AI 环境搭建。

## 核心体系

### Claude Code 六层记忆架构

| 层级 | 类型 | 位置 | 作用域 |
|------|------|------|--------|
| 1 | 用户记忆 | `~/.claude/CLAUDE.md` | 全局偏好、编码风格 |
| 2 | 项目记忆 | 项目根 `.CLAUDE.md` | 项目特定配置 |
| 3 | 对话记忆 | 当前会话 | 任务上下文 |
| 4 | 工具记忆 | `/memory` | 跨对话信息 |
| 5 | MCP 记忆 | MCP 服务器 | 外部服务数据 |
| 6 | 记忆文件 | `~/.claude/memory/` | 持久化知识库 |

详见 [[sources/ai-coding-tools-摘要|AI 编程工具来源摘要]]。

### MCP（Model Context Protocol）

Anthropic 开源通信标准，让 Claude Code 连接本地文件系统、API 服务、数据库、开发工具。

**三大作用域**：Local（`.mcp.json`）→ Project（团队共享）→ User（`~/.claude/settings.json`），优先级 Local > Project > User。

**10 个必备 MCP**：文件系统、GitHub、Sequential Thinking、Context 7、Brave Search、Puppeteer、PostgreSQL、Fetch、Slack、Memory。

> [!warning] Windows 配置
> 必须使用 `cmd /c` 前缀：`claude mcp add <name> -- cmd /c "npx -y <package>"`。

### Agents 与 Commands

- **Commands**：可复用快捷命令（`~/.claude/commands/xxx.md`），通过 `/` 前缀调用
- **Agents**：自动化执行单元（`~/.claude/agents/xxx.md`），可被自动调用处理子任务
- **Hooks**：事件钩子系统，在 `preTool`/`postTool`/`preTask`/`postTask` 时机触发自定义脚本

### Skills 技能系统

Skills 是 Claude Code 的**知识注入与规范约束**机制，区别于 Agent 的自主执行：

| 维度 | Skill | Agent |
|------|-------|-------|
| 定位 | 操作手册/专家指南 | 独立员工 |
| 触发 | 自动（上下文匹配） | Tool call |
| 消耗 | Token 低 | Token 高 + 新对话流 |

**社区推荐**：everything-claude-code（50+ Skills）、superpowers（TDD + 代码审查）、claude-mem（跨会话记忆）。

### 思考模式与上下文管理

四种思考深度：`think` → `think hard` → `think harder` → `ultrathink`，逐级增加 Token 消耗和响应时间。

三个标准化文档保持上下文清晰：需求文档（目标约束）、项目状态（进度问题）、代办清单（事项追踪）。`/compact` 压缩上下文，`/clear` 清空会话。

### PI Agent 扩展生态

- **pi-subagents**：子代理委派（链式/并行/后台执行）
- **pi-web-search**：网页搜索能力
- **plannotator**：交互式计划审查（浏览器 UI）
- **taskplane**：多代理任务编排（Supervisor + Worker + Reviewer + Merger）
- **pi-kanban**：Web 看板仪表盘
- **pi-gsd**：GSD 结构化开发框架（6 阶段工作流 + 57 条命令）

### AI 环境搭建

```bash
# NVM 管理 Node → Claude Code CLI → CCSwitch API 代理
npm install -g @anthropic-ai/claude-code
ccswitch status
```

配合 AI 辅助工具：DeepWiki（代码 Wiki 化）+ 沉浸式翻译。

## 关联页面

- [[sources/ai-coding-tools-摘要|AI 编程工具来源摘要]]
- [[concepts/WSL2与Windows开发环境|WSL2 开发环境]] — WSL2 + Claude Code 集成
- [[concepts/Linux Shell环境|Linux Shell 环境]] — Tmux 多会话工作流
- [[concepts/Dev工具集|Dev 工具集]] — 笔记系统与 Obsidian 插件
- [[concepts/Git操作完全指南|Git 操作指南]] — 版本控制与代码管理
