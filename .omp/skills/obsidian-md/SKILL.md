---
name: obsidian-md
description: Obsidian Markdown 语法规范 — wikilink、callout、frontmatter、标签、嵌入、尖括号包裹等。**在本 Wiki 项目中，任何涉及创建、编写、编辑 .md 文件的操作都必须使用此 skill。** 无论用户说"写个页面"、"创建笔记"、"记录一下"、"更新文档"、"添加概念"、"完善草稿"、"写一篇关于X的wiki"还是任何会输出 markdown 内容的任务，都先触发此 skill 确保正确的 Obsidian 语法：wikilink 替代标准链接、callout 替代纯文本警告、frontmatter 必填、泛型尖括号用反引号包裹等。不要只在用户明确说"Obsidian"时才触发——只要涉及写 markdown 文件就要用
user-invocable: true
---

# Obsidian Markdown 生成器

在本 Wiki 项目中，任何创建或编辑 markdown 文件的操作都应使用此 skill。生成的内容遵循 Obsidian 特有的语法规范。即使任务看起来简单（如"写个页面"、"更新索引"），也要先过一遍 Obsidian 语法规则——wikilink、callout、frontmatter、尖括号包裹等细节容易被忽略。

## 核心语法

### 内部链接 (Wikilink)
```markdown
[[页面名称]]
[[页面名称|显示别名]]
[[页面名称#标题]]
[[页面名称#标题|别名]]
```

### 嵌入 (Transclusion)
```markdown
![[页面名称]]
![[页面名称#标题]]
![[image.png]]
![[audio.mp3]]
```

### Callout (标注块)
类型: note, abstract, info, tip, success, question, warning, failure, danger, bug, example, quote

```markdown
> [!note] 标题
> 内容行
> 可以多行

> [!warning] 注意
> 这是一个警告

> [!tip]- 可折叠
> 默认折叠的内容

> [!info]+ 默认展开
> 默认展开的折叠块
```

### Frontmatter (元数据)
```markdown
---
title: 页面标题
updated: 2026-05-05
tags: [tag1, tag2]
aliases: [别名1, 别名2]
cssclasses: []
---
```

### 标签
```markdown
#tag
#parent/child
#game-dev/rendering
```

### 块引用 (Block Reference)
```markdown
[[页面名^block-id]]
一段文本 ^block-id
```

### 脚注
```markdown
这是一段文本[^1]

[^1]: 脚注内容
```

### 高亮
```markdown
==高亮文本==
```

### Mermaid 图表
```markdown
\```mermaid
graph TD
    A[开始] --> B[结束]
\```
```

### 任务列表
```markdown
- [ ] 未完成
- [x] 已完成
- [-] 取消
```

### 数学公式 (LaTeX)
```markdown
$E = mc^2$

$$
\int_0^\infty e^{-x^2} dx = \frac{\sqrt{\pi}}{2}
$$
```

### 表格
表格与前文之间必须有**一个空行**，否则 Obsidian 无法正确渲染：

```markdown
这是一段文字。

| 列 A | 列 B | 列 C |
|------|------|------|
| 值 1 | 值 2 | 值 3 |
| 值 4 | 值 5 | 值 6 |
```

### 属性列表 (Properties)
在 frontmatter 中定义，也可用行内语法：
```markdown
主题:: 渲染管线
状态:: 草稿
```

### 尖括号包裹

在正文（代码块外部）出现尖括号 `<>` 时，必须用反引号包裹整个表达式。Obsidian 会将裸 `<T>` 解析为 HTML 标签，导致内容从页面消失。

```markdown
❌ IEquatable<T> 是一种泛型接口        → 渲染异常：<T> 被当作 HTML 标签
✅ `IEquatable<T>` 是一种泛型接口      → 正确渲染为行内代码

❌ 使用 Span<int> 切片数据              → 渲染异常
✅ 使用 `Span<int>` 切片数据            → 正确渲染
```

尖括号常见场景：泛型参数（`<T>`、`<int>`、`<string>`）、泛型类型（`List<T>`、`Dictionary<K,V>`）、约束（`where T : struct`）。

> [!warning] 代码块内的尖括号不要包裹
> ```csharp 内部是代码环境，Obsidian 不会将其中 `<T>` 解析为 HTML，保持原样即可。

### 嵌套代码块

当代码块内部需要展示另一个代码块时（例如在 markdown 示例中展示代码块语法），外层必须使用比内层更多的反引号。Markdown 解析器以**连续最长匹配**的反引号串作为围栏：外层反引号数必须严格大于内层。

````markdown
```markdown
这是普通代码块，里面没有嵌套代码块
```
````

如果代码块内容中包含 ```` ``` ````（三个反引号），外层改用 ```` ```` ````（四个反引号）：

`````markdown
````markdown
```csharp
public void Hello() {
    Console.WriteLine("Hello, World!");
}
```
````
`````

更深嵌套以此类推，每多一层就多加一个反引号：

``````markdown
`````markdown
````python
```python
print("三层嵌套")
```
````
`````
``````

> [!tip] 规则：外层反引号数 > 内层反引号数
> 判断标准很简单——看内容中出现的**最长连续反引号串**，外层必须比它多至少一个。例如内容中有 ```` ``` ````，外层至少 ```` ```` ````。不确定时，宁可多写一个。

## 生成规则

1. **所有页面必须有 frontmatter**，至少包含 `title` 和 `updated` 字段
2. **交叉引用优先用 wikilink** `[[ ]]`，不要用 markdown 标准链接
3. **关键概念首次出现时用 wikilink**，即使目标页面尚不存在
4. **重要提示用 callout**，不要用纯文本强调
5. **标签用小写 + 连字符**，如 `#game-dev/rendering`
6. **嵌入图片/文件用 `![[ ]]`** 语法而非 `![alt](url)`
7. **任务清单用 `- [ ]`** 语法
8. **表格前必须有空行**：表格与前一段文字之间保留一个空行，确保 Obsidian 正确渲染
9. **尖括号用反引号包裹**：正文中出现 `<T>`、`<int>` 等泛型尖括号时必须用反引号包裹，防止 Obsidian 将其解析为 HTML 标签导致渲染异常。代码块内部除外。
10. **保持 Obsidian 兼容性**：不使用 HTML 标签，优先使用 Obsidian 原生语法
11. **嵌套代码块外层加反引号**：当代码块内容中包含 ```` ``` ```` 时，外层必须使用 ```` ```` ````（四个反引号）；更深嵌套依此类推，每层多加一个反引号。原则：外层反引号数严格大于内容中最长的连续反引号串。
