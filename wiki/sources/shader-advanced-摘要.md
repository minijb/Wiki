---
title: Shader高级特性-摘要
updated: 2026-06-02
tags: [shader, unity, rendering, transparency, lighting]
source: raw/gamedev/rendering/shader-advanced.md
---

# Shader高级特性 — 摘要

## 主题

本文档深入 Unity Shader 的高级话题：透明渲染与混合、模板测试、法线贴图与切线空间、GrabPass 玻璃折射、表面着色器与曲面细分、几何着色器、前向/延迟渲染路径以及阴影系统。

## 核心知识点

### 透明渲染
- **Alpha Test**：`clip(texColor.a - _Cutoff)` 丢弃不满足条件的片元，Queue=AlphaTest
- **Alpha Blend**：`Blend SrcAlpha OneMinusSrcAlpha`，Queue=Transparent，ZWrite Off
- **深度写入半透明**：双 Pass（先 ZWrite On/ColorMask 0 写深度，再正常 Blend）
- **双面渲染**：`Cull Front` 渲染背面 + `Cull Back` 渲染正面

### 模板测试
8 位整数/像素的通用遮罩。使用 `Ref`（参考值）、`Comp`（比较函数）、`Pass/Fail`（操作）配置。可用于镜面反射、轮廓描边等效果。

### 法线贴图与 TBN 矩阵
- **切线空间**：高自由度、可 UV 动画、可压缩
- **世界空间**：计算稍复杂但可复用光照方向
- `TANGENT_SPACE_ROTATION` 宏自动构建旋转矩阵
- `UnpackNormal` 处理平台差异的压缩格式
- 法线 z 分量由 xy 导出：$z = \sqrt{1 - (x^2 + y^2)}$

### GrabPass 折射
`ComputeGrabScreenPos` 获取屏幕空间坐标，结合法线贴图偏移实现扭曲折射：
`offset = normal.xy * bumpScale * texelSize.xy`

### 表面着色器与曲面细分
`#pragma surface surf BlinnPhong vertex:disp tessellate:tessFixed`
曲面细分四种方式：固定、边长、距离、Phong。水面效果通过正弦顶点动画 + 双向法线采样实现。

### 几何着色器
位于顶点着色器后、光栅化前。输入图元（point/line/triangle），输出流（PointStream/LineStream/TriangleStream）。`Append` 追加顶点，`RestartStrip` 重置条带。maxvertexcount 应尽量小（1-20 为最佳性能）。

### 前向渲染 vs 延迟渲染
- **Forward**：每光源每物体一个 Pass。Base Pass（一次）+ Add Pass（每额外像素光源一次）。Add Pass 使用 `Blend One One`。
- **Deferred**：G-Buffer Pass + Lighting Pass。不支持抗锯齿和半透明。
- 光照衰减通过 `_LightTexture0` 纹理查找或公式计算。

### 阴影系统
`SHADOW_COORDS(n)` 声明阴影坐标 → `TRANSFER_SHADOW(o)` 传递 → `SHADOW_ATTENUATION(i)` 获取阴影值。需 `#include "AutoLight.cginc"`。

## 相关概念

- [[Shader高级特性]] — 概念页面
- [[Unity Shader基础]] — ShaderLab 基础
- [[渲染管线理论]] — 图形学理论基础
