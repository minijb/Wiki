---
title: "PI Agent"
type: entity
updated: 2026-05-09
aliases: [pi-coding-agent, pi agent]
tags: [pi-agent, ai-coding, tool, cli]
---

# PI Agent

## 概述

PI 是由 [@mariozechner](https://github.com/mariozechner) 创建的 AI 编码代理 CLI 工具。通过 `npm install -g @mariozechner/pi-coding-agent` 安装。其扩展系统允许从 npm 或 git 安装插件，每个插件提供额外的工具、命令或 UI。

与 Claude Code 不同，PI 是独立的开源编码代理，拥有自己的扩展生态。

## 核心机制

- **扩展系统** — 通过 `pi install npm:<package>` 安装插件，插件可注册工具、slash 命令、TUI 覆盖层
- **TUI 界面** — 终端内交互界面，支持多标签、覆盖层、快捷键
- **配置层级** — 项目级 `.pi/` > 用户级 `~/.pi/agent/` > 扩展内置默认

## 扩展生态

| 扩展 | 功能 |
|------|------|
| pi-subagents | 子代理委派，链式/并行执行 |
| @ollama/pi-web-search | Web 搜索 + 网页抓取 |
| @plannotator/pi-extension | 交互式计划审查（浏览器 UI） |
| @juicesharp/rpiv-ask-user-question | 结构化问卷 |
| @juicesharp/rpiv-todo | 持久化任务列表 |
| pi-kanban | Web 仪表盘 |
| taskplane | 多代理任务编排 |
| pi-gsd | GSD 结构化开发框架（6 阶段工作流） |

## 对比

PI 与 Claude Code 同为 AI 编码代理 CLI，但 PI 的扩展系统更开放——任何 npm 包均可作为扩展发布，社区贡献的门槛更低。

## 关联页面

- [[sources/PI-Agent-扩展插件-摘要|PI Agent 扩展插件]] — 各扩展的详细安装和使用说明
