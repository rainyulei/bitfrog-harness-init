# Modules

> 本文件是**业务层**,由 bitfrog Phase 2 brainstorm 产出填充。升级模式下**完全重写**。

---

## 模块清单

<!-- 由 brainstorm Q2 产出填充 -->
{模块列表,每个模块含职责一句话}

---

## 边界规则表

<!-- 由 brainstorm Q3 产出填充 -->

| 规则 | 含义 | 检测方式 | 严重度 |
|-----|------|---------|-------|
| {规则 1} | {含义} | hook 自动 / reviewer 审查 | CRITICAL / MAJOR / MINOR |
| ... | ... | ... | ... |

---

## 数据流方向

<!-- 由 brainstorm Q4 产出填充 -->
{单向 / 双向约束,如 A → B → C,C 不能反向调用 A}

---

## 外部依赖规范

<!-- 由 brainstorm Q5 产出填充 -->
{每个外部依赖的超时 / 重试 / 兜底要求,带代码示例}

---

## 项目专属触发器

<!-- 由 brainstorm Q14 产出填充 -->
{哪些目录/模块改动时必须先读某些上下文或必须问用户}

示例:
- "改 auth/* 必须先读 session.ts 的并发注释"
- "改 migration 必须先问用户回滚策略"
- "改 payment/* 必须先读 settlement.ts"
