# 架构决策记录

> 三层分层:**Architectural**(长期不归档)/ **Operational**(中期季度 review)/ **Learned**(短期月度归档)。
> 写入时机和归档规则见 `.claude/rules/memory.md`。

---

## Architectural

<!-- 长期决策:技术栈 / 架构模式 / 核心 tradeoff / 不可变约束 -->
<!-- 不归档,永远保留全部历史 -->
<!-- 写入时机:brainstorm 对齐的架构选择 / 项目重大转型 / 用户明确说是架构决定 -->

<!-- 由 brainstorm Q13 预填初始架构决策 -->

---

## Operational

<!-- 中期约定:模块用什么 library / pattern 怎么写 / 命名约定 / 工具链选择 -->
<!-- 每季度 review,过时标注 deprecated,不删 -->
<!-- 写入时机:plan 执行中决定的实现模式 / brainstorm 对齐但未到架构级 / reviewer 多次指出同一问题沉淀 -->

---

## Learned

<!-- 短期沉淀:feedback-log 同类 3 次后的短规则 -->
<!-- 每月归档,active 状态超 3 个月未触发 → archived → 归档到 feedback-log#Archived -->
<!-- 规则够普适 → 升级到 CLAUDE.md 或 hook -->

---
