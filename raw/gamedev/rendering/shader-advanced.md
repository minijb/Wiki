---
title: Shader进阶：透明、几何、表面着色与光照系统
type: source
updated: 2026-06-02
tags:
  - shader
  - unity
  - rendering
  - transparency
  - lighting
  - geometry-shader
---

# Shader进阶：透明、几何、表面着色与光照系统

## 一、透明渲染与混合

### 1.1 渲染顺序与深度值

不透明物体先渲染近处再渲染远处（利用深度值Depth）。透明物体则需要从远到近渲染（画家算法），因为后渲染的透明物体需要混合到已渲染的像素上。

**渲染队列顺序：**

| 队列 | 值 | 说明 |
|:-----|:---|:-----|
| Background | 1000 | 最先渲染（背景） |
| Geometry | 2000 | 不透明物体（默认） |
| AlphaTest | 2450 | 透明度测试 |
| Transparent | 3000 | 半透明对象，从后往前 |
| Overlay | 4000 | 最后渲染（叠加效果） |

### 1.2 透明渲染的深度问题

渲染透明物体时如果写入深度缓冲会导致后面的物体被遮挡。解决方案：
- 不写入深度缓存（`ZWrite Off`）— 这样后面的物体不知道前面有透明物体，会完整绘制出来，混合阶段叠加上去
- 不能简单关闭深度测试或设为Always（会覆盖不透明物体）

### 1.3 透明度测试（Alpha Test）

使用 `clip` 函数丢弃不满足条件的片元——要么完全透明，要么完全不透明。

```c++
Properties {
    _Cutoff ("Alpha Cutoff", Range(0, 1)) = 0.5
}

SubShader {
    Tags {"Queue"="AlphaTest" "IgnoreProjector"="True" "RenderType"="TransparentCutout"}
    // ...
}

// 片元着色器中：
fixed4 texColor = tex2D(_MainTex, i.uv);
clip(texColor.a - _Cutoff);  // 如果 alpha < Cutoff，丢弃该片元
// 等价于 if ((texColor.a - _Cutoff) < 0.0) discard;
```

- `clip(float4 x)`：如果 x 的任意分量 < 0，则丢弃该片元
- RenderType 设为 `TransparentCutout`
- 不需要关闭深度写入

### 1.4 透明度混合（Alpha Blending）

**混合公式：**
$$
DST_{color} = SrcAlpha \times SrcColor + (1-SrcAlpha) \times DstColor
$$

```c++
SubShader {
    Tags {"Queue"="Transparent" "IgnoreProjector"="True" "RenderType"="Transparent"}
    Pass {
        ZWrite Off
        Blend SrcAlpha OneMinusSrcAlpha
        // ...
    }
}
```

片元着色器中输出带Alpha的颜色：
```c++
return fixed4(ambient + diffuse, texColor.a * _AlphaScale);
```

### 1.5 开启深度写入的半透明

使用两个 Pass 解决自身遮挡问题：

```c++
Pass {
    ZWrite On
    ColorMask 0   // 不输出任何颜色，只写深度
}
Pass {
    ZWrite Off
    Blend SrcAlpha OneMinusSrcAlpha
    // 正常光照计算
}
```

第一个 Pass 写入深度但不输出颜色，可以剔除被自身遮挡的片元。

### 1.6 双面渲染（Cull）

透明渲染因关闭深度写入会导致渲染混乱（看不到背面），解决方案是两个 Pass：

```c++
// 第一个 Pass：只渲染背面
Pass {
    Cull Front
    ZWrite Off
    Blend SrcAlpha OneMinusSrcAlpha
}
// 第二个 Pass：只渲染正面
Pass {
    Cull Back
    ZWrite Off
    Blend SrcAlpha OneMinusSrcAlpha
}
```

Cull 可选值：`Back`（默认，剔除背面）、`Front`（剔除正面）、`Off`（不剔除）。

### 1.7 常见混合类型汇总

```c++
// 正常透明度混合
Blend SrcAlpha OneMinusSrcAlpha

// 柔和相加（Soft Additive）
Blend OneMinusDstColor One

// 正片叠底（Multiply）
Blend DstColor Zero

// 两倍相乘（2x Multiply）
Blend DstColor SrcColor

// 变暗（Darken）
BlendOp Min
Blend One One

// 变亮（Lighten）
BlendOp Max
Blend One One

// 滤色（Screen）
Blend OneMinusDstColor One
// 等同于 Blend One OneMinusSrcColor

// 线性减淡（Linear Dodge）
Blend One One
```

---

## 二、模板测试（Stencil Test）

模板缓冲区是每像素 8 位整数的通用遮罩，用于保存或丢弃像素。

流程：GPU 读取模板缓冲区中该片元位置的模板值，与参考值比较。不通过测试的片元被舍弃。无论片元是否通过测试，都可以修改模板缓冲区。

```c++
Stencil {
    Ref 2              // 参考值
    Comp Always        // 比较函数（Always=始终通过）
    Pass Replace       // 通过后写入操作（Replace=替换为Ref值）
}
```

比较函数：`Always`, `Never`, `Less`, `Equal`, `LEqual`, `Greater`, `NotEqual`, `GEqual`。
操作：`Keep`, `Zero`, `Replace`, `IncrSat`, `DecrSat`, `Invert`, `IncrWrap`, `DecrWrap`。

---

## 三、法线贴图与切线空间

### 3.1 法线贴图原理

法线贴图存储切线空间的法线信息，通过扰动法线方向来模拟表面凹凸细节，而不增加几何复杂度。

**模型空间法线 vs 切线空间法线：**

| 模型空间 | 切线空间 |
|:---------|:---------|
| 实现简单，计算简单 | 自由度高，相对切线 |
| 边界平滑 | 可进行UV动画 |
| — | 可重用、可压缩 |

### 3.2 TBN矩阵

将切线空间法线变换到世界/模型空间：

**T**angent（切线）、**B**inormal（副切线）、**N**ormal（法线）。

```c++
// 计算副切线
float3 binormal = cross(normalize(v.normal), normalize(v.tangent.xyz)) * v.tangent.w;
// 构建 TBN 矩阵（模型空间到切线空间）
float3x3 rotation = float3x3(v.tangent.xyz, binormal, v.normal);
```

`v.tangent.w` 用于确定副切线的正确方向（+1 或 -1）。

快捷宏：`TANGENT_SPACE_ROTATION` 等价于上面的矩阵构建。

### 3.3 在切线空间计算光照

将光照方向和视角方向变换到切线空间，直接使用切线空间法线：

```c++
TANGENT_SPACE_ROTATION;
o.lightDir = mul(rotation, ObjSpaceLightDir(v.vertex)).xyz;
o.viewDir = mul(rotation, ObjSpaceViewDir(v.vertex)).xyz;

// 片元着色器
fixed4 packedNormal = tex2D(_BumpMap, i.uv.zw);
fixed3 tangentNormal = UnpackNormal(packedNormal);
tangentNormal.xy *= _BumpScale;
tangentNormal.z = sqrt(1.0 - saturate(dot(tangentNormal.xy, tangentNormal.xy)));

fixed3 diffuse = _LightColor0.rgb * albedo * max(0, dot(tangentNormal, tangentLightDir));
```

### 3.4 在世界空间计算

将切线空间法线变换到世界空间（存储 TtoW 矩阵的行）：

```c++
// 顶点着色器：构建世界空间 TBN 并存储
o.TtoW0 = float4(worldTangent.x, worldBinormal.x, worldNormal.x, worldPos.x);
o.TtoW1 = float4(worldTangent.y, worldBinormal.y, worldNormal.y, worldPos.y);
o.TtoW2 = float4(worldTangent.z, worldBinormal.z, worldNormal.z, worldPos.z);

// 片元着色器：将切线空间法线变换到世界空间
bump = normalize(half3(dot(i.TtoW0.xyz, bump), dot(i.TtoW1.xyz, bump), dot(i.TtoW2.xyz, bump)));
```

世界空间方法的优势：可以复用世界空间的光照方向和视角方向，不需要每个片元都做空间变换。

### 3.5 UnpackNormal 注意事项

Unity 会根据不同平台使用不同的法线打包方式，因此推荐使用 `UnpackNormal` 而不是手动计算：
- 如果纹理未标记为 "Normal map"：`tangentNormal.xy = (packedNormal.xy * 2 - 1) * _BumpScale; tangentNormal.z = sqrt(1.0 - saturate(dot(tangentNormal.xy, tangentNormal.xy)));`
- 标记为 "Normal map"：直接 `tangentNormal = UnpackNormal(packedNormal);`

法线 z 分量由 xy 导出：$z = \sqrt{1 - (x^2 + y^2)}$。如果不需要偏移（xy=0），则 z=1，即普通法线。

---

## 四、玻璃折射与GrabPass

### 4.1 GrabPass 抓取屏幕

```c++
GrabPass { }                        // 每批次抓取一次
GrabPass { "ExampleTextureName" }   // 第一次遇到时抓取，同一帧内多个子着色器可共享
```

`GrabPass` 将当前帧缓冲区内容抓取到纹理中，通过 `_GrabTexture` 引用。

### 4.2 ComputeGrabScreenPos

```c++
srcPos = ComputeGrabScreenPos(o.pos);  // 返回齐次坐标下的屏幕坐标值
```

输入：裁剪空间坐标（`clipPos`），通常由 `UnityObjectToClipPos` 计算。
输出：float4，表示屏幕空间坐标，可直接用于采样 `_GrabTexture`。

### 4.3 折射偏移实现

```c++
// 通过法线贴图的法线方向对屏幕图像进行扭曲偏移
float2 offset = normal.xy * _BumpScale * _RefractionTex_TexelSize.xy;
float2 grabUV = srcPos.xy / srcPos.w + offset;
fixed4 refractedColor = tex2D(_GrabTexture, grabUV);
```

`_RefractionTex_TexelSize` 是纹理的纹素大小（Texel Size），即每个像素在UV坐标中的大小。采样时需要做透视除法（`srcPos.xy / srcPos.w`）得到正确的屏幕UV。

---

## 五、表面着色器（Surface Shader）

表面着色器是 Unity 对顶点/片元着色器的更高层抽象，默认帮我们实现了光照计算。

```c++
#pragma surface surf BlinnPhong addshadow fullforwardshadows vertex:disp tessellate:tessFixed nolightmap
```

- `surf`：表面函数名
- `BlinnPhong`：光照模型
- `addshadow`：生成阴影投射Pass
- `fullforwardshadows`：支持所有光源类型的阴影
- `vertex:disp`：自定义顶点修改函数
- `tessellate:tessFixed`：曲面细分函数

### 5.1 曲面细分（Tessellation）

四种细分方式：
- **固定数量**：返回固定值
- **基于边长**：根据三角形边长动态调整
- **基于距离**：根据相机距离动态调整
- **Phong细分**：基于Phong平滑的细分

```c++
float _Tess;

float4 tessFixed() {
    return _Tess;  // 固定细分级别
}

void disp(inout appdata v) {
    // 根据位移贴图调整顶点位置
    float d = tex2Dlod(_DispTex, float4(v.texcoord.xy, 0, 0)).r * _Displacement;
    v.vertex.xyz += v.normal * d;
}
```

### 5.2 水面效果

通过顶点动画（正弦波）和法线贴图动画实现动态水面：

```c++
// 顶点着色器：正弦波浪
fixed height = sin(_Time.y * _WaveSpeed + v.vertex.z * _WaveGap + v.vertex.x) * _WaveHeight;
v.vertex.xyz += v.normal * height;

// 法线贴图双向采样增强动态效果
float2 speed = _Time.x * float2(_WaveSpeed, _WaveSpeed) * _NormalSpeed;
fixed3 bump1 = UnpackNormal(tex2D(_NormalTex, IN.uv_FoamTex.xy + speed)).rgb;
fixed3 bump2 = UnpackNormal(tex2D(_NormalTex, IN.uv_FoamTex.xy - speed)).rgb;
```

双向采样（正方向和反方向）使法线变化更自然，增强波浪的动态效果。

---

## 六、几何着色器（Geometry Shader）

几何着色器位于顶点着色器之后、光栅化之前。它可以把（一个或多个）顶点转变为完全不同的基本图形，生成比原来多得多的顶点。

### 6.1 基础声明

```c++
[maxvertexcount(N)]
void ShaderName (PrimitiveType InputVertexType InputName[NumElements],
                 inout StreamOutputObjectVertexType OutputName) {
    // 几何着色器实现
}
```

- `[maxvertexcount(N)]`：单次调用输出的最大顶点数。出于性能考虑应尽可能小（1-20为最佳，27-40时性能下降到峰值50%）
- 每次调用输出的标量个数 = maxvertexcount × 输出顶点类型结构体中标量个数

### 6.2 输入图元类型

| 前缀 | 输入图元拓扑 |
|:-----|:-----------|
| `point` | 点列表 |
| `line` | 线列表或线条带 |
| `triangle` | 三角形列表或三角形带 |
| `lineadj` | 线列表/带及其邻接图元 |
| `triangleadj` | 三角形列表/带及其邻接图元 |

输入参数以图元类型作为前缀，**必须对应输入装配阶段的图元拓扑类型**。

### 6.3 输出流类型

| 流类型 | 输出 |
|:-------|:-----|
| `PointStream<OutputVertexType>` | 点列表 |
| `LineStream<OutputVertexType>` | 线条带 |
| `TriangleStream<OutputVertexType>` | 三角形带 |

几何着色器输出的图元拓扑类型必须是**线条带或三角形带**。线条列表和三角形列表可借助 `RestartStrip` 输出。

### 6.4 Append 和 RestartStrip

```c++
OutputName.Append(gout);      // 将顶点追加到输出流
OutputName.RestartStrip();    // 结束当前条带，开始新条带
```

`RestartStrip` 用于结束当前的基元条带并开始新的条带。如果当前条带没有足够的顶点以填满基元拓扑结构，末端的不完整基元将被丢弃。

### 6.5 应用示例

**三角形→点（质心）：**

```c++
[maxvertexcount(1)]
void geom(triangle v2g input[3], inout PointStream<g2f> outStream) {
    float4 vertex = (input[0].vertex + input[1].vertex + input[2].vertex) / 3;
    g2f o;
    o.vertex = UnityObjectToClipPos(vertex);
    outStream.Append(o);
}
```

**三角形→线框（三边各两个顶点 = 6个顶点）：**

```c++
[maxvertexcount(6)]
void geom(triangle v2g input[3], inout LineStream<g2f> outStream) {
    for (int i = 0; i < 3; i++) {
        g2f o;
        o.vertex = UnityObjectToClipPos(input[i].vertex);
        outStream.Append(o);
        int next = (i + 1) % 3;
        o.vertex = UnityObjectToClipPos(input[next].vertex);
        outStream.Append(o);
    }
}
```

---

## 七、前向渲染与延迟渲染

### 7.1 渲染路径（Rendering Path）

设置：Project Setting > Player > Other Settings > Rendering Path。主要两种：
- **Forward Rendering（前向渲染）**
- **Deferred Rendering（延迟渲染）**

### 7.2 前向渲染

原理：每个对象每个片元逐光源计算光照。每个光源需要一个 Pass，结果在帧缓冲中混合。

**Unity 前向渲染的处理方式：**
- 场景中最亮的平行光总是逐像素处理
- `Not Important` 光源按逐顶点或球谐函数（SH）处理
- `Important` 光源按逐像素处理
- 超过 Quality Setting 中 Pixel Light Count 数量后，其余光源降级

**两种 Pass：**

| | ForwardBase | ForwardAdd |
|:--|:------------|:-----------|
| 执行次数 | 一次 | 每个额外逐像素光源一次 |
| 光照 | 环境光 + 自发光 + 最重要的平行光 | 其他逐像素光 |
| 阴影 | 默认支持 | 默认不支持 |
| 混合 | 不混合 | `Blend One One` |
| 编译指令 | `#pragma multi_compile_fwdbase` | `#pragma multi_compile_fwdadd` |

**Addition Pass 关键点：**
- 不计算环境光和自发光（否则会重复叠加）
- 使用 `Blend One One` 将结果叠加
- 使用宏区分光源类型：

```c++
#ifdef USING_DIRECTIONAL_LIGHT
    fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz);
#else
    fixed3 worldLightDir = normalize(_WorldSpaceLightPos0.xyz - i.worldPos.xyz);
#endif
```

### 7.3 光照衰减

**纹理查找法（Unity内置）：**

```c++
// 点光源衰减
float3 lightCoord = mul(unity_WorldToLight, float4(i.worldPos, 1)).xyz;
fixed atten = tex2D(_LightTexture0, dot(lightCoord, lightCoord).rr).UNITY_ATTEN_CHANNEL;

// 聚光灯衰减
float4 lightCoord = mul(unity_WorldToLight, float4(i.worldPos, 1));
fixed atten = (lightCoord.z > 0) * tex2D(_LightTexture0, lightCoord.xy / lightCoord.w + 0.5).w
            * tex2D(_LightTextureB0, dot(lightCoord, lightCoord).rr).UNITY_ATTEN_CHANNEL;
```

原理：将世界坐标转到光源空间，计算到光源中心的距离平方，通过查找 `_LightTexture0` 纹理获取衰减值。`UNITY_ATTEN_CHANNEL` 宏获取衰减值所在分量。

**公式计算法：**
```c++
float distance = length(_WorldSpaceLightPos0.xyz - i.worldPos.xyz);
atten = 1.0 / distance;
```

纹理查找法的缺点：纹理大小影响精度、不直观，但性能更好。

### 7.4 延迟渲染

两个 Pass：
1. **G-Buffer Pass**：只计算哪些片元可见，将几何信息（漫反射色、高光色、法线、自发光等）写入多个渲染目标（MRT）
2. **Lighting Pass**：利用 G-Buffer 对每个光源进行光照计算

**G-Buffer 布局（Unity）：**
- RT0：RGB 漫反射颜色，A 未使用
- RT1：RGB 高光反射颜色，A 高光指数
- RT2：RGB 世界空间法线，A 未使用
- RT3：自发光 + lightmap + 反射探针
- 深度缓冲 + 模板缓冲

**延迟渲染缺点：**
- 不支持真正的抗锯齿
- 不能处理半透明物体
- 对显卡有要求（需支持 MRT、Shader Model 3.0+、深度渲染纹理、双面模板缓冲）

---

## 八、阴影系统

### 8.1 阴影映射原理

将相机放到光源位置渲染深度图（Shadow Map），再在正常渲染时比较当前片元在光源空间的深度与 Shadow Map 中的深度：
- 当前深度 > Shadow Map 深度 → 被遮挡 → 处于阴影中

### 8.2 Unity 阴影设置

**物体设置：**
- Cast Shadows（On/Off/Two Sided/Shadows Only）
- Receive Shadows（是否接收阴影）

**光源设置：**
- Shadow Type（No Shadows/Hard Shadows/Soft Shadows）
- Strength、Resolution、Bias、Normal Bias 等

### 8.3 接收阴影的 Shader 代码

```c++
#include "AutoLight.cginc"

struct v2f {
    float4 pos : SV_POSITION;
    float3 worldNormal : TEXCOORD0;
    float3 worldPos : TEXCOORD1;
    SHADOW_COORDS(2)              // 声明阴影坐标（使用TEXCOORD2）
};

v2f vert(a2v v) {
    v2f o;
    o.pos = UnityObjectToClipPos(v.vertex);
    o.worldNormal = UnityObjectToWorldNormal(v.normal);
    o.worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;
    TRANSFER_SHADOW(o);           // 传递阴影坐标
    return o;
}

fixed4 frag(v2f i) : SV_Target {
    // ... 光照计算 ...
    fixed shadow = SHADOW_ATTENUATION(i);  // 获取阴影衰减
    return fixed4(ambient + (diffuse + specular) * atten * shadow, 1.0);
}
```

关键宏：`SHADOW_COORDS(n)`、`TRANSFER_SHADOW(o)`、`SHADOW_ATTENUATION(i)`。

### 8.4 默认 Shadow Caster Pass

Unity 的默认阴影投射Pass：

```c++
struct v2f {
    V2F_SHADOW_CASTER;
};

v2f vert(appdata_base v) {
    v2f o;
    TRANSFER_SHADOW_CASTER_NORMALOFFSET(o)
    return o;
}

float4 frag(v2f i) : SV_Target {
    SHADOW_CASTER_FRAGMENT(i)
}
```

如果物体只有一面投射阴影，可将 Cast Shadows 设为 Two Sided。

---

## 九、Shader优化补充

### 9.1 纹理查找优化

使用纹理存衰减值的弊端：纹理大小影响精度、不直观不方便。但可以显著提高性能（避免实时距离计算）。

### 9.2 光源管理

- 设置 Pixel Light Count（Edit > Project Settings > Quality）控制逐像素光源数量
- 设置光源 Render Mode：Important（逐像素）/ Not Important（逐顶点或SH）/ Auto（Unity自动决定）
- Auto 模式根据光源位置、强度等自动确定重要性
