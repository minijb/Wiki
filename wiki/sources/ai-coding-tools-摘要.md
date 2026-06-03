---
title: "AI 编程工具完全指南 — 摘要"
type: source-summary
updated: 2026-06-02
source: "raw/tools/ai-coding/ai-coding-tools.md"
tags: [ai-coding, claude-code, pi-agent, mcp, skills, agents]
---

# AI 编程工具完全指南

## 来源

`raw/tools/ai-coding/ai-coding-tools.md` — AI 编程工具完整指南：Claude Code 全面教程（安装/六层记忆架构/MCP 服务器配置及故障排除/Agents 子代理/Commands 自定义命令/Hooks 钩子系统/Skills 技能开发/Output Style 输出风格/Mode Alias/思考模式/速查表）、WSL2+Tmux 进阶工作流、CCSwitch API 代理配置、PI Agent 扩展生态（子代理/搜索/看板/任务编排/GSD 框架）、AI 环境搭建、社区推荐资源

## 要点

1. **记忆系统（六层架构）** — 用户记忆（`~/.claude/CLAUDE.md`）→ 项目记忆（`.CLAUDE.md`）→ 对话记忆 → 工具记忆（`/memory`）→ MCP 记忆 → 记忆文件目录（`~/.claude/memory/`）
2. **MCP 协议** — 三大作用域（Local/Project/User）、命令行和配置文件两种安装方式、Windows 下必须使用 `cmd /c` 前缀、10 个必备 MCP 服务器（文件系统/GitHub/Sequential Thinking/Context 7 等）、故障排除常见错误
3. **Agents 与 Commands** — Commands（`~/.claude/commands/` 自定义快捷命令）、Agents（`~/.claude/agents/` 子代理自动化单元）、Hooks（preTool/postTool/preTask/postTask 四类事件钩子）
4. **Skills 技能系统** — Plugin vs Skill 区别、Skill vs Agent 对比（知识注入 vs 自主执行）、SKILL.md 编写规范（name/description 元数据 + 核心指令）、自动触发 vs 手动触发、8 个官方基础插件、4 个社区高星插件（everything-claude-code 72k+/superpowers 78k+/claude-mem 34k+/CCPlugins）
5. **思考模式** — `think`/`think hard`/`think harder`/`ultrathink` 四级深度，逐级增加 Token 消耗
6. **进阶工作流** — WSL2 中安装 Claude Code、Windows Terminal 分屏快捷键、Tmux 多会话配置（Prefix/Ctrl+b → Ctrl+a 重映射）、后台运行（`nohup`/`systemd`）
7. **AI 环境搭建** — NVM → Claude Code CLI → CCSwitch（桌面/CLI 双模式）→ MiniMax API Key 配置
8. **PI Agent 扩展生态** — pi-subagents（链式/并行/后台子代理委派）、pi-web-search（网页搜索）、plannotator（交互式计划审查浏览器 UI）、taskplane（Supervisor+Worker+Reviewer+Merger 多代理编排）、pi-kanban（Web 仪表盘）、pi-gsd（6 阶段结构化开发框架）
9. **其他工具** — DeepWiki（代码 Wiki 化）+ 沉浸式翻译、zread.ai（中文支持）

## 关联 Wiki 页面

- [[concepts/AI编程工具|AI 编程工具]] — 概念页
- [[concepts/WSL2与Windows开发环境|WSL2 开发环境]] — WSL2 配置
- [[concepts/Linux Shell环境|Linux Shell 环境]] — Tmux 终端复用
