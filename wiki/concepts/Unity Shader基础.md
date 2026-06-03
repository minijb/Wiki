---
title: Unity Shader基础
updated: 2026-06-02
tags: [shader, unity, rendering, hlsl]
aliases: [Shader基础, ShaderLab基础]
---

# Unity Shader基础

Unity Shader 基础涵盖 ShaderLab 结构、HLSL/CG 类型系统、内置函数库、纹理操作、基本光照计算和批处理机制。

## 核心结构

```
Shader "Name" {
    Properties { ... }
    SubShader { Tags { ... } Pass { ... } }
    FallBack "..."
}
```

- **Properties**：材质面板可见属性，桥接材质和 Shader。类型映射：Color/Vector → `float4`，Range/Float → `float`，2D → `sampler2D`
- **SubShader**：按平台/显卡适配，多个 SubShader 选第一个可运行的
- **Pass**：单次几何体渲染，可多个 Pass。特殊 Pass：`UsePass`（复用）、`GrabPass`（抓取屏幕）
- **FallBack**：所有 SubShader 不可用时的后备

## 渲染队列

| 队列 | 值 | 用途 |
|:-----|:---|:-----|
| Background | 1000 | 天空盒等 |
| Geometry | 2000 | 不透明物体（默认） |
| AlphaTest | 2450 | `clip` 裁剪透明 |
| Transparent | 3000 | 半透明混合 |
| Overlay | 4000 | 叠加效果 |

LOD 控制着色器细节层次（VertexLit=100 → Parallax Specular=600）。

## 状态命令

- **`ZTest`**：深度比较函数（Less/LEqual/Greater/Always...）
- **`ZWrite On/Off`**：是否写入深度缓冲
- **`Blend Src Dst`**：颜色混合因子。常见：`SrcAlpha OneMinusSrcAlpha`（透明度）、`One One`（加法）、`DstColor Zero`（正片叠底）
- **`Cull Back/Front/Off`**：面剔除
- **`ColorMask RGB/A/0`**：输出通道掩码
- **`Offset Factor, Units`**：Z 缓冲区偏移（解决 Z-Fighting）

## 精度类型

- `fixed`：最低精度（11位），适合颜色
- `half`：中等精度（16位），适合 UV/法线
- `float`：最高精度（32位），适合世界空间位置

纹理采样器支持精度变体：`sampler2D_half`（移动平台）、`sampler2D_float`（HDR）。

## 语义与命名规范

- `a2v`（application to vertex）：`POSITION`、`NORMAL`、`TANGENT`、`TEXCOORD0~n`、`COLOR`
- `v2f`（vertex to fragment）：`SV_POSITION`、`SV_TARGET`、`COLOR0`/`COLOR1`、`TEXCOORD0~7`
- `SV_` 前缀为 System Value，在管线中有特殊含义

## 常用内置函数

| 函数 | 用途 |
|:-----|:-----|
| `UnityObjectToClipPos(v)` | 模型空间→裁剪空间 |
| `UnityObjectToWorldNormal(n)` | 法线转世界空间 |
| `UnityWorldSpaceViewDir(pos)` | 世界空间观察方向 |
| `UnityWorldSpaceLightDir(pos)` | 世界空间光照方向 |
| `TRANSFORM_TEX(uv, tex)` | 应用纹理 Tiling/Offset |
| `tex2D(sampler, uv)` | 纹理采样 |

内置结构体：`appdata_base`、`appdata_tan`、`appdata_full`、`v2f_img`。

## 基本光照

- **漫反射**：$C = (C_{light} \cdot m_{diffuse}) \cdot max(0, \hat{n} \cdot \hat{l})$（兰伯特定律）
- **半兰伯特**：用 $0.5(\hat{n} \cdot \hat{l}) + 0.5$ 替代 max，背面不纯黑
- **高光**：$C = (C_{light} \cdot m_{spec}) \cdot max(0, \hat{n} \cdot \hat{h})^{gloss}$（Blinn-Phong，光泽度 8-256）
- 需 `Tags {"LightMode"="ForwardBase"}` 获取光照变量。平行光方向通过 `_WorldSpaceLightPos0.xyz` 获取
- 纹理颜色参与环境光和漫反射，高光通常不乘纹理色

## 纹理

- `_MainTex_ST`：`.xy` = Tiling（缩放），`.zw` = Offset（偏移）
- `TRANSFORM_TEX` 宏自动应用 UV 变换
- 可通过噪声纹理采样实现 UV 扰动和描边燃烧效果

## 批处理机制

| 机制 | 说明 |
|:-----|:-----|
| Static Batching | 静态物体合并大网格，减少 buffer binding |
| Dynamic Batching | 低端设备，小网格动态合并 |
| SRP Batcher | URP 默认，减少 DrawCall 间 CPU 开销 |
| GPU Instancing | 同 Mesh+Material 批量渲染 |
| Material Property Block | 不生成新材质实例，保持合批能力 |

## 四元数旋转

`Quaternion.LookRotation`、`Quaternion.Slerp`、`Quaternion.Euler` 等避免欧拉角万向锁。欧拉角按顺规 XYZ 应用。

## 相关页面

- [[Shader高级特性]] — 透明、几何着色器与光照系统
- [[渲染管线理论]] — GAMES101 图形学理论
- [[OpenGL学习笔记]] — OpenGL/GLSL 对比
- [[Unity性能优化]] — 批处理与渲染优化
- [[shader-basics-摘要]] — 原始来源摘要
