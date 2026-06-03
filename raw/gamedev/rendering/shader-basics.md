---
title: Shader基础与Unity内置结构
type: source
updated: 2026-06-02
tags:
  - shader
  - unity
  - rendering
  - hlsl
  - cg
  - shaderlab
---

# Shader基础与Unity内置结构

> [!note] 来源
> 综合来自 `shader_入门精要--1_shader_基础`、`Shader1 -- 基础`、`unity - shader` 等原子笔记，以及 Unity 官方文档。

## 一、Shader基础结构

Unity Shader 由以下层次组成：`Shader` > `Properties` > `SubShader` > `Pass` > `FallBack`。

```mermaid
flowchart TD
    A[Shader \"Custom/Name\"] --> B[Properties 属性块]
    A --> C[SubShader 列表]
    A --> D[FallBack 后备]
    C --> E[SubShader 0]
    C --> F[SubShader 1]
    C --> G["..."]
    E --> H["Tags + RenderSetup"]
    E --> I[Pass 0]
    E --> J[Pass 1]
    E --> K["..."]
```

### 1.1 Properties（属性）

连接材质和Shader的桥梁，结构为 `Name ("displayName", PropertyType) = DefaultValue`。

```hlsl
Properties
{
    _Color ("Color Tint", Color) = (1.0, 1.0, 1.0, 1.0)
    _MainTex ("Main Texture", 2D) = "white" {}
    _Gloss ("Gloss", Range(8.0, 256)) = 20
}
```

| ShaderLab属性类型 | CG/HLSL变量类型 |
|:-----------------|:----------------|
| Color, Vector    | float4, half4, fixed4 |
| Range, Float     | float, half, fixed    |
| 2D               | sampler2D             |
| Cube             | samplerCube           |
| 3D               | sampler3D             |

属性通过 **name** 引用（以下划线开头），在材质检视面板显示 **display name**。默认值规则：

- `Range` / `Float`：单个数字
- `Color` / `Vector`：四个数字 `(r,g,b,a)`
- 2D纹理默认值：空字符串或内置纹理 `"white"`, `"black"`, `"gray"`, `"bump"`, `"red"`
- 非2D纹理（立方体、3D、2D数组）：空字符串

### 1.2 SubShader

```hlsl
SubShader
{
    // 可选的标签和渲染状态
    [Tags]
    [RenderSetup]

    Pass {
        // ...
    }
}
```

Unity 选择**第一个能在当前硬件上运行的 SubShader**。

**常见 Tags：**

| Tag | 说明 |
|:----|:-----|
| `Queue` | 渲染队列（Background=1000, Geometry=2000, AlphaTest=2450, Transparent=3000, Overlay=4000） |
| `RenderType` | 着色器分类（Opaque, Transparent, TransparentCutout 等），用于着色器替换 |
| `DisableBatching` | 禁用批处理 |
| `ForceNoShadowCasting` | 强制不投射阴影 |
| `IgnoreProjector` | 常用于半透明物体，忽略投影器影响 |

**RenderType 常用值：**

| 类型 | 描述 |
|:-----|:-----|
| Opaque | 不透明着色器（法线、自发光、反射、地形） |
| Transparent | 半透明着色器（透明、粒子、字体） |
| TransparentCutout | 透明度测试（植被双通道） |
| Background | 天空盒 |
| Overlay | 光晕/闪光 |
| TreeOpaque | 地形引擎树皮 |
| TreeTransparentCutout | 地形引擎树叶 |
| TreeBillboard | 地形引擎广告牌树 |
| Grass | 地形引擎草 |
| GrassBillboard | 地形引擎广告牌草 |

**LOD (Level of Detail)：**

控制着色器细节层次。当 LOD 值小于设定值时，Shader 不会工作。

Unity 内建着色器 LOD 设置：
- VertexLit = 100
- Diffuse = 200
- Bumped, Specular = 300
- Bumped Specular = 400
- Parallax = 500
- Parallax Specular = 600

可通过 `Shader.maximumLOD` 和 `Shader.globalMaximumLOD` 控制。

### 1.3 Pass

Pass 代码块使几何体被渲染一次。一个 SubShader 可以有多个 Pass。

```hlsl
Pass {
    [Name]
    [Tags]
    [RenderSetup]
    // shader code
}
```

**特殊 Pass：**
- `UsePass "ShaderName/PassName"` — 复用其他 Shader 的 Pass
- `GrabPass { }` 或 `GrabPass { "TextureName" }` — 抓取当前屏幕内容到纹理

**Pass 状态命令：**

| 命令 | 说明 |
|:-----|:-----|
| `Cull Back\|Front\|Off` | 面剔除 |
| `ZTest Less\|Greater\|LEqual\|GEqual\|Equal\|NotEqual\|Always` | 深度测试 |
| `ZWrite On\|Off` | 深度写入 |
| `Blend SrcFactor DstFactor` | 颜色混合 |
| `ColorMask RGB\|A\|0\|任意组合` | 颜色通道写入掩码 |
| `Offset Factor, Units` | Z 缓冲区深度偏移（解决 Z-Fighting） |

**深度测试四种情况：**

| 测试 | 写入 | 效果 |
|:-----|:-----|:-----|
| 通过 | 开启 | 写深度 + 写颜色 |
| 通过 | 关闭 | 不写深度 + 写颜色 |
| 失败 | 开启 | 不写深度 + 不写颜色 |
| 失败 | 关闭 | 不写深度 + 不写颜色 |

`ZTest Off` = `ZTest Always`。**Z-Fighting**：两个多边形 Z 值极其接近时交替闪烁，用 `Offset` 解决。

**Blend 混合因子：**

| 因子 | 含义 |
|:-----|:-----|
| One | 1 |
| Zero | 0 |
| SrcColor | 源颜色 |
| SrcAlpha | 源 Alpha |
| DstColor | 目标颜色 |
| DstAlpha | 目标 Alpha |
| OneMinusSrcColor | 1 - 源颜色 |
| OneMinusSrcAlpha | 1 - 源 Alpha |
| OneMinusDstColor | 1 - 目标颜色 |
| OneMinusDstAlpha | 1 - 目标 Alpha |

常见混合类型：
- `Blend SrcAlpha OneMinusSrcAlpha` — 传统透明度
- `Blend One OneMinusSrcAlpha` — 预乘透明度
- `Blend One One` — 加法 / 线性减淡
- `Blend OneMinusDstColor One` — 软加法 / 滤色
- `Blend DstColor Zero` — 正片叠底
- `Blend DstColor SrcColor` — 2x 乘法
- `BlendOp Min / Max` + `Blend One One` — 变暗 / 变亮

### 1.4 FallBack

所有 SubShader 均不可用时的后备着色器。如 `FallBack "Diffuse"`。也可设为 `FallBack Off`。

---

## 二、HLSL/CG 基础类型与语义

### 2.1 精度类型

| 类型 | 精度 | 说明 |
|:-----|:-----|:-----|
| `float` | 最高精度，32位 | 世界空间位置、复杂计算 |
| `half` | 中等精度，16位，范围 -60000 ~ +60000 | UV 坐标、法线方向 |
| `fixed` | 最低精度，11位，范围 -2.0 ~ +2.0 | 颜色值 |

复合类型：`float3`, `float4`, `float4x4`（方形矩阵）。某些平台仅支持方形矩阵。

### 2.2 纹理采样器

```hlsl
sampler2D _MainTex;        // 2D纹理
samplerCUBE _Cubemap;      // 立方体贴图
sampler2D_half _MainTex;   // 低精度采样器（移动平台）
sampler2D_float _MainTex;  // 完整浮点精度（HDR）
samplerCUBE_float _Cubemap;
```

### 2.3 语义（Semantics）

语义是赋给 Shader 输入输出的字符串，告诉 GPU 从哪里读取数据、输出到哪里。`SV_` 前缀的语义（System Value）在渲染管线中有特殊含义。

**应用到顶点着色器（a2v 输入语义）：**

| 语义 | 意义 |
|:-----|:-----|
| `POSITION` | 顶点位置（模型空间） |
| `NORMAL` | 法线向量（模型空间） |
| `TANGENT` | 切线向量（模型空间，float4，w 存副切线方向） |
| `TEXCOORD0` ~ `TEXCOORDn` | 纹理坐标（第 n 套 UV） |
| `COLOR` | 顶点颜色 |

**顶点到片元着色器（v2f 传递语义）：**

| 语义 | 意义 |
|:-----|:-----|
| `SV_POSITION` | 裁剪空间位置（System Value） |
| `SV_TARGET` | 片元着色器输出到帧缓冲的颜色 |
| `COLOR0`, `COLOR1` | 颜色传递 |
| `TEXCOORD0` ~ `TEXCOORD7` | 纹理坐标 / 自定义数据传递 |

**命名规范：**
- `a2v`：application to vertex（顶点着色器输入结构体）
- `v2f`：vertex to fragment（顶点着色器输出 / 片元着色器输入结构体）

### 2.4 完整应用示例

```hlsl
Shader "Book/C5/Simple Shader"
{
    Properties { }
    SubShader
    {
        Pass {
            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag

            struct a2v {
                float4 ver : POSITION;    // 模型空间坐标
                float3 normal : NORMAL;   // 法线
                float4 texcoord : TEXCOORD0; // 纹理坐标
            };

            struct v2f {
                float4 pos : SV_POSITION;
                float3 color : COLOR0;
            };

            v2f vert(a2v v) {
                v2f o;
                o.pos = UnityObjectToClipPos(v.ver);
                o.color = v.normal * 0.5 + fixed3(0.5, 0.5, 0.5);
                return o;
            }

            fixed4 frag(v2f i) : SV_TARGET {
                return fixed4(i.color, 1.0);
            }
            ENDCG
        }
    }
    FallBack "Diffuse"
}
```

---

## 三、内置文件与变量

### 3.1 常用内置文件

```hlsl
#include "UnityCG.cginc"
```

| 文件 | 说明 |
|:-----|:-----|
| `UnityCG.cginc` | 常用帮助函数、宏、结构体 |
| `UnityShaderVariables.cginc` | 自动包含，内置全局变量 |
| `Lighting.cginc` | 内置光照模型（Surface Shader 自动包含） |
| `HLSLSupport.cginc` | 跨平台宏和定义（自动包含） |

### 3.2 UnityCG.cginc 内置结构体

**顶点着色器输入结构体：**

```hlsl
struct appdata_base {
    float4 vertex : POSITION;
    float3 normal : NORMAL;
    float4 texcoord : TEXCOORD0;
    UNITY_VERTEX_INPUT_INSTANCE_ID
};

struct appdata_tan {
    float4 vertex : POSITION;
    float4 tangent : TANGENT;
    float3 normal : NORMAL;
    float4 texcoord : TEXCOORD0;
    UNITY_VERTEX_INPUT_INSTANCE_ID
};

struct appdata_full {
    float4 vertex : POSITION;
    float4 tangent : TANGENT;
    float3 normal : NORMAL;
    float4 texcoord : TEXCOORD0;
    float4 texcoord1 : TEXCOORD1;
    float4 texcoord2 : TEXCOORD2;
    float4 texcoord3 : TEXCOORD3;
    fixed4 color : COLOR;
    UNITY_VERTEX_INPUT_INSTANCE_ID
};
```

**顶点着色器输出结构体（片元输入）：**
- `v2f_img`：裁剪空间位置 + 纹理坐标

### 3.3 常用内置函数

| 函数 | 说明 |
|:-----|:-----|
| `UnityObjectToClipPos(v)` | 模型空间 → 裁剪空间 |
| `UnityObjectToWorldNormal(n)` | 模型空间法线 → 世界空间（已单位化） |
| `UnityObjectToWorldDir(d)` | 模型空间方向 → 世界空间（已单位化） |
| `UnityWorldToObjectDir(d)` | 世界空间方向 → 模型空间（已单位化） |
| `WorldSpaceViewDir(localPos)` | 模型顶点 → 世界空间观察方向（未单位化） |
| `ObjSpaceViewDir(v)` | 模型顶点 → 模型空间观察方向（未单位化） |
| `WorldSpaceLightDir(localPos)` | 模型顶点 → 世界空间光照方向（未单位化） |
| `ObjSpaceLightDir(v)` | 模型顶点 → 模型空间光照方向（未单位化） |
| `UnityWorldToClipPos(pos)` | 世界空间 → 裁剪空间 |
| `UnityViewToClipPos(pos)` | 观察空间 → 裁剪空间 |
| `UnityObjectToViewPos(pos)` | 模型空间 → 观察空间 |

### 3.4 时间变量

| 变量 | 类型 | 说明 |
|:-----|:-----|:-----|
| `_Time` | float4 | (t/20, t, t\*2, t\*3) |
| `_SinTime` | float4 | 正弦时间 (t/8, t/4, t/2, t) |
| `_CosTime` | float4 | 余弦时间 (t/8, t/4, t/2, t) |
| `unity_DeltaTime` | float4 | 时间增量 (dt, 1/dt, smoothDt, 1/smoothDt) |

### 3.5 Properties 变量与 CG 变量对应

在 Pass 中声明与 Properties 同名的变量以接收材质面板值：

| ShaderLab属性类型 | CG变量类型 |
|:-----------------|:-----------|
| Color, Vector    | float4, half4, fixed4 |
| Range, Float     | float, half, fixed    |
| 2D               | sampler2D             |
| Cube              | samplerCube           |
| 3D                | sampler3D             |

---

## 四、顶点着色器与片元着色器

### 4.1 顶点着色器

```hlsl
struct appdata {
    float4 vertex : POSITION;
    float2 uv : TEXCOORD0;
    float3 normal : NORMAL;
    float4 tangent : TANGENT;
};

struct v2f {
    float2 uv : TEXCOORD0;
    float4 vertex : SV_POSITION;
    UNITY_FOG_COORDS(1)
};

v2f vert(appdata v) {
    v2f o;
    o.vertex = UnityObjectToClipPos(v.vertex);
    o.uv = TRANSFORM_TEX(v.uv, _MainTex);
    UNITY_TRANSFER_FOG(o, o.vertex);
    return o;
}
```

### 4.2 片元着色器

```hlsl
fixed4 frag(v2f i) : SV_Target {
    fixed4 col = tex2D(_MainTex, i.uv);
    UNITY_APPLY_FOG(i.fogCoord, col);
    return col;
}
```

`SV_Target` 标识输出到帧缓冲的颜色。Alpha 通道仅在混合模式开启时有效。

---

## 五、纹理系统

### 5.1 基础纹理映射

```hlsl
Properties {
    _MainTex ("Main Tex", 2D) = "white" {}
}

// CG 代码中：
sampler2D _MainTex;        // 纹理采样器
float4 _MainTex_ST;         // 缩放(xy)和偏移(zw)
```

`_MainTex_ST.xy` = Tiling（缩放），`_MainTex_ST.zw` = Offset（偏移）。默认值 `(1,1,0,0)`。

```hlsl
// UV 变换宏
o.uv = TRANSFORM_TEX(v.texcoord, _MainTex);
// 等价于: o.uv = v.texcoord.xy * _MainTex_ST.xy + _MainTex_ST.zw;
```

### 5.2 tex2D 采样

```hlsl
fixed4 col = tex2D(_MainTex, i.uv);  // 根据 UV 采样纹理颜色
fixed3 albedo = tex2D(_MainTex, i.uv).rgb * _Color.rgb; // 反照率
```

纹理属性（分辨率、格式、Wrap Mode、Filter Mode）在纹理导入设置中配置，可为不同平台设置不同参数。

### 5.3 UV 扰动效果

通过对噪声纹理采样来偏移主纹理 UV：

```hlsl
// 滚动扭曲纹理
i.uvDistTex.x += ((_Time.x + randomSeed) * _DistortTexXSpeed) % 1;
i.uvDistTex.y += ((_Time.x + randomSeed) * _DistortTexYSpeed) % 1;

// 采样噪声纹理作为偏移强度
half distortAmnt = (tex2D(_DistortTex, i.uvDistTex).r - 0.5) * 0.2 * _DistortAmount;

// 扰动主纹理 UV
i.uv.x += distortAmnt;
i.uv.y += distortAmnt;
```

### 5.4 描边燃烧效果

通过叠加轮廓纹理和噪声扰动实现燃烧描边：

| 属性 | 说明 |
|:-----|:-----|
| `_OutlineWidth` | 轮廓宽度 |
| `_OutlineGlow` | 发光强度 |
| `_OutlineAlpha` | 透明度 |
| `_OutlineColor` | 轮廓颜色 |
| `_OutlineDistortAmount` | 失真强度 |
| `_OutlineTexXSpeed/YSpeed` | 轮廓纹理滚动速度 |
| `_OutlineDistortTexXSpeed/YSpeed` | 失真纹理滚动速度 |

---

## 六、基本光照计算

### 6.1 光与物体的交互

光与物体相交有两种结果：
- **散射（Scattering）**：不改变密度和颜色，只改变方向 → 分为反射和折射
- **吸收（Absorption）**：改变密度和颜色，不改变方向

使用不同部分计算散射方向：
- **高光反射（Specular）**：物体表面如何反射光线
- **漫反射（Diffuse）**：折射、吸收、散射后的出射光

### 6.2 标准光照模型四部分

| 分量 | 公式 | 说明 |
|:-----|:-----|:-----|
| 自发光 Emissive | $C_{emission} = m_{emission}$ | 自己发光但不作为光源 |
| 环境光 Ambient | $C_{ambient} = g_{ambient}$ | 通过 `UNITY_LIGHTMODEL_AMBIENT` 获取 |
| 漫反射 Diffuse | 兰伯特定律（下详） | 与观察方向无关 |
| 高光 Specular | Blinn-Phong（下详） | 与观察方向有关 |

### 6.3 漫反射（兰伯特定律）

$$
C_{diffuse} = (C_{light} \cdot m_{diffuse}) \cdot max(0, \hat{n} \cdot \hat{l})
$$

`m_diffuse` 为漫反射颜色，`n̂` 为法线方向，`l̂` 为光源方向。`saturate(x)` = `max(0, min(1, x))`。

**逐顶点漫反射**（在 vert 中计算）：

```hlsl
v2f vert(a2v v) {
    v2f o;
    o.pos = UnityObjectToClipPos(v.vertex);
    fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;
    fixed3 worldNormal = normalize(UnityObjectToWorldNormal(v.normal));
    fixed3 worldLight = normalize(_WorldSpaceLightPos0.xyz);
    fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * saturate(dot(worldNormal, worldLight));
    o.color = ambient + diffuse;
    return o;
}
```

**逐片元漫反射**（在 frag 中计算，更平滑）：

```hlsl
fixed4 frag(v2f i) : SV_Target {
    fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT;
    fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz);
    fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * saturate(dot(i.worldNormal, worldLightDir));
    return fixed4(ambient + diffuse, 1.0);
}
```

> [!warning] 必须设置 Tags
> `Tags {"LightMode"="ForwardBase"}` 才能获取 Unity 的光照变量。平行光方向通过 `normalize(_WorldSpaceLightPos0.xyz)` 获取。

### 6.4 半兰伯特模型

$$
C_{diffuse} = (C_{light} \cdot m_{diffuse})(\alpha(\hat{n} \cdot \hat{l}) + \beta)
$$

其中 $\alpha = \beta = 0.5$。背面不再纯黑，而是呈现渐变：

```hlsl
fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * (0.5 + 0.5 * dot(worldNormal, worldLightDir));
```

### 6.5 高光反射（Blinn-Phong 模型）

使用半程向量简化计算：

$$
\hat{h} = \frac{\hat{v} + \hat{l}}{\left | \hat{v} + \hat{l} \right |}
$$

$$
C_{specular} = (C_{light} \cdot m_{specular}) \cdot max(0, \hat{n} \cdot \hat{h})^{M_{gloss}}
$$

$M_{gloss}$ 为光泽度（Gloss），控制光斑大小，常用 8-256。半程向量比反射向量更好算。

```hlsl
fixed4 frag(v2f i) : SV_Target {
    fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT.xyz;
    fixed3 worldNormal = normalize(i.worldNormal);
    fixed3 worldLightDir = normalize(UnityWorldSpaceLightDir(i.worldPos));
    fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * max(0, dot(worldNormal, worldLightDir));
    fixed3 viewDir = normalize(UnityWorldSpaceViewDir(i.worldPos));
    fixed3 halfDir = normalize(worldLightDir + viewDir);
    fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(max(0, dot(worldNormal, halfDir)), _Gloss);
    return fixed4(ambient + diffuse + specular, 1.0);
}
```

也可以使用 `reflect(i, n)` 实现经典 Phong 模型。

### 6.6 纹理与光照结合

纹理颜色（albedo）参与环境光和漫反射，高光通常不乘纹理色：

```hlsl
fixed3 albedo = tex2D(_MainTex, i.uv).rgb * _Color.rgb;
fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT * albedo;
fixed3 diffuse = _LightColor0.rgb * albedo * saturate(dot(normal, lightDir));
fixed3 specular = _LightColor0.rgb * _Specular.rgb * pow(saturate(dot(normal, halfDir)), _Gloss);
```

---

## 七、Shader调试与优化基础

### 7.1 调试工具

- **Frame Debugger**：Window > Analysis > Frame Debugger，逐帧查看渲染过程
- **RenderDoc**：跨平台图形调试器，可截帧分析 Draw Call、Shader 变量、纹理输入输出
- **Nsight Graphics**：NVIDIA GPU 专用调试工具

### 7.2 Material Property Block

原本相同材质才能合批（颜色不同就生成新材质实例）。使用 Material Property Block 可在不生成新材质的前提下修改属性值，保持 SRP Batcher / GPU Instancing 的合批能力。

### 7.3 性能提示

- 使用尽可能低的精度（`fixed` > `half` > `float`），移动平台尤其重要
- 减少纹理采样次数（合并贴图通道）
- 避免在片元着色器中进行复杂数学运算（尽量移到顶点着色器）
- 减少分支（if/else）和动态循环
- 利用 Unity 的批处理机制（Static/Dynamic Batching, SRP Batcher, GPU Instancing）
- Constant Buffer：将不变数据存于 GPU 常量缓冲，按更新频率分组（per-frame / per-material / per-draw）

---

## 八、Unity 批处理机制

### 8.1 批处理概念

将多个 DrawCall 合并为一个，减少 CPU 和 GPU 之间的通信开销。

### 8.2 Static Batching

- 静态（非移动）对象利用预计算信息减少资源消耗
- 只合并**相同材质**的物体
- 将静态物体合并为大网格（存储在 GPU 的 Vertex/Index Buffer 中）
- 不一定减少 DrawCall 次数，但让 CPU 在"设置渲染状态→提交 DrawCall"上更高效（避免重复 buffer binding）

### 8.3 Dynamic Batching

为低端设备设计。只有当批处理 CPU 开销 < DrawCall 开销时才有优化效果。在新设备上可能反而降低性能。

### 8.4 SRP Batcher（URP 默认开启）

所有材质球在显存中占有固定的 CBuffer。如果材质球内容不变，CPU 无需 SetUp → Upload，减少 CPU 渲染开销。**不减少 DrawCall**，而是减少 DrawCall 之间的 CPU 工作量。

适用场景：大量不同材质球但使用相同 Shader Variant 的物体。

### 8.5 GPU Instancing

适用于大量**同 Mesh + 同 Material** 的重复物体（建筑、树、草等）。

---

## 九、Unity 四元数（Quaternion）

四元数用于表示旋转，避免欧拉角的万向锁问题：

| 函数 | 说明 |
|:-----|:-----|
| `Quaternion.LookRotation` | 根据方向创建旋转 |
| `Quaternion.Angle` | 两旋转间角度 |
| `Quaternion.Euler` | 欧拉角转四元数 |
| `Quaternion.Slerp` | 球面线性插值 |
| `Quaternion.FromToRotation` | 从一个方向到另一个方向 |
| `Quaternion.identity` | 无旋转 |

欧拉角按顺规 XYZ 应用：(0, 90, 90) 代表先绕 +Z 旋转 90°，再绕 +Y 旋转 90°。

---

## 相关页面

- [[Shader高级特性]] — 透明、几何着色器与光照系统
- [[渲染管线理论]] — GAMES101 图形学理论
- [[OpenGL学习笔记]] — OpenGL/GLSL 对比
- [[Shader基础与Unity内置结构-摘要]] — 精简摘要
