# 02 — 创建四个 Skill 文件

## 目标

在 `.claude/skills/` 下创建四个核心技能定义文件，每个通过 YAML frontmatter 定义，可通过 `/skill-name` 调用。

## Skill 清单

| 文件 | 命令 | 职责 |
|------|------|------|
| `ingest.md` | `/ingest` | 阅读来源 → 与人类讨论要点 → 写摘要页面 → 更新 index.md → 更新相关实体/概念页面 → 追加 log.md |
| `query.md` | `/query` | 先读 index.md 定位，再调 QMD 精准检索 → 阅读并综合 → 生成答案（支持 markdown/表格/幻灯片/图表） → 可选归档回 wiki/ |
| `lint.md` | `/lint` | QMD 全局搜索检孤立页面/断链 → 检查矛盾声明 → 陈旧内容 → 缺失交叉引用 → 数据缺口 → 建议新来源/新问题 |
| `refine.md` | `/refine` | 读取 drafts/ 中不完善笔记 → 指出遗漏、模糊、矛盾之处 → 与用户讨论 → 根据讨论优化 → 完善后标记可归档 |

## 关键设计决策

- **Skills 承载工作流，CLAUDE.md 承载约定**：四种操作是"行为"，用可调用的 Skill 定义
- `/query` 和 `/lint` 依赖 QMD 作为主要检索手段
- 每个 Skill 需包含完整工作流步骤（含失败处理和边界情况）

## 验证

1. 确认四个 `.md` 文件存在于 `.claude/skills/` 目录下
2. 确认每个文件包含 YAML frontmatter（name, description）
3. 确认 `/ingest` 流程完整覆盖：阅读→讨论→写摘要→更新index→更新wiki→写log
4. 确认 `/query` 和 `/lint` 中已集成 QMD 调用步骤
5. 确认 `/refine` 包含讨论迭代流程和归档条件判定
6. 分别用 `/ingest`、`/query`、`/lint`、`/refine` 命令测试调用是否成功
