---
title: A* 寻路算法 — 来源摘要
type: source-summary
source: raw/cs/algorithms/A-star寻路算法.md
updated: 2026-05-05
tags: [algorithm, pathfinding, gamedev]
---

# A* 寻路算法 — 来源摘要

基于 [[A-star算法|A-star寻路算法]]（源文件）的消化摘要。

## 一句话

A* 是结合 Dijkstra 最短路径保证和贪心最佳优先搜索效率的启发式搜索算法，核心公式 `f(n) = g(n) + h(n)`。

## 关键知识点

- **评估函数**：`f(n) = g(n) + h(n)`，g 为实际代价，h 为启发式估算（必须 admissible）
- **启发式函数**：三种常用距离 — [[曼哈顿距离]]（4方向网格）、[[欧几里得距离]]（任意方向）、[[对角线距离]]（8方向网格）
- **Unity 集成**：NavMesh 内部使用 A* 变体；自定义实现需 PriorityQueue + Closed Set + 路径回溯
- **优化变体**：[[跳点搜索]]（JPS，均匀网格剪枝）、[[分层寻路]]（HPA*，先粗后细）

## 待深入

- Job System + Burst 多线程寻路的具体实现
- NavMesh 与自定义网格寻路的性能对比
