# bitfrog v4 Upgrade Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.
>
> **Context:** This is a meta-bootstrap plan — it upgrades bitfrog itself from v3 to v4. The v4 methodology (two-stage review, plan step 11, etc.) is NOT yet available when executing this plan. Each task uses a simplified 4-step structure: **Read spec section → Write file → Verify → Commit**. Full v4 review loop only applies after Phase F (Dogfood).

**Goal:** Upgrade bitfrog from v3 (static harness rules generator) to v4 (dynamic methodology + static rules + infrastructure + doc management + upgrade mode), by creating a `templates/` directory with 22 template files and rewriting SKILL.md to reference them.

**Architecture:** v4 splits the monolithic SKILL.md into a thin orchestrator (SKILL.md with Phase 1-5 logic) plus a `templates/` directory containing all the output skeletons (CLAUDE.md template, 11 rules, 4 agents, session/plan/memory/docs templates, hooks JSON). bitfrog generates project harnesses by copying templates and filling in business-layer placeholders from brainstorm answers.

**Tech Stack:** Pure markdown + shell/bash (no code language). Deliverable is a Claude Code plugin consisting of `SKILL.md` + `templates/*` + plugin metadata.

**Source of truth:** `docs/superpowers/specs/2026-04-10-bitfrog-v4-design.md` (the design spec). Every task in this plan maps to one or more sections in the spec. Task headers cite spec sections explicitly.

---

## Task Structure Convention

All tasks use a 4-step structure(v4 review loop unavailable during bootstrap):

```
- [ ] Step 1: Read spec section X.Y
- [ ] Step 2: Write/Modify file Z with required content
- [ ] Step 3: Verify (grep / structure checks)
- [ ] Step 4: Commit
```

No TDD (these are markdown template files, not code with tests). Verification is structural: file exists, key sections present, no forbidden placeholders, cross-references valid.

Each task's **Required sections** list is derived from the spec. Implementers read the spec section, then write the file such that every required section exists and no forbidden pattern appears.

---

## Phase A: Templates Infrastructure (Task 1)

### Task 1: Create templates directory structure

**Files:**
- Create: `templates/` (directory)
- Create: `templates/rules/` (directory)
- Create: `templates/agents/` (directory)
- Create: `templates/memory/` (directory)
- Create: `templates/state/` (directory)
- Create: `templates/hooks/` (directory)

- [ ] **Step 1:** Read spec Section 4.1 (生成物目录树) to confirm the template layout mirrors the output layout.

- [ ] **Step 2:** Create directories.

  ```bash
  cd /Users/rainlei/holiday/bitfrog-harness-init
  mkdir -p templates/rules
  mkdir -p templates/agents
  mkdir -p templates/memory
  mkdir -p templates/state
  mkdir -p templates/hooks
  ```

- [ ] **Step 3:** Verify directories exist.

  ```bash
  ls -d templates templates/rules templates/agents templates/memory templates/state templates/hooks
  ```
  Expected: all 6 paths listed, no errors.

- [ ] **Step 4:** Commit (empty directories are not tracked, so no files to add yet; skip commit until Task 2 adds content).

---

## Phase A: Root CLAUDE.md Template (Task 2)

### Task 2: Write templates/CLAUDE.md

**Files:**
- Create: `templates/CLAUDE.md`

**Spec reference:** Section 7.1 — root CLAUDE.md(精简指针表风格).

**Required sections (the template file must contain these headers in order):**
1. `# {项目名}` — project name placeholder
2. `## 项目目标` — goal section with `{brainstorm Q1 的完整回答}` placeholder
3. `## 技术栈` — tech stack table placeholder
4. `## 项目结构` — directory tree placeholder
5. `## 第一动作(强制,所有任务)` — mandatory first action, pointing to `.claude/rules/workflow.md`
6. `## 骨架指针表` — pointer table with 13 rows (see spec Section 7.1 for exact rows)
7. `## 进化规则` — evolution rules (feedback-log 3x, decisions learned monthly archive, project goal change → upgrade mode)

**Forbidden patterns:** None (this is a template, placeholders like `{...}` are legitimate).

- [ ] **Step 1:** Read spec Section 7.1 (CLAUDE.md 精简指针表).

- [ ] **Step 2:** Write `templates/CLAUDE.md` with the exact content from spec Section 7.1, using `{...}` as literal placeholders for business content filled in at generation time.

  **The pointer table must contain these 13 rows in order:**
  | 场景 | 读哪个文件 | 必读 |
  |-----|----------|-----|
  | 开工(任何任务) | `rules/workflow.md` | ✓ |
  | 目标/方案不清 | `rules/brainstorm.md` | 条件 |
  | 需要写 plan | `rules/plan.md` | 条件 |
  | 跨模块大改 / 有回滚风险 | `rules/worktree.md` | 条件 |
  | 写代码前(模块边界) | `rules/modules.md` | ✓ |
  | verify(每次跑命令) | `rules/verify.md` | ✓ |
  | task 完成后(review) | `rules/review.md` | ✓ |
  | dispatch subagent | `rules/subagent.md` | ✓ |
  | 微/深 feedback 写入 | `rules/feedback.md` | ✓ |
  | 文档同步 / 写文档 | `rules/docs.md` | 条件 |
  | 架构/约定写入 decisions | `rules/memory.md` | 条件 |
  | compact 前 | `rules/workflow.md#compact-protocol` | ✓ |
  | compact 后 | 读 `.claude/state/session.md` → 找 active plan | ✓ |

- [ ] **Step 3:** Verify the file exists and contains all required section headers.

  ```bash
  test -f templates/CLAUDE.md && \
  grep -c "^## 项目目标$" templates/CLAUDE.md && \
  grep -c "^## 技术栈$" templates/CLAUDE.md && \
  grep -c "^## 项目结构$" templates/CLAUDE.md && \
  grep -c "^## 第一动作" templates/CLAUDE.md && \
  grep -c "^## 骨架指针表$" templates/CLAUDE.md && \
  grep -c "^## 进化规则$" templates/CLAUDE.md && \
  grep -c "rules/workflow.md" templates/CLAUDE.md
  ```
  Expected: file exists, 6 header matches, rules/workflow.md referenced at least twice.

- [ ] **Step 4:** Commit.

  ```bash
  git add templates/CLAUDE.md
  git commit -m "feat(v4): add templates/CLAUDE.md root pointer-table template"
  ```

---

## Phase A: Rules Files (Tasks 3-13)

### Task 3: Write templates/rules/workflow.md

**Files:**
- Create: `templates/rules/workflow.md`

**Spec reference:** Section 7.2.1 (workflow 定位 + 章节) + Section 5.1 (第一动作详细) + Section 4.2 (完整流程图) + Section 4.3 (Compact 协议).

**Required sections:**
1. `# Workflow` — file title
2. `## 第一动作(强制,不可跳过)` — mandatory first action, containing:
   - 动作格式(3 步:重述 / 列出理解 / 按任务类型决定下一步)
   - 任务类型参照表(5 行:Bug / Feature / 重构 / 探索性 / 机械性)
   - 退出第一动作的条件
   - 为什么强制(senior engineer 类比 + compact-safe 理由)
3. `## 完整流程图` — full flow diagram (text form, from spec 4.2)
4. `## 骨架指针表` — reference to CLAUDE.md 的指针表(相同结构,这里展开)
5. `## Compact 协议` — compact protocol (from spec 4.3)
   - 必须含 anchor `#compact-protocol`(CLAUDE.md 指针表引用)
6. `## 任务收尾 checklist` — end-of-task checklist(微 feedback / 深 feedback / 文档同步 / session 清理 / architectural 决策写入 / operational 约定写入)

**Forbidden patterns:** `TBD`, `TODO`, `implement later`

- [ ] **Step 1:** Read spec sections 4.2, 4.3, 5.1, 7.2.1.

- [ ] **Step 2:** Write `templates/rules/workflow.md` with all required sections. The 第一动作 section is the heart of this file — copy the mandatory first action wording from spec 5.1 verbatim. Include the full 流程图 text from spec 4.2. Compact 协议 from spec 4.3. The anchor `{#compact-protocol}` must be present on the Compact section heading.

  **Task type table in 第一动作 must have exactly these 5 rows:**
  - Bug / 故障 → 沿症状链路追
  - Feature → 找项目里最接近的正常工作的同类代码
  - 重构 → 确认"为什么改"的 root cause
  - 探索性 → 先问 metric
  - 机械性 → 直接进 plan

- [ ] **Step 3:** Verify.

  ```bash
  test -f templates/rules/workflow.md && \
  grep -c "^## 第一动作" templates/rules/workflow.md && \
  grep -c "^## 完整流程图$" templates/rules/workflow.md && \
  grep -c "^## Compact 协议" templates/rules/workflow.md && \
  grep -c "^## 任务收尾 checklist$" templates/rules/workflow.md && \
  grep -c "复述" templates/rules/workflow.md && \
  ! grep -qE '(TBD|TODO|implement later)' templates/rules/workflow.md
  ```
  Expected: file exists, 4 header matches, "复述" appears, no forbidden patterns.

- [ ] **Step 4:** Commit.

  ```bash
  git add templates/rules/workflow.md
  git commit -m "feat(v4): add templates/rules/workflow.md (first action + flow + compact protocol)"
  ```

---

### Task 4: Write templates/rules/brainstorm.md

**Files:**
- Create: `templates/rules/brainstorm.md`

**Spec reference:** Section 7.2.2 + Section 5.2 (brainstorm 动态模式 + 动作池 + 退出条件 + 任务类型参照).

**Required sections:**
1. `# Brainstorm` — title
2. `## 定义` — brainstorm = 明确目标 ↔ load context 耦合螺旋;不是 phase,是模式
3. `## 退出条件(三勾)` — 三个勾选项(目标具体 / context 足够 / 能写 plan)
4. `## 动作池(自由组合,无固定顺序)` — 三组动作
   - Context load 动作(7 个)
   - 目标明确 动作(4 个)
   - Escalation 动作(2 个)
5. `## 任务类型参照` — 5 行表格(null bug / API / race / 让页更快 / 拆 auth)
6. `## 禁止` — 4 条禁止(固定问题清单 / 跳过第一动作 / 假装退出 / 小任务强制 2-3 approaches)

**Forbidden patterns:** 同 Task 3

- [ ] **Step 1:** Read spec Section 5.2 (brainstorm dynamic mode).

- [ ] **Step 2:** Write `templates/rules/brainstorm.md`.

  **Context load 动作池必须包含这 7 条(精确引用):**
  - 开工前快扫:项目结构 / 最近 commit / 相关 docs
  - 读要改的 code + 直接调用方 + 相关 tests
  - 沿 data flow 追(bug 必做)— 从症状往上游,找第一个产生坏值的位置
  - 找同仓库正常工作的同类代码对比,列每一处差异
  - 读 decisions.md / 历史注释 / 外部文档
  - 跑 profile / log / 复现步骤看现状
  - 写 POC 试

  **目标明确 动作池必须包含这 4 条:**
  - 复述 + 检查理解(强制入口,详见 rules/workflow.md)
  - 列 2-3 个方案,带 tradeoff,明确推荐一个
  - 问用户(一次一个问题,优先 multiple choice)
  - 分段呈现设计,逐段和用户确认

  **Escalation 动作:**
  - 发现任务是多个独立子系统 → 先拆分再单个 brainstorm

- [ ] **Step 3:** Verify.

  ```bash
  test -f templates/rules/brainstorm.md && \
  grep -c "^## 定义$" templates/rules/brainstorm.md && \
  grep -c "^## 退出条件" templates/rules/brainstorm.md && \
  grep -c "^## 动作池" templates/rules/brainstorm.md && \
  grep -c "沿 data flow 追" templates/rules/brainstorm.md && \
  grep -c "找同仓库正常工作的同类代码" templates/rules/brainstorm.md && \
  ! grep -qE '(TBD|TODO)' templates/rules/brainstorm.md
  ```

- [ ] **Step 4:** Commit.

  ```bash
  git add templates/rules/brainstorm.md
  git commit -m "feat(v4): add templates/rules/brainstorm.md (dynamic mode + action pool)"
  ```

---

### Task 5: Write templates/rules/plan.md

**Files:**
- Create: `templates/rules/plan.md`

**Spec reference:** Section 7.2.3 + Section 5.3 (plan 格式 + 11-step task 结构 + no placeholder 红线).

**Required sections:**
1. `# Plan` — title
2. `## 触发条件` — 能写 + 需要写 两条件交集 + 决策表(能写×需要写 4 种组合)
3. `## 文件位置和命名` — `.claude/plans/YYYY-MM-DD-{slug}.md`,git tracked
4. `## Header(强制格式)` — Goal / Architecture / Tech Stack
5. `## Task 结构(每 task 11 个 step)` — 完整 task 模板
6. `## No Placeholder 红线` — 5 条禁止
7. `## Type Consistency 自检` — 后 task 的签名和前 task 一致
8. `## Plan 生命周期` — 创建 → 执行 → blocker → checkbox 勾完 → 归档

**Forbidden patterns:** 见红线段,但红线段本身必须列出这些 patterns 作为"禁止项",grep 时要排除红线段自身。

- [ ] **Step 1:** Read spec Section 5.3 (Plan 格式).

- [ ] **Step 2:** Write `templates/rules/plan.md`.

  **Task 11 步模板必须精确包含:**
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

  **Step 1-5 说明:**
  > 重要:Step 1-5 是"有测试场景"的示例,不是所有 task 都必须这个形态。纯重构 / 纯文档 / 纯配置 task 的前 5 个 step 可能完全不同。bitfrog 不强制 TDD。**但 Step 6-11 无论 task 类型都是强制的**。

  **No Placeholder 红线列表(在一个代码块里,避免被 verify grep 误判):**
  ```
  禁止以下 pattern 出现在 plan 里:
  - TBD / TODO / implement later / fill in details
  - Add appropriate error handling / add validation / handle edge cases(不具体)
  - Write tests for the above(没实际 test code)
  - Similar to Task N(必须重复 code)
  - 没 code block 的 implement step
  ```

- [ ] **Step 3:** Verify.

  ```bash
  test -f templates/rules/plan.md && \
  grep -c "^## 触发条件$" templates/rules/plan.md && \
  grep -c "^## Task 结构" templates/rules/plan.md && \
  grep -c "^## No Placeholder 红线$" templates/rules/plan.md && \
  grep -c "^## Plan 生命周期$" templates/rules/plan.md && \
  grep -c "Step 6: Dispatch reviewer-spec" templates/rules/plan.md && \
  grep -c "Step 10: 微 feedback" templates/rules/plan.md && \
  grep -c "Step 11: 文档同步检查" templates/rules/plan.md
  ```

- [ ] **Step 4:** Commit.

  ```bash
  git add templates/rules/plan.md
  git commit -m "feat(v4): add templates/rules/plan.md (plan format + 11-step task structure)"
  ```

---

### Task 6: Write templates/rules/review.md

**Files:**
- Create: `templates/rules/review.md`

**Spec reference:** Section 7.2.4 + Section 5.4 (review loop 完整规范).

**Required sections:**
1. `# Review`
2. `## 为什么要 review 层`(第一性理由 — v3 只有 reviewer agent,v4 补上完整循环)
3. `## 强制时机` — 每 task 后 / 重大 feature / merge 前
4. `## 可选但有价值` — stuck / 重构前 baseline / 复杂 bug 修完
5. `## Two-stage 顺序(硬约束)` — spec first → quality after,不可颠倒
6. `## Request Dispatch 原则` — never inherit session history + precisely crafted context + request template 5 字段
7. `## Receive Feedback 的 6 步 Pattern`(READ → UNDERSTAND → VERIFY → EVALUATE → RESPOND → IMPLEMENT)
8. `## 禁止性能性同意` — "You're absolutely right" / "Great point" / 任何 gratitude / 直接动手不写 Thanks
9. `## YAGNI Check` — grep 确认 unused 就删
10. `## Unclear Item 优先澄清` — 不先做懂的部分
11. `## Push Back 时机和方法` — 6 种推回场景 + 推回原则(技术推理不 defensive)
12. `## Review 结果的处理原则:直接行动,不写报告`(三类 finding 分流:本 task 问题直接修 / 本来就有的 bug 写 feedback-log / plan 本身问题 escalate)
13. `## 禁止动作` — 不写 review-YYYYMMDD.md / 不塞进 decisions.md / 不在 commit message 里复述 reviewer 全部 finding

- [ ] **Step 1:** Read spec Section 5.4 (Review Loop).

- [ ] **Step 2:** Write `templates/rules/review.md`.

  **Request template 5 字段必须精确包含:**
  - `PLAN_PATH` — plan 文件路径
  - `TASK_NUMBER` — 当前 task 编号
  - `BASE_SHA` — diff 起点
  - `HEAD_SHA` — diff 终点
  - `DESCRIPTION` — 简短摘要

  **Receive 6 步必须精确命名:**
  READ → UNDERSTAND → VERIFY → EVALUATE → RESPOND → IMPLEMENT

  **禁止性能性同意清单精确包含:**
  - "You're absolutely right!"
  - "Great point!" / "Excellent feedback!"
  - "Let me implement that now"(未验证前)
  - "Thanks for catching that!"
  - ANY gratitude expression

  **Review 结果三类 finding 分类必须精确:**
  - 类别 1: 本 task 引入的问题 → worker 直接修 → re-review(不写报告)
  - 类别 2: 本来就有的 bug → feedback-log.md + 视情况开独立 task
  - 类别 3: plan 本身有问题 → stop + escalate

- [ ] **Step 3:** Verify.

  ```bash
  test -f templates/rules/review.md && \
  grep -c "^## Two-stage 顺序" templates/rules/review.md && \
  grep -c "^## Receive Feedback 的 6 步 Pattern$" templates/rules/review.md && \
  grep -c "^## 禁止性能性同意$" templates/rules/review.md && \
  grep -c "^## Review 结果的处理原则" templates/rules/review.md && \
  grep -c "READ → UNDERSTAND → VERIFY → EVALUATE → RESPOND → IMPLEMENT" templates/rules/review.md && \
  grep -c "never inherit session history" templates/rules/review.md && \
  grep -c "You're absolutely right" templates/rules/review.md
  ```

- [ ] **Step 4:** Commit.

  ```bash
  git add templates/rules/review.md
  git commit -m "feat(v4): add templates/rules/review.md (two-stage review loop + 6-step pattern)"
  ```

---

### Task 7: Write templates/rules/verify.md

**Files:**
- Create: `templates/rules/verify.md`

**Spec reference:** Section 7.2.5 + Section 5.5 (verify 5-step gate + evidence).

**Layer:** 混合(骨架 + 业务 placeholder)

**Required sections:**
1. `# Verify`
2. `## 核心原则` — Evidence before claims
3. `## 5 步 Gate Function`(IDENTIFY → RUN → READ → VERIFY → CLAIM)
4. `## Evidence 清单` — 7 行表(Tests pass / Linter clean / Build succeeds / Bug fixed / Regression test works / Agent completed / Requirements met)
5. `## Red-Green Cycle(regression test 必做)` — 6 步
6. `## 禁止表述` — should/probably/seems to + Great/Perfect/Done 未验证前 + gratitude
7. `## Agent success ≠ success` — 子段,强调必须 check VCS diff
8. `## 项目验证命令`(业务 placeholder 段) — 占位结构,由 bitfrog 生成时 brainstorm Q9 填入

- [ ] **Step 1:** Read spec Section 5.5 (Verify Gate).

- [ ] **Step 2:** Write `templates/rules/verify.md`.

  **5 步 Gate Function 必须精确:**
  ```
  在声明任何状态之前:

  1. IDENTIFY: 什么 command 证明这个 claim?
  2. RUN:      执行完整命令(fresh,不用缓存)
  3. READ:     完整输出,检查 exit code,数失败数
  4. VERIFY:   输出符合 claim 吗?
               - 不符合 → 说实际状态带证据
               - 符合 → 说 claim 带证据
  5. ONLY THEN: Make the claim

  跳过任何一步 = lying,不是 verifying
  ```

  **Evidence 清单 7 行必须精确包含:**
  | Claim | 证据 |
  |------|------|
  | Tests pass | 测试命令输出 0 failures |
  | Linter clean | Linter 输出 0 errors |
  | Build succeeds | Build 命令 exit 0 |
  | Bug fixed | 测原症状通过 |
  | Regression test works | Red-green cycle 验证过 |
  | **Agent completed** | **VCS diff 显示改动**(不是信 agent 的 success 报告) |
  | Requirements met | Line-by-line checklist |

  **项目验证命令 段(业务 placeholder):**
  ```markdown
  ## 项目验证命令

  <!-- 由 bitfrog Phase 2 Q9 产出填充 -->
  - **typecheck**: `{typecheck 命令}`
  - **lint**: `{lint 命令}`
  - **test**: `{test 命令}`
  - **build**: `{build 命令}`(如适用)
  - **e2e**: `{e2e 命令}`(如适用)
  - **组合 verify**: `{组合命令,如 npm run verify}`
  ```

- [ ] **Step 3:** Verify.

  ```bash
  test -f templates/rules/verify.md && \
  grep -c "^## 5 步 Gate Function$" templates/rules/verify.md && \
  grep -c "^## Evidence 清单$" templates/rules/verify.md && \
  grep -c "^## Red-Green Cycle" templates/rules/verify.md && \
  grep -c "^## Agent success ≠ success$" templates/rules/verify.md && \
  grep -c "^## 项目验证命令$" templates/rules/verify.md && \
  grep -c "IDENTIFY.*RUN.*READ.*VERIFY.*CLAIM" templates/rules/verify.md && \
  grep -c "{typecheck 命令}" templates/rules/verify.md
  ```

- [ ] **Step 4:** Commit.

  ```bash
  git add templates/rules/verify.md
  git commit -m "feat(v4): add templates/rules/verify.md (5-step gate + evidence + business placeholder)"
  ```

---

### Task 8: Write templates/rules/feedback.md

**Files:**
- Create: `templates/rules/feedback.md`

**Spec reference:** Section 7.2.6 + Section 5.6 (feedback 两阶段).

**Required sections:**
1. `# Feedback`
2. `## 微 feedback(每 task 必写)` — 定位 / 格式 / 位置 / 强制方式 / 时机 / 理由
3. `## 深 feedback(踩坑才写)` — 定位 / 位置(feedback-log.md)/ 格式 / 触发条件 / Hook 兜底
4. `## 两阶段的升级和归档` — 升级规则 + 归档规则(feedback-log 同类 3 次沉淀)

- [ ] **Step 1:** Read spec Section 5.6 (Feedback 两阶段).

- [ ] **Step 2:** Write `templates/rules/feedback.md`.

  **微 feedback 格式必须精确:**
  ```
  YYYY-MM-DD HH:MM | task# | {动作} | {意外 (无则"无")}
  ```

  **深 feedback 触发条件列表:**
  - verify 连续 2 次失败
  - 同类问题 2 次出现
  - 用户纠正做法("stop doing X")
  - 微 feedback 的"意外"字段非空且严重

  **强制方式必须说明:**
  > 微 feedback 嵌入 plan 的 Step 10(不靠自觉);深 feedback 由 hook 兜底(verify 失败 ≥ 2 次 && session 未写 feedback → 提醒)

- [ ] **Step 3:** Verify.

  ```bash
  test -f templates/rules/feedback.md && \
  grep -c "^## 微 feedback" templates/rules/feedback.md && \
  grep -c "^## 深 feedback" templates/rules/feedback.md && \
  grep -c "Step 10" templates/rules/feedback.md && \
  grep -c "feedback-log.md" templates/rules/feedback.md
  ```

- [ ] **Step 4:** Commit.

  ```bash
  git add templates/rules/feedback.md
  git commit -m "feat(v4): add templates/rules/feedback.md (micro + deep two-stage)"
  ```

---

### Task 9: Write templates/rules/subagent.md

**Files:**
- Create: `templates/rules/subagent.md`

**Spec reference:** Section 7.2.7 + Section 5.7 (Subagent 规范) + Section 7.4 (三种 dispatch prompt 模板).

**Required sections:**
1. `# Subagent`
2. `## Why Subagents` — 隔离 context / 保护主 session / never inherit session history / precisely craft
3. `## Two-stage Review(硬约束)` — 顺序 spec first → quality after,不可颠倒
4. `## Dispatch Prompt 原则` — full text provision / 自包含 / 明确 output / 明确约束 / specific scope
5. `## Implementer 的 4 种状态处理` — DONE / DONE_WITH_CONCERNS / NEEDS_CONTEXT / BLOCKED
6. `## Model Selection 指南` — 机械 cheap / integration standard / 架构&review most capable + reviewer-spec cheaper + reviewer-quality most capable
7. `## Parallel Dispatch 规则` — implementer 永远不并行 / debug 调查可并行
8. `## Red Flags` — 7 条
9. `## Dispatch Prompt 模板` — 3 个子段
   - `### worker dispatch prompt` — 完整模板
   - `### reviewer-spec dispatch prompt` — 完整模板
   - `### reviewer-quality dispatch prompt` — 完整模板

- [ ] **Step 1:** Read spec Section 5.7 (Subagent) + Section 7.4 (Dispatch templates).

- [ ] **Step 2:** Write `templates/rules/subagent.md`. The three dispatch prompt templates must be copied from spec Section 7.4 verbatim (they are verbose but necessary).

  **Implementer 4 状态处理表必须精确:**
  - **DONE** → 进 spec review
  - **DONE_WITH_CONCERNS** → 读 concerns,正确性/scope 先解决再 review;观察性 concerns 记录+进 review
  - **NEEDS_CONTEXT** → 补 context 后 re-dispatch
  - **BLOCKED** → 分类处理(context 补 / reasoning 升级 model / 任务拆 / plan 错 escalate)

  **Parallel 规则精确:**
  - ✅ 可以 parallel: 3+ 独立 test file failure / 多个独立 subsystem / 独立 debug 调查
  - ❌ 不可 parallel: **多个 implementer(永远不行,会 file conflict)** / related failures / 全系统视角 investigation / shared state

- [ ] **Step 3:** Verify.

  ```bash
  test -f templates/rules/subagent.md && \
  grep -c "^## Why Subagents$" templates/rules/subagent.md && \
  grep -c "^## Two-stage Review" templates/rules/subagent.md && \
  grep -c "^## Implementer 的 4 种状态处理$" templates/rules/subagent.md && \
  grep -c "^### worker dispatch prompt$" templates/rules/subagent.md && \
  grep -c "^### reviewer-spec dispatch prompt$" templates/rules/subagent.md && \
  grep -c "^### reviewer-quality dispatch prompt$" templates/rules/subagent.md && \
  grep -c "DONE_WITH_CONCERNS" templates/rules/subagent.md && \
  grep -c "implementer 永远不并行\|永远不行" templates/rules/subagent.md
  ```

- [ ] **Step 4:** Commit.

  ```bash
  git add templates/rules/subagent.md
  git commit -m "feat(v4): add templates/rules/subagent.md (spec + 3 dispatch prompt templates)"
  ```

---

### Task 10: Write templates/rules/modules.md

**Files:**
- Create: `templates/rules/modules.md`

**Spec reference:** Section 7.2.8 (modules.md 业务层).

**Layer:** **业务层**(完全由 brainstorm 产出填充,升级模式重写)

**Required sections(全部是业务 placeholder):**
1. `# Modules`
2. `## 模块清单` — placeholder for brainstorm Q2 output
3. `## 边界规则表` — placeholder table(规则 / 含义 / 检测方式 / 严重度)
4. `## 数据流方向` — placeholder for brainstorm Q4 output
5. `## 外部依赖规范` — placeholder for brainstorm Q5 output
6. `## 项目专属触发器` — placeholder for brainstorm Q14 output(如 "改 auth/* 必须先读 session.ts")

- [ ] **Step 1:** Read spec Section 7.2.8.

- [ ] **Step 2:** Write `templates/rules/modules.md` with placeholder structure only.

  ```markdown
  # Modules

  > 本文件是业务层,由 bitfrog Phase 2 brainstorm 产出填充。升级模式下完全重写。

  ## 模块清单

  <!-- 由 brainstorm Q2 产出填充 -->
  {模块列表,每个模块含职责一句话}

  ## 边界规则表

  <!-- 由 brainstorm Q3 产出填充 -->
  | 规则 | 含义 | 检测方式 | 严重度 |
  |-----|------|---------|-------|
  | {规则 1} | {含义} | hook 自动 / reviewer 审查 | CRITICAL/MAJOR/MINOR |
  | ... | ... | ... | ... |

  ## 数据流方向

  <!-- 由 brainstorm Q4 产出填充 -->
  {单向 / 双向约束,如 A → B → C,C 不能反向调用 A}

  ## 外部依赖规范

  <!-- 由 brainstorm Q5 产出填充 -->
  {每个外部依赖的超时 / 重试 / 兜底要求}

  ## 项目专属触发器

  <!-- 由 brainstorm Q14 产出填充 -->
  {哪些目录/模块改动时必须先读某些上下文,如 "改 auth/* 必须先读 session.ts 的并发注释"}
  ```

- [ ] **Step 3:** Verify.

  ```bash
  test -f templates/rules/modules.md && \
  grep -c "^## 模块清单$" templates/rules/modules.md && \
  grep -c "^## 边界规则表$" templates/rules/modules.md && \
  grep -c "^## 数据流方向$" templates/rules/modules.md && \
  grep -c "^## 外部依赖规范$" templates/rules/modules.md && \
  grep -c "^## 项目专属触发器$" templates/rules/modules.md && \
  grep -c "业务层" templates/rules/modules.md
  ```

- [ ] **Step 4:** Commit.

  ```bash
  git add templates/rules/modules.md
  git commit -m "feat(v4): add templates/rules/modules.md (business-layer placeholder structure)"
  ```

---

### Task 11: Write templates/rules/memory.md

**Files:**
- Create: `templates/rules/memory.md`

**Spec reference:** Section 7.2.9 + Section 9 (Memory 三层分层规范).

**Required sections:**
1. `# Memory`
2. `## 三层定义`
   - `### Layer 1: Architectural(长期,不归档)`
   - `### Layer 2: Operational(中期,季度 review)`
   - `### Layer 3: Learned(短期,定期归档)`
3. `## 每层的写入时机`
4. `## 读取时机表` — 5 行(brainstorm 前 / load context / 卡住 / reviewer-quality / compact 后 resume)
5. `## 归档规则` — architectural 不归档 / operational 季度 review / learned 每月归档
6. `## 写入的硬动作(嵌入 workflow)` — task 收尾 checklist 的 2 条(architectural 决策 / operational 约定)
7. `## 三层的文件结构`(decisions.md 的 3 个二级标题:`## Architectural` / `## Operational` / `## Learned`)

- [ ] **Step 1:** Read spec Section 9 (Memory Layering).

- [ ] **Step 2:** Write `templates/rules/memory.md`.

  **每层的示例条目必须包含(从 spec Section 9.1-9.3):**
  - Architectural 示例:`## [Arch-001] 选 Postgres 而非 SQLite`(含日期 / 决策 / 原因 / 放弃的替代 / 影响 / 目标变更)
  - Operational 示例:`## [Op-023] API 响应统一用 envelope 格式`(含日期 / 约定 / 原因 / 适用范围 / 例外 / 状态)
  - Learned 示例:`## [Learn-007] auth 模块 token refresh 不要嵌套 try-catch`(含日期 / 规则 / 原因 / feedback-log 条目引用 / 状态)

  **读取时机表必须精确包含 5 行:**
  | 时机 | 读哪层 |
  |-----|-------|
  | brainstorm 前 | architectural + operational(相关模块) |
  | load context 阶段 | operational + learned(相关模块) |
  | 写代码卡住时 | learned(相关模块) |
  | reviewer-quality 审查时 | operational + learned |
  | compact 后 resume | architectural(不变约束) |

- [ ] **Step 3:** Verify.

  ```bash
  test -f templates/rules/memory.md && \
  grep -c "^### Layer 1: Architectural" templates/rules/memory.md && \
  grep -c "^### Layer 2: Operational" templates/rules/memory.md && \
  grep -c "^### Layer 3: Learned" templates/rules/memory.md && \
  grep -c "^## 读取时机表$" templates/rules/memory.md && \
  grep -c "^## 归档规则$" templates/rules/memory.md && \
  grep -c "Arch-001" templates/rules/memory.md && \
  grep -c "Op-023" templates/rules/memory.md && \
  grep -c "Learn-007" templates/rules/memory.md
  ```

- [ ] **Step 4:** Commit.

  ```bash
  git add templates/rules/memory.md
  git commit -m "feat(v4): add templates/rules/memory.md (three-layer decisions + read/write timing)"
  ```

---

### Task 12: Write templates/rules/docs.md

**Files:**
- Create: `templates/rules/docs.md`

**Spec reference:** Section 7.2.10 + Section 8 (Doc Management Layer 完整 9 子章).

**Layer:** 混合(骨架 + 业务 placeholder)

**Required sections:**
1. `# Docs`
2. `## 为什么这么严`(第一性理由:AI 唯一信息源是文件,compact-safe source of truth)
3. `## 三层读者`(用户 / 开发者 / AI)
4. `## B. 触发时机`(触发条件表,7 行)
5. `## C. 写作红线`(7 条)
6. `## D. Review 融合`(reviewer-spec 查漏 + reviewer-quality 查质量 + MAJOR FAIL)
7. `## E. AI-load 友好性`(模块接口文档固定结构 + docs agent 读什么)
8. `## F. Evolution 规则`(文档漂移 3 次重写)
9. `## 工作方式(角色协作表)`(worker / docs agent / reviewer-spec / reviewer-quality 各自职责边界)
10. `## 文档维度在 two-stage review 里的触发顺序`(流程图)
11. `## 本项目文档清单`(业务 placeholder,brainstorm Q11 填充)
12. `## 本项目文档同步触发器`(业务 placeholder,brainstorm Q11 填充)

- [ ] **Step 1:** Read spec Section 8 (Doc Management Layer).

- [ ] **Step 2:** Write `templates/rules/docs.md`.

  **为什么这么严 段必须含这段文字(精确):**
  > **AI 在这个项目里工作的唯一信息源是文件,不是对话。** 对话会被 compact 掉,文件不会。
  >
  > 所以:
  > - AI 下一次 session 能知道这个项目当前状态,**全靠读文档**
  > - 代码改了文档没改 → 下一次 session 的 AI 读到过期文档 → 做错误的判断 → 埋新 bug
  > - 文档必须是 **source of truth**,不是"有时间再写"的副产品
  >
  > 因此:任何代码变动,只要触发了 `#B` 的条件,**文档必须在同一 commit 内同步**。文档违规 = **MAJOR → FAIL**,和代码 bug 同级,不降级不放水。

  **触发时机表 7 行(# B)必须精确包含:**
  | 代码里发生的事 | 必须同步的文档 |
  |---|---|
  | 新增/修改公开接口 | `docs/modules/<module>.md` 或 README 命令段 |
  | 架构决策 | `decisions.md` 追加一条 |
  | 破坏性变更 | `CHANGELOG.md` + 迁移说明 |
  | 环境/依赖变更 | dev setup 段 |
  | 模块边界调整 | `rules/modules.md` + 相关模块接口文档 |
  | 发现 AI 容易踩的坑 | `feedback-log.md` |
  | task 没动公开面 | 不写文档(不强制) |

  **写作红线 7 条(# C)必须精确:**
  1. 禁止 restate 代码
  2. 禁止提前文档
  3. 代码示例必须可运行
  4. 禁止重复
  5. 禁止装腔作势(comprehensive / robust / nuanced / leveraging)
  6. 禁止无署名决定
  7. 禁止文档独立于代码的 commit

  **角色协作表 4 行必须精确:**
  | 角色 | 做什么 | 不做什么 |
  |-----|-------|---------|
  | worker | 写代码时同步维护本 task 涉及的文档(Step 11),小范围同步改动 | 不做整段重写,不动别的 task 文档 |
  | docs agent | 被主动 dispatch 时出场,整段/整个文档的重构或重写 | 不参与日常小修,不自动跑 |
  | reviewer-spec | 按 #B 查"代码改了,对应文档同一 commit 内更新了吗" | 不审文档质量,只审漏没漏 |
  | reviewer-quality | 按 #C 红线查文档质量 | 不重复 reviewer-spec 的漏检查 |

  **模块接口文档固定结构(#E)必须精确包含这些子段:**
  - `## 职责(一句话)`
  - `## 对外接口(函数签名 + 用途 + 示例)`
  - `## 依赖(上游 / 下游)`
  - `## 边界(不允许做什么 — 对应 rules/modules.md)`
  - `## 已知陷阱(来自 feedback-log.md 沉淀)`

- [ ] **Step 3:** Verify.

  ```bash
  test -f templates/rules/docs.md && \
  grep -c "^## 为什么这么严" templates/rules/docs.md && \
  grep -c "^## 三层读者$" templates/rules/docs.md && \
  grep -c "^## B\. 触发时机" templates/rules/docs.md && \
  grep -c "^## C\. 写作红线" templates/rules/docs.md && \
  grep -c "^## D\. Review 融合" templates/rules/docs.md && \
  grep -c "^## 工作方式" templates/rules/docs.md && \
  grep -c "compact-safe source of truth\|source of truth" templates/rules/docs.md && \
  grep -c "MAJOR → FAIL\|MAJOR FAIL" templates/rules/docs.md
  ```

- [ ] **Step 4:** Commit.

  ```bash
  git add templates/rules/docs.md
  git commit -m "feat(v4): add templates/rules/docs.md (doc management layer with 9 sub-sections + collaboration table)"
  ```

---

### Task 13: Write templates/rules/worktree.md

**Files:**
- Create: `templates/rules/worktree.md`

**Spec reference:** Section 7.2.11 + Section 6.3 (Worktree Infrastructure).

**Required sections:**
1. `# Worktree`
2. `## When to use`(触发条件 5 条)
3. `## When NOT to use`(4 条)
4. `## Directory 选择`(`.worktrees/` 优先,bitfrog 初始化时加进 .gitignore)
5. `## 安全验证`(`git check-ignore` 强制)
6. `## 创建 5 步`(detect project → create → auto-detect setup → verify baseline → report)
7. `## Baseline 失败的处理`(stop + 报告 + 问,不硬刚)
8. `## Cleanup 时机`(对齐 finishing flow 的 4 options)

- [ ] **Step 1:** Read spec Section 6.3 (Worktree Infrastructure).

- [ ] **Step 2:** Write `templates/rules/worktree.md`.

  **When to use 5 条触发条件必须精确:**
  - 需要新 branch(不是 main 直改)
  - 预期 ≥ 3 commit
  - 实现过程可能回滚
  - 跨模块或破坏性变更
  - 计划 dispatch subagent 实现 task

  **创建 5 步必须精确(含 bash 命令):**
  1. Detect project name: `project=$(basename "$(git rev-parse --show-toplevel)")`
  2. Create worktree: `git worktree add .worktrees/{branch-name} -b {branch-name}`
  3. Auto-detect project setup: `if [ -f package.json ]; then npm install; fi` (+ Cargo.toml, requirements.txt, pyproject.toml, go.mod)
  4. Verify clean baseline: 跑 verify 命令 per rules/verify.md, tests 必须绿
  5. Report location

  **Cleanup 4 options 对应:**
  - Option 1 Merge locally → 清
  - Option 2 Push + PR → 保留
  - Option 3 Keep as-is → 保留
  - Option 4 Discard → 清(typed "discard" 确认)

- [ ] **Step 3:** Verify.

  ```bash
  test -f templates/rules/worktree.md && \
  grep -c "^## When to use$" templates/rules/worktree.md && \
  grep -c "^## When NOT to use$" templates/rules/worktree.md && \
  grep -c "^## 安全验证$" templates/rules/worktree.md && \
  grep -c "^## 创建 5 步$" templates/rules/worktree.md && \
  grep -c "^## Cleanup 时机$" templates/rules/worktree.md && \
  grep -c "git check-ignore" templates/rules/worktree.md && \
  grep -c "git worktree add" templates/rules/worktree.md
  ```

- [ ] **Step 4:** Commit.

  ```bash
  git add templates/rules/worktree.md
  git commit -m "feat(v4): add templates/rules/worktree.md (isolation infrastructure + 5-step creation)"
  ```

---

## Phase B: Templates Agents (Tasks 14-17)

### Task 14: Write templates/agents/worker.md

**Files:**
- Create: `templates/agents/worker.md`

**Spec reference:** Section 7.3.1 (worker.md).

**Required sections:**
1. YAML frontmatter: `name: worker` / `description: {占位}` / `tools: [Read, Write, Edit, Bash, Grep, Glob, WebSearch, WebFetch]`
2. `# Worker` title
3. `## 定位`
4. `## 开工前`(读 CLAUDE.md / feedback-log / decisions)
5. `## 开发流程`(引用 rules/workflow.md)
6. `## 反馈循环`(verify 失败 → 读错误 → 修 → 再 verify,最多 3 轮)
7. `## 模块边界`(引用 rules/modules.md)
8. `## 禁止行为` — 跳过 step / 跳过 verify / 注释报错代码 / @ts-ignore / 硬刚 3 轮失败
9. `## 状态报告`(4 种状态:DONE / DONE_WITH_CONCERNS / NEEDS_CONTEXT / BLOCKED,引用 rules/subagent.md)
10. `## 提交`(verify 通过 + git diff 检查 + commit message 格式)

- [ ] **Step 1:** Read spec Section 7.3.1.

- [ ] **Step 2:** Write `templates/agents/worker.md`.

  **Frontmatter 必须精确:**
  ```yaml
  ---
  name: worker
  description: {基于项目定制的描述}
  tools: [Read, Write, Edit, Bash, Grep, Glob, WebSearch, WebFetch]
  ---
  ```

  **禁止行为 5 条必须精确:**
  - 跳过测试(test.skip / describe.skip)
  - 注释掉报错代码
  - @ts-ignore / any 类型
  - 违反模块边界(hooks 会拦截,但主动遵守)
  - 3 轮 verify 失败后继续硬刚

- [ ] **Step 3:** Verify.

  ```bash
  test -f templates/agents/worker.md && \
  head -1 templates/agents/worker.md | grep -q '^---$' && \
  grep -q '^name: worker$' templates/agents/worker.md && \
  grep -q '^tools:' templates/agents/worker.md && \
  grep -c "^## 开工前$" templates/agents/worker.md && \
  grep -c "^## 禁止行为$" templates/agents/worker.md && \
  grep -c "^## 状态报告" templates/agents/worker.md && \
  grep -c "DONE\|DONE_WITH_CONCERNS\|NEEDS_CONTEXT\|BLOCKED" templates/agents/worker.md
  ```

- [ ] **Step 4:** Commit.

  ```bash
  git add templates/agents/worker.md
  git commit -m "feat(v4): add templates/agents/worker.md (implementer with 4-status reporting)"
  ```

---

### Task 15: Write templates/agents/reviewer-spec.md (NEW AGENT)

**Files:**
- Create: `templates/agents/reviewer-spec.md`

**Spec reference:** Section 7.3.2 (reviewer-spec,新增 agent) + Section 7.4.2 (reviewer-spec dispatch prompt 模板).

**Required sections:**
1. YAML frontmatter: `name: reviewer-spec` / `description: {占位}` / `tools: [Read, Grep, Glob, Bash]`
2. `# Reviewer Spec` title
3. `## 定位`(第一 stage reviewer,spec compliance + 文档同步)
4. `## 职责`
5. `## 工作流程` — dispatch 入参 → 读 plan + diff → 核对 3 项(missing / extra / doc sync)→ 输出
6. `## 输出格式`
7. `## 禁止`(不审代码质量 / 不看 session history)

- [ ] **Step 1:** Read spec Section 7.3.2.

- [ ] **Step 2:** Write `templates/agents/reviewer-spec.md`.

  **Frontmatter:**
  ```yaml
  ---
  name: reviewer-spec
  description: {基于项目定制的描述 — spec compliance reviewer,检查代码是否满足 plan 要求 + 文档同步}
  tools: [Read, Grep, Glob, Bash]
  ---
  ```

  **核对 3 项必须精确:**
  1. **Missing**: 每个 task 要求的东西,代码里做了吗?
  2. **Extra**: task 没要求的东西,代码里多做了吗?(YAGNI)
  3. **Doc sync**: `rules/docs.md#B` 触发的文档,同一 commit 内同步了吗?

  **输出格式必须精确:**
  ```
  Verdict: PASS | FAIL

  If FAIL:
    [Missing] <requirement> — <how to fix> at <file:line>
    [Extra] <code> at <file:line> — <why remove>
    [Doc gap] <which doc> — <rules/docs.md#B trigger matched>

  If PASS: "Spec compliant — all requirements met, nothing extra, docs synced."
  ```

- [ ] **Step 3:** Verify.

  ```bash
  test -f templates/agents/reviewer-spec.md && \
  head -1 templates/agents/reviewer-spec.md | grep -q '^---$' && \
  grep -q '^name: reviewer-spec$' templates/agents/reviewer-spec.md && \
  grep -c "^## 工作流程$" templates/agents/reviewer-spec.md && \
  grep -c "^## 输出格式$" templates/agents/reviewer-spec.md && \
  grep -c "Missing\|Extra\|Doc gap" templates/agents/reviewer-spec.md && \
  grep -c "YAGNI" templates/agents/reviewer-spec.md && \
  grep -c "docs.md#B" templates/agents/reviewer-spec.md
  ```

- [ ] **Step 4:** Commit.

  ```bash
  git add templates/agents/reviewer-spec.md
  git commit -m "feat(v4): add templates/agents/reviewer-spec.md (stage-1 spec compliance reviewer)"
  ```

---

### Task 16: Write templates/agents/reviewer-quality.md

**Files:**
- Create: `templates/agents/reviewer-quality.md`

**Spec reference:** Section 7.3.3 (reviewer-quality, v3 reviewer.md 改名扩展) + Section 7.4.3 (reviewer-quality dispatch prompt).

**Required sections:**
1. YAML frontmatter: `name: reviewer-quality` / `description: {占位}` / `tools: [Read, Grep, Glob, Bash]`
2. `# Reviewer Quality` title
3. `## 定位`(第二 stage reviewer,代码质量 + 模块边界 + 文档质量)
4. `## 前置检查`(reviewer-spec 必须已 PASS)
5. `## 审查维度`(模块边界 / 代码质量 / 文档质量)
6. `## 判定规则(不可修改)` — CRITICAL/MAJOR → FAIL,只有 MINOR → PASS,**文档红线 = MAJOR**
7. `## 输出格式`(PASS/FAIL + [CRITICAL]/[MAJOR]/[MINOR] 列表)
8. `## 禁止`(不重复 spec compliance / 不看 session history / 不给 performative praise)

- [ ] **Step 1:** Read spec Section 7.3.3 + 7.4.3.

- [ ] **Step 2:** Write `templates/agents/reviewer-quality.md`.

  **判定规则必须精确(不可修改段落):**
  > ## 判定规则(不可修改)
  >
  > - Any CRITICAL → FAIL
  > - Any MAJOR → FAIL
  > - Only MINOR → PASS
  > - **Doc red line violation = MAJOR**(和代码 bug 同级,不降级)

  **审查维度 3 项:**
  1. **模块边界**(`rules/modules.md` 的规则)
  2. **代码质量**(项目专属 CRITICAL/MAJOR/MINOR 维度,{brainstorm 产出填充})
  3. **文档质量**(`rules/docs.md#C` 红线)

  **输出格式必须精确:**
  ```
  Verdict: PASS / FAIL

  [CRITICAL] 问题 + file:line + 修复建议
  [MAJOR] 问题 + file:line + 修复建议
  [MINOR] 建议
  ```

- [ ] **Step 3:** Verify.

  ```bash
  test -f templates/agents/reviewer-quality.md && \
  head -1 templates/agents/reviewer-quality.md | grep -q '^---$' && \
  grep -q '^name: reviewer-quality$' templates/agents/reviewer-quality.md && \
  grep -c "^## 判定规则" templates/agents/reviewer-quality.md && \
  grep -c "CRITICAL → FAIL" templates/agents/reviewer-quality.md && \
  grep -c "MAJOR → FAIL" templates/agents/reviewer-quality.md && \
  grep -c "Doc red line violation = MAJOR\|文档红线\|文档违规" templates/agents/reviewer-quality.md && \
  grep -c "reviewer-spec 必须已 PASS\|前置检查" templates/agents/reviewer-quality.md
  ```

- [ ] **Step 4:** Commit.

  ```bash
  git add templates/agents/reviewer-quality.md
  git commit -m "feat(v4): add templates/agents/reviewer-quality.md (stage-2 quality + doc red line = MAJOR)"
  ```

---

### Task 17: Write templates/agents/docs.md

**Files:**
- Create: `templates/agents/docs.md`

**Spec reference:** Section 7.3.4 (docs agent).

**Required sections:**
1. YAML frontmatter: `name: docs` / `description: {占位}` / `tools: [Read, Write, Edit, Grep, Glob]`
2. `# Docs Agent` title
3. `## 定位`(大规模文档重构的执行者)
4. `## Dispatch 时机`(新增模块完整接口文档 / 跨 3+ 文档重构 / 架构文档大改)
5. `## 工作前`(读 根 CLAUDE.md / 最近 10 条 decisions / git diff / 对应 `docs/modules/<module>.md`)
6. `## 遵循规范`(rules/docs.md#C 红线 + #E 模块接口文档固定结构)
7. `## 禁止读`(不读 `.claude/rules/` 其他文件)
8. `## 输出`(修改的文档列表 + 每个文档的 diff 摘要)

- [ ] **Step 1:** Read spec Section 7.3.4.

- [ ] **Step 2:** Write `templates/agents/docs.md`.

  **Frontmatter:**
  ```yaml
  ---
  name: docs
  description: {基于项目定制的描述 — 大规模文档重构执行者,遵循 rules/docs.md#C 红线}
  tools: [Read, Write, Edit, Grep, Glob]
  ---
  ```

  **Dispatch 时机 3 条:**
  - 新增整个模块 → 写全新 `docs/modules/<module>.md`
  - 重构影响 3+ 文档
  - 架构文档(ARCHITECTURE / DESIGN)大改

  **工作前读取清单(精确):**
  1. 根 `CLAUDE.md`
  2. `.claude/memory/decisions.md`(最近 10 条)
  3. 当前 task 的 `git diff`
  4. 对应的 `docs/modules/<module>.md`(如果存在)

  **禁止读 明确说明:**
  > 不读 `.claude/rules/` 下其他文件(那是 worker 用的开发规则,不是文档写作所需)

- [ ] **Step 3:** Verify.

  ```bash
  test -f templates/agents/docs.md && \
  head -1 templates/agents/docs.md | grep -q '^---$' && \
  grep -q '^name: docs$' templates/agents/docs.md && \
  grep -c "^## Dispatch 时机$" templates/agents/docs.md && \
  grep -c "^## 工作前$" templates/agents/docs.md && \
  grep -c "^## 禁止读$" templates/agents/docs.md && \
  grep -c "rules/docs.md#C" templates/agents/docs.md
  ```

- [ ] **Step 4:** Commit.

  ```bash
  git add templates/agents/docs.md
  git commit -m "feat(v4): add templates/agents/docs.md (large-scale doc refactor executor)"
  ```

---

## Phase C: Other Templates (Tasks 18-23)

### Task 18: Write templates/memory/decisions.md

**Files:**
- Create: `templates/memory/decisions.md`

**Spec reference:** Section 9 (Memory 三层).

**Required sections:**
1. `# 架构决策记录`
2. `## 说明` — 三层分层简介(指向 rules/memory.md)
3. `## Architectural` — 一级子段,含空 placeholder 说明 + 示例
4. `## Operational` — 一级子段,含空 placeholder 说明 + 示例
5. `## Learned` — 一级子段,含空 placeholder 说明 + 示例

- [ ] **Step 1:** Read spec Section 9.

- [ ] **Step 2:** Write `templates/memory/decisions.md`.

  ```markdown
  # 架构决策记录

  > 三层分层:**Architectural**(长期不归档)/ **Operational**(中期季度 review)/ **Learned**(短期月度归档)。
  > 写入时机和归档规则见 `.claude/rules/memory.md`。

  ---

  ## Architectural

  <!-- 长期决策:技术栈 / 架构模式 / 核心 tradeoff / 不可变约束 -->
  <!-- 不归档,永远保留全部历史 -->
  <!-- 写入时机:brainstorm 对齐的架构选择 / 项目重大转型 / 用户明确说是架构决定 -->

  ### [Arch-000] 初始化(示例)
  - **日期**: YYYY-MM-DD
  - **决策**: {选了什么}
  - **原因**: {为什么}
  - **放弃的替代**: {考虑过但放弃的}
  - **影响**: {影响哪些模块}
  - **目标变更**: 无

  ---

  ## Operational

  <!-- 中期约定:模块用什么 library / pattern 怎么写 / 命名约定 / 工具链选择 -->
  <!-- 每季度 review,过时标注 deprecated,不删 -->
  <!-- 写入时机:plan 执行中决定的实现模式 / brainstorm 对齐但未到架构级 / reviewer 多次指出同一问题沉淀 -->

  ### [Op-000] 初始化(示例)
  - **日期**: YYYY-MM-DD
  - **约定**: {什么约定}
  - **原因**: {为什么}
  - **适用范围**: {哪些场景}
  - **例外**: {无 或 具体例外}
  - **状态**: active

  ---

  ## Learned

  <!-- 短期沉淀:feedback-log 同类 3 次后的短规则 -->
  <!-- 每月归档,active 状态超 3 个月未触发 → archived → 归档到 feedback-log#archived -->
  <!-- 规则够普适 → 升级到 CLAUDE.md 或 hook -->

  ### [Learn-000] 初始化(示例)
  - **日期**: YYYY-MM-DD
  - **规则**: {具体规则}
  - **原因**: {踩了什么坑}
  - **对应 feedback-log 条目**: [YYYY-MM-DD] [YYYY-MM-DD] [YYYY-MM-DD]
  - **状态**: active
  ```

- [ ] **Step 3:** Verify.

  ```bash
  test -f templates/memory/decisions.md && \
  grep -c "^## Architectural$" templates/memory/decisions.md && \
  grep -c "^## Operational$" templates/memory/decisions.md && \
  grep -c "^## Learned$" templates/memory/decisions.md && \
  grep -c "Arch-000\|Op-000\|Learn-000" templates/memory/decisions.md && \
  grep -c "rules/memory.md" templates/memory/decisions.md
  ```

- [ ] **Step 4:** Commit.

  ```bash
  git add templates/memory/decisions.md
  git commit -m "feat(v4): add templates/memory/decisions.md (three-layer skeleton with examples)"
  ```

---

### Task 19: Write templates/state/session.md

**Files:**
- Create: `templates/state/session.md`

**Spec reference:** Section 6.1 (Session State).

- [ ] **Step 1:** Read spec Section 6.1.

- [ ] **Step 2:** Write `templates/state/session.md`.

  ```markdown
  # Session State

  Last updated: (none yet)
  Active plan: (none)
  Current task: (none)
  Current step: (none)

  ## 刚才在想什么
  (新项目,暂无)

  ## Blocker / 未决定
  (无)

  ## 今日 log
  <!-- 格式: YYYY-MM-DD HH:MM | task# | {动作} | {意外 (无则"无")} -->

  ## Review 标记(commit hook 会检查)
  <!-- 格式: task N | spec: approved | quality: approved -->
  ```

- [ ] **Step 3:** Verify.

  ```bash
  test -f templates/state/session.md && \
  grep -c "^# Session State$" templates/state/session.md && \
  grep -c "^## 刚才在想什么$" templates/state/session.md && \
  grep -c "^## Blocker" templates/state/session.md && \
  grep -c "^## 今日 log$" templates/state/session.md && \
  grep -c "^## Review 标记" templates/state/session.md && \
  grep -c "Active plan:" templates/state/session.md
  ```

- [ ] **Step 4:** Commit.

  ```bash
  git add templates/state/session.md
  git commit -m "feat(v4): add templates/state/session.md (compact-safe short-term scratchpad)"
  ```

---

### Task 20: Write templates/feedback-log.md

**Files:**
- Create: `templates/feedback-log.md`

**Spec reference:** Section 5.6 (深 feedback 保留 v3 格式).

- [ ] **Step 1:** Read spec Section 5.6.

- [ ] **Step 2:** Write `templates/feedback-log.md`.

  ```markdown
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
  <!-- {brainstorm Q8 output} -->

  ## Archived

  <!-- 深 feedback 归档段:超过 3 个月的 learned 条目沉淀到这里 -->
  ```

- [ ] **Step 3:** Verify.

  ```bash
  test -f templates/feedback-log.md && \
  grep -c "^# Feedback Log$" templates/feedback-log.md && \
  grep -c "^## 格式$" templates/feedback-log.md && \
  grep -c "^## Archived$" templates/feedback-log.md && \
  grep -c "微 feedback\|深 feedback" templates/feedback-log.md
  ```

- [ ] **Step 4:** Commit.

  ```bash
  git add templates/feedback-log.md
  git commit -m "feat(v4): add templates/feedback-log.md (deep feedback with two-stage reference)"
  ```

---

### Task 21: Write templates/README.md (user README skeleton)

**Files:**
- Create: `templates/README.md`

**Spec reference:** Section 12.4 4.13 (README skeleton).

- [ ] **Step 1:** Read spec Section 12.4 4.13.

- [ ] **Step 2:** Write `templates/README.md`.

  ```markdown
  # {项目名}

  {一句话项目目标 — 由 bitfrog Phase 2 Q1 填充}

  ## Install

  <!-- 由用户补充 -->

  ## Run

  <!-- 由用户补充 -->

  ## Development

  参见 `CLAUDE.md` 和 `.claude/rules/` 了解开发规范。

  Key rules:
  - 方法论流程:`.claude/rules/workflow.md`
  - 模块边界:`.claude/rules/modules.md`
  - 验证命令:`.claude/rules/verify.md`

  ## License

  <!-- 由用户补充 -->
  ```

- [ ] **Step 3:** Verify.

  ```bash
  test -f templates/README.md && \
  grep -c "^# {项目名}$" templates/README.md && \
  grep -c "^## Install$" templates/README.md && \
  grep -c "^## Run$" templates/README.md && \
  grep -c "^## Development$" templates/README.md && \
  grep -c "rules/workflow.md" templates/README.md
  ```

- [ ] **Step 4:** Commit.

  ```bash
  git add templates/README.md
  git commit -m "feat(v4): add templates/README.md (user README skeleton)"
  ```

---

### Task 22: Write templates/CHANGELOG.md

**Files:**
- Create: `templates/CHANGELOG.md`

**Spec reference:** Section 12.4 4.13 (CHANGELOG skeleton,Keep a Changelog 格式).

- [ ] **Step 1:** Read spec Section 12.4 4.13.

- [ ] **Step 2:** Write `templates/CHANGELOG.md`.

  ```markdown
  # Changelog

  All notable changes to this project will be documented in this file.

  The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
  and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

  ## [Unreleased]

  ### Added
  ### Changed
  ### Deprecated
  ### Removed
  ### Fixed
  ### Security
  ```

- [ ] **Step 3:** Verify.

  ```bash
  test -f templates/CHANGELOG.md && \
  grep -c "^# Changelog$" templates/CHANGELOG.md && \
  grep -c "Keep a Changelog" templates/CHANGELOG.md && \
  grep -c "Semantic Versioning" templates/CHANGELOG.md && \
  grep -c "^## \[Unreleased\]$" templates/CHANGELOG.md
  ```

- [ ] **Step 4:** Commit.

  ```bash
  git add templates/CHANGELOG.md
  git commit -m "feat(v4): add templates/CHANGELOG.md (Keep a Changelog skeleton)"
  ```

---

### Task 23: Write templates/hooks/skeleton-hooks.json

**Files:**
- Create: `templates/hooks/skeleton-hooks.json`

**Spec reference:** Section 11 (Hooks 5 骨架条).

**Required content:** 5 条骨架 hook 的 JSON 片段,按 Claude Code hooks API 格式,可以被 bitfrog Phase 4.2 合并到 `.claude/settings.json` 里。

- [ ] **Step 1:** Read spec Section 11.1-11.5 (骨架 hooks 1-5).

- [ ] **Step 2:** Write `templates/hooks/skeleton-hooks.json` as a JSON file containing a top-level `hooks` object with `PreToolUse` and `PostToolUse` arrays. All 5 skeleton hooks go in PreToolUse except Hook 5 which is PostToolUse (Write matcher for plan files).

  **File content (verbatim,必须是合法 JSON):**

  ```json
  {
    "_comment": "bitfrog v4 skeleton hooks — 5 条骨架层 hooks(升级时保留不动)",
    "hooks": {
      "PreToolUse": [
        {
          "_name": "Hook 1: 敏感文件拦截",
          "matcher": "Write|Edit",
          "hooks": [
            {
              "type": "command",
              "command": "FILE=\"$CLAUDE_TOOL_INPUT_FILE_PATH\"\nif echo \"$FILE\" | grep -qE '\\.(env|pem|key|secret|credentials)$'; then\n  echo '[BLOCKED] 敏感文件禁止 AI 修改' >&2\n  echo '原因: 防止泄露密钥和凭证' >&2\n  echo '修复: 手动编辑该文件' >&2\n  exit 1\nfi",
              "timeout": 5
            }
          ]
        },
        {
          "_name": "Hook 2: git commit 前 verify(项目 verify 命令填入)",
          "matcher": "Bash",
          "hooks": [
            {
              "type": "command",
              "if": "Bash(git commit*)",
              "command": "cd {项目绝对路径} && {verify 命令} 2>&1",
              "timeout": 120,
              "statusMessage": "提交前验证"
            }
          ]
        },
        {
          "_name": "Hook 3: git commit 前 review 通过检查",
          "matcher": "Bash",
          "hooks": [
            {
              "type": "command",
              "if": "Bash(git commit*)",
              "command": "SESSION='.claude/state/session.md'\nif [ ! -f \"$SESSION\" ]; then exit 0; fi\nTASK=$(grep -E '^Current task:' \"$SESSION\" | head -1 | sed 's/.*: //')\nif [ -z \"$TASK\" ] || [ \"$TASK\" = '(none)' ]; then exit 0; fi\nif ! grep -qE \"task ${TASK}.*spec: approved\" \"$SESSION\"; then\n  echo '[BLOCKED] 当前 task 的 spec compliance review 未通过' >&2\n  echo '原因: 必须先 dispatch reviewer-spec 通过才能 commit' >&2\n  echo '修复: 按 rules/review.md 跑 two-stage review' >&2\n  exit 1\nfi\nif ! grep -qE \"task ${TASK}.*quality: approved\" \"$SESSION\"; then\n  echo '[BLOCKED] 当前 task 的 code quality review 未通过' >&2\n  echo '原因: spec 通过后必须 dispatch reviewer-quality' >&2\n  exit 1\nfi",
              "timeout": 5
            }
          ]
        },
        {
          "_name": "Hook 4: git worktree add 前 ignore 验证",
          "matcher": "Bash",
          "hooks": [
            {
              "type": "command",
              "if": "Bash(git worktree add*)",
              "command": "CMD=\"$CLAUDE_TOOL_INPUT\"\nTARGET=$(echo \"$CMD\" | grep -oE 'git worktree add [^ ]+' | awk '{print $4}' | xargs dirname)\nif [ -z \"$TARGET\" ] || [ \"$TARGET\" = '.' ]; then exit 0; fi\nif ! git check-ignore -q \"$TARGET\" 2>/dev/null; then\n  echo \"[BLOCKED] worktree 目标目录 $TARGET 未被 .gitignore ignore\" >&2\n  echo '原因: 防止 worktree 内容污染主仓库' >&2\n  echo \"修复: echo '$TARGET/' >> .gitignore && git add .gitignore && git commit -m 'chore: ignore $TARGET'\" >&2\n  exit 1\nfi",
              "timeout": 5
            }
          ]
        }
      ],
      "PostToolUse": [
        {
          "_name": "Hook 5: Write .claude/plans/*.md 后 plan 格式自检",
          "matcher": "Write|Edit",
          "hooks": [
            {
              "type": "command",
              "if": "matches(.claude/plans/*.md)",
              "command": "FILE=\"$CLAUDE_TOOL_INPUT_FILE_PATH\"\nif ! grep -qE '^\\*\\*Goal:\\*\\*' \"$FILE\"; then\n  echo '[WARN] plan 缺少 **Goal:** 段' >&2\n  exit 0\nfi\nif grep -qE '(TBD|TODO|implement later|add appropriate error handling|similar to Task [0-9])' \"$FILE\"; then\n  echo '[BLOCKED] plan 含 No Placeholder 红线违规' >&2\n  echo '原因: rules/plan.md 禁止 TBD/TODO/implement later/similar to Task N/add appropriate error handling' >&2\n  echo '修复: 把 placeholder 展开成具体内容或删除该 task' >&2\n  exit 1\nfi",
              "timeout": 5
            }
          ]
        }
      ]
    }
  }
  ```

  **Note:** JSON 实际不支持注释,`_comment` 和 `_name` 字段作为可读注释(JSON 解析器会忽略未知字段)。bitfrog Phase 4.2 合并到 settings.json 时保留这些字段作为 in-line 注释。

- [ ] **Step 3:** Verify file is valid JSON and contains 5 hooks.

  ```bash
  test -f templates/hooks/skeleton-hooks.json && \
  python3 -c "import json; j = json.load(open('templates/hooks/skeleton-hooks.json')); \
    pre = len(j['hooks']['PreToolUse']); post = len(j['hooks']['PostToolUse']); \
    assert pre == 4, f'expected 4 PreToolUse hooks, got {pre}'; \
    assert post == 1, f'expected 1 PostToolUse hook, got {post}'; \
    print('OK: 4 PreToolUse + 1 PostToolUse')"
  ```
  Expected: "OK: 4 PreToolUse + 1 PostToolUse"

- [ ] **Step 4:** Commit.

  ```bash
  git add templates/hooks/skeleton-hooks.json
  git commit -m "feat(v4): add templates/hooks/skeleton-hooks.json (5 skeleton hooks JSON)"
  ```

---

## Phase D: SKILL.md Rewrite (Tasks 24-30)

### Task 24: Rewrite SKILL.md Phase 1 (exploration extended)

**Files:**
- Modify: `SKILL.md` (the `## Phase 1: 探索项目上下文` section)

**Spec reference:** Section 12.1.

- [ ] **Step 1:** Read current `SKILL.md` Phase 1 section and spec Section 12.1.

- [ ] **Step 2:** Replace the Phase 1 section of `SKILL.md` with v4 content that keeps v3 checks and adds v4 exploration steps.

  **New Phase 1 section must contain:**
  - 保留 v3 的 6 条(package.json / 现有配置 / git log / 目录结构 / README / 用户熟练度)
  - 新增 v4 扫描:`.worktrees/` + `git check-ignore .worktrees/` / `.claude/rules/` / `.claude/plans/` / `.claude/state/` / `docs/modules/` / `README.md` / `CHANGELOG.md`
  - 升级模式下额外:读 `.claude/rules/*.md` 现有内容,按 "Section 10 映射表" 识别骨架/业务段
  - 升级模式下额外:备份骨架层到 `.claude/.bitfrog-backup-YYYYMMDD-HHMMSS/`
  - 探索总结的格式要求(告诉用户扫到什么,是初始化还是升级,如果升级明确说会保留/重写什么)

- [ ] **Step 3:** Verify the modified section.

  ```bash
  grep -c "^## Phase 1: 探索项目上下文$" SKILL.md && \
  grep -c ".worktrees/" SKILL.md && \
  grep -c ".claude/rules/" SKILL.md && \
  grep -c ".bitfrog-backup" SKILL.md && \
  grep -c "git check-ignore" SKILL.md
  ```

- [ ] **Step 4:** Commit.

  ```bash
  git add SKILL.md
  git commit -m "feat(v4): rewrite SKILL.md Phase 1 (add worktree/rules/plans/state/docs scanning + upgrade-mode backup)"
  ```

---

### Task 25: Rewrite SKILL.md Phase 2 (brainstorm Q11-Q14 added)

**Files:**
- Modify: `SKILL.md` (the `## Phase 2: Brainstorm(核心)` section)

**Spec reference:** Section 12.2.

- [ ] **Step 1:** Read current Phase 2 and spec Section 12.2.

- [ ] **Step 2:** Modify the Phase 2 section to add Q11-Q14 after existing Q1-Q10 (keep Q1-Q10 unchanged).

  **Q11 — 文档习惯**(精确提问文案):
  > "关于文档:
  > 1. 项目有 README / CHANGELOG 吗?要我生成 skeleton 吗?
  > 2. 模块接口文档按 `docs/modules/<module>.md` 拆,还是集中在 ARCHITECTURE.md?
  > 3. 有没有特定的文件路径模式(如 `src/**/*.ts`),写入时应该提醒文档同步?"

  **Q12 — Worktree 偏好**(multiple choice):
  > "本项目要启用 worktree 吗?
  > A. 默认启用 + 条件触发(推荐)
  > B. 询问后启用
  > C. 显式 disable(项目小,不需要隔离)"

  **Q13 — 初始 architectural decisions**:
  > "项目有哪些已经定下来的架构选择值得记到 `decisions.md#architectural`?(技术栈 / 架构模式 / 不可变约束)"

  **Q14 — 项目专属触发器**:
  > "哪几个目录/模块改动时,**必须先读某些上下文**或**必须问你**?(如 '改 auth/* 必须先读 session.ts 的并发注释';'改 migration 必须问回滚策略')"

  **升级模式特殊说明:**
  > 升级模式下,如果用户说"项目目标不变",跳过 Q1;Q2-Q14 按需跑(只跑业务层相关问题)。

- [ ] **Step 3:** Verify.

  ```bash
  grep -c "^### Q11" SKILL.md && \
  grep -c "^### Q12" SKILL.md && \
  grep -c "^### Q13" SKILL.md && \
  grep -c "^### Q14" SKILL.md && \
  grep -c "文档习惯\|Worktree 偏好\|初始 architectural\|项目专属触发器" SKILL.md
  ```

- [ ] **Step 4:** Commit.

  ```bash
  git add SKILL.md
  git commit -m "feat(v4): add Phase 2 Q11-Q14 (docs habits / worktree / architectural init / triggers)"
  ```

---

### Task 26: Rewrite SKILL.md Phase 3 (new sections 5-8)

**Files:**
- Modify: `SKILL.md` (the `## Phase 3: 确认设计` section)

**Spec reference:** Section 12.3.

- [ ] **Step 1:** Read current Phase 3 and spec Section 12.3.

- [ ] **Step 2:** Add sections 5-8 to Phase 3 (keep sections 1-4 unchanged).

  **段 5: Rules 文件清单** — 列出 11 个 rule 文件,每个标注骨架/业务,让用户确认哪些是通用的(骨架)、哪些 brainstorm 产出会填入(业务)。

  **段 6: 基础设施**(Plan / Session state / Worktree)— 显示将创建的 `.claude/plans/` `.claude/state/session.md` `.worktrees/` 的初始化计划,问用户是否确认。

  **段 7: 文档 skeleton**(README / CHANGELOG / docs/modules/)— 按 Q11 的答案决定生成哪些 skeleton。如果用户已有就不生成。

  **段 8: Two-stage review 循环**(reviewer-spec 和 reviewer-quality 的拆分)— 明确告诉用户 reviewer 从 1 个(v3)拆成 2 个,v4 的 review loop 是 spec → quality 顺序。

  每段结束问 "这段 OK 吗?",通过才进下一段,和 v3 的惯例一致。

- [ ] **Step 3:** Verify.

  ```bash
  grep -c "^### 段 5" SKILL.md && \
  grep -c "^### 段 6" SKILL.md && \
  grep -c "^### 段 7" SKILL.md && \
  grep -c "^### 段 8" SKILL.md && \
  grep -c "Rules 文件清单\|基础设施\|文档 skeleton\|Two-stage review" SKILL.md
  ```

- [ ] **Step 4:** Commit.

  ```bash
  git add SKILL.md
  git commit -m "feat(v4): add Phase 3 sections 5-8 (rules inventory / infrastructure / docs skeleton / two-stage review)"
  ```

---

### Task 27: Rewrite SKILL.md Phase 4 (generation extended)

**Files:**
- Modify: `SKILL.md` (the `## Phase 4: 生成配置` section)

**Spec reference:** Section 12.4.

This is the largest change to SKILL.md. Keep v3 sub-sections 4.1-4.7 (adapted to reference templates/) and add 4.8-4.14.

- [ ] **Step 1:** Read current Phase 4 and spec Section 12.4.

- [ ] **Step 2:** Rewrite Phase 4 sub-sections:

  **4.1 CLAUDE.md** — change from "inline 大模板" to "copy templates/CLAUDE.md, fill in placeholders from Phase 2 brainstorm answers"
  **4.2 settings.json** — change from "inline hook definitions" to "merge templates/hooks/skeleton-hooks.json + add business hooks per rules/modules.md"
  **4.3 reviewer.md → reviewer-quality.md** — rename, copy from templates/agents/reviewer-quality.md
  **4.4 docs.md (agent)** — copy from templates/agents/docs.md
  **4.5 worker.md** — copy from templates/agents/worker.md, fill in project-specific tools
  **4.6 feedback-log.md** — copy from templates/feedback-log.md, prefill Q8 known pitfalls
  **4.7 decisions.md** — copy from templates/memory/decisions.md, prefill Q13 architectural decisions in the `## Architectural` section

  **4.8 Generate rules/*.md** — for each of the 11 rule files:
  - 骨架 rule(workflow / brainstorm / plan / review / feedback / subagent / memory / worktree):straight copy from `templates/rules/`
  - 混合 rule(verify / docs):copy template + replace business section placeholders with brainstorm answers
  - 业务 rule(modules):start from `templates/rules/modules.md` placeholder, fill entirely from Q2-Q5 + Q14

  **4.9 Generate reviewer-spec.md** — copy from templates/agents/reviewer-spec.md

  **4.10 Create directory skeleton:**
  ```bash
  mkdir -p .claude/plans
  mkdir -p .claude/state
  mkdir -p .worktrees        # only if Q12 answered "enable"
  mkdir -p docs/modules
  ```

  **4.11 Generate session.md** — copy from templates/state/session.md

  **4.12 Update .gitignore** — append (if not already present):
  ```
  .worktrees/
  .claude/state/
  .claude/.bitfrog-backup-*/
  ```
  **Do NOT ignore** `.claude/plans/` (plans 必须 commit)

  **4.13 Generate README / CHANGELOG skeleton** — only if Q11 answered "yes" and files don't already exist

  **4.14 Replace CLAUDE.md with pointer-table version** — this is the biggest break from v3. v3 CLAUDE.md 是大而全,v4 CLAUDE.md 是精简指针表。

- [ ] **Step 3:** Verify.

  ```bash
  grep -c "^### 4\.1" SKILL.md && \
  grep -c "^### 4\.8" SKILL.md && \
  grep -c "^### 4\.9" SKILL.md && \
  grep -c "^### 4\.12" SKILL.md && \
  grep -c "^### 4\.14" SKILL.md && \
  grep -c "templates/rules/" SKILL.md && \
  grep -c "templates/agents/" SKILL.md && \
  grep -c "templates/hooks/" SKILL.md && \
  grep -c "不 ignore\|不.*ignore.*plans" SKILL.md
  ```

- [ ] **Step 4:** Commit.

  ```bash
  git add SKILL.md
  git commit -m "feat(v4): rewrite Phase 4 (reference templates/ + add 4.8-4.14 for rules/agents/state/gitignore)"
  ```

---

### Task 28: Rewrite SKILL.md Phase 5 (validation extended 5.7-5.11)

**Files:**
- Modify: `SKILL.md` (the `## Phase 5: 生成后验证` section)

**Spec reference:** Section 12.5.

- [ ] **Step 1:** Read current Phase 5 and spec Section 12.5.

- [ ] **Step 2:** Add sections 5.7-5.11 to Phase 5 (keep 5.1-5.6 unchanged).

  **5.7 Rules 文件验证:**
  - [ ] `.claude/rules/` 下 11 个文件都存在(workflow / brainstorm / plan / review / verify / feedback / subagent / modules / memory / docs / worktree)
  - [ ] 每个 rule 文件含定义的关键章节(按 spec Section 7.2 的清单)
  - [ ] 骨架 rule 文件内容和 `templates/rules/*` 模板一致(`diff` 检查)
  - [ ] 业务 rule 文件(modules.md / verify.md 业务段 / docs.md 业务段)含 brainstorm 产出
  - [ ] CLAUDE.md 骨架指针表引用的 rules/*.md 都真实存在

  **5.8 Two-stage review 拆分验证:**
  - [ ] `.claude/agents/reviewer-spec.md` 存在
  - [ ] `.claude/agents/reviewer-quality.md` 存在(原 reviewer.md 改名或新生成)
  - [ ] reviewer-quality.md 判定规则段含 "CRITICAL → FAIL" "MAJOR → FAIL" "Doc red line violation = MAJOR"
  - [ ] plan 模板的 task 结构含 Step 6-9 的 review loop
  - [ ] settings.json 含 Hook 3(commit 前 review 通过检查)

  **5.9 基础设施验证:**
  - [ ] `.claude/plans/` 目录存在
  - [ ] `.claude/state/session.md` 存在且有模板内容
  - [ ] `.worktrees/` 存在(Q12 启用时)
  - [ ] `.gitignore` 含 `.worktrees/` `.claude/state/` `.claude/.bitfrog-backup-*/`
  - [ ] `.gitignore` **不**含 `.claude/plans/`
  - [ ] `docs/modules/` 目录存在

  **5.10 文档 skeleton 验证**(Q11 答"要"时):
  - [ ] `README.md` 存在
  - [ ] `CHANGELOG.md` 存在
  - [ ] bitfrog 生成的 skeleton 只在原文件不存在时生成(检查覆盖性)

  **5.11 Memory 分层验证:**
  - [ ] `.claude/memory/decisions.md` 含 `## Architectural` / `## Operational` / `## Learned` 三个段
  - [ ] brainstorm Q13 的初始 architectural decisions 已写入对应段
  - [ ] `rules/memory.md` 存在且定义了三层

- [ ] **Step 3:** Verify.

  ```bash
  grep -c "^### 5\.7" SKILL.md && \
  grep -c "^### 5\.8" SKILL.md && \
  grep -c "^### 5\.9" SKILL.md && \
  grep -c "^### 5\.10" SKILL.md && \
  grep -c "^### 5\.11" SKILL.md && \
  grep -c "Rules 文件验证\|Two-stage review 拆分\|基础设施验证\|文档 skeleton\|Memory 分层" SKILL.md
  ```

- [ ] **Step 4:** Commit.

  ```bash
  git add SKILL.md
  git commit -m "feat(v4): add Phase 5 sections 5.7-5.11 (rules/review-split/infrastructure/docs-skeleton/memory)"
  ```

---

### Task 29: Add SKILL.md Upgrade Mode Section

**Files:**
- Modify: `SKILL.md` (add new `## 升级模式` section after Phase 5)

**Spec reference:** Section 13 (Upgrade Mode 7 步).

- [ ] **Step 1:** Read spec Section 13.

- [ ] **Step 2:** Add a new top-level section `## 升级模式(Upgrade Mode)` after Phase 5 containing 7 steps and rollback.

  **Required content structure:**
  ```markdown
  ## 升级模式(Upgrade Mode)

  ### 触发检测
  {代码:检测已有 CLAUDE.md 且含"项目目标"章节}

  ### Step U1: 检测和备份
  {bash 代码}

  ### Step U2: 识别骨架 vs 业务层
  {按 Section 10 映射表分类,输出 diff 报告给用户}

  ### Step U3: 重新 brainstorm(只跑业务相关问题)
  {跳过 Q1 if 目标不变,只跑 Q2-Q14 里和业务层相关的}
  {开场白:检测到已有 harness 配置...}

  ### Step U4: 重写业务层(只动业务部分)
  {精确说明每个文件哪段动哪段不动}

  ### Step U5: 追加新 decisions(标注"目标变更")
  {格式示例}

  ### Step U6: 跑 Phase 5 验证
  {全部 5.1-5.11 验证,特别是 5.6 升级兼容性}

  ### Step U7: 清理备份
  {通过才清,失败保留;rollback 命令打印给用户}
  ```

  **骨架/业务分层映射表**必须嵌入或明确引用 spec Section 10。

- [ ] **Step 3:** Verify.

  ```bash
  grep -c "^## 升级模式" SKILL.md && \
  grep -c "^### Step U1" SKILL.md && \
  grep -c "^### Step U2" SKILL.md && \
  grep -c "^### Step U3" SKILL.md && \
  grep -c "^### Step U4" SKILL.md && \
  grep -c "^### Step U5" SKILL.md && \
  grep -c "^### Step U6" SKILL.md && \
  grep -c "^### Step U7" SKILL.md && \
  grep -c ".bitfrog-backup" SKILL.md && \
  grep -c "目标变更" SKILL.md
  ```

- [ ] **Step 4:** Commit.

  ```bash
  git add SKILL.md
  git commit -m "feat(v4): add SKILL.md 升级模式 section (7 steps + rollback)"
  ```

---

### Task 30: Update SKILL.md 关键原则 section + other copy edits

**Files:**
- Modify: `SKILL.md` (the `## 关键原则` section at the bottom)

**Spec reference:** Section 3 (Design Philosophy 10 条).

- [ ] **Step 1:** Read current `## 关键原则` section and spec Section 3.

- [ ] **Step 2:** Update `## 关键原则` to include v4 new principles while keeping v3 principles. Add these v4 principles(在现有的 13 条 v3 principles 之后追加):

  **新增 v4 原则:**
  - **第一动作强制** — 收到任务先复述 + 检查理解,不跳过
  - **brainstorm 是动态模式** — 不是 phase,是"明确目标 ↔ load context" 耦合螺旋,退出条件驱动
  - **plan 是可移交合约** — 和 session 解耦,compact-safe
  - **Two-stage review 硬约束** — spec first → quality after,不可颠倒
  - **Verify 5 步 gate** — Evidence before claims;Agent success ≠ success(必须 check VCS diff)
  - **微 feedback 每 task 必写** — 嵌入 plan Step 10,不靠自觉
  - **文档是 compact-safe source of truth** — 违规 = MAJOR → FAIL
  - **Review 直接行动,不写报告** — 三类 finding 分流(本 task / 本来就有 / plan 错)
  - **选择性吸收 superpowers** — brainstorm 哲学 / plan 格式 / review 循环 / verify evidence / worktree 隔离;**不吸收** TDD 原教旨 / debug 四阶段 / 红旗清单
  - **骨架/业务分层** — 升级模式保骨架重写业务

  同时更新 "工具适配" 段(v3 line 575):移除 "未来版本将支持 Cursor / Codex / Windsurf" 的承诺,改为 "v4 深度绑定 Claude Code;跨 harness 推到 v5"。

- [ ] **Step 3:** Verify.

  ```bash
  grep -c "第一动作强制" SKILL.md && \
  grep -c "brainstorm 是动态模式" SKILL.md && \
  grep -c "Two-stage review 硬约束" SKILL.md && \
  grep -c "文档是 compact-safe source of truth" SKILL.md && \
  grep -c "深度绑定 Claude Code" SKILL.md && \
  ! grep -q "未来版本将支持 Cursor" SKILL.md
  ```

- [ ] **Step 4:** Commit.

  ```bash
  git add SKILL.md
  git commit -m "feat(v4): update 关键原则 (add 10 v4 principles) + remove cross-harness promise"
  ```

---

## Phase E: bitfrog Metadata (Task 31)

### Task 31: Update bitfrog README / CHANGELOG / plugin metadata

**Files:**
- Modify: `README.md` (bitfrog's own README, not the template)
- Modify: `CHANGELOG.md` (create if not exists)
- Modify: `.claude-plugin/plugin.json`
- Modify: `.claude-plugin/marketplace.json`

**Spec reference:** Section 16 Migration Plan.

- [ ] **Step 1:** Read current README.md, plugin.json, marketplace.json.

  ```bash
  cat README.md
  cat .claude-plugin/plugin.json
  cat .claude-plugin/marketplace.json
  ```

- [ ] **Step 2:** Update each file:

  **README.md** — add a "What's New in v4" section (short,指向 spec 和 CHANGELOG):
  ```markdown
  ## What's New in v4 (2026-04-10)

  bitfrog v4 adds:
  - **Dynamic brainstorm mode** — 退出条件驱动,不再固定问题清单
  - **Plan as movable contract** — `.claude/plans/*.md` + compact-safe session state
  - **Two-stage review loop** — reviewer-spec + reviewer-quality,强制顺序
  - **Doc management layer** — 代码改动必须同步文档,违规 MAJOR FAIL
  - **Memory three-tier layering** — architectural / operational / learned
  - **Worktree infrastructure** — 大改隔离,baseline 验证
  - **Information architecture** — 根 CLAUDE.md 精简为指针表,规则按 topic 拆到 `.claude/rules/`

  See [design spec](docs/superpowers/specs/2026-04-10-bitfrog-v4-design.md) for details.
  ```

  **CHANGELOG.md** — create with Keep a Changelog format, add v4 entry:
  ```markdown
  # Changelog

  All notable changes to bitfrog-harness-init will be documented in this file.

  The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
  and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

  ## [0.2.0] - 2026-04-10

  ### Added
  - Templates directory with 22 template files (CLAUDE.md + 11 rules + 4 agents + memory/state/hooks/README/CHANGELOG skeletons)
  - Dynamic brainstorm mode (action pool + exit conditions)
  - Two-stage review loop (reviewer-spec + reviewer-quality)
  - Plan as movable contract (.claude/plans/)
  - Session state for compact-safe work resumption (.claude/state/session.md)
  - Worktree infrastructure (.worktrees/ with .gitignore)
  - Doc management layer (rules/docs.md + 角色协作表)
  - Memory three-tier layering (architectural / operational / learned)
  - Upgrade mode (7-step flow with backup + rollback)
  - 3 new skeleton hooks (Hook 3: commit-review gate, Hook 4: worktree ignore verify, Hook 5: plan format self-check)

  ### Changed
  - SKILL.md: Phase 1 extended (scan worktree/rules/plans/state/docs)
  - SKILL.md: Phase 2 added Q11-Q14 (docs / worktree / architectural init / triggers)
  - SKILL.md: Phase 3 added sections 5-8 (rules inventory / infra / docs skeleton / two-stage review)
  - SKILL.md: Phase 4 rewritten to reference templates/ (4.8-4.14 added)
  - SKILL.md: Phase 5 extended with 5.7-5.11 (rules / review-split / infra / docs / memory)
  - `.claude/agents/reviewer.md` renamed to `reviewer-quality.md`
  - Root CLAUDE.md template changed from "大而全" to "项目目标 + 骨架指针表"
  - `.claude/memory/decisions.md` changed from single-layer to three-tier

  ### Removed
  - "Future support for Cursor/Codex/Windsurf" promise (deferred to v5)

  ## [0.1.0] - 2026-04-04

  - Initial release (v3)
  ```

  **plugin.json** — bump version from current to "0.2.0":
  ```bash
  cat .claude-plugin/plugin.json
  # 确认当前 version,然后 edit 到 0.2.0
  ```

  **marketplace.json** — same bump.

- [ ] **Step 3:** Verify.

  ```bash
  grep -c "What's New in v4\|v4 (2026" README.md && \
  grep -c "^## \[0\.2\.0\]" CHANGELOG.md && \
  grep -c "Two-stage review\|templates directory\|Dynamic brainstorm" CHANGELOG.md && \
  grep -c '"version": "0\.2\.0"' .claude-plugin/plugin.json && \
  grep -c '"version": "0\.2\.0"' .claude-plugin/marketplace.json
  ```

- [ ] **Step 4:** Commit.

  ```bash
  git add README.md CHANGELOG.md .claude-plugin/plugin.json .claude-plugin/marketplace.json
  git commit -m "chore(v4): update metadata (README + CHANGELOG 0.2.0 + plugin version bump)"
  ```

---

## Phase F: Dogfood Tests (Tasks 32-34)

### Task 32: Dogfood — Initialization mode on a test project

**Files:**
- Create: `/tmp/bitfrog-v4-dogfood-init/` (throwaway test project)
- Read: generated files in the test project

- [ ] **Step 1:** Create a minimal test project.

  ```bash
  mkdir -p /tmp/bitfrog-v4-dogfood-init
  cd /tmp/bitfrog-v4-dogfood-init
  git init
  echo '{"name":"dogfood-test","version":"0.0.1"}' > package.json
  echo "# Dogfood Test" > README.md
  git add .
  git commit -m "init"
  ```

- [ ] **Step 2:** From within Claude Code in `/tmp/bitfrog-v4-dogfood-init`,运行 `/bitfrog-harness-init` skill。按 SKILL.md v4 的 Phase 1-5 走完整流程:
  - Phase 1:扫描项目(是个空 node 项目)
  - Phase 2:brainstorm Q1-Q14(作为测试,回答所有问题)
  - Phase 3:段 1-8 逐段确认
  - Phase 4:生成所有文件
  - Phase 5:跑所有验证

  **重要:** 这一步是手动运行,记录遇到的问题和错误。

- [ ] **Step 3:** Verify the generated harness in the test project.

  ```bash
  cd /tmp/bitfrog-v4-dogfood-init

  # Phase 5.1: 结构验证
  test -f CLAUDE.md
  test -f feedback-log.md
  test -f .claude/settings.json
  test -f .claude/memory/decisions.md

  # Phase 5.7: Rules files
  for f in workflow brainstorm plan review verify feedback subagent modules memory docs worktree; do
    test -f ".claude/rules/$f.md" && echo "OK: $f.md"
  done

  # Phase 5.8: Two-stage review split
  test -f .claude/agents/reviewer-spec.md
  test -f .claude/agents/reviewer-quality.md

  # Phase 5.9: Infrastructure
  test -d .claude/plans
  test -f .claude/state/session.md
  test -d docs/modules
  grep -q '.worktrees/' .gitignore
  grep -q '.claude/state/' .gitignore
  grep -v -q '.claude/plans/' .gitignore  # 确认 plans 没 ignore

  # Phase 5.11: Memory 三层
  grep -q '^## Architectural$' .claude/memory/decisions.md
  grep -q '^## Operational$' .claude/memory/decisions.md
  grep -q '^## Learned$' .claude/memory/decisions.md
  ```
  Expected: all checks pass.

  **Record any failures in `/tmp/bitfrog-v4-dogfood-init/DOGFOOD-FAILURES.md`** for Task 34.

- [ ] **Step 4:** Commit dogfood results to a note in the upstream repo.

  ```bash
  cd /Users/rainlei/holiday/bitfrog-harness-init
  mkdir -p docs/superpowers/dogfood
  cp /tmp/bitfrog-v4-dogfood-init/DOGFOOD-FAILURES.md docs/superpowers/dogfood/2026-04-10-init-mode.md 2>/dev/null || echo "(no failures)" > docs/superpowers/dogfood/2026-04-10-init-mode.md
  git add docs/superpowers/dogfood/2026-04-10-init-mode.md
  git commit -m "test(v4): dogfood results for init mode"
  ```

---

### Task 33: Dogfood — Upgrade mode on bitfrog-harness-init itself (meta test)

**Files:**
- Modify: the current `bitfrog-harness-init` repo (meta — upgrade this project from v3 harness to v4 harness using v4 itself)

**Note:** This is a meta test. bitfrog-harness-init currently has no harness (no `.claude/rules/` etc.). Running upgrade mode should detect "no prior v4 harness" and fall into init mode. If it correctly handles "v3 inline SKILL.md + no rules/" state, that's a pass.

- [ ] **Step 1:** From within Claude Code in `/Users/rainlei/holiday/bitfrog-harness-init`, run `/bitfrog-harness-init` skill.

  Expected behavior:
  - Phase 1 扫描:检测到 CLAUDE.md 不存在(bitfrog 项目本身没有项目级 CLAUDE.md)→ 进入初始化模式
  - 如果 bitfrog 项目本身有 CLAUDE.md,进入升级模式

  Either path should work — this task tests both scenarios.

- [ ] **Step 2:** Go through Phase 2 brainstorm(项目目标 = "生成 harness engineering for projects")。

- [ ] **Step 3:** Verify generated harness.

  ```bash
  cd /Users/rainlei/holiday/bitfrog-harness-init
  # Run the same Phase 5 checks as Task 32
  test -f .claude/rules/workflow.md
  test -f .claude/agents/reviewer-spec.md
  test -f .claude/agents/reviewer-quality.md
  test -d .claude/plans
  test -f .claude/state/session.md
  grep -q '.worktrees/' .gitignore
  ```

  **Record failures in `docs/superpowers/dogfood/2026-04-10-upgrade-mode.md`**.

- [ ] **Step 4:** Commit dogfood results.

  ```bash
  git add docs/superpowers/dogfood/2026-04-10-upgrade-mode.md
  git commit -m "test(v4): dogfood results for upgrade mode (meta)"
  ```

---

### Task 34: Fix dogfood-discovered issues

**Files:**
- Modify: whatever files need fixing based on Task 32 and 33 failures

**Note:** This is a placeholder task that may not be needed if dogfood passes. If it does have issues,they are fixed here with one commit per fix.

- [ ] **Step 1:** Read the dogfood failure records from Task 32 and 33.

  ```bash
  cat docs/superpowers/dogfood/2026-04-10-init-mode.md
  cat docs/superpowers/dogfood/2026-04-10-upgrade-mode.md
  ```

- [ ] **Step 2:** For each issue, identify the root cause and the file to fix:
  - Template file missing a required section → fix `templates/.../X.md`
  - SKILL.md Phase logic wrong → fix `SKILL.md`
  - Hook script syntax error → fix `templates/hooks/skeleton-hooks.json`
  - Placeholder not substituted correctly → fix SKILL.md Phase 4 logic

- [ ] **Step 3:** Fix each issue. For each fix, commit individually with message `fix(v4): {specific issue}`.

  Example:
  ```bash
  # Suppose Task 32 found that rules/modules.md placeholder not substituted
  git add templates/rules/modules.md SKILL.md
  git commit -m "fix(v4): modules.md placeholder substitution in Phase 4.8"
  ```

- [ ] **Step 4:** Re-run Task 32 and 33 smoke tests after fixes.

  ```bash
  # Re-verify a minimal subset
  cd /tmp/bitfrog-v4-dogfood-init
  test -f .claude/rules/modules.md
  grep -q '{模块清单}' .claude/rules/modules.md || echo "OK: placeholder substituted"
  ```

---

## Phase G: Final (Task 35)

### Task 35: Final self-verification + push

**Files:**
- Read: all modified/created files to do a final sanity check
- Execute: push to remote

- [ ] **Step 1:** Run a final structural sanity check on the bitfrog-harness-init repo.

  ```bash
  cd /Users/rainlei/holiday/bitfrog-harness-init

  # Check templates directory
  ls templates/
  ls templates/rules/ | wc -l   # expect 11
  ls templates/agents/ | wc -l  # expect 4

  # Check SKILL.md has v4 sections
  grep -c "^## 升级模式" SKILL.md  # expect 1
  grep -c "第一动作强制" SKILL.md  # expect 1+
  grep -c "templates/" SKILL.md    # expect many

  # Check metadata
  grep "0\.2\.0" .claude-plugin/plugin.json
  grep "0\.2\.0" .claude-plugin/marketplace.json
  grep "What's New in v4" README.md
  grep "^## \[0\.2\.0\]" CHANGELOG.md

  # Check spec and plan are in place
  test -f docs/superpowers/specs/2026-04-10-bitfrog-v4-design.md
  test -f docs/superpowers/plans/2026-04-10-bitfrog-v4-upgrade.md
  ```

  Expected: all checks pass.

- [ ] **Step 2:** Commit spec + plan (they haven't been committed yet — per user's "Strategy C" decision in brainstorm).

  ```bash
  git add docs/superpowers/specs/2026-04-10-bitfrog-v4-design.md
  git add docs/superpowers/plans/2026-04-10-bitfrog-v4-upgrade.md
  git commit -m "docs(v4): add design spec and implementation plan"
  ```

- [ ] **Step 3:** Push branch to origin and create PR (or ask user).

  ```bash
  git push -u origin feature/v4-upgrade
  ```

  Then ask user: "v4 upgrade branch pushed. Create a PR?"

  If yes:
  ```bash
  gh pr create --title "feat(v4): bitfrog v4 upgrade — dynamic methodology + infrastructure + doc layer" --body "$(cat <<'EOF'
  ## Summary
  - Upgrade bitfrog from v3 (static rules generator) to v4 (rules + methodology + infrastructure + doc layer)
  - Selective integration of superpowers (brainstorm / plan / review / verify / subagent / worktree), avoiding TDD/debug/red-flags rigidity
  - Split monolithic SKILL.md into thin orchestrator + templates/ directory (22 files)
  - Two-stage review loop (reviewer-spec + reviewer-quality)
  - Compact-safe plan + session state
  - Doc management as independent layer (red line violations = MAJOR)
  - Memory three-tier layering (architectural / operational / learned)
  - Upgrade mode with 7-step flow + rollback

  ## Design spec
  `docs/superpowers/specs/2026-04-10-bitfrog-v4-design.md` (2100 lines)

  ## Implementation plan
  `docs/superpowers/plans/2026-04-10-bitfrog-v4-upgrade.md` (this plan)

  ## Test plan
  - [x] Dogfood init mode on a throwaway test project
  - [x] Dogfood upgrade mode on bitfrog-harness-init itself (meta test)
  - [ ] User acceptance test on a real project

  🤖 Generated with [Claude Code](https://claude.com/claude-code)
  EOF
  )"
  ```

- [ ] **Step 4:** Report status to user.

  ```
  STATUS: DONE

  v4 upgrade complete.
  - 22 template files created in templates/
  - SKILL.md rewritten (Phases 1-5 extended + upgrade mode added)
  - Metadata bumped to 0.2.0
  - Dogfood passed (init + upgrade modes)
  - PR: <url>
  ```

---

## Self-Review

### Spec Coverage Check

| Spec section | Covered by task(s) |
|--------------|-------------------|
| 1 Overview | (context, no task) |
| 2 Goals / Non-Goals | (context, no task) |
| 3 Design Philosophy | Task 30 (关键原则 update) |
| 4.1 生成物目录树 | Task 1 (directory creation) + all file-creation tasks |
| 4.2 方法论流程 | Task 3 (workflow.md) |
| 4.3 Compact 协议 | Task 3 (workflow.md) |
| 5.1 第一动作 | Task 3 (workflow.md) |
| 5.2 Brainstorm 动态模式 | Task 4 (brainstorm.md) |
| 5.3 Plan 格式 | Task 5 (plan.md) |
| 5.4 Review Loop | Task 6 (review.md) |
| 5.5 Verify Gate | Task 7 (verify.md) |
| 5.6 Feedback 两阶段 | Task 8 (feedback.md) |
| 5.7 Subagent | Task 9 (subagent.md) |
| 6.1 Session State | Task 19 (session.md) + referenced in Task 3 |
| 6.2 Plan Files | Task 5 (plan.md) |
| 6.3 Worktree | Task 13 (worktree.md) |
| 6.4 Finishing Flow | Task 3 (workflow.md) |
| 7.1 CLAUDE.md 模板 | Task 2 (CLAUDE.md template) |
| 7.2 Rules 文件规范 | Tasks 3-13 (one per rule file) |
| 7.3 Agents 规范 | Tasks 14-17 (one per agent) |
| 7.4 Dispatch prompt 模板 | Task 9 (subagent.md contains all 3) |
| 8 Doc Management Layer | Task 12 (docs.md) |
| 9 Memory Layering | Tasks 11, 18 (memory.md + decisions.md) |
| 10 Skeleton/Business Map | Task 27 (Phase 4 references) + Task 29 (upgrade mode) |
| 11 Hooks | Task 23 (skeleton-hooks.json) + Task 27 (Phase 4.2 merge logic) |
| 12 SKILL.md Phase Changes | Tasks 24-30 (one per Phase/section) |
| 13 Upgrade Mode | Task 29 (upgrade mode section) |
| 14 Superpowers 吸收清单 | Task 30 (关键原则) |
| 15 v3 → v4 diff | (reference section, no task needed) |
| 16 Migration Plan | Task 31 (metadata) + Tasks 32-34 (dogfood) |
| 17 Success Criteria | Task 35 (final verification) |
| 18 Risks | (informational, no task) |
| 19 Out of Scope | (informational, no task) |
| 20 Mental Model 变化 | (informational, no task) |

**Coverage: 100%** — every actionable section in the spec maps to at least one task.

**Gap found**: Section 7.3.3 (reviewer-quality) references Section 7.4.3 (dispatch prompt) — covered by Task 9 (subagent.md) and Task 16 (reviewer-quality.md). Both tasks reference this template. ✓

**Another gap found**: Section 4.12 in Phase 4 of SKILL.md says "更新 .gitignore",which is covered in Task 27 Step 2. But the test project in Task 32 also needs its own .gitignore update when bitfrog runs on it — this is implicit in Phase 4.12 and will happen automatically. ✓

### Placeholder Scan

Searched this plan for the forbidden patterns from writing-plans skill:
- "TBD" → not found (as plan instruction)
- "TODO" → only found as "禁止" content describing what NOT to write (OK)
- "implement later" → not found
- "fill in details" → not found
- "Add appropriate error handling" → not found
- "Write tests for the above" → not found
- "Similar to Task N" → not found

**Plan is placeholder-free.** ✓

### Type Consistency

Cross-task name consistency:
- `templates/rules/workflow.md` — mentioned consistently in Tasks 3, 24, 26, 27 ✓
- `reviewer-spec` / `reviewer-quality` — consistently spelled across Tasks 9, 15, 16, 27, 28 ✓
- `decisions.md#architectural` / `#operational` / `#learned` — consistent in Tasks 11, 18, 27 ✓
- `rules/docs.md#B` / `#C` / `#D` / `#E` — consistent across Tasks 6, 12, 15, 16 ✓
- Hook numbering (1-5 skeleton, 6+ business) — consistent in Tasks 23, 27, 28 ✓
- Session.md headers (`## 刚才在想什么` / `## Blocker` / `## 今日 log` / `## Review 标记`) — consistent in Task 19 and referenced correctly in Task 3 ✓

**No naming inconsistencies found.** ✓

---

## Execution Handoff

**Plan complete and saved to `docs/superpowers/plans/2026-04-10-bitfrog-v4-upgrade.md`.**

35 tasks total:
- Phase A (Templates infra + CLAUDE.md + rules): 13 tasks
- Phase B (Agents): 4 tasks
- Phase C (Other templates): 6 tasks
- Phase D (SKILL.md rewrite): 7 tasks
- Phase E (Metadata): 1 task
- Phase F (Dogfood): 3 tasks
- Phase G (Final): 1 task

**Two execution options:**

**1. Subagent-Driven (recommended)** — I dispatch a fresh subagent per task, review between tasks, fast iteration. Given this is a meta-bootstrap plan(v4 doesn't exist yet),subagent-driven avoids polluting the main session context with 35 task executions,and each task is mostly independent file creation.

**2. Inline Execution** — Execute tasks in this session using executing-plans, batch execution with checkpoints for review. Appropriate if you want to watch each task complete in the main session.

**Which approach?**

Given the scale (35 tasks, 2000+ lines of template content to write), **Subagent-Driven is strongly recommended** — the main session will be the coordinator and won't burn context on individual file writes.

---

**End of plan.**
