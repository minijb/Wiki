---
title: Unity Shader基础
updated: 2026-06-02
tags: [shader, unity, rendering, hlsl]
aliases: [Shader基础, ShaderLab基础]
---

# Unity Shader基础

Unity Shader 基础涵盖 ShaderLab 结构、HLSL/CG 类型系统、内置函数库、纹理操作和基本光照计算。

## 核心结构

```
Shader "Name" {
    Properties { ... }
    SubShader { Tags { ... } Pass { ... } }
    FallBack "..."
}
```

- **Properties**：材质面板可见属性，桥接材质和 Shader
- **SubShader**：按平台/显卡适配，多个 SubShader 选第一个可运行的
- **Pass**：单次几何体渲染，可多个 Pass
- **FallBack**：所有 SubShader 不可用时的后备

## 渲染队列

| 队列 | 值 | 用途 |
|:-----|:---|:-----|
| Background | 1000 | 天空盒等 |
| Geometry | 2000 | 不透明物体（默认） |
| AlphaTest | 2450 | `clip` 裁剪透明 |
| Transparent | 3000 | 半透明混合 |
| Overlay | 4000 | 叠加效果 |

## 状态命令

- **`ZTest`**：深度比较函数（Less/LEqual/Greater/Always...）
- **`ZWrite On/Off`**：是否写入深度缓冲
- **`Blend Src Dst`**：颜色混合因子
- **`Cull Back/Front/Off`**：面剔除
- **`ColorMask RGB/A/0`**：输出通道掩码

## 精度类型

- `fixed`：最低精度（11位），适合颜色
- `half`：中等精度（16位），适合 UV/法线
- `float`：最高精度（32位），适合世界空间位置

## 常用内置函数

| 函数 | 用途 |
|:-----|:-----|
| `UnityObjectToClipPos(v)` | 模型空间→裁剪空间 |
| `UnityObjectToWorldNormal(n)` | 法线转世界空间 |
| `TRANSFORM_TEX(uv, tex)` | 应用纹理 Tiling/Offset |
| `tex2D(sampler, uv)` | 纹理采样 |

## 基本光照

- **漫反射**：$C = (C_{light} \cdot m_{diffuse}) \cdot max(0, \hat{n} \cdot \hat{l})$（兰伯特定律）
- **半兰伯特**：用 $0.5(\hat{n} \cdot \hat{l}) + 0.5$ 替代 max，背面不纯黑
- **高光**：$C = (C_{light} \cdot m_{spec}) \cdot max(0, \hat{n} \cdot \hat{h})^{gloss}$（Blinn-Phong）
- 需 `Tags {"LightMode"="ForwardBase"}` 获取光照变量

## 纹理

- `_MainTex_ST`：`.xy` = Tiling（缩放），`.zw` = Offset（偏移）
- `TRANSFORM_TEX` 宏自动应用 UV 变换
- 可通过噪声纹理采样实现 UV 扰动效果

## 相关页面

- [[Shader高级特性]] — 透明、几何着色器与光照系统
- [[渲染管线理论]] — GAMES101 图形学理论
- [[Shader基础与Unity内置结构-摘要]] — 原始来源摘要
