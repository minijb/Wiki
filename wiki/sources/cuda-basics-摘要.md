---
title: "CUDA 基础 — 摘要"
type: source-summary
updated: 2026-06-02
tags: [cuda, gpu, parallel-computing]
source: "raw/cs/gpgpu/cuda-basics.md"
---

# CUDA 基础 — 摘要

来源：`raw/cs/gpgpu/cuda-basics.md`

## 概述

CUDA 并行计算平台完整介绍：GPU 硬件架构（SM / 线程束 / SIMT）、内存模型（寄存器 / 共享内存 / 全局内存 / 常量内存 / 纹理内存）、编程基础（cudaMalloc / cudaMemcpy / Kernel 启动）、环境配置与开发工具链。

## 要点

- **SM 架构**：SM 通过 SIMT 架构以 Warp（32 线程）为基本执行单元并发执行线程
- **线程层次**：Grid → Block → Thread；Block 大小建议 32 的倍数
- **物理 vs 逻辑**：Grid/Block 是逻辑划分，SM 是物理执行层；一个 kernel 分配到多个 SM
- **内存层次**：寄存器（线程私有，最快）→ 共享内存（Block 内共享）→ 全局内存（所有线程，最慢）
- **共享内存生命周期**：与 Block 一致，Block 结束后释放
- **Warp 分化**：分支结构导致 Warp 内线程串行执行，严重影响性能
- **内存传输**：PCI 总线带宽远低于显存，尽量减少 Host-Device 数据拷贝
- **编译**：使用 `nvcc` 编译 `.cu` 文件；VS Code 可安装 vscode-cudacpp 和 Nsight 插件

## 关联页面

- [[concepts/CUDA并行计算|CUDA 并行计算]] — 概念综合页
- [[concepts/C++并发与异步|C++ 并发与异步]] — CPU 并行模型对比
