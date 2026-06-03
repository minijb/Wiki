---
title: LearnOpenGL 学习笔记
type: source
updated: 2026-06-02
tags:
  - opengl
  - glsl
  - graphics
  - rendering
  - gpu
---

# LearnOpenGL 学习笔记

> 基于 LearnOpenGL 教程的实践笔记，涵盖窗口创建、渲染管线、着色器编程、缓冲对象系统和调试方法。

---

## 一、OpenGL 基础概念

### 1.1 OpenGL 本质

OpenGL 是一个**状态机**。所有状态与 `OpenGL Context` 关联。操作 OpenGL 就是操作不同的状态（如绑定缓冲区、操作缓冲区、使用当前缓冲区渲染）。操作方法是通过 state-using functions 来改变 Context 中的状态。

### 1.2 OpenGL 对象（Object）

OpenGL 使用对象来管理状态子集合。可将其类比为 C 结构体：

```c
struct object_name {
    float  option1;
    int    option2;
    char[] name;
};

// OpenGL 上下文可视化为一个大结构体
struct OpenGL_Context {
    object_name* object_Window_Target;
    // ...
};
```

**对象使用模式：**

```c
// 1. 创建对象
unsigned int objectId = 0;
glGenObject(1, &objectId);
// 2. 绑定到上下文
glBindObject(GL_WINDOW_TARGET, objectId);
// 3. 设置当前绑定对象的选项
glSetObjectOption(GL_WINDOW_TARGET, GL_OPTION_WINDOW_WIDTH, 800);
glSetObjectOption(GL_WINDOW_TARGET, GL_OPTION_WINDOW_HEIGHT, 600);
// 4. 解绑（恢复默认）
glBindObject(GL_WINDOW_TARGET, 0);
```

### 1.3 OpenGL 扩展（Extensions）

扩展与硬件绑定，需检测可用性：

```c++
if (GL_ARB_extension_name) {
    // 支持扩展 → 使用现代特性
} else {
    // 不支持 → 回退到旧方式
}
```

---

## 二、渲染缓冲对象

### 2.1 顶点缓冲对象（VBO）

GPU 获取顶点数据前需要先在 GPU 内存中分配 VBO：

```c++
unsigned int VBO;
glGenBuffers(1, &VBO);                                           // 1. 生成缓冲
glBindBuffer(GL_ARRAY_BUFFER, VBO);                              // 2. 绑定为数组缓冲
glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW);  // 3. 拷贝数据到 GPU
```

**BufferData 的 Usage 参数：**

| 模式 | 说明 |
|:-----|:-----|
| `GL_STATIC_DRAW` | 数据设置一次，使用多次 |
| `GL_DYNAMIC_DRAW` | 数据频繁修改，使用多次 |
| `GL_STREAM_DRAW` | 数据设置一次，使用几次 |

### 2.2 链接顶点属性（Vertex Attributes）

指定 OpenGL 如何解释顶点数据：

```c++
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(float), (void*)0);
glEnableVertexAttribArray(0);
```

**参数（按顺序）：**
1. 顶点属性位置（对应 Shader 中 `layout (location = 0)`）
2. 属性分量数量（vec3 = 3）
3. 数据类型
4. 是否归一化
5. 步长（Stride）= 顶点数据之间的间隔
6. 起始偏移量

每个顶点属性从**当前绑定到 `GL_ARRAY_BUFFER` 的 VBO** 中获取数据。`glVertexAttribPointer` 调用时绑定的是哪个 VBO，该属性就关联那个 VBO。

### 2.3 顶点数组对象（VAO）

VAO 存储所有顶点属性配置，通过绑定 VAO 即可恢复状态。

**VAO 存储的内容：**
- `glEnableVertexAttribArray` / `glDisableVertexAttribArray` 调用
- `glVertexAttribPointer` 设置的顶点属性配置
- 与顶点属性关联的 VBO
- **绑定的 EBO**（直接存储在 VAO 中）

![VAO 结构示意](https://learnopengl-cn.github.io/img/01/04/vertex_array_objects.png)

**配置阶段：**

```c++
unsigned int VAO;
glGenVertexArrays(1, &VAO);

glBindVertexArray(VAO);                          // 绑定 VAO
glBindBuffer(GL_ARRAY_BUFFER, VBO);              // 绑定 VBO
glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW);
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(float), (void*)0);
glEnableVertexAttribArray(0);
// VBO 可以在此解绑（VAO 已记录 attribute pointer）
glBindBuffer(GL_ARRAY_BUFFER, 0);
glBindVertexArray(0);                            // 解绑 VAO
```

**绘制阶段：**

```c++
glUseProgram(shaderProgram);
glBindVertexArray(VAO);
glDrawArrays(GL_TRIANGLES, 0, 3);
```

### 2.4 元素缓冲对象（EBO/索引缓冲）

减少重复顶点，通过索引引用顶点：

```c++
unsigned int indices[] = {
    0, 1, 3,  // 第一个三角形
    1, 2, 3   // 第二个三角形
};

unsigned int EBO;
glGenBuffers(1, &EBO);
glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, EBO);
glBufferData(GL_ELEMENT_ARRAY_BUFFER, sizeof(indices), indices, GL_STATIC_DRAW);

// 绘制时使用 glDrawElements
glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, EBO);
glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_INT, 0);
```

**注意**：VAO 也绑定 EBO。一个 VAO 中可以有多个 VBO（保存 attribute）和 EBO（保存索引）。

### 2.5 绑定/解绑顺序

> [!warning] 顺序至关重要
> 错误的解绑顺序会导致渲染黑屏或无输出。

**绑定顺序：** VAO → VBO（设置 attribute）→ EBO → 解绑 VBO → 解绑 VAO → 解绑 EBO

VAO 中存储 attribute pointer 数组，因此可以随时解绑 VBO。绑定的 EBO 直接存储在 VAO 中，解绑 VAO 后才可以安全解绑 EBO。

---

## 三、着色器与 GLSL

### 3.1 GLSL 基础语法

```glsl
#version 330 core
layout (location = 0) in vec3 aPos;

void main() {
    gl_Position = vec4(aPos.x, aPos.y, aPos.z, 1.0);
}
```

**基本类型：** `int`, `float`, `double`, `uint`, `bool`

**向量类型：**

| 类型 | 含义 |
|:-----|:-----|
| `vecn` | n 个 float 分量的默认向量 |
| `bvecn` | n 个 bool 分量 |
| `ivecn` | n 个 int 分量 |
| `uvecn` | n 个 unsigned int 分量 |
| `dvecn` | n 个 double 分量 |

**分量访问方式：** `.xyzw`（位置）或 `.rgba`（颜色）或 `.stpq`（纹理）。支持混合重组：如 `vec3.xxyy`。

### 3.2 着色器间数据传递

GLSL 使用 `in` 和 `out` 关键字。顶点着色器的输出必须与片元着色器的输入匹配（名称和类型一致）。

```glsl
// 顶点着色器
out vec4 vertexColor;
void main() {
    vertexColor = vec4(0.5, 0.0, 0.0, 1.0);
}

// 片元着色器
in vec4 vertexColor;
out vec4 FragColor;
void main() {
    FragColor = vertexColor;
}
```

### 3.3 Uniform（全局变量）

Uniform 是全局的，在所有着色器阶段都可访问。值在程序对象中保持，直到重置或更新。

```glsl
// 片元着色器
uniform vec4 ourColor;
out vec4 FragColor;
void main() {
    FragColor = ourColor;
}
```

```c++
// C++ 端设置
float timeValue = glfwGetTime();
float greenValue = (sin(timeValue) / 2.0f) + 0.5f;
int vertexColorLocation = glGetUniformLocation(shaderProgram, "ourColor");
glUseProgram(shaderProgram);
glUniform4f(vertexColorLocation, 0.0f, greenValue, 0.0f, 1.0f);
```

**glUniform 后缀规则：**

| 后缀 | 含义 |
|:-----|:-----|
| `f` | 函数需要一个 float |
| `i` | 函数需要一个 int |
| `ui` | 函数需要一个 unsigned int |
| `3f` | 函数需要 3 个 float |
| `fv` | 函数需要一个 float 向量/数组 |

### 3.4 纹理采样器（Sampler）

GLSL 中使用 `sampler2D` 等类型访问纹理。uniform 类型的采样器从 C++ 端绑定纹理单元。

```glsl
uniform sampler2D ourTexture;

void main() {
    FragColor = texture(ourTexture, TexCoord);
}
```

### 3.5 顶点着色器输入限制

顶点着色器的输入变量（顶点属性）至少保证 16 个位置可用。每个属性使用 `layout (location = N)` 指定位置。

### 3.6 GLSL 内置变量

| 内置变量 | 说明 |
|:---------|:-----|
| `gl_Position` | 顶点着色器输出：裁剪空间位置（vec4） |
| `gl_PointSize` | 顶点着色器输出：点的大小（float） |
| `gl_FragCoord` | 片元着色器输入：窗口坐标 |
| `gl_FrontFacing` | 片元着色器输入：是否是正面（bool） |
| `gl_FragDepth` | 片元着色器输出：深度值（float） |

### 3.7 纹理采样器类型

| GLSL 类型 | 对应纹理 |
|:----------|:--------|
| `sampler2D` | 2D 纹理 |
| `sampler3D` | 3D 纹理 |
| `samplerCube` | 立方体贴图 |
| `sampler2DShadow` | 2D 阴影贴图 |
| `sampler2DArray` | 2D 纹理数组 |

纹理采样函数：`texture(sampler, coord)` 返回纹理颜色。

### 3.8 常用 GLSL 内置函数

| 函数 | 说明 |
|:-----|:-----|
| `normalize(x)` | 单位化向量 |
| `dot(x, y)` | 点乘 |
| `cross(x, y)` | 叉乘 |
| `reflect(I, N)` | 计算反射方向 |
| `refract(I, N, eta)` | 计算折射方向 |
| `length(x)` | 向量长度 |
| `distance(p0, p1)` | 两点距离 |
| `mix(x, y, a)` | 线性混合（x*(1-a) + y*a） |
| `clamp(x, min, max)` | 钳制到 [min, max] |
| `smoothstep(edge0, edge1, x)` | Hermite 平滑插值 |
| `pow(x, y)` | x 的 y 次幂 |
| `step(edge, x)` | x < edge → 0，否则 1 |

### 3.9 着色器编译流程

```c++
// 1. 创建着色器对象
unsigned int vertexShader = glCreateShader(GL_VERTEX_SHADER);
unsigned int fragmentShader = glCreateShader(GL_FRAGMENT_SHADER);

// 2. 附加源代码并编译
glShaderSource(vertexShader, 1, &vertexShaderSource, NULL);
glCompileShader(vertexShader);

// 3. 检查编译状态
int success;
char infoLog[512];
glGetShaderiv(vertexShader, GL_COMPILE_STATUS, &success);
if (!success) {
    glGetShaderInfoLog(vertexShader, 512, NULL, infoLog);
    std::cout << "ERROR::SHADER::VERTEX::COMPILATION_FAILED\n" << infoLog << std::endl;
}

// 4. 创建着色器程序并链接
unsigned int shaderProgram = glCreateProgram();
glAttachShader(shaderProgram, vertexShader);
glAttachShader(shaderProgram, fragmentShader);
glLinkProgram(shaderProgram);

// 5. 检查链接状态
glGetProgramiv(shaderProgram, GL_LINK_STATUS, &success);
if (!success) {
    glGetProgramInfoLog(shaderProgram, 512, NULL, infoLog);
    std::cout << "ERROR::SHADER::PROGRAM::LINKING_FAILED\n" << infoLog << std::endl;
}

// 6. 使用
glUseProgram(shaderProgram);

// 7. 清理（链接后可以删除着色器对象，程序已包含编译后的代码）
glDeleteShader(vertexShader);
glDeleteShader(fragmentShader);
```

> [!tip] 着色器对象可删除
> 一旦链接到程序对象，着色器对象不再需要。`glDeleteShader` 仅标记删除，实际释放在程序解绑后。

---

## 四、标准化设备坐标（NDC）与帧缓冲

### 4.1 NDC（Normalized Device Coordinates）

屏幕坐标中心为 (0,0)，范围 (-1,-1) 左下角到 (1,1) 右上角。所有超出 NDC 范围的顶点被裁剪。

### 4.2 清除缓冲

```c++
glClearColor(r, g, b, a);  // 设置清屏颜色
glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT | GL_STENCIL_BUFFER_BIT);
```

### 4.3 GPU 管线概览

```
应用 → 顶点着色器 → 曲面细分 → 几何着色器 → 裁剪 → 光栅化 → 片元着色器 → 帧缓冲
```

每个 Pass 指定：
- 物体、相机、MVP 矩阵等
- 帧缓冲（Framebuffer）及输入/输出纹理
- 顶点和片元着色器
- 一个帧缓冲可以绑定多个渲染目标（MRT）

---

## 五、完整的绘制流程示例

以下是一个从顶点数据到屏幕输出的完整流程：

```c++
// === 初始化阶段（只运行一次） ===

// 1. 创建 VAO
unsigned int VAO;
glGenVertexArrays(1, &VAO);
glBindVertexArray(VAO);

// 2. 创建并填充 VBO
unsigned int VBO;
glGenBuffers(1, &VBO);
glBindBuffer(GL_ARRAY_BUFFER, VBO);
glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW);

// 3. 创建并填充 EBO
unsigned int EBO;
glGenBuffers(1, &EBO);
glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, EBO);
glBufferData(GL_ELEMENT_ARRAY_BUFFER, sizeof(indices), indices, GL_STATIC_DRAW);

// 4. 设置顶点属性指针
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(float), (void*)0);
glEnableVertexAttribArray(0);

// 5. 解绑
glBindBuffer(GL_ARRAY_BUFFER, 0);
glBindVertexArray(0);
// 注意：解绑 VAO 之后才解绑 EBO
glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, 0);

// === 渲染循环（每帧运行） ===

glUseProgram(shaderProgram);
glBindVertexArray(VAO);
glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_INT, 0);
glBindVertexArray(0);
```

### 关键要点

1. VAO 存储 attribute pointer 数组和 EBO 绑定，因此可以随时解绑 VBO
2. EBO 直接存储在 VAO 中，解绑 VAO 前不要解绑 EBO
3. 渲染循环中只需 bind VAO → draw → unbind VAO
4. 多个 VAO 可以复用同一个 VBO（只需每个 VAO 设置自己的 attribute pointer）

---

## 六、Shader 调试与错误处理

### 6.1 调试工具

| 工具 | 说明 |
|:-----|:-----|
| **Nsight Graphics** | NVIDIA 图形调试器，支持截帧、Shader 调试 |
| **RenderDoc** | 跨平台图形调试器，支持截帧分析、查看 DrawCall/纹理/Shader 变量 |
| **glGetError()** | 运行时错误码查询 |

### 6.2 错误处理

```c++
// 检查 OpenGL 错误
GLenum err;
while ((err = glGetError()) != GL_NO_ERROR) {
    std::cerr << "OpenGL error: " << err << std::endl;
}
```

### 6.3 常见问题排查

- **黑屏无输出**：检查着色器编译/链接状态、VAO 绑定状态、VBO 数据是否正确
- **着色器编译失败**：使用 `glGetShaderInfoLog` 获取错误信息，检查 GLSL 语法和版本号
- **链接失败**：检查顶点着色器输出与片元着色器输入的变量名是否完全匹配（名称+类型）
- **VAO/EBO 顺序错误**：EBO 必须在 VAO 绑定时绑定，解绑时必须在 VAO 之后
- **glDrawArrays 无输出**：检查是否调用了 `glUseProgram` 和 `glBindVertexArray`

---

## 七、参考资料

- [LearnOpenGL 官网](https://learnopengl.com/)
- [LearnOpenGL CN 中文站](https://learnopengl-cn.github.io/)
- [OpenGL 文档](https://docs.gl/)
- [OpenGL Tutorial](https://www.opengl-tutorial.org/)
- [Khronos OpenGL Wiki](https://www.khronos.org/opengl/wiki/)

---

## 相关页面

- [[OpenGL学习笔记]] — 概念页面
- [[Unity Shader基础]] — HLSL/CG 对比 GLSL
- [[渲染管线理论]] — GPU 管线原理
- [[learn-opengl-notes-摘要]] — 精简摘要
