# Feedback Log

深 feedback 记录 — AI 出错、卡住、输出不符合预期时写入。同类 3 次 → CLAUDE.md 加规则 + settings.json 加 hook。

两阶段:
- **微 feedback** 每 task 必写,位置 `.claude/state/session.md` 的"今日 log"段(见 `.claude/rules/feedback.md`)
- **深 feedback** 踩坑才写,位置本文件

## 格式

### [YYYY-MM-DD] 问题简述
- **场景**: 在做什么时出的问题
- **错误**: 具体错了什么
- **原因**: 为什么会出错
- **修复**: 怎么修的
- **分类**: [模块边界|测试|类型|AI 调用|其他]
- **规则更新**: 无 / CLAUDE.md 加了 xxx / hook 加了 xxx

---

<!-- 由 brainstorm Q8(已知的坑)预填在这里 -->

## Archived

<!-- 深 feedback 归档段:超过 3 个月的 learned 条目沉淀到这里 -->
