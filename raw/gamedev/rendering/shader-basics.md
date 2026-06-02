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
---

# Shader基础与Unity内置结构

## 一、Shader基础结构

Unity Shader 由以下层次组成：`Shader` > `Properties` > `SubShader` > `Pass`。

### 1.1 Properties（属性）

连接材质和Shader的桥梁，结构为 `Name ("displayName", PropertyType) = DefaultValue`。

| ShaderLab属性类型 | CG变量类型              |
|:-----------------|:---------------------|
| Color, Vector    | float4, half4, fixed4 |
| Range, Float      | float, half, fixed    |
| 2D                | sampler2D             |
| Cube              | samplerCube           |
| 3D                | sampler3D             |

属性通过 **name** 引用（通常以下划线开头），在材质检视面板显示 **display name**。每个属性在等号后给出默认值：

- `_Range` 和 `_Float` 默认值是单个数字
- `_Color` 和 `_Vector` 默认值是括在圆括号中的四个数字
- 2D纹理默认值为空字符串或内置默认纹理：`"white"`, `"black"`, `"gray"`, `"bump"`, `"red"`
- 非2D纹理（立方体、3D、2D数组）默认值为空字符串

### 1.2 SubShader

Unity 中的每个着色器包含一个子着色器列表。当 Unity 渲染网格时，选择在用户显卡上运行的第一个子着色器。结构为：

```hlsl
Subshader { [Tags] [CommonState] Passdef [Passdef ...] }
```

**常见 Tags：**
- `Queue` — 渲染队列（Background=1000, Geometry=2000, AlphaTest=2450, Transparent=3000, Overlay=4000）
- `RenderType` — 着色器分类，如 Opaque、Transparent、TransparentCutout、Background、Overlay
- `DisableBatching` — 禁用批处理
- `ForceNoShadowCasting` — 强制不投射阴影
- `IgnoreProjector` — 常用于半透明物体

**RenderType 常用值：**

| 类型 | 描述 |
|:-----|:-----|
| Opaque | 不透明着色器（法线、自发光、反射、地形） |
| Transparent | 半透明着色器（透明、粒子、字体） |
| TransparentCutout | 蒙皮透明着色器（植被） |
| Background | 天空盒着色器 |
| Overlay | 光晕/闪光着色器 |
| TreeOpaque | 地形引擎中的树皮 |
| TreeTransparentCutout | 地形引擎中的树叶 |
| TreeBillboard | 地形引擎中的广告牌树 |
| Grass | 地形引擎中的草 |
| GrassBillboard | 地形引擎中的广告牌草 |

**LOD (Level of Detail)**：控制着色器细节层次。当 LOD 值小于设定值时，相应的 Shader/SubShader 不会工作。

Unity内建着色器LOD设置：
- VertexLit: 100
- Decal, Reflective VertexLit: 150
- Diffuse: 200
- Diffuse Detail, Reflective Bumped: 250
- Bumped, Specular: 300
- Bumped Specular: 400
- Parallax: 500
- Parallax Specular: 600

可通过 `Shader.maximumLOD` 针对单个着色器设定，或 `Shader.globalMaximumLOD` 全局设定。

### 1.3 Pass

Pass 代码块使游戏对象的几何体被渲染一次。可以有多个 Pass。

```glsl
Pass {
    [Name]
    [Tags]
    [RenderSetup]
    // shader code
}
```

**特殊 Pass：**
- `UsePass` — 使用其他Shader的Pass
- `GrabPass` — 抓取屏幕结果并存储在纹理中

**Pass 状态命令：**

| 命令 | 说明 |
|:-----|:-----|
| `Cull Back\|Front\|Off` | 剔除模式 |
| `ZTest Less\|Greater\|LEqual\|GEqual\|Equal\|NotEqual\|Always` | 深度测试 |
| `ZWrite On\|Off` | 深度写入 |
| `Blend SrcFactor DstFactor` | 混合模式 |
| `ColorMask RGB\|A\|0\|任意组合` | 颜色通道写入 |
| `Offset Factor, Units` | Z缓冲区深度偏移（解决Z-Fighting） |

**深度测试规则（四种情况）：**
1. 深度测试通过，深度写入开启 → 写深度缓冲，写颜色缓冲
2. 深度测试通过，深度写入关闭 → 不写深度缓冲，写颜色缓冲
3. 深度测试失败，深度写入开启 → 不写深度缓冲，不写颜色缓冲
4. 深度测试失败，深度写入关闭 → 不写深度缓冲，不写颜色缓冲

`ZTest Off` 等同于 `ZTest Always`，关闭深度测试等于完全通过。

**Z-Fighting**：当两个面在同一平面下Z值极其接近时，会出现交替显示/闪烁现象。使用 `Offset` 可分出先后顺序。

**Blend（混合）因子：**
- `One`：源或目标的完整值
- `Zero`：0
- `SrcColor`：源的颜色值
- `SrcAlpha`：源的Alpha值
- `DstColor`：目标的颜色值
- `DstAlpha`：目标的Alpha值
- `OneMinusSrcColor`：1-源颜色
- `OneMinusSrcAlpha`：1-源Alpha
- `OneMinusDstColor`：1-目标颜色
- `OneMinusDstAlpha`：1-目标Alpha

常见混合类型：
- `Blend SrcAlpha OneMinusSrcAlpha` — 传统透明度
- `Blend One OneMinusSrcAlpha` — 预乘透明度
- `Blend One One` — 加法/线性减淡
- `Blend OneMinusDstColor One` — 软加法/滤色
- `Blend DstColor Zero` — 乘法/正片叠底
- `Blend DstColor SrcColor` — 2x乘法

**ColorMask**：设置颜色写入通道（RGB|A|0|组合）。可用于渲染阴影贴图时关闭所有颜色通道，或在不写入渲染目标的情况下填充模板缓冲区。

### 1.4 FallBack

所有 SubShader 都无法运行时使用的后备着色器。如 `FallBack "Diffuse"`。

---

## 二、HLSL/CG 基础类型与语义

### 2.1 精度类型

| 类型 | 精度 |
|:-----|:-----|
| `float` | 最高精度，32位存储 |
| `half` | 中等精度，16位，范围 -60000 ~ +60000 |
| `fixed` | 最低精度，11位，范围 -2.0 ~ +2.0 |

复合类型：`float3/4`、`float4x4`（方形矩阵）。某些平台仅支持方形矩阵。

### 2.2 纹理采样器

```c++
sampler2D _MainTex;       // 2D纹理
samplerCUBE _Cubemap;     // 立方体贴图
sampler2D_half _MainTex;  // 低精度采样器（移动平台）
sampler2D_float _MainTex; // 完整浮点精度采样器（HDR）
```

### 2.3 语义（Semantics）

赋给Shader输入输出的字符串，表达参数含义，让 Shader 知道从哪里读取数据、把数据输出到哪里。

**应用到顶点着色器（a2v 输入语义）：**
| 语义 | 意义 |
|:-----|:-----|
| `POSITION` | 顶点位置（模型空间） |
| `NORMAL` | 法线向量（模型空间） |
| `TANGENT` | 切线向量（模型空间） |
| `TEXCOORD0~n` | 纹理坐标（第n套UV） |
| `COLOR` | 顶点颜色 |

**顶点到片元着色器（v2f 传递语义）：**
| 语义 | 意义 |
|:-----|:-----|
| `SV_POSITION` | 裁剪空间位置（System Value） |
| `SV_TARGET` | 片元着色器输出颜色 |
| `COLOR0/1` | 颜色传递 |
| `TEXCOORD0~7` | 纹理坐标传递 |

**命名规范：**
- `a2v`：application to vertex（顶点着色器输入结构体）
- `v2f`：vertex to fragment（顶点着色器输出/片元着色器输入结构体）

### 2.4 时间变量

| 变量 | 类型 | 说明 |
|:-----|:-----|:-----|
| `_Time` | float4 | (t/20, t, t*2, t*3) |
| `_SinTime` | float4 | 正弦时间 (t/8, t/4, t/2, t) |
| `_CosTime` | float4 | 余弦时间 (t/8, t/4, t/2, t) |
| `unity_DeltaTime` | float4 | 时间增量 (dt, 1/dt, smoothDt, 1/smoothDt) |

---

## 三、内置文件与变量

### 3.1 常用内置文件

```glsl
#include "UnityCG.cginc"
```

| 文件 | 说明 |
|:-----|:-----|
| `UnityCG.cginc` | 常用帮助函数、宏、结构体 |
| `UnityShaderVariables.cginc` | 自动包含，内置全局变量 |
| `Lighting.cginc` | 内置光照模型（Surface Shader自动包含） |
| `HLSLSupport.cginc` | 跨平台宏和定义（自动包含） |

### 3.2 UnityCG.cginc 内置结构体

**顶点着色器输入：**
- `appdata_base`：顶点位置、法线、第一组纹理坐标
- `appdata_tan`：顶点位置、切线、法线、第一组纹理坐标
- `appdata_full`：顶点位置、切线、法线、四组纹理坐标、顶点颜色
- `appdata_img`：顶点位置、第一组纹理坐标

**顶点着色器输出：**
- `v2f_img`：裁剪空间位置、纹理坐标

### 3.3 常用内置函数

| 函数 | 说明 |
|:-----|:-----|
| `UnityObjectToClipPos(v)` | 模型空间→裁剪空间 |
| `UnityObjectToWorldNormal(n)` | 模型空间法线→世界空间（已单位化） |
| `UnityObjectToWorldDir(d)` | 模型空间方向→世界空间（已单位化） |
| `UnityWorldToObjectDir(d)` | 世界空间方向→模型空间（已单位化） |
| `WorldSpaceViewDir(localPos)` | 世界空间观察方向（未单位化） |
| `ObjSpaceViewDir(v)` | 模型空间观察方向（未单位化） |
| `WorldSpaceLightDir(localPos)` | 世界空间光照方向（未单位化） |
| `ObjSpaceLightDir(v)` | 模型空间光照方向（未单位化） |
| `UnityWorldToClipPos(pos)` | 世界空间→裁剪空间 |
| `UnityViewToClipPos(pos)` | 观察空间→裁剪空间 |

---

## 四、顶点着色器与片元着色器

### 4.1 顶点着色器

接收 `appdata` 结构体，输出 `v2f` 结构体。顶点着色器负责将顶点从模型空间变换到裁剪空间。

```c++
struct appdata {
    float4 vertex : POSITION;   // 顶点位置
    float2 uv : TEXCOORD0;      // 纹理坐标
    float3 normal : NORMAL;     // 法线
    float4 tangent : TANGENT;   // 切线
};

struct v2f {
    float2 uv : TEXCOORD0;           // UV传递
    float4 vertex : SV_POSITION;     // 裁剪空间位置
    UNITY_FOG_COORDS(1)              // 雾效坐标（宏）
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

接收 `v2f` 结构体，输出颜色到帧缓冲。

```c++
fixed4 frag(v2f i) : SV_Target {
    fixed4 col = tex2D(_MainTex, i.uv);   // 采样纹理
    UNITY_APPLY_FOG(i.fogCoord, col);      // 应用雾效
    return col;                            // 返回最终颜色
}
```

`SV_Target` 标识片元着色器输出的颜色值，最终存储到帧缓冲。Alpha 只有在混合模式开启时才有效。

---

## 五、纹理系统

### 5.1 基础纹理映射

在 Properties 中声明纹理后，需要在 CG 代码中声明两个变量：

```c++
sampler2D _MainTex;       // 纹理采样器
float4 _MainTex_ST;        // 纹理的缩放(xy)和偏移(zw)
```

`_MainTex_ST.xy` 存储缩放值（Tiling），`_MainTex_ST.zw` 存储偏移值（Offset）。默认值：`(1,1,0,0)`。

UV 变换使用宏：
```c++
o.uv = TRANSFORM_TEX(v.texcoord, _MainTex);
// 等价于 o.uv = v.texcoord.xy * _MainTex_ST.xy + _MainTex_ST.zw;
```

### 5.2 tex2D 采样与纹理属性

```c++
fixed4 col = tex2D(_MainTex, i.uv);  // 在纹理上根据UV取颜色
```

`albedo = tex2D(_MainTex, i.uv).rgb * _Color.rgb` — 纹理颜色乘上 tint 颜色得到反照率。

可在纹理导入设置中为不同平台设置不同最大分辨率和格式。常用 Wrap Mode 和 Filter Mode 可直接在纹理属性面板修改。

### 5.3 UV扰动效果

通过对噪声纹理采样来扰动主纹理的UV坐标：

```c++
// 滚动扭曲纹理UV
i.uvDistTex.x += ((_Time.x + randomSeed) * _DistortTexXSpeed) % 1;
i.uvDistTex.y += ((_Time.x + randomSeed) * _DistortTexYSpeed) % 1;

// 计算扭曲强度
half distortAmnt = (tex2D(_DistortTex, i.uvDistTex).r - 0.5) * 0.2 * _DistortAmount;

// 扰动主纹理UV
i.uv.x += distortAmnt;
i.uv.y += distortAmnt;
```

原理：对噪声图进行采样作为偏移量，施加到主纹理的UV坐标上，产生扭曲/热浪效果。

### 5.4 描边燃烧效果

通过叠加轮廓纹理和噪声扰动纹理来实现燃烧描边。关键属性包括：

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

光与物体相交产生两种结果：
- **散射**（Scattering）：不改变密度和颜色，只改变方向。分为反射和折射
- **吸收**（Absorption）：改变密度和颜色，不改变方向

使用不同部分计算不同的散射方向：
- **高光反射**（Specular）：物体表面如何反射光线
- **漫反射**（Diffuse）：折射、吸收、散射后的出射光

### 6.2 标准光照模型

四个组成部分：
- **Emissive（自发光）**：$C_{emission} = m_{emission}$，自己发光但不作为光源
- **Ambient（环境光）**：$C_{ambient} = g_{ambient}$，全局变量 `UNITY_LIGHTMODEL_AMBIENT`
- **Diffuse（漫反射）**：兰伯特定律，与观察方向无关
- **Specular（高光反射）**：镜面反射，与观察方向有关

### 6.3 漫反射（兰伯特定律）

$$
C_{diffuse} = (C_{light} \cdot m_{diffuse}) \cdot max(0, \hat{n} \cdot \hat{l})
$$

其中 $m_{diffuse}$ 为漫反射颜色，$\hat{n}$ 为法线方向，$\hat{l}$ 为光源方向。`saturate` 函数将结果钳制到 [0, 1]。

```c++
// saturate 等价于 max(0, min(1, x))
float saturate(float x) {
    return max(0.0, min(1.0, x));
}
```

**逐顶点漫反射（在vert中计算）：**

```c++
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

**逐片元漫反射（在frag中计算，效果更平滑）：**

```c++
fixed4 frag(v2f i) : SV_Target {
    fixed3 ambient = UNITY_LIGHTMODEL_AMBIENT;
    fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz);
    fixed3 diffuse = _LightColor0.rgb * _Diffuse.rgb * saturate(dot(i.worldNormal, worldLightDir));
    return fixed4(ambient + diffuse, 1.0);
}
```

注意：必须设置 `Tags {"LightMode"="ForwardBase"}` 才能获取 Unity 的光照变量。对于平行光，方向使用 `normalize(_WorldSpaceLightPos0.xyz)`。

### 6.4 半兰伯特模型

$$
C_{diffuse} = (C_{light} \cdot m_{diffuse})(\alpha(\hat{n} \cdot \hat{l}) + \beta)
$$

其中 $\alpha = \beta = 0.5$。背面不会完全变黑，而是呈现渐变。

```c++
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

$M_{gloss}$ 为光泽度（Gloss），控制光斑大小，常用 8-256。半程向量比反射向量更好算，且 cos 容忍度更低（光斑更小更真实）。

```c++
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

也可以使用 CG 内置的 `reflect(i, n)` 函数计算反射方向（Phong模型），其中 `i` 为入射方向，`n` 为法线。

### 6.6 纹理与光照结合

纹理颜色（albedo）参与环境光和漫反射的计算，高光通常不乘纹理（因为纯白色高光）：

```c++
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

### 7.2 Material Property Block

原本相同材质才能合批（颜色不同就会生成新材质实例）。使用 Material Property Block 可以在不生成新材质的前提下修改属性值，保持 SRP Batcher / GPU Instancing 的合批能力。

### 7.3 Constant Buffer

将很少变动的数据直接放到 GPU 内的常量缓冲区。每帧更新，且每次提交整个 Constant Buffer。需要合理安排变量布局：按更新频率分组（per-frame / per-material / per-draw）。

### 7.4 性能提示

- 使用尽可能低的精度（fixed > half > float），移动平台尤其重要
- 减少纹理采样次数（合并贴图通道）
- 避免在片元着色器中进行复杂数学运算（尽量移到顶点着色器）
- 减少分支（if/else）和动态循环
- 利用 Unity 的批处理机制（Static/Dynamic Batching, SRP Batcher, GPU Instancing）

---

## 八、Unity四元数（Quaternion）

四元数用于表示旋转，避免欧拉角的万向锁问题。常用函数：

| 函数 | 说明 |
|:-----|:-----|
| `Quaternion.LookRotation` | 根据方向创建旋转 |
| `Quaternion.Angle` | 两旋转间角度 |
| `Quaternion.Euler` | 欧拉角转四元数 |
| `Quaternion.Slerp` | 球面线性插值 |
| `Quaternion.FromToRotation` | 从一个方向到另一个方向的旋转 |
| `Quaternion.identity` | 无旋转 |

欧拉角按顺规 XYZ 应用旋转：(0, 90, 90) 代表先绕 +Z 轴旋转 90 度，再绕 +Y 轴旋转 90 度。
