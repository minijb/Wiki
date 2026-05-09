---
name: obsidian-md
description: 生成 Obsidian 风格的 markdown 文件，支持 wikilink、callout、frontmatter、标签、嵌入等特有语法
user-invocable: true
---

# Obsidian Markdown 生成器

当用户需要生成 Obsidian 风格的 markdown 内容时，使用此 skill。生成的内容遵循 Obsidian 特有的语法规范。

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
date: 2026-05-05
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

## 生成规则

1. **所有页面必须有 frontmatter**，至少包含 `title` 和 `date` 字段
2. **交叉引用优先用 wikilink** `[[ ]]`，不要用 markdown 标准链接
3. **关键概念首次出现时用 wikilink**，即使目标页面尚不存在
4. **重要提示用 callout**，不要用纯文本强调
5. **标签用小写 + 连字符**，如 `#game-dev/rendering`
6. **嵌入图片/文件用 `![[ ]]`** 语法而非 `![alt](url)`
7. **任务清单用 `- [ ]`** 语法
8. **表格前必须有空行**：表格与前一段文字之间保留一个空行，确保 Obsidian 正确渲染
9. **保持 Obsidian 兼容性**：不使用 HTML 标签，优先使用 Obsidian 原生语法
