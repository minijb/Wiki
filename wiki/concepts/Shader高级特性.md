---
title: Shader高级特性
updated: 2026-06-02
tags: [shader, unity, rendering, transparency, geometry-shader]
aliases: [Shader进阶, 透明与几何着色器]
---

# Shader高级特性

Unity Shader 高级话题包括透明渲染、模板测试、法线映射、GrabPass、几何着色器、表面着色器与曲面细分、前向/延迟渲染路径、阴影系统和现代渲染技术。

## 透明渲染

### 透明度测试（Alpha Test）
使用 `clip(alpha - threshold)` 丢弃片元。完全透明或完全不透明，不需要关闭深度写入。Queue=AlphaTest，RenderType=TransparentCutout。

### 透明度混合（Alpha Blend）
`Blend SrcAlpha OneMinusSrcAlpha`，需要 `ZWrite Off`。Queue=Transparent。渲染顺序：先不透明 → 再透明（从远到近）。

### 深度写入半透明
双 Pass 方案：Pass 0（ZWrite On + ColorMask 0 写深度）+ Pass 1（正常 Blend）。解决自身遮挡问题。

### 双面透明
两个 Pass：`Cull Front` 渲染背面 → `Cull Back` 渲染正面。

### 常见混合类型
`Blend OneMinusDstColor One`（柔和相加/滤色）、`Blend DstColor Zero`（正片叠底）、`BlendOp Min/Max` + `Blend One One`（变暗/变亮）、`Blend One One`（线性减淡）。

## 模板测试（Stencil）

8 位/像素遮罩。配置项：Ref（参考值）、Comp（Always/Less/Equal...）、Pass/Fail/ZFail（Keep/Replace/Incr...）。可用于镜面反射、轮廓描边等效果。模板测试在深度测试之前。

## 法线贴图

- **切线空间计算**：光/视线变换到切线空间，高自由度 + 可 UV 动画 + 可压缩
- **世界空间计算**：法线变换到世界空间（用 TtoW 矩阵行存储），计算稍复杂但可复用光照方向
- TBN 矩阵：Tangent × Binormal × Normal。副切线方向由 `v.tangent.w`（+1 或 -1）确定
- `UnpackNormal` 处理平台差异的压缩格式，`_BumpScale` 控制强度
- 法线 z 分量由 xy 导出：$z = \sqrt{1 - (x^2 + y^2)}$

## GrabPass 玻璃折射

```c++
GrabPass { "_RefractionTex" }  // 抓取屏幕到纹理
// 片元着色器中
float2 offset = normal.xy * _BumpScale * _RefractionTex_TexelSize.xy;
float2 grabUV = screenPos.xy / screenPos.w + offset;  // 透视除法
fixed4 refracted = tex2D(_GrabTexture, grabUV);
```

`ComputeGrabScreenPos` 获取屏幕空间坐标。`_RefractionTex_TexelSize` 为纹素大小。

## 表面着色器与曲面细分

表面着色器是 Unity 对顶点/片元着色器的更高层抽象。`#pragma surface surf BlinnPhong vertex:disp tessellate:tessFixed`

四种细分方式：固定数量、基于边长、基于距离、Phong 细分。水面效果通过正弦顶点动画 + 法线贴图双向采样（正反方向）实现动态感。

## 几何着色器

> [!info] 管线位置
> Vertex Shader → Geometry Shader → Rasterization

- `[maxvertexcount(N)]` 限制最大输出顶点数，应尽可能小（1-20 最佳）
- 输入图元：triangle/line/point + 邻接类型（lineadj/triangleadj）
- 输出流：TriangleStream/LineStream/PointStream
- `Append` 追加顶点，`RestartStrip` 开始新条带
- 应用：三角形→点（质心）、三角形→线框、草叶生成

## 前向渲染路径

| | ForwardBase | ForwardAdd |
|:--|:------------|:-----------|
| 执行次数 | 1 | 每额外逐像素光源 1 |
| 光照 | 环境+自发光+主平行光 | 其他逐像素光 |
| 混合 | 无 | `Blend One One` |
| 编译指令 | `multi_compile_fwdbase` | `multi_compile_fwdadd` |

Addition Pass 不计算环境光和自发光，需区分光源类型计算衰减。

## 光照衰减

- 平行光：无衰减
- 点光源/聚光灯：通过 `_LightTexture0` 纹理查找（将世界坐标转到光源空间，计算距离平方查找）
- 公式法：`atten = 1.0 / distance`
- `UNITY_ATTEN_CHANNEL` 获取衰减所在分量

## 延迟渲染

两个 Pass：G-Buffer Pass（写入 MRT：漫反射色、高光色+指数、世界法线、自发光+探针）+ Lighting Pass（利用 G-Buffer 逐光源计算）。

缺点：不支持真正的抗锯齿、不能处理半透明物体、需 MRT+SM3.0。

## 阴影

核心宏：`SHADOW_COORDS(n)` → `TRANSFER_SHADOW(o)` → `SHADOW_ATTENUATION(i)`。

需 `#include "AutoLight.cginc"` 和 `#pragma multi_compile_fwdbase`。Shadow Caster Pass 通过 `V2F_SHADOW_CASTER` 和 `TRANSFER_SHADOW_CASTER_NORMALOFFSET` 实现。

光源设置：Shadow Type（No/Hard/Soft）、Strength、Resolution、Bias/Normal Bias（避免 Shadow Acne）。

## 现代渲染技术

- **Mesh Shader**（DX12 Ultimate / NVIDIA Turing+）：替代传统 Vertex→Geometry 管线，Amplification Shader（可选）+ Mesh Shader 直接在 GPU 端生成几何体
- **Render Graph**（Unity 6 / SRP Core 17）：自动资源生命周期管理、显式渲染依赖图、更好的 GPU 内存管理
- **DLSS 5 / Neural Rendering**：全神经网络渲染，RTX Neural Shaders 替代复杂 BRDF 计算

## 相关页面

- [[Unity Shader基础]] — ShaderLab基础与基本光照
- [[渲染管线理论]] — GPU 管线与图形学理论
- [[光线追踪入门]] — 材质系统（Lambertian/Metal/Dielectric）
- [[OpenGL学习笔记]] — OpenGL GLSL 对比
- [[Unity性能优化]] — 渲染优化
- [[shader-advanced-摘要]] — 原始来源摘要
