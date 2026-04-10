# Plan

Plan 文件格式 + 生命周期 + 自检规则。

---

## 本质

Plan 是**可移交的执行合约**。不是"列步骤",是"让执行可以被中断、恢复、验证、被另一个 subagent 接手"。

**自包含** —— 另一次 session 的 AI 读了 plan 就能照做,不需要回看对话。

---

## 触发条件

两条件的交集:

### 1. 能写(brainstorm 退出条件已达成)

- 目标具体到"改哪个文件让它做什么"
- context 足够支撑至少 1 个可行方案
- 能写出让另一个人照做的 plan

### 2. 需要写(任一 ✓ 即触发)

- 跨 session / 预期 compact 会发生
- 需要 dispatch 给 subagent 执行
- 改动风险大,需要每步 checkpoint 可回滚
- 步骤多到一次 session 做不完
- 多人协作,plan 是交接物

### 决策表

| 能写 | 需要写 | 做什么 |
|-----|-------|-------|
| ✅ | ✅ | 写 plan |
| ✅ | ❌ | 跳过 plan,直接写代码 |
| ❌ | — | brainstorm 没做够,回去 |

---

## 文件位置和命名

`.claude/plans/YYYY-MM-DD-{slug}.md`

- 日期是创建日期
- slug 是短描述(lowercase + hyphen,如 `auth-refactor`)
- **不**被 gitignore —— plan 是可移交合约,必须 commit 进版本控制

---

## Header(强制格式)

每个 plan 文件必须以这个格式开头:

```markdown
# {Feature Name} Implementation Plan

**Goal:** {One sentence describing what this builds}

**Architecture:** {2-3 sentences about approach}

**Tech Stack:** {Key technologies/libraries involved}

---
```

---

## Task 结构(每 task 11 个 step)

```markdown
### Task N: {Component Name}

**Files:**
- Create: `exact/path/to/file.ext`
- Modify: `exact/path/to/existing.ext:lines`
- Test: `tests/exact/path/to/file.test.ext`

- [ ] **Step 1: Write failing test**(如适用)
- [ ] **Step 2: Run test**(expected: FAIL)
- [ ] **Step 3: Implement**
- [ ] **Step 4: Run test**(expected: PASS)
- [ ] **Step 5: Commit**
- [ ] **Step 6: Dispatch reviewer-spec**(参考 rules/review.md)
- [ ] **Step 7: Fix spec gaps**(loop until PASS)
- [ ] **Step 8: Dispatch reviewer-quality**
- [ ] **Step 9: Fix quality issues**(loop until PASS)
- [ ] **Step 10: 微 feedback 追加到 session.md**
- [ ] **Step 11: 文档同步检查**(对照 rules/docs.md#B)
```

> **重要:Step 1-5 是"有测试场景"的示例,不是所有 task 都必须这个形态。** 纯重构 / 纯文档 / 纯配置 task 的前 5 个 step 可能完全不同(比如 "Step 1: 读现有实现" / "Step 2: 设计新结构" / "Step 3: 应用改动" / "Step 4: 跑 verify" / "Step 5: Commit")。bitfrog 不强制 TDD。
>
> **但以下 Step 无论 task 类型都是强制的:**
> - **Step 6-9**: Two-stage review loop(reviewer-spec → fix → reviewer-quality → fix)
> - **Step 10**: 微 feedback 追加到 session.md(每 task 必写)
> - **Step 11**: 文档同步检查(对照 rules/docs.md#B)

---

## No Placeholder 红线

**以下 pattern 严禁出现在 plan 里**(Hook 5 会自动检测并阻断):

```
禁止列表:
- TBD / TODO / implement later / fill in details
- Add appropriate error handling / add validation / handle edge cases(不具体)
- Write tests for the above(没实际 test code)
- Similar to Task N(必须重复 code,reader 可能 out of order)
- 没 code block 的 implement step(描述做什么但不 show 怎么做)
- 引用未在任何 task 里定义的 types / functions / methods
```

**每个 step 必须包含执行者需要的实际内容**:精确 file paths / 完整 code blocks / 精确 commands with expected output。

---

## Type Consistency 自检

写完 plan 后必须自查:**后面 task 用到的函数名、方法签名、属性名,和前面 task 定义的是否一致?**

`clearLayers()` 在 Task 3 和 `clearFullLayers()` 在 Task 7 是 bug。

---

## Plan 生命周期

```
brainstorm 退出 → 判断是否需要 plan(触发条件)
  ↓ 需要
写 plan 文件(.claude/plans/YYYY-MM-DD-{slug}.md)
  → TaskCreate 镜像 plan 的 checkbox(task 级追踪)
  ↓
执行时:
  勾 checkbox + 更新 TaskCreate state + 写 session.md
  ↓
遇到 blocker:
  stop + escalate(不硬刚)→ 报告 BLOCKED / NEEDS_CONTEXT
  ↓
所有 checkbox 勾完:
  → Finishing Flow(rules/workflow.md)
  ↓
归档:
  plan 文件保留在 .claude/plans/ 作为历史记录(不删)
```

**Plan 是 git tracked 的**:`.claude/plans/` 不被 gitignore。plan 是项目历史的一部分,要和代码一起版本控制。

---

## 和其他 rules 的关系

- **rules/workflow.md** — plan 在完整流程图的 brainstorm 之后
- **rules/brainstorm.md** — brainstorm 退出条件 = plan 的输入
- **rules/review.md** — plan 的 Step 6-9 引用 review loop
- **rules/feedback.md** — plan 的 Step 10 引用微 feedback
- **rules/docs.md** — plan 的 Step 11 引用文档同步
- **rules/subagent.md** — plan 的执行可以 dispatch 给 subagent
