---
title: Shader基础与Unity内置结构-摘要
updated: 2026-06-02
tags: [shader, unity, rendering]
source: raw/gamedev/rendering/shader-basics.md
---

# Shader基础与Unity内置结构 — 摘要

## 主题

本文档系统梳理了 Unity Shader 的基础知识体系，从 ShaderLab 结构到 HLSL/CG 类型系统、内置函数库，覆盖纹理系统和基本光照计算。

## 核心知识点

### ShaderLab 层次结构
Unity Shader 由 `Shader > Properties > SubShader > Pass` 组成。Properties 连接材质和着色器，SubShader 根据平台适配选择，Pass 执行实际渲染。关键标签包括 Queue（渲染队列 1000-4000）、RenderType（Opaque/Transparent/Cutout 等）和 LOD（细节层次 100-600）。

### Pass 状态控制
深度测试（ZTest）、深度写入（ZWrite）、混合（Blend）和剔除（Cull）是 Pass 层的核心状态。Z-Fighting 通过 Offset 解决。常见混合模式包括传统透明度（`SrcAlpha OneMinusSrcAlpha`）、加法（`One One`）、乘法（`DstColor Zero`）。

### HLSL/CG 类型系统
三种精度：`float`（32位）、`half`（16位）、`fixed`（11位）。语义（POSITION、NORMAL、SV_POSITION、SV_TARGET 等）让 GPU 知道数据流向。a2v/v2f 是标准命名约定。

### 内置函数库
`UnityCG.cginc` 提供核心工具函数（`UnityObjectToClipPos`、`UnityObjectToWorldNormal` 等）和预定义结构体（`appdata_base`、`appdata_full` 等）。`Lighting.cginc` 内置光照模型。

### 纹理系统
`_MainTex_ST` 存储缩放/偏移，`TRANSFORM_TEX` 宏应用 UV 变换。UV 扰动通过噪声纹理采样实现动态扭曲效果。

### 基本光照
漫反射（Lambert 定律）、半兰伯特模型、高光反射（Blinn-Phong 半程向量法）。必须设置 `ForwardBase` 才能获取 Unity 光照变量。纹理颜色（albedo）参与环境和漫反射，高光通常不乘纹理色。

### 优化基础
Material Property Block 避免生成重复材质以保持合批；Constant Buffer 将不频繁变化的数据存于 GPU；精度选择（能低则低）、减少采样和分支。

## 相关概念

- [[Unity Shader基础]] — 概念页面
- [[Shader高级特性]] — 透明、几何着色器、光照系统
- [[渲染管线理论]] — GAMES101 图形学理论
