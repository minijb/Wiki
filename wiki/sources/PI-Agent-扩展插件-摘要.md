---
title: "PI Agent 扩展插件"
type: source-summary
updated: 2026-05-09
source: "raw/tools/ai-coding/PI-Agent-扩展插件.md"
tags: [pi-agent, ai-coding, extensions, plugins]
---

# PI Agent 扩展插件

## 来源

原始笔记 `raw/tools/ai-coding/PI-Agent-扩展插件.md`，基于 npm 注册表和 GitHub 搜索整理。

## 要点

- PI 是由 @mariozechner 创建的 AI 编码代理（非 Claude Code），通过 `pi install npm:<package>` 安装扩展
- **pi-subagents** — 子代理委派，支持链式/并行/后台执行，内置 10 种专业子代理，用 Markdown frontmatter 定义代理配置
- **@ollama/pi-web-search** — 为 PI 添加 Web 搜索和网页抓取能力，安装即用
- **@plannotator/pi-extension** — 交互式计划审查，浏览器 UI 标注/审批，含 idle→planning→executing 状态机
- **@juicesharp/rpiv-ask-user-question** — 结构化问卷，代理以带类型选项的问卷向用户提问而非自由猜测
- **@juicesharp/rpiv-todo** — 持久化 Todo 覆盖层，存活于 /reload 和对话压缩，配合 pi-kanban 展示看板列
- **pi-kanban** — Web 仪表盘，提供会话浏览、看板、子代理可视化，12 个 /kanban 命令
- **taskplane** — 多代理编排系统，4 种角色（Supervisor/Worker/Reviewer/Merger），Git worktree 隔离并行执行
- **pi-gsd** — GSD 结构化开发框架，6 阶段工作流 + 57 条命令 + WXP 预处理引擎

## 关联页面

- [[entities/PI-Agent|PI Agent]] — PI 编码代理实体页
