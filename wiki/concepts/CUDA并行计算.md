---
title: "CUDA 并行计算"
type: concept
updated: 2026-06-02
tags: [cuda, gpu, parallel-computing, nvidia]
aliases: [CUDA, GPU编程, 并行计算]
---

# CUDA 并行计算

CUDA (Compute Unified Device Architecture) 是 NVIDIA 的并行计算平台，使开发者能够利用 GPU 进行通用计算。GPU 通过 PCI 总线与 CPU 连接——CPU 称为主机端 (host)，GPU 称为设备端 (device)。

## 硬件架构

GPU 的核心是 **SM（Streaming Multiprocessor，流式多处理器）**，包含 CUDA 核心、共享内存和寄存器。SM 可并发执行数百个线程，并发能力取决于 SM 资源数。

### 线程层次

| 层次 | 说明 | 通信方式 |
|------|------|----------|
| Grid | 一次 kernel 启动的所有线程 | 全局内存 |
| Block | Grid 的子划分 | 共享内存 + `__syncthreads()` |
| Thread | 最小执行单元 | 寄存器 + 局部内存 |

### SIMT 与 Warp

SM 采用 **SIMT（单指令多线程）** 架构，基本执行单元是 **Warp（线程束，32 线程）**。Warp 中所有线程同时执行相同指令，但各自有独立的寄存器状态。遇到分支时可能出现**线程束分化**导致性能下降。

> [!important] Block 大小应为 32 的倍数
> 因为 Warp 是 32 线程，block 大小设为 32 的倍数可最大化 SM 利用率。

## 内存模型

GPU 内存从快到慢：

- **寄存器**：线程私有，最快
- **共享内存 (Shared Memory)**：Block 内共享，接近寄存器速度，生命周期与 block 一致
- **全局内存 (Global Memory)**：所有线程可访问，最慢但容量最大
- **常量内存 / 纹理内存**：只读，有缓存

> [!tip] 优化关键
> 合并访问、利用共享内存缓存全局数据、避免 bank conflict、减少 Host-Device 数据传输。

## 编程模型

使用 `__global__` 标记 kernel 函数，`<<<grid, block>>>` 语法启动 kernel。内存通过 `cudaMalloc` / `cudaMemcpy` / `cudaFree` 管理。

## 关联页面

- [[sources/cuda-basics-摘要|CUDA 基础来源摘要]]
- [[concepts/C++并发与异步|C++ 并发与异步]] — CPU 并行模型对比
- [[concepts/CSAPP系统级主题|CSAPP 系统级主题]] — 计算机体系结构基础
