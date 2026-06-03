---
title: "CUDA 并行计算基础"
type: source
updated: 2026-06-02
tags: [cuda, gpu, parallel-computing, nvidia, simt, memory-model]
aliases: [CUDA基础, CUDA编程, GPU编程]
description: CUDA 并行计算平台全文：GPU 硬件架构（SM/线程束/内存层次）、编程模型（grid/block/thread）、内存管理（cudaMalloc/cudaMemcpy）、环境配置与开发工具
---

# CUDA 并行计算基础

> [!note] 核心理念
> CUDA (Compute Unified Device Architecture) 是 NVIDIA 的并行计算平台与编程模型。GPU 通过 **PCI 总线** 与 CPU 连接——CPU 称为**主机端 (host)**，GPU 称为**设备端 (device)**。程序在 host 上发起 kernel 调用，在 device 上并行执行成千上万个线程。

## 1. CUDA 硬件架构

### 1.1 核心概念：SM (Streaming Multiprocessor)

GPU 硬件的核心组件是 **SM（流式多处理器）**。SM 包含 CUDA 核心、共享内存、寄存器等，可以并发执行数百个线程——并发能力取决于 SM 拥有的资源数。

```c
int dev = 0;
cudaDeviceProp devProp;
cudaGetDeviceProperties(&devProp, dev);
std::cout << "使用GPU device " << dev << ": " << devProp.name << std::endl;
std::cout << "SM的数量：" << devProp.multiProcessorCount << std::endl;
std::cout << "每个线程块的共享内存大小：" << devProp.sharedMemPerBlock / 1024.0 << " KB" << std::endl;
std::cout << "每个线程块的最大线程数：" << devProp.maxThreadsPerBlock << std::endl;
std::cout << "每个SM的最大线程数：" << devProp.maxThreadsPerMultiProcessor << std::endl;
std::cout << "每个SM的最大线程束数：" << devProp.maxThreadsPerMultiProcessor / 32 << std::endl;
```

### 1.2 线程层次：Grid → Block → Thread

| 层次 | 说明 | 通信方式 |
|------|------|----------|
| **Grid（网格）** | 一次 kernel 启动的所有线程集合 | 全局内存 |
| **Block（线程块）** | Grid 的子划分，包含若干线程 | 共享内存 + `__syncthreads()` |
| **Thread（线程）** | 最小执行单元 | 局部内存、寄存器 |

- 一个 kernel 对应一个 grid
- grid 分为多个 block
- block 包含多个线程
- **block 大小一般设置为 32 的倍数**（原因见下文线程束）

### 1.3 SIMT 与线程束 (Warp)

SM 采用 **SIMT (Single-Instruction, Multiple-Thread，单指令多线程)** 架构，基本执行单元是**线程束 (warp)**——包含 **32 个线程**。

> [!important] Warp 的关键特性
> - Warp 中所有线程同时执行相同的指令
> - 每个线程有独立的指令地址计数器和寄存器状态
> - 遇到分支结构时，部分线程可能进入分支而另一些等待 → **线程束分化 (warp divergence)** 导致性能下降

当 block 被分配到某个 SM 上时，进一步划分为多个 warp。但一个 SM 同时并发的 warp 数是有限的——受共享内存和寄存器资源限制。

**物理执行 vs 逻辑划分**：grid 和 block 只是逻辑划分，一个 kernel 的所有线程在物理层不一定同时并发。grid/block 配置不同，性能会有显著差异。

### 1.4 调度机制

- 一个线程只能在一个 SM 中调度
- 一个 SM 一般可以调度多个 block
- 一个 kernel 会被分配到多个 SM 上执行
- SM 要为每个 block 分配共享内存，为每个 warp 中的线程分配独立寄存器

→ **SM 配置直接影响支持的 block/warp 并发数量**。

---

## 2. CUDA 内存模型

```
┌─────────────────────────────────────────────────────┐
│                    Global Memory                      │
│                 (所有线程可访问，最慢)                   │
│  ┌──────────────────────┐  ┌──────────────────────┐  │
│  │      Block 0          │  │      Block 1          │  │
│  │  ┌──────────────────┐│  │  ┌──────────────────┐│  │
│  │  │   Shared Memory   ││  │  │   Shared Memory   ││  │
│  │  │ (block内线程共享)  ││  │  │ (block内线程共享)  ││  │
│  │  └──────────────────┘│  │  └──────────────────┘│  │
│  │  Thread0 Thread1 ... │  │  Thread0 Thread1 ... │  │
│  │  [Reg]  [Reg]        │  │  [Reg]  [Reg]        │  │
│  │  [Local] [Local]     │  │  [Local] [Local]     │  │
│  └──────────────────────┘  └──────────────────────┘  │
└─────────────────────────────────────────────────────┘
         只读内存：Constant Memory / Texture Memory
```

### 2.1 各类内存对比

| 内存类型 | 位置 | 访问范围 | 生命周期 | 速度 | 用途 |
|----------|------|----------|----------|------|------|
| **Register（寄存器）** | SM 片上 | 单线程 | 线程 | 最快 | 局部变量 |
| **Local Memory** | 显存 | 单线程 | 线程 | 慢 | 寄存器溢出/大数组 |
| **Shared Memory** | SM 片上 | Block 内所有线程 | Block | 接近寄存器 | 线程间数据共享 |
| **Global Memory** | 显存 | 所有线程 | 整个应用 | 最慢 (~400-800 cycles) | Host-Device 数据交换 |
| **Constant Memory** | 显存（缓存） | 所有线程（只读） | 整个应用 | 缓存命中时快 | 常量参数 |
| **Texture Memory** | 显存（缓存） | 所有线程（只读） | 整个应用 | 缓存命中时快 | 图像/2D 空间局部性 |

> [!warning] Shared Memory 生命周期
> Shared Memory 的生命周期与线程块一致——block 执行结束后释放。

### 2.2 内存访问优化原则

1. **合并访问 (Coalesced Access)**：相邻线程访问相邻的内存地址，减少内存事务数量
2. **使用 Shared Memory**：将频繁访问的 global memory 数据缓存到 shared memory
3. **避免 Bank Conflict**：Shared memory 分为 32 个 bank，同一 warp 中多个线程访问同一 bank 的不同地址会串行化
4. **减少 Host-Device 传输**：PCI 总线带宽远低于显存带宽，尽量在 GPU 上完成计算

---

## 3. CUDA 编程基础

### 3.1 基本内存管理

```c
// Device 上分配内存
cudaError_t cudaMalloc(void** devPtr, size_t size);

// 释放 Device 内存
cudaError_t cudaFree(void* devPtr);

// Host ↔ Device 数据传输
cudaError_t cudaMemcpy(void* dst, const void* src, size_t count, cudaMemcpyKind kind);
```

`cudaMemcpyKind` 控制传输方向：

| 枚举值 | 方向 |
|--------|------|
| `cudaMemcpyHostToHost` | Host → Host |
| `cudaMemcpyHostToDevice` | Host → Device |
| `cudaMemcpyDeviceToHost` | Device → Host |
| `cudaMemcpyDeviceToDevice` | Device → Device |

### 3.2 Kernel 启动

Kernel 使用 `<<<gridDim, blockDim>>>` 语法启动：

```c
// Kernel 定义
__global__ void vectorAdd(float* A, float* B, float* C, int N) {
    int i = blockIdx.x * blockDim.x + threadIdx.x;
    if (i < N) {
        C[i] = A[i] + B[i];
    }
}

// Kernel 启动
int blockSize = 256;
int gridSize = (N + blockSize - 1) / blockSize;
vectorAdd<<<gridSize, blockSize>>>(d_A, d_B, d_C, N);
```

### 3.3 函数限定符

| 限定符 | 调用位置 | 执行位置 |
|--------|----------|----------|
| `__global__` | Host | Device（kernel 函数） |
| `__device__` | Device | Device（辅助函数） |
| `__host__` | Host | Host（可省略，默认） |

### 3.4 完整示例：向量加法

```c
#include <stdio.h>
#include <cuda_runtime.h>

__global__ void vectorAdd(float* A, float* B, float* C, int N) {
    int idx = blockIdx.x * blockDim.x + threadIdx.x;
    if (idx < N) {
        C[idx] = A[idx] + B[idx];
    }
}

int main() {
    int N = 1 << 20;  // 1M 元素
    size_t size = N * sizeof(float);

    // 1. Host 分配内存
    float *h_A = (float*)malloc(size);
    float *h_B = (float*)malloc(size);
    float *h_C = (float*)malloc(size);

    // 初始化数据
    for (int i = 0; i < N; i++) {
        h_A[i] = 1.0f;
        h_B[i] = 2.0f;
    }

    // 2. Device 分配内存
    float *d_A, *d_B, *d_C;
    cudaMalloc(&d_A, size);
    cudaMalloc(&d_B, size);
    cudaMalloc(&d_C, size);

    // 3. Host → Device
    cudaMemcpy(d_A, h_A, size, cudaMemcpyHostToDevice);
    cudaMemcpy(d_B, h_B, size, cudaMemcpyHostToDevice);

    // 4. 启动 Kernel
    int blockSize = 256;
    int gridSize = (N + blockSize - 1) / blockSize;
    vectorAdd<<<gridSize, blockSize>>>(d_A, d_B, d_C, N);

    // 5. Device → Host
    cudaMemcpy(h_C, d_C, size, cudaMemcpyDeviceToHost);

    // 6. 验证
    float maxError = 0.0f;
    for (int i = 0; i < N; i++)
        maxError = fmax(maxError, fabs(h_C[i] - 3.0f));
    printf("Max error: %f\n", maxError);

    // 7. 清理
    cudaFree(d_A); cudaFree(d_B); cudaFree(d_C);
    free(h_A); free(h_B); free(h_C);

    return 0;
}
```

---

## 4. 环境配置

### 4.1 前置要求

- NVIDIA GPU（支持 CUDA）
- CUDA Toolkit 安装
- C++ 编译器（Linux: `build-essential`；Windows: Visual Studio）

### 4.2 VS Code 插件

| 插件 | 用途 |
|------|------|
| **vscode-cudacpp** | CUDA C++ 语法高亮与补全 |
| **Nsight Visual Studio Code Edition** | NVIDIA 官方调试与性能分析 |

### 4.3 编译与运行

```bash
# 使用 nvcc 编译
nvcc -o vector_add vector_add.cu

# 运行
./vector_add

# 查看 GPU 信息
nvidia-smi
```

### 4.4 常用 API 速查

| API | 用途 |
|-----|------|
| `cudaGetDeviceCount(&count)` | 获取可用 GPU 数量 |
| `cudaSetDevice(0)` | 选择 GPU |
| `cudaGetDeviceProperties(&prop, 0)` | 获取 GPU 属性 |
| `cudaDeviceSynchronize()` | 等待所有 GPU 操作完成 |
| `cudaGetLastError()` | 获取最近一次错误 |
| `cudaMallocManaged(&ptr, size)` | 统一内存（自动迁移） |

---

## 5. 参考资源

| 资源 | 链接 |
|------|------|
| 谭升的博客 — GPU 编程系列 | https://face2ai.com/program-blog/#GPU编程（CUDA） |
| 知乎 CUDA 快速教程 | https://zhuanlan.zhihu.com/p/34587739 |
| 知乎 CUDA 资源列表 | https://zhuanlan.zhihu.com/p/346910129 |
| NVIDIA CUDA 编程指南 | https://docs.nvidia.com/cuda/cuda-c-programming-guide/ |
| NVIDIA 技术博客 — CUDA Refresher | https://developer.nvidia.com/blog/cuda-refresher-cuda-programming-model/ |
| B站 CUDA 入门视频 | https://www.bilibili.com/video/BV1vJ411D73S/ |

## 参见

- [[concepts/C++并发与异步|C++ 并发与异步]] — CPU 并行模型对比
- [[concepts/CSAPP系统级主题|CSAPP 系统级主题]] — 计算机体系结构基础
