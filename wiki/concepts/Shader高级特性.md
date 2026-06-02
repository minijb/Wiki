---
title: Shader高级特性
updated: 2026-06-02
tags: [shader, unity, rendering, transparency, geometry-shader]
aliases: [Shader进阶, 透明与几何着色器]
---

# Shader高级特性

Unity Shader 高级话题包括透明渲染、模板测试、法线映射、GrabPass、几何着色器、光照路径和阴影。

## 透明渲染

### 透明度测试（Alpha Test）
使用 `clip(alpha - threshold)` 丢弃片元。完全透明或完全不透明，不需要关闭深度写入。RenderType=TransparentCutout。

### 透明度混合（Alpha Blend）
`Blend SrcAlpha OneMinusSrcAlpha`，需要 `ZWrite Off`。渲染顺序：先不透明 → 再透明（从远到近）。

### 双面透明
两个 Pass：`Cull Front` 渲染背面 → `Cull Back` 渲染正面。

## 模板测试（Stencil）

8 位/像素遮罩。配置项：Ref（参考值）、Comp（Always/Less/Equal...）、Pass/Fail/ZFail（Keep/Replace/Incr...）。

## 法线贴图

- **切线空间计算**：光/视线变换到切线空间
- **世界空间计算**：法线变换到世界空间（用 TtoW 矩阵行存储）
- TBN 矩阵：Tangent × Binormal × Normal
- `UnpackNormal` 处理压缩格式，`_BumpScale` 控制强度

## GrabPass 玻璃折射

```c++
GrabPass { "_RefractionTex" }  // 抓取屏幕到纹理
// 片元着色器中
float2 offset = normal.xy * bumpScale * texelSize.xy;
float2 grabUV = screenPos.xy / screenPos.w + offset;
fixed4 refracted = tex2D(_GrabTexture, grabUV);
```

## 几何着色器

> [!info] 管线位置
> Vertex Shader → Geometry Shader → Rasterization

- `[maxvertexcount(N)]` 限制最大输出顶点数
- 输入：triangle/line/point + 邻接类型
- 输出：TriangleStream/LineStream/PointStream
- `Append` 追加顶点，`RestartStrip` 开始新条带

## 前向渲染路径

| | ForwardBase | ForwardAdd |
|:--|:------------|:-----------|
| 执行次数 | 1 | 每光源 1 |
| 光照 | 环境+自发光+主平行光 | 其他逐像素光 |
| 混合 | 无 | `Blend One One` |

## 阴影

核心宏：`SHADOW_COORDS(n)` → `TRANSFER_SHADOW(o)` → `SHADOW_ATTENUATION(i)`。

需 `#include "AutoLight.cginc"` 和 `#pragma multi_compile_fwdbase`。

## 曲面细分

`#pragma surface surf ... tessellate:tessFixed`
四种方式：固定数量、基于边长、基于距离、Phong 细分。

## 相关页面

- [[Unity Shader基础]] — ShaderLab基础与基本光照
- [[渲染管线理论]] — 图形学理论
- [[Shader高级特性-摘要]] — 原始来源
