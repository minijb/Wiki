---
title: "Luban 配置工具 — 来源摘要"
type: source-summary
updated: 2026-06-03
tags: [unity, luban, config, data-table, tooling, 面试]
source: "raw/gamedev/gameplay/luban-config.md"
---

# Luban 配置工具 — 来源摘要

来源文件：[[raw/gamedev/gameplay/luban-config.md]]

## 核心内容

Luban 是 focus-creative-games 开源的游戏配置表解决方案，支持 Excel → 多格式（JSON/Binary/Lua/C#）转换管线。

### 关键要点

1. **安装与集成**：通过 git URL 安装 luban_unity 包，配置 gen_config.bat 脚本指定 Excel 源目录和输出目录
2. **运行时加载**：通过回调函数控制加载逻辑，支持 Resources/AssetBundle/网络下载
3. **类型系统**：支持 int/float/string/bool/enum/list/map/bean/datetime/ForeignKey 等丰富类型
4. **多格式输出**：同一套 Excel 可输出 JSON（开发）、Binary（发布）、Lua（热更）等多种格式
5. **编译期校验**：类型检查/外键完整性/范围验证/唯一性约束在生成阶段暴露，非运行时崩溃
6. **XLua 配合**：Luban + XLua 是 Unity 热更新常见组合，配置表可走 Lua 热更通道

## 参见

- [[concepts/luban配置工具|Luban 配置工具]]
- [[concepts/XLua热补丁|XLua 热补丁]]
- [[concepts/Unity脚本架构|Unity 脚本架构]]
