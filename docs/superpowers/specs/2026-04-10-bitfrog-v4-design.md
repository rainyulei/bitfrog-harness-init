# bitfrog v4 Design Spec

**Date:** 2026-04-10
**Author:** rainyulei
**Status:** Draft — pending user approval before handoff to writing-plans

---

## 1. Overview

bitfrog v3 生成一套静态的 harness 配置(CLAUDE.md + hooks + agents + feedback-log + decisions.md),解决"AI 在项目里做事时应该遵守哪些硬规则"。它的强项是**机械化检查**:hooks 拦截、verify 循环、reviewer 判定、模块边界。用户在实战中确认这些做得好。

但 v3 有一整层是空的 —— **动态方法论**:
- AI 不知道收到任务后的**第一动作**是什么
- AI 不知道**什么时候** brainstorm、**什么时候**直接动手
- AI 不知道**怎么写 plan**、写什么样的 plan 才能跨 session 执行
- compact 之后 AI 忘了自己在做什么
- AI 经常忘记回写 feedback-log
- 大项目的 CLAUDE.md 变成一堵墙,无法按需加载
- subagent 的调用、review 循环、文档管理都没有规范

bitfrog v4 补齐这一层,同时保留 v3 的机械化检查不变。核心策略是**选择性集成 superpowers 的好部分**,**不集成**它的鸡肋部分。

## 2. Goals and Non-Goals

### 2.1 Goals

1. AI 收到任务后有**明确的第一动作**(复述 + 检查理解),不自信跳过 context
2. AI 有**动态可伸缩的 brainstorm 模式**(动作池 + 退出条件),小任务 30 秒大任务几小时同一个框架
3. AI 写出的 **plan 是自包含、可移交、compact-safe 的合约**
4. AI 每 task 强制走 **two-stage review loop**(spec compliance → code quality)
5. AI 的 **verify 有 5 步 gate**,杜绝 "should pass" 式性能同意
6. AI 的**微 feedback 每 task 必写**(嵌入 plan),深 feedback 由 hook 兜底
7. compact 后 AI **按 session.md + active plan 无缝 resume**
8. 大项目的 CLAUDE.md 精简为**指针表**,方法论细节在 `.claude/rules/*.md` 按需加载
9. 文档管理是**独立层**,代码变动强制同步文档,违规 MAJOR FAIL
10. subagent 使用有**完整规范**(隔离 context / two-stage review / parallel 规则 / 4 种状态处理)

### 2.2 Non-Goals

- **跨 harness 支持**:v4 深度绑定 Claude Code,Cursor / Codex / Windsurf 推后到 v5
- **TDD 原教旨主义**:bitfrog 的测试策略更灵活,不强制每个 bug 先写 failing test
- **自己实现 skill 机制**:不生成 wrapper skill,内容嵌入 CLAUDE.md + rules/,superpowers 只作为"吸收内容的来源"(不 runtime 依赖)
- **完美版本**:v4 追求"解决用户 3 个主要痛点 + 4 个次要痛点",不追求覆盖所有场景

### 2.3 Pain Points → Solution Map

| 用户原话痛点 | v4 解法 |
|-------------|--------|
| 缺开发方法论(load 什么 context / 怎么开发 / 怎么交流 plan) | `rules/workflow.md` + `rules/brainstorm.md` + `rules/plan.md` + 强制第一动作 |
| compact 后任务忘记 | plan 文件作为 source of truth + `session.md` 便签 + compact 协议 |
| 经常忘记 feedback | 微 feedback 嵌入 plan Step 10(不靠自觉)+ 深 feedback hook 兜底 |
| brainstorm 不适用小任务 | brainstorm 重定义为动态模式,退出条件驱动,小任务 30 秒自然退出 |
| debug skill 鸡肋 / 红旗清单没用 | 只借鉴 trace data flow + find working examples,不要四阶段仪式 |
| 大 CLAUDE.md 变长 | 根 CLAUDE.md 只放指针表,`rules/*.md` 按需加载 |
| subagent 怎么驱动 | `rules/subagent.md`(完整规范)+ dispatch prompt 模板 |
| doc 标准散乱 | `rules/docs.md` 独立层 + 工作方式协作表 + reviewer 融合检查 |
| 记忆问题 | `rules/memory.md` + decisions.md 三层(architectural / operational / learned)+ 归档规则 |
| review 层被漏 | `rules/review.md` + two-stage review + reviewer agent 拆分 + Hook 3 commit 兜底 |

---

## 3. Design Philosophy

### 3.1 机械化检查保留,动态方法论新增

v3 的 hooks / verify / reviewer / 模块边界是用户认可的强项,v4 原样保留。v4 新增的是"方法论"层,和机械化检查**并列**,不替换。

### 3.2 选择性集成,不全盘 wrap

superpowers 的好部分(brainstorming 的"一次一个问题"、writing-plans 的 checkbox + no placeholder、subagent-driven-development 的 two-stage review、verification-before-completion 的 evidence-before-claim、systematic-debugging 的 data-flow trace、using-git-worktrees 的 isolation、code-review 的完整循环)吸收到 bitfrog 生成的文件里,**不运行时依赖**。生成的 harness 可以独立工作,不需要用户安装 superpowers。

### 3.3 Brainstorm 是动态模式,不是固定流程

brainstorm 不是"phase",是"**明确目标 ↔ load context** 的耦合螺旋"模式。小任务的 brainstorm 可能只有 30 秒,大任务的 brainstorm 可能几小时。同一套**动作池**和**退出条件**覆盖所有场景。

### 3.4 Plan 是可移交合约,和 session 解耦

plan 不是"执行步骤",是"让执行可以被中断、恢复、验证、甚至被另一个 subagent 接手"的合约。`.claude/plans/*.md` + `.claude/state/session.md` 两者组合实现 compact-safe 工作恢复。

### 3.5 第一动作决定一切

AI 收到任何任务的第一动作永远是**复述 + 检查理解**,不是看 code,不是动手。这是 senior engineer 的本能。强制写进 workflow 入口,不可跳过。

### 3.6 文档是 compact-safe source of truth

AI 在项目里工作的唯一信息源是**文件**,不是对话。对话会被 compact,文件不会。因此文档必须和代码一起演进,文档违规 = MAJOR FAIL,和代码 bug 同级。

### 3.7 Review 结果直接行动,不写报告

review 发现的问题(代码 / 文档)直接修,不生产"review 报告文档"。review 发现的本来就有的 bug 写到 feedback-log 而不是 review 报告。

### 3.8 分层按需加载

根 CLAUDE.md 只放"项目目标 + 骨架指针表",方法论和业务规则按 topic 拆到 `.claude/rules/*.md`,AI 按任务类型从指针表跳到对应 rule 文件。

### 3.9 Hook 不只是拦截,还是硬 checkpoint

v4 的新 hooks 不只做"敏感文件拦截",还用来强制 review 通过才能 commit、强制 worktree 目录被 ignore、强制 plan 文件不含 placeholder、提醒文档同步。用 hook 把"AI 记得做某件事"降级成"AI 不做就被拦",去除靠自觉的不确定性。

### 3.10 骨架/业务分层 + 进化兼容

所有生成物按"骨架层(跨项目不变)"和"业务层(随项目目标变)"分层。升级模式保留骨架层,只重写业务层,不丢历史。

---

## 4. Architecture Overview

### 4.1 生成物目录树

```
project/
├── README.md                          # 用户文档 skeleton(可选生成)
├── CHANGELOG.md                       # 用户文档 skeleton(可选生成)
├── CLAUDE.md                          # 根:项目目标 + 骨架指针表(稳定短小)
├── feedback-log.md                    # 深 feedback 日志(保留 v3)
├── .gitignore                         # 追加 .worktrees/ .claude/state/ .claude/.bitfrog-backup-*/
├── .worktrees/                        # ★ Worktree 隔离区(默认启用)
├── docs/
│   └── modules/                       # 模块接口文档
├── .claude/
│   ├── settings.json                  # hooks(5 骨架 + N 业务)
│   ├── agents/
│   │   ├── worker.md                  # 实现者
│   │   ├── reviewer-spec.md           # ★ Two-stage review #1
│   │   ├── reviewer-quality.md        # 原 reviewer.md 改名:Two-stage review #2
│   │   └── docs.md                    # 文档执行者
│   ├── rules/                         # ★ 分层规则(11 个)
│   │   ├── workflow.md
│   │   ├── brainstorm.md
│   │   ├── plan.md
│   │   ├── review.md
│   │   ├── verify.md
│   │   ├── feedback.md
│   │   ├── subagent.md
│   │   ├── modules.md
│   │   ├── memory.md
│   │   ├── docs.md
│   │   └── worktree.md
│   ├── plans/                         # ★ Plan 文件(可移交合约,git tracked)
│   │   └── YYYY-MM-DD-{slug}.md
│   ├── state/                         # ★ Session 短期便签(gitignored)
│   │   └── session.md
│   └── memory/
│       └── decisions.md               # 三层(architectural / operational / learned)
```

### 4.2 完整方法论流程

```
收到任务
  ↓
[强制] 复述 + 检查理解 (rules/workflow.md)
  ↓
brainstorm 模式(动态,rules/brainstorm.md)
  动作池:
    Context load  : 看 code / trace data flow / find working examples /
                    读 decisions / 读外部文档 / profile / POC / 快扫项目
    目标明确      : 列 2-3 方案 + tradeoff / 问用户(一次一个 + MCQ 优先) /
                    分段呈现设计
    Escalation   : 发现任务过大 → 拆子项目
  退出条件(三勾):
    □ 目标具体到"改哪个文件让它做什么"
    □ context 足够支撑至少 1 个可行方案
    □ 能写出让另一个人照做的 plan
  ↓
[条件] plan (能写 + 需要写, rules/plan.md)
  写到 .claude/plans/YYYY-MM-DD-{slug}.md
  checkbox + bite-sized step + no placeholder
  ↓
[条件] 创建 worktree (触发任一, rules/worktree.md)
  新分支 / 多 commit / 回滚风险 / 跨模块 / 需要 subagent
  ↓
写代码
  ↓
verify (rules/verify.md 5 步 gate)
  ↓
dispatch reviewer-spec (rules/review.md + rules/subagent.md)
  ↓ 有 gap → 回写代码
  ↓ PASS
dispatch reviewer-quality
  ↓ 有 issue → 回写代码
  ↓ PASS
勾 checkbox + 微 feedback (rules/feedback.md) + 文档同步检查 (rules/docs.md#B)
  ↓ 下一 step
所有 checkbox 勾完
  ↓
Finishing Flow (rules/workflow.md):
  跑完整测试 → Determine base → Present 4 options (merge/PR/keep/discard)
  → Execute + Cleanup worktree
  ↓
深 feedback (if 踩坑) → session.md 清理
```

### 4.3 Compact 协议

**Compact 前**(AI 感知上下文窗口接近上限时):
1. 强制更新 `.claude/state/session.md`:active plan / current task / current step / 刚才在想什么 / blocker
2. Hook 兜底:距离上次更新 > 30 min → 提醒

**Compact 后**(或新 session 开工):
1. 读 `.claude/state/session.md`
2. 读 session.md 里 active plan 指向的 `.claude/plans/*.md`
3. 找第一个未勾 checkbox
4. 从那里继续

---

## 5. Methodology Layer — Detailed Design

### 5.1 第一动作(rules/workflow.md)

**强制,不可跳过,对所有任务都做。**

**动作格式**(长短不限,关键在"暴露理解"不在长度):
1. 用自己的话把任务重述 1-3 句
2. 列出你现在理解到的:
   - 目标是什么(尽量具体到"改哪个区域让它做什么")
   - 成功的样子是什么(怎么算做完)
   - 当前的模糊点(心里不确定的地方)
3. 按任务类型决定下一步:

| 任务类型 | 复述后的自然下一步 |
|---------|-----------------|
| **Bug / 故障** | 沿症状链路追:"X 崩了" → "X 由什么触发?" → "触发点前的状态是什么?" → 直到第一个产生坏值的地方 |
| **Feature** | 先找项目里最接近的**正常工作**的同类代码,推断项目模式 |
| **重构** | 先确认"为什么改"的 root cause(用户说"难维护"时,具体是什么难?) |
| **探索性**("让它更快""让它更好") | 先问 metric("更快多少算够"),不问就是瞎优化 |
| **机械性**(改 100 个文件的 API) | 跳过复述之后的动作,直接进 plan |

**退出第一动作**:能对自己说出"我知道目标是 X,我接下来要做 Y"—— 这个动作就完成了,进入动态 brainstorm。说不出来就继续复述 / 问问题,不要急着看 code。

**为什么强制**:senior engineer 收到任务的第一反应就是"我理解你是要 X,对吗?"。AI 倾向于跳过这一步直接看 code,结果 context 看了一堆但目标理解错了。**强制第一动作是复述,把理解偏差在最便宜的时刻暴露**。

### 5.2 Brainstorm 动态模式(rules/brainstorm.md)

**定义**:brainstorm 是"**明确目标**"和"**load context**"两件事**同时发生的那段时间**。它不是一个独立节点,是两个节点的纠缠状态。是一个模式,不是一个 phase。

为什么两者必须同时?因为两件事互相驱动、螺旋推进:
- 看一眼 code → 目标细化一层("优化这个页面" → "消除 N+1")
- 目标细化了 → 知道该 load 什么 context("N+1 相关的 ORM 查询")
- 读 context → 目标又细化一层("改成 eager load 模式")
- 继续循环,直到两件事同时收敛到"目标具体 + context 足够"

#### 5.2.1 退出条件(三勾必须都 ✓)

- [ ] 目标具体到"改哪个文件的哪个函数,让它做什么"
- [ ] context 足够支撑至少 1 个可行的实现方案
- [ ] 能写出让另一个人照做的 plan(即使不写)

**三个 ✓ = brainstorm 结束。** 任何一个 ✗ = brainstorm 还没做够,回到动作池。

#### 5.2.2 动作池(自由组合,无固定顺序)

**Context load 动作**:
- 开工前快扫:项目结构 / 最近 commit / 相关 docs
- 读要改的 code + 直接调用方 + 相关 tests
- **沿 data flow 追**:从症状/坏值往上游,找第一个产生坏值的位置(bug 必做,借鉴 systematic-debugging)
- **找同仓库正常工作的同类代码对比**,列每一处差异(借鉴 systematic-debugging)
- 读 `decisions.md` / 历史注释 / 外部文档
- 跑 profile / log / 复现步骤看现状
- 写 POC 试

**目标明确 动作**:
- **复述 + 检查理解**(强制入口,已在 5.1 定义)
- 列 2-3 个方案,带 tradeoff,明确推荐一个
- 问用户(一次一个问题,优先 multiple choice)
- 分段呈现设计,逐段和用户确认(大任务)

**Escalation 动作**:
- 发现任务是多个独立子系统 → 先拆分再单个 brainstorm

#### 5.2.3 任务类型参照(不是强制流程,是"常见形态")

| 任务类型 | 典型 brainstorm 形态 | 时长 |
|---------|-------------------|------|
| null 崩溃 bug | 看一眼函数 + 看调用方 → 退出 | 30 秒 |
| 加个 API endpoint | 读现有 API 模式 + 读 schema + 列 1 方案 → 退出 | 5 分钟 |
| race condition | 读并发代码 + 列 3 方案 + 可能 POC 复现 → 退出 | 20 分钟 |
| "让这页更快" | 跑 profile + 读瓶颈 + 问 metric + 列方案 → 退出 | 1 小时 |
| 拆 auth 模块 | 读全模块 + 读所有调用方 + 多轮问用户 + 读历史 decisions + 写 POC → 退出 | 几小时多轮 |

**同一套退出条件**,brainstorm 的形态从 30 秒到几小时自然伸缩。不需要 S/M/L 分级。

#### 5.2.4 禁止

- 不要用固定问题清单
- 不要跳过第一动作
- 不要假装退出条件达成("看了 10 行就说我懂了,开始写")
- 不要在小任务上强制走"2-3 approaches"仪式

### 5.3 Plan 格式(rules/plan.md)

**本质**:plan 是**可移交的执行合约**。不是"列步骤",是"让执行可以被中断、恢复、验证、被另一个 subagent 接手"。**自包含** —— 另一次 session 的 AI 读了 plan 就能照做,不需要回看对话。

#### 5.3.1 触发条件(两条件交集)

1. **能写**:brainstorm 退出条件已达成(目标明确 + context 足够)
2. **需要写**:需要自包含执行(任一 ✓ 即触发)
   - 跨 session / 预期 compact 会发生
   - 需要 dispatch 给 subagent 执行
   - 改动风险大,需要每步 checkpoint 可回滚
   - 步骤多到一次 session 做不完
   - 多人协作,plan 是交接物

**能写 + 需要写**:写 plan。
**能写 + 不需要写**:跳过 plan,直接写代码。
**不能写**:brainstorm 没做够,回去。

#### 5.3.2 文件位置和命名

`.claude/plans/YYYY-MM-DD-{slug}.md`

- 日期是创建日期
- slug 是短描述(lowercase + hyphen,如 `auth-refactor`)
- **不**被 gitignore —— plan 是可移交合约,必须 commit 进版本控制

#### 5.3.3 Header(强制格式)

```markdown
# {Feature Name} Implementation Plan

**Goal:** {One sentence describing what this builds}

**Architecture:** {2-3 sentences about approach}

**Tech Stack:** {Key technologies/libraries involved}

---
```

#### 5.3.4 Task 结构(每 task 11 个 step)

```markdown
### Task N: {Component Name}

**Files:**
- Create: `exact/path/to/file.ts`
- Modify: `exact/path/to/existing.ts:123-145`
- Test: `tests/exact/path/to/file.test.ts`

- [ ] **Step 1: Write failing test**(如果适用)
  ```typescript
  test('specific behavior', () => {
    expect(fn(input)).toBe(expected);
  });
  ```

- [ ] **Step 2: Run test**(expected: FAIL)
  Run: `npm test -- tests/path/file.test.ts`
  Expected: FAIL with "fn is not defined"

- [ ] **Step 3: Implement**
  ```typescript
  export function fn(input: X): Y {
    // minimal impl
  }
  ```

- [ ] **Step 4: Run test**(expected: PASS)
  Run: `npm test -- tests/path/file.test.ts`
  Expected: PASS

- [ ] **Step 5: Commit**
  ```bash
  git add tests/path/file.test.ts src/path/file.ts
  git commit -m "feat: add specific feature"
  ```

- [ ] **Step 6: Dispatch reviewer-spec**
  See rules/review.md for request template

- [ ] **Step 7: Fix spec gaps**(loop until PASS)

- [ ] **Step 8: Dispatch reviewer-quality**

- [ ] **Step 9: Fix quality issues**(loop until PASS)

- [ ] **Step 10: 微 feedback 追加到 session.md**
  `date "+%Y-%m-%d %H:%M" | {task#} | {动作} | {意外or无}`

- [ ] **Step 11: 文档同步检查**
  对照 rules/docs.md#B 判断本 task 是否触发文档同步
  如触发,在同一 commit 内更新(已在 Step 5 之前处理)
```

**重要:Step 1-5 是"有测试场景"的示例,不是所有 task 都必须这个形态**。纯重构 / 纯文档 / 纯配置 task 的前 5 个 step 可能完全不同(比如 "Step 1: 读现有实现" / "Step 2: 设计新结构" / "Step 3: 应用改动" / "Step 4: 跑 verify" / "Step 5: Commit")。bitfrog 不强制 TDD。

**但以下 Step 是**强制**的,无论 task 类型**:
- **Step 6-9**: Two-stage review loop(reviewer-spec → fix → reviewer-quality → fix)
- **Step 10**: 微 feedback 追加到 session.md(每 task 必写)
- **Step 11**: 文档同步检查(对照 rules/docs.md#B)

#### 5.3.5 No Placeholder 红线(Hook 5 会自检)

**严禁**以下任一出现在 plan 里:
- `TBD` / `TODO` / `implement later` / `fill in details`
- `Add appropriate error handling` / `add validation` / `handle edge cases`(不具体)
- `Write tests for the above`(没实际 test code)
- `Similar to Task N`(必须重复 code,读者可能 out of order)
- 没 code block 的 "implement" step

#### 5.3.6 Type Consistency 自检

写完 plan 后自查:后面 task 用到的函数名 / 方法签名 / 属性名,和前面 task 定义的是否一致?`clearLayers()` 在 Task 3 和 `clearFullLayers()` 在 Task 7 是 bug。

#### 5.3.7 Plan 生命周期

```
brainstorm 退出 → 判断是否需要 plan
  ↓ 需要
写 plan 文件 → TaskCreate 镜像 plan 的 checkbox
  ↓
执行时:勾 checkbox + 更新 TaskCreate state + 写 session.md
  ↓
遇到 blocker → stop + escalate(不硬刚)
  ↓
所有 checkbox 勾完 → Finishing Flow
  ↓
归档(plan 文件保留在 .claude/plans/ 作为历史)
```

### 5.4 Review Loop(rules/review.md)

#### 5.4.1 为什么要 review 层(第一性理由)

bitfrog v3 的 reviewer agent 是 review 的**执行者**,但没有定义:
- **什么时候**请求 review
- **请求时怎么打包 context**(避免把整个对话塞给 reviewer)
- **收到 review feedback 后怎么响应**(技术严谨 vs 性能性同意)
- **推回 reviewer 的权利**

v4 补上完整的 review **循环**。

#### 5.4.2 强制时机

**强制**:
- 每个 task 完成后(plan 的 Step 6 和 Step 8)
- 重大 feature 完成后
- Merge 前

**可选但有价值**:
- Stuck 时(fresh perspective)
- 重构前 baseline 检查
- 复杂 bug 修完

#### 5.4.3 Two-stage 顺序(硬约束,不可颠倒)

1. **reviewer-spec** 第一阶段:对照 plan 检查 spec compliance + 文档同步
2. **reviewer-quality** 第二阶段(**只在 spec PASS 后**):代码质量 + 模块边界 + 文档质量

**顺序不可颠倒**:quality review 先于 spec 是 red flag。理由:quality 是对"做对的东西"评价质量,spec 不对根本不是在评价同一个东西。

#### 5.4.4 Request Dispatch 原则(借鉴 requesting-code-review)

- **Never inherit session history** — reviewer 拿到 precisely crafted context,不是 requester 的对话
- **保护主 session 的 context window** 给协调工作
- Request 包含:
  - `PLAN_PATH`:plan 文件路径
  - `TASK_NUMBER`:当前 task 编号
  - `BASE_SHA` / `HEAD_SHA`:精确 diff 边界
  - `DESCRIPTION`:简短摘要

完整 dispatch prompt 模板见 Section 7.4。

#### 5.4.5 Receive Feedback 的 6 步 Pattern(借鉴 receiving-code-review)

```
收到 review feedback 时:

1. READ:    完整读完 feedback,不要反应
2. UNDERSTAND: 用自己的话 restate 要求(或者问)
3. VERIFY:  对照 codebase 实际情况
4. EVALUATE: 对 THIS codebase 技术正确吗?
5. RESPOND: 技术确认 或 reasoned pushback
6. IMPLEMENT: 一次一条,每条测
```

#### 5.4.6 禁止性能性同意

**Never**:
- "You're absolutely right!"
- "Great point!" / "Excellent feedback!"
- "Let me implement that now"(未验证前)
- "Thanks for catching that!"
- **任何 gratitude expression**

**Instead**:
- Restate 技术要求
- 问澄清问题
- 推回 with 技术推理(如果错)
- **直接动手 fix,动作胜于话语**

**理由**:如果你要写 "Thanks",DELETE IT,写 "Fixed: {做了什么}"。

#### 5.4.7 YAGNI Check

reviewer 说 "implement X properly"(加字段 / 加 database / 加 export)时:
```
先 grep codebase 看 X 有没有被用:
  - 没用 → "This isn't called. Remove it (YAGNI)?"
  - 有用 → 按 reviewer 说的 implement
```

#### 5.4.8 Unclear Item 优先澄清

如果 review 有多条 feedback,部分懂部分不懂:
- **不要**先做懂的那部分,后问不懂的
- **先**问清楚不懂的,然后一起做
- **理由**:items may be related,部分理解 = 错误实现

#### 5.4.9 Push Back 时机

推回 reviewer 的时机:
- Suggestion 破坏现有功能
- Reviewer 缺完整 context
- YAGNI 违反(unused feature)
- 对 this codebase 技术错误
- 有 legacy / 兼容性原因
- 和 user 之前的架构决策冲突

**推回方法**:
- 技术推理,不 defensive
- 引用具体 tests / code
- 如果推回错了 → "You were right,verified X does Y. Implementing now." **不长篇道歉,不 defending 为什么推回**

#### 5.4.10 Review 结果的处理原则:直接行动,不写报告

**reviewer 返回的 finding 分三类,分别处理**:

**类别 1: 本 task 引入的问题(代码 or 文档)**
- worker 直接修 → re-review → 通过后勾 checkbox
- **不写**"review 报告"文档,不开独立 task
- 理由:问题和 context 都在手上,修比记录便宜

**类别 2: review 发现的是本来就有的 bug(不是本 task 引入的)**
- 写到 `feedback-log.md`(深 feedback)
- 视情况:
  - 小 + 和本 task 相关 → 在本 task 一起修
  - 大 or 和本 task 无关 → 开独立 task 到下一个 plan,不阻塞本 task
- 理由:不让"无关 bug"污染当前 task 的 review 循环

**类别 3: review 发现 plan 本身有问题**
- Stop → escalate 到人(AskUserQuestion 或对话)
- **不**自作主张改 plan
- 理由:plan 是合约,合约有问题要和用户对齐再改

**禁止动作**:
- 不要把 review finding 写成独立的 `review-YYYYMMDD.md` 文件
- 不要把 review 对话塞到 decisions.md
- 不要在 commit message 里复述 reviewer 的所有 finding

### 5.5 Verify Gate(rules/verify.md)

#### 5.5.1 核心原则

**Evidence before claims, always.** 没跑过验证命令就说"通过了"是说谎不是效率。

#### 5.5.2 5 步 Gate Function

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

#### 5.5.3 Evidence 清单

| Claim | 需要的证据 | 不足 |
|------|----------|-----|
| Tests pass | 测试命令输出 0 failures | "上次跑过" / "应该通过" |
| Linter clean | Linter 输出 0 errors | Partial check / extrapolation |
| Build succeeds | Build 命令 exit 0 | "Linter passed"(linter ≠ compiler) |
| Bug fixed | 测原症状通过 | "代码改了假设修好" |
| Regression test works | **Red-green cycle 验证过** | 测试通过一次 |
| **Agent completed** | **VCS diff 显示改动** | **Agent reports "success"(不可信)** |
| Requirements met | Line-by-line checklist | Tests passing alone |

#### 5.5.4 Red-Green Cycle(regression test 必做)

写一个 regression test 时:
```
1. Write failing test
2. Run → must PASS(with fix applied)
3. Revert the fix
4. Run → must FAIL(证明测试真的测到了 bug)
5. Restore the fix
6. Run → must PASS
```

不走 red-green cycle 的 "regression test" 是假的 —— 可能测的是别的东西。

#### 5.5.5 禁止表述

- "should / probably / seems to"
- 在验证前的 "Great / Perfect / Done / Awesome"
- 任何 gratitude expression
- 任何 wording imply success without running verification

#### 5.5.6 Agent success ≠ success

Subagent 自报"成功"必须 check VCS diff 才算数。**理由**:agent 可能报告 DONE 但实际没改代码,或改的不是要求的地方。

### 5.6 Feedback 两阶段(rules/feedback.md)

#### 5.6.1 微 feedback(每 task 必写)

**定位**:task 级状态 log,便宜动作,不是复盘。

**格式**:
```
YYYY-MM-DD HH:MM | task# | {动作} | {意外 (无则"无")}
```

**位置**:`.claude/state/session.md` 的"今日 log"段。

**强制方式**:嵌入 plan 的 **Step 10**(不靠自觉)。

**时机**:每完成一个 task 的 Step 10。

**理由**:便宜动作一定写,累积成低成本的活动 trace,既给未来 session 参考,也是深 feedback 的触发器。

#### 5.6.2 深 feedback(踩坑才写,信息密度高)

**定位**:踩坑后的复盘,为了进化规则。

**位置**:`feedback-log.md`(保留 v3 格式)。

**格式**(保留 v3):
```markdown
### [YYYY-MM-DD] 问题简述
- **场景**: 在做什么时出的问题
- **错误**: 具体错了什么
- **原因**: 为什么会出错
- **修复**: 怎么修的
- **分类**: [模块边界 | 测试 | 类型 | AI 调用 | 其他]
- **规则更新**: 无 / CLAUDE.md 加了 xxx / hook 加了 xxx
```

**触发条件**(任一):
- verify 连续 2 次失败
- 同类问题 2 次出现
- 用户纠正做法("stop doing X")
- 微 feedback 的"意外"字段非空且严重

**Hook 兜底**:verify 失败 ≥ 2 次 && session 未写 deep feedback → 提醒(Hook 6 也可以设计成这个)。

#### 5.6.3 两阶段的升级和归档

**升级**:微 feedback 的"意外"字段非空 → 自动升级触发深 feedback。

**归档**(保留 v3 进化机制):
- feedback-log 同类 3 次 → 沉淀到 rules/ 或 hook
- 深 feedback 归档到 `feedback-log.md#archived` 段,规则沉淀到 CLAUDE.md 或 settings.json

### 5.7 Subagent 使用规范(rules/subagent.md)

#### 5.7.1 Why Subagents

- **隔离 context**:subagent 有自己的 context,不污染主 session
- **保护主 session 的 context window**:主 session 留给协调工作
- **Never inherit session history**:subagent 拿到精心设计的 prompt,不是对话 history
- **Precisely craft instructions + context**:给 subagent 正好够用的 full text,不多不少

#### 5.7.2 Two-stage Review(硬约束)

每个 task 都走:
1. dispatch **reviewer-spec** → 通过
2. dispatch **reviewer-quality** → 通过
3. 才能勾 task 的 checkbox

**顺序不可颠倒**。详见 rules/review.md。

#### 5.7.3 Dispatch Prompt 原则

- **Full text provision**:把 task 的全文 + 相关 context 直接塞进 prompt,**不要让 subagent 自己读 plan 文件**
- **自包含**:所有需要的 context 都在 prompt 里
- **明确 output 格式**:告诉 subagent 返回什么(状态 + 具体内容)
- **明确约束**:"don't change X", "fix tests only", "don't refactor production code"
- **Specific scope**:一个 subagent 一个清晰的 problem domain,不是"fix all bugs"

完整模板见 Section 7.4。

#### 5.7.4 Implementer 的 4 种状态处理

Subagent 完成工作后返回四种状态之一:

**DONE**:进 spec review。

**DONE_WITH_CONCERNS**:读 concerns 再决定。
- 关于正确性/scope 的 concerns → 先解决再 review
- 关于观察的 concerns(如"这个文件越来越大")→ 记录 + 进 review

**NEEDS_CONTEXT**:补 context 后 re-dispatch。

**BLOCKED**:分类处理。
- Context 问题 → 补 context + re-dispatch 同 model
- 需要更多 reasoning → re-dispatch with 更强 model
- Task 太大 → 拆成更小的 task
- Plan 错了 → escalate 给人

**Never**:
- 忽略 escalation
- 同 model 重试但不改任何东西
- 假装 BLOCKED 是 DONE

#### 5.7.5 Model Selection 指南

用能 handle the task 的最便宜 model 来节省成本:

| 任务类型 | Model |
|---------|-------|
| 机械实现(1-2 文件,清晰 spec,isolated function) | **Cheap model** |
| 整合任务(多文件,pattern matching,debug) | **Standard model** |
| 架构、设计、**review(quality)** | **Most capable model** |

**核心**:
- **reviewer-spec** 的工作是机械对照 plan + diff + docs.md#B 触发条件 → 可以用 **cheaper model**(结构化 checking,不需要深度判断)
- **reviewer-quality** 的工作是代码质量 + 模块边界 + 文档红线判定 → 必须 **most capable model**(FAIL 判定的准确性直接影响整个 review loop 的质量)

#### 5.7.6 Parallel Dispatch 规则

**可以 parallel**:
- 3+ 独立 test file failure(不同 root cause)
- 多个独立 subsystem 的 bug
- 独立的 debug 调查

**不可以 parallel**:
- 多个 **implementer** subagent(**永远不行**,会 file conflict)
- Related failures(修一个可能修其他)
- 需要全系统视角的 investigation
- Shared state / 会编辑同一文件

#### 5.7.7 Red Flags

- 并行 dispatch 多个 implementer
- 让 subagent 自己读 plan 文件(应 provide full text)
- 跳过 scene-setting context
- 先 code quality 后 spec compliance(顺序错)
- 让 implementer 的 self-review 代替真正的 review
- Move to next task while review has open issues

---

## 6. Infrastructure Layer

### 6.1 Session State(`.claude/state/session.md`)

**目的**:步骤级短期便签,补充 plan(task 级合约)不够细的地方。支持 compact-resume。

**Gitignored**:session 是本地工作状态,不应 commit。

**内容模板**:

```markdown
# Session State
Last updated: {ISO timestamp}
Active plan: .claude/plans/YYYY-MM-DD-{slug}.md
Current task: #N
Current step: Step M — {description}

## 刚才在想什么
{1-3 句:刚试了什么,下一秒要做什么,未 commit 的临时决定}

## Blocker / 未决定
{如有,列出}

## 今日 log
{task# | 动作 | 意外}

## Review 标记(commit hook 会检查)
task N | spec: approved | quality: approved
```

**更新时机**:
1. 每完成一个 step
2. 离开工位前
3. **Compact 前强制**
4. Review 结果变化时

**读取时机**:
1. Compact 后第一动作
2. 新 session 开工第一动作
3. AI "不知道在做什么"时

### 6.2 Plan Files(`.claude/plans/*.md`)

**目的**:task 级可移交合约。跨 session 恢复的 source of truth。

**Git tracked**:`.claude/plans/` **不**被 gitignore。plan 是项目历史的一部分,要和代码一起版本控制。

**生命周期**:
```
brainstorm 退出 → 判断是否需要 plan
  ↓ 需要
写 plan 文件 → TaskCreate 镜像 checkbox
  ↓
执行 → 勾 checkbox + 写 session.md
  ↓
Blocker → stop + escalate(不硬刚)
  ↓
所有 checkbox 勾完 → Finishing Flow
  ↓
归档(保留在 .claude/plans/ 作为历史)
```

### 6.3 Worktree Infrastructure(`.worktrees/`)

**目的**:大任务隔离工作空间,支持 subagent dispatch + 多 feature 并行 + 回滚安全。

**默认状态**:启用(bitfrog 初始化时创建空目录 + 加入 `.gitignore`)。brainstorm Q12 可以 disable。

**触发条件**(AI 在 plan 阶段判断,任一 ✓ 即创建):
- 需要新 branch(不是 main 直改)
- 预期 ≥ 3 commit
- 实现过程可能回滚
- 跨模块或破坏性变更
- 计划 dispatch subagent 实现 task

**不创建**(任一 ✓):
- 小 bug 修复(1-3 行)
- 文档修改
- 单文件单函数改动
- 纯配置调整

**创建流程**(5 步,借鉴 using-git-worktrees):

1. **Detect project name**
   ```bash
   project=$(basename "$(git rev-parse --show-toplevel)")
   ```

2. **Create worktree**
   ```bash
   git worktree add .worktrees/{branch-name} -b {branch-name}
   cd .worktrees/{branch-name}
   ```

3. **Auto-detect 并跑 project setup**
   ```bash
   if [ -f package.json ]; then npm install; fi
   if [ -f Cargo.toml ]; then cargo build; fi
   if [ -f requirements.txt ]; then pip install -r requirements.txt; fi
   if [ -f pyproject.toml ]; then poetry install; fi
   if [ -f go.mod ]; then go mod download; fi
   ```

4. **Verify clean baseline**
   跑 verify 命令(从 `rules/verify.md` 的业务段拿),tests 必须绿才继续。

5. **Report location**
   ```
   Worktree ready at {full_path}
   Tests passing ({N} tests, 0 failures)
   Ready to implement {feature}
   ```

**Baseline 失败的处理**:stop,报告,问用户是否继续。**不硬刚**。

**Safety**:Hook 4 验证 worktree 父目录被 `.gitignore` ignore,否则阻断。

**Cleanup**(对齐 Finishing Flow 的 4 options):
- Option 1 Merge locally → 清
- Option 2 Push + PR → 保留
- Option 3 Keep as-is → 保留
- Option 4 Discard → 清(需要 typed "discard")

### 6.4 Finishing Flow(rules/workflow.md)

所有 task 的 checkbox 勾完后执行,借鉴 finishing-a-development-branch:

```
Step 1: Verify tests
  跑完整测试套件
  ↓ 失败 → stop, 报告失败, 不进下一步
  ↓ 通过

Step 2: Determine base branch
  git merge-base HEAD main 或 git merge-base HEAD master
  或问用户

Step 3: Present 4 options(不加解释)
  1. Merge back to base locally
  2. Push + Create PR
  3. Keep as-is (稍后处理)
  4. Discard this work
  Which option?

Step 4: Execute choice
  Option 1: checkout base → merge → verify → 删 branch
  Option 2: push -u origin → gh pr create
  Option 3: 报告,保留
  Option 4: 要求 typed "discard" → delete branch

Step 5: Cleanup worktree
  Options 1 & 4: git worktree remove
  Options 2 & 3: 保留

Step 6: Deep feedback if 踩坑

Step 7: Clear session.md active plan
```

---

## 7. Information Architecture & File Specs

### 7.1 根 CLAUDE.md(精简指针表)

**从 v3 的"大而全"改为"项目目标 + 指针表"**:

```markdown
# {项目名}

{一句话描述}

## 项目目标

{brainstorm Q1 的完整回答 — 所有业务规则的依据}
{目标变了,重跑 /bitfrog-harness-init 升级模式}

## 技术栈

| 层 | 技术 | 版本 |
|---|------|------|
| ... | ... | ... |

## 项目结构

{目录树 + 每个目录职责一行}

## 第一动作(强制,所有任务)

收到任务 → **复述 + 检查理解** → 按任务类型决定下一步
详细流程:`.claude/rules/workflow.md`

## 骨架指针表

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

## 进化规则

- `feedback-log.md` 同类 3 次 → 沉淀到 `rules/` 或 hook
- `decisions.md#learned` 每月归档
- 项目目标重大变化 → 重跑 `/bitfrog-harness-init` 升级模式
```

**核心变化**:所有"怎么做"的细节搬到 `rules/*.md`。CLAUDE.md 只保留**稳定不变的东西**(项目目标 / 指针表 / 进化规则)。

### 7.2 Rules 文件规范(11 个)

每个文件给:**定位(一句话)+ 关键章节清单 + 层级(骨架 / 业务 / 混合)**。

#### 7.2.1 `rules/workflow.md` — 骨架

**定位**:开工总流程 + 第一动作 + 完整流程图 + compact 协议。AI 每次 session 的入口。

**章节**:
- 第一动作(强制):复述 + 检查理解 + 按任务类型决定下一步
- 骨架指针表的展开版("去哪读哪个 rule")
- 完整流程图(第一动作 → brainstorm → plan → worktree → 写代码 → verify → review → 勾 checkbox → finishing)
- Compact 协议(compact 前必更新 session.md / compact 后读 session.md 找 active plan)
- 任务收尾 checklist(微 feedback / 深 feedback / 文档同步 / session 清理)

#### 7.2.2 `rules/brainstorm.md` — 骨架

**定位**:brainstorm 的动态模式定义。不是固定流程,是动作池 + 退出条件。

**章节**:
- 定义(明确目标 ↔ load context 的耦合螺旋)
- 退出条件(三勾)
- 动作池(Context load 7 个 + 目标明确 4 个 + Escalation 2 个)
- 任务类型参照表(bug / feature / refactor / explore / 机械性)
- 不要做什么(不要固定问题清单 / 不要跳过第一动作 / 不要假装退出条件达成)

详细内容见 Section 5.2。

#### 7.2.3 `rules/plan.md` — 骨架

**定位**:plan 文件格式 + 生命周期 + 自检。

**章节**:
- 触发条件(能写 + 需要写 两条件交集)
- 文件位置和命名
- Header 格式强制
- Task 结构(11 个 step 的 checkbox 模板)
- No Placeholder 红线
- Type consistency 自检
- Handoff(worker 或 subagent 接手)
- Plan 生命周期

详细内容见 Section 5.3。

#### 7.2.4 `rules/review.md` — 骨架

**定位**:review 循环规范 + 请求/接收规范 + 处理原则。

**章节**:
- 时机(强制 + 可选)
- Two-stage 顺序(spec first → quality after)
- Request template(WHAT / PLAN / BASE_SHA / HEAD_SHA / DESCRIPTION)
- Dispatch 原则(never inherit session history / full text provision)
- Receive 6 步 pattern
- 禁止性能性同意清单
- YAGNI check
- Push back 时机和方法
- **review 结果处理原则**(三类 finding 分流,不写报告)
- Unclear item 优先澄清

详细内容见 Section 5.4。

#### 7.2.5 `rules/verify.md` — 混合(骨架 + 业务)

**定位**:verify 的完整规范 + 项目专属验证命令。

**骨架章节**:
- 5 步 Gate Function
- Evidence 清单表
- 禁止表述
- Red-Green Cycle 模板
- Agent success ≠ success 原则

**业务章节**(brainstorm 产出):
- 项目的具体验证命令(typecheck / lint / test / build / e2e)
- 每个命令的超时和期望退出码
- 组合 verify 命令(`npm run verify` 或等效)

详细内容见 Section 5.5。

#### 7.2.6 `rules/feedback.md` — 骨架

**定位**:微 feedback + 深 feedback 两阶段规范。

**章节**:
- 两阶段定义
- 微 feedback 格式和位置(session.md)
- 微 feedback 强制方式(嵌入 plan Step 10)
- 深 feedback 格式和位置(feedback-log.md)
- 深 feedback 触发条件
- Hook 兜底机制
- 升级规则(微 → 深)
- 归档规则(沉淀到 CLAUDE.md / hook)

详细内容见 Section 5.6。

#### 7.2.7 `rules/subagent.md` — 骨架

**定位**:subagent 使用的完整规范。

**章节**:
- Why subagents
- Two-stage review 顺序
- Dispatch prompt 原则(full text / 自包含 / 明确 output / 明确约束)
- 4 种状态处理(DONE / DONE_WITH_CONCERNS / NEEDS_CONTEXT / BLOCKED)
- Model selection 指南
- Parallel dispatch 规则(implementer 永远不并行)
- Red flags
- Never inherit session history 原则

详细内容见 Section 5.7。

#### 7.2.8 `rules/modules.md` — **业务**(brainstorm 产出)

**定位**:这个项目的模块边界规则。完全由 brainstorm Q2-Q5 产出填充。

**章节**:
- 模块清单(brainstorm Q2 产出)
- 边界规则表(规则 / 含义 / 检测方式 [hook 自动 / reviewer 审查] / 严重度)
- 数据流方向(brainstorm Q4 产出,单向 / 双向约束)
- 外部依赖规范(brainstorm Q5 产出,超时 / 重试 / 兜底)
- 项目专属触发器(改 auth/* 必须读 session.ts 的并发注释 等)

**升级模式完全重写**。

#### 7.2.9 `rules/memory.md` — 骨架

**定位**:decisions.md 三层分层 + 写入/归档规则。

**章节**:
- 三层定义(architectural / operational / learned)
- 每层的写入时机
- 每层的读取时机
- 归档规则

详细内容见 Section 9。

#### 7.2.10 `rules/docs.md` — 混合(骨架 + 业务)

**定位**:完整的文档管理规范 + 工作方式 + 项目专属文档清单。

**骨架章节**:
- **为什么严**(AI 依赖文档作为 compact-safe source of truth)
- 三层读者(用户 / 开发者 / AI)
- 触发时机表(#B)
- 写作红线(#C)
- Review 融合(reviewer-spec 查漏,reviewer-quality 查质量,违规 MAJOR FAIL)
- AI-load 友好性(根 CLAUDE.md 精简 / rules 按 topic / 模块接口文档固定结构)
- Evolution 规则
- **工作方式(角色协作表)**

**业务章节**(brainstorm 产出):
- 本项目的文档清单(README / CHANGELOG / docs/modules/X.md / ARCHITECTURE.md)
- 本项目的文档同步触发器

详细内容见 Section 8。

#### 7.2.11 `rules/worktree.md` — 骨架

**定位**:worktree 基础设施 + 触发条件 + 安全验证。

**章节**:
- When to use(触发条件清单)
- When NOT to use
- Directory 选择(`.worktrees/` 优先)
- 安全验证(`git check-ignore` 强制)
- 创建 5 步
- Baseline 失败的处理
- Cleanup 时机

详细内容见 Section 6.3。

### 7.3 Agents 规范(4 个)

#### 7.3.1 `agents/worker.md`

**定位**:实现者。按 plan 执行 task,每 step verify,完成后 self-review,提交给 review loop。

**Tools**:`Read, Write, Edit, Bash, Grep, Glob, WebSearch, WebFetch`

**主要职责**:
- 开工前读 `CLAUDE.md` + `feedback-log.md` + `decisions.md`
- 按 plan 的 task 逐步执行,严格跟 step 顺序
- 每个 step 跑 verify(引用 `rules/verify.md` 的 5 步 gate)
- Step 11 判断文档同步(引用 `rules/docs.md#B`)
- 遇到 blocker 按 4 种状态报告(引用 `rules/subagent.md`)
- 模块边界遵守(引用 `rules/modules.md`)
- 禁止:跳过 step / 跳过 verify / 注释报错代码 / `@ts-ignore` / 硬刚 3 轮 verify 失败

#### 7.3.2 `agents/reviewer-spec.md`(★ 新增)

**定位**:第一 stage reviewer。对照 plan 检查 spec compliance + 文档同步。

**Tools**:`Read, Grep, Glob, Bash`(只读用)

**主要职责**:
- Dispatch 入参:plan 文件路径 / current task 编号 / BASE_SHA / HEAD_SHA
- 读 plan 的 task 定义 + `git diff BASE..HEAD`
- 核对:
  - task 要求做的事,代码里做了吗?(missing = gap)
  - task 没要求的事,代码里多做了吗?(extra = YAGNI gap)
  - `rules/docs.md#B` 触发的文档,同一 commit 内同步了吗?(漏 = gap)
- 输出:PASS / FAIL 列表 + 每个 gap 的 `file:line` + 如何修
- **禁止**:审代码质量(那是 reviewer-quality 的事)
- **禁止**:看 session history

#### 7.3.3 `agents/reviewer-quality.md`(原 `reviewer.md` 改名 + 扩展)

**定位**:第二 stage reviewer。审查代码质量 + 文档质量 + 模块边界。

**Tools**:`Read, Grep, Glob, Bash`(只读 + verify)

**主要职责**:
- Dispatch 入参同 reviewer-spec
- 前置检查:reviewer-spec 必须已经 PASS(spec 没通过不审 quality)
- 跑 `rules/verify.md` 的项目验证命令,不过直接 FAIL
- 按维度审查:
  - 模块边界(`rules/modules.md` 的规则)
  - 代码质量(项目 CRITICAL / MAJOR / MINOR 维度)
  - 文档质量(`rules/docs.md#C` 红线 → **违规 = MAJOR**)
- 输出格式(保留 v3):
  ```
  Verdict: PASS / FAIL
  [CRITICAL] 问题 + file:line + 修复建议
  [MAJOR] 问题 + file:line + 修复建议
  [MINOR] 建议
  ```
- **判定规则(不可修改)**:
  - Any CRITICAL → FAIL
  - Any MAJOR → FAIL
  - 只有 MINOR → PASS
  - **Doc red line violation = MAJOR**(和代码 bug 同级)

#### 7.3.4 `agents/docs.md`

**定位**:大规模文档重构的执行者(worker 的文档能力不够时 dispatch)。

**Tools**:`Read, Write, Edit, Grep, Glob`

**主要职责**:
- Dispatch 时机:新增模块完整接口文档 / 跨 3+ 文档的重构 / 架构文档大改
- 开工前读:根 CLAUDE.md + 最近 10 条 `decisions.md` + 本次 task 的 `git diff` + 对应的 `docs/modules/<module>.md`
- 严格遵守 `rules/docs.md#C` 红线
- 遵守 `rules/docs.md#E` 的模块接口文档固定结构
- **禁止读** `.claude/rules/` 其他文件(那是 worker 用的)
- 输出:修改的文档列表 + 每个文档的 diff 摘要

### 7.4 Subagent Dispatch Prompt 模板

#### 7.4.1 worker dispatch prompt

```
You are executing Task {task_number} from plan {plan_path}.

**Plan reference:** {plan_path}
**Current task full text:**
{full_task_text_from_plan}

**Context you need:**
- Project goal: {from_claude_md}
- Relevant module boundaries: {from_rules_modules_md}
- Verification commands: {from_rules_verify_md_business_section}
- Known pitfalls for this area: {from_feedback_log_recent_matching}

**Your job:**
1. Execute each step IN ORDER (do not skip, do not reorder)
2. After each code-touching step, run verification per rules/verify.md 5-step gate
3. For Step 11 (doc sync), check rules/docs.md#B triggers against your diff
4. Report status as one of: DONE / DONE_WITH_CONCERNS / NEEDS_CONTEXT / BLOCKED
5. NEVER inherit session history — this prompt is your complete context

**Constraints:**
- Do NOT read .claude/plans/ yourself — this prompt contains the full task text
- Do NOT @ts-ignore or skip tests to force verify pass
- Do NOT attempt more than 3 verify cycles without reporting BLOCKED
- Follow rules/modules.md boundaries strictly

**Output:**
- Git commit(s) with code + doc changes together
- Status: DONE / DONE_WITH_CONCERNS / NEEDS_CONTEXT / BLOCKED
- If concerns/blocked: specific description
```

#### 7.4.2 reviewer-spec dispatch prompt

```
You are the SPEC COMPLIANCE reviewer for Task {task_number}.

**Your ONLY job:** Check whether the code changes satisfy what the plan's task
requires — nothing more, nothing less.

**Plan reference:** {plan_path}
**Task full text:**
{full_task_text_from_plan}

**Diff to review:**
- Base SHA: {base_sha}
- Head SHA: {head_sha}
- Run: git diff {base_sha}..{head_sha}

**Checklist:**
1. Missing: For each requirement in the task, is there code that implements it?
2. Extra: Is there code the task didn't ask for? (YAGNI)
3. Doc sync: Does the diff trigger rules/docs.md#B? If yes, docs updated in SAME commits?

**Constraints:**
- Do NOT review code quality (that's reviewer-quality's job)
- Do NOT read session history — only plan, diff, rules/docs.md#B
- Do NOT assume "what the user probably wanted" — stick to the task text

**Output format:**
Verdict: PASS | FAIL

If FAIL:
  [Missing] <requirement> — <how to fix> at <file:line>
  [Extra] <code> at <file:line> — <why remove>
  [Doc gap] <which doc> — <rules/docs.md#B trigger matched>

If PASS: "Spec compliant — all requirements met, nothing extra, docs synced."
```

#### 7.4.3 reviewer-quality dispatch prompt

```
You are the CODE QUALITY reviewer for Task {task_number}.

**Precondition:** reviewer-spec has already PASSED this task.
If no evidence of a prior spec pass, STOP and report:
"Spec compliance review must run first."

**Your job:** Audit code quality, module boundaries, and doc quality.

**Plan reference:** {plan_path}
**Diff to review:**
- Base SHA: {base_sha}
- Head SHA: {head_sha}

**Dimensions:**
1. Module boundaries (rules/modules.md)
2. Code quality (project's CRITICAL/MAJOR/MINOR dimensions)
3. Doc quality (rules/docs.md#C red lines):
   - Doc RESTATE code instead of explain why?
   - "Will support X in future" placeholder?
   - Code example that cannot actually run?
   - Duplicate content that could drift?
   - Dressing-up words ("comprehensive/robust/nuanced")?

**Verify first:**
Run {verify_command_from_rules_verify_md}.
If verify fails → automatic FAIL, do not proceed.

**Judgment rules (UNMODIFIABLE):**
- Any CRITICAL → FAIL
- Any MAJOR → FAIL
- Only MINOR → PASS
- **Doc red line violation = MAJOR** (same severity as code bugs)

**Output format:**
Verdict: PASS | FAIL

[CRITICAL] <issue> at <file:line> — <fix>
[MAJOR] <issue> at <file:line> — <fix>
[MINOR] <issue> at <file:line> — <optional fix>

If PASS: "Quality approved. Strengths: <1-3 items>. Notes: <observations if any>."

**Constraints:**
- Do NOT re-check spec compliance
- Do NOT read session history
- Do NOT give performative praise — technical observations only
```

---

## 8. Doc Management Layer

**文档管理作为独立层,不是边角。** 这是 v4 相对 v3 的核心补强之一。

### 8.1 为什么这么严(第一性原理)

**AI 在这个项目里工作的唯一信息源是文件,不是对话。** 对话会被 compact 掉,文件不会。

所以:
- AI 下一次 session 能知道这个项目当前状态,**全靠读文档**
- 代码改了文档没改 → 下一次 session 的 AI 读到过期文档 → 做错误的判断 → 埋新 bug
- 文档必须是 **source of truth**,不是"有时间再写"的副产品

因此:任何代码变动,只要触发了 `#B` 的条件,**文档必须在同一 commit 内同步**。文档违规 = **MAJOR → FAIL**,和代码 bug 同级,不降级不放水。

### 8.2 三层读者(硬分层,不可混)

| 层 | 读者 | 文档 | 位置 |
|---|-----|-----|-----|
| **给用户的** | 跑这个项目的人 | README / 使用示例 / CHANGELOG | 根目录 |
| **给开发者的** | 改这个项目的人 | dev setup / CONTRIBUTING / 运行验证 | 根目录或 `docs/` |
| **给 AI 的** | 下一次 session 的 AI | CLAUDE.md / `.claude/rules/*.md` / `decisions.md` / 模块接口 | `.claude/` 和 `docs/modules/` |

**每类文档严禁混内容**:README 不写 AI-load 的指针表,CLAUDE.md 不写用户跑项目的命令。

### 8.3 触发时机(硬触发 #B)

触发条件 → 必须同步的文档:

| 代码里发生的事 | 必须同步的文档 |
|---|---|
| 新增/修改公开接口(exported function, public API, command flag) | `docs/modules/<module>.md` 或 README 的命令段 |
| 架构决策(选了 X 而不是 Y,且有 tradeoff) | `.claude/memory/decisions.md` 追加一条 |
| 破坏性变更 | `CHANGELOG.md` + 迁移说明 |
| 环境/依赖变更 | dev setup 段 |
| 模块边界调整 | `.claude/rules/modules.md` + 相关模块的接口文档 |
| 发现"这个项目 AI 容易踩的坑" | `feedback-log.md`(见 rules/feedback.md) |
| 本次 session 的 task 没动任何公开面 | **不写文档**(不强制) |

**对应 plan 的 Step 11**:task 收尾时判断是否触发,触发则在同一 commit 内更新。

### 8.4 写作红线 #C(强制禁止)

1. **禁止 restate 代码**:不写 `"This function parses JSON"`,代码自己说。写 `"We parse JSON here because the upstream API only supports text/plain and gives us a JSON-shaped string; we need to re-interpret it"`
2. **禁止提前文档**:不写未实现的接口、"将来会支持 Y"、TODO 占位
3. **代码示例必须可运行**:写了示例就要能 copy-paste 跑起来。不能运行的示例 → 删掉或改成可运行的
4. **禁止重复**:一个事实只在一处说,其他地方引用,不 copy paste
5. **禁止装腔作势**:`comprehensive / robust / nuanced / leveraging` 这类词直接删
6. **禁止无署名决定**:decisions.md 每条决定必须有日期和触发原因
7. **禁止文档独立于代码的 commit**:文档更新必须和对应代码在同一个 commit

### 8.5 Review 融合 #D

**reviewer-spec 职责扩展**:对照 plan 检查 spec compliance 时,**同时检查**本 task 的代码改动是否触发了 `#B` 的任一条件,如触发,对应文档是否在同一 commit 内更新。

文档缺失 = spec gap = 不通过。

**reviewer-quality 职责扩展**:审查代码质量时,**同时审查**本 commit 涉及的文档是否违反 `#C` 的红线。如违反 → **MAJOR**(和代码的 MAJOR 同级)→ FAIL。

### 8.6 AI-load 友好性 #E

**根 CLAUDE.md** 永远保持精简(项目目标 + 骨架指针表),不写细节。

**模块接口文档** 放在 `docs/modules/<module>.md`,每个模块一个文件,结构固定:

```markdown
# Module: <name>

## 职责(一句话)
## 对外接口(函数签名 + 用途 + 示例)
## 依赖(上游 / 下游)
## 边界(不允许做什么 — 对应 rules/modules.md)
## 已知陷阱(来自 feedback-log.md 沉淀)
```

**docs agent 被 dispatch 时**,它只读:
1. 根 CLAUDE.md
2. `.claude/memory/decisions.md`(最近 10 条)
3. 当前 task 的 git diff
4. 对应的 `docs/modules/<module>.md`

**不读** `.claude/rules/` 其他文件(那是 worker 用的)。

### 8.7 Evolution 规则 #F

保持 bitfrog 现有的 feedback-log 驱动 evolution,文档层新增:
- **某个文档被 AI 多次误读或漂移超过 3 次** → 这个文档的结构有问题 → 进入升级模式 brainstorm 重写该文档

### 8.8 工作方式(角色协作表)

| 角色 | 在文档维度上做什么 | 不做什么 |
|-----|-----------------|---------|
| **worker** | 写代码时同步维护本 task 涉及的文档(plan Step 11 触发)。只做**小范围同步改动** — 改一两行、追加一个接口说明、追加 CHANGELOG 一行 | 不做整段重写、不动别的 task 的文档 |
| **docs agent** | 被 worker 或 workflow **主动 dispatch** 时出场,做**整段/整个文档的重构或重写** | 不参与日常小修,不自动跑 |
| **reviewer-spec** | 按 `rules/docs.md#B` 检查:代码改了触发条件的东西,对应文档是否在**同一 commit 内**更新 | 不审文档质量,只审"漏没漏" |
| **reviewer-quality** | 按 `rules/docs.md#C` 红线检查文档质量 | 不重复 reviewer-spec 的"漏没漏"检查 |

**worker 自修 vs dispatch docs agent 的边界**:

| 情况 | 选哪个 | 理由 |
|-----|-------|-----|
| 改动只涉及"加一行""改一个参数说明""追加 CHANGELOG 一条" | worker 自修 | 上下文在手上 |
| 新增整个模块 → 写全新 `docs/modules/<module>.md` | dispatch docs agent | docs agent 有专门模板 |
| 重构影响 3+ 个文档 | dispatch docs agent | 跨文档一致性需要独立 context |
| 只是 typo / 格式 | worker 自修 | 不值得 dispatch |

### 8.9 文档维度在 two-stage review 里的触发顺序

```
worker 写代码
  ↓ 改动命中 rules/docs.md#B 任一触发条件?
    是 → 决定 小修 or 大改
           小修 → worker 自己改(Step 11)
           大改 → dispatch docs agent(Step 11 的替代路径)
    否 → 跳过 Step 11
  ↓
commit(代码 + 文档 同一 commit,强制)
  ↓
reviewer-spec:文档同步了吗?(漏 = spec gap)
  ↓ 通过
reviewer-quality:文档红线违反了吗?(违反 = MAJOR → FAIL)
  ↓ 通过
勾 checkbox
```

---

## 9. Memory Layering(rules/memory.md)

**decisions.md 三层分层**,避免"一个文件变墙"。

### 9.1 Layer 1: Architectural(长期,不归档)

**内容**:
- 选了什么技术栈和为什么
- 核心架构模式(monolith / microservice / layered / event-driven)
- 重大 tradeoff(选 X 不选 Y,Y 的代价)
- 不可变约束(如"必须 SOC2 compliant")

**写入时机**:
- brainstorm 对齐的架构选择
- 项目重大转型
- 用户明确说"这是架构决定"

**格式**:
```markdown
## [Arch-001] 选 Postgres 而非 SQLite
- **日期**: 2026-04-10
- **决策**: 用 Postgres 作为主数据库
- **原因**: 需要 full-text search 和 JSONB
- **放弃的替代**: SQLite(不支持 FTS5 over 10GB)
- **影响**: 所有 repository 层、部署复杂度 +1
- **目标变更**: 无
```

**不归档**:architectural 层永远保留全部历史。

### 9.2 Layer 2: Operational(中期,季度 review)

**内容**:
- 模块用哪个 library(axios vs fetch / zod vs joi)
- 项目里某个 pattern 的具体写法(错误处理约定 / 命名约定 / 测试文件结构)
- 非架构级但影响日常实现的决定
- 工具链选择(eslint config / prettier 风格)

**写入时机**:
- plan 执行中决定的实现模式(worker 在 Step 11 写)
- brainstorm 对齐但未到架构级的决定
- reviewer 多次指出同一问题,沉淀为约定

**格式**:
```markdown
## [Op-023] API 响应统一用 envelope 格式
- **日期**: 2026-03-15
- **约定**: `{ data: T, error: null }` 或 `{ data: null, error: ErrorDetail }`
- **原因**: 客户端错误处理一致
- **适用范围**: 所有 REST endpoint
- **例外**: 文件下载(直接返回 binary)
- **状态**: active
```

**归档规则**:
- 每季度(或每次升级模式)review
- 过时的标注 `status: deprecated`,不删除
- 升级到架构级,复制到 architectural,operational 标注 `promoted to Arch-XXX`

### 9.3 Layer 3: Learned(短期,定期归档)

**内容**:
- feedback-log 同类 3 次后沉淀的短规则
- "这个地方 AI 容易踩的坑"
- 具体的"别这么写"经验

**写入时机**:
- feedback-log 同类 3 次 → 升级为 learned
- 进化机制自动触发(或 reviewer-quality 发现重复问题提出)

**格式**:
```markdown
## [Learn-007] auth 模块的 token refresh 不要嵌套 try-catch
- **日期**: 2026-04-05
- **规则**: auth/refresh.ts 的 try-catch 只包 network call,不包 parse
- **原因**: 嵌套 catch 会吃掉真实错误,改了 3 次才发现
- **对应 feedback-log 条目**: [2026-04-01] [2026-04-03] [2026-04-05]
- **状态**: active
```

**归档规则**:
- 每月归档
- 超过 3 个月没被触发的 `active` 降为 `archived`
- learned 归档到 `feedback-log.md` 的"已归档"段
- 规则够普适 → 进 CLAUDE.md 或 hook

### 9.4 读取时机

| 时机 | 读哪层 |
|-----|-------|
| brainstorm 前(建立约束) | **architectural**(全部)+ **operational**(相关模块) |
| load context 阶段 | **operational**(相关模块)+ **learned**(相关模块) |
| 写代码卡住时 | **learned**(相关模块)— "之前踩过什么坑" |
| reviewer-quality 审查时 | **operational**(约定)+ **learned**(重犯检查) |
| compact 后 resume | session.md → active plan → **architectural**(不变约束) |

### 9.5 写入的硬动作(嵌入 workflow)

在 task 收尾 checklist 里加两条:

```
- [ ] 本 task 是否产生新的 architectural 决策?(有 → decisions.md#architectural)
- [ ] 本 task 是否产生新的 operational 约定?(有 → decisions.md#operational)
```

learned 层不在 task 收尾时写,由 feedback-log 同类 3 次自动触发升级。

---

## 10. Skeleton vs Business Layer Map

升级模式的 source of truth。表中每个文件按层级分类:

| 文件 | 层级 | 升级时处理 | 原因 |
|-----|-----|----------|------|
| **CLAUDE.md(根)** | 混合 | 骨架指针表 + 进化规则保留;项目目标 + 技术栈 + 项目结构可能重写 | 项目目标变化是升级触发,必须更新;指针表是方法论骨架 |
| **feedback-log.md** | 历史 | **全部保留**,只追加 | 历史是资产 |
| **decisions.md** | 历史 | **全部保留**,追加新决策,新决策标注"目标变更" | 历史决策是 context |
| `rules/workflow.md` | **骨架** | 不动 | 方法论入口,跨项目通用 |
| `rules/brainstorm.md` | **骨架** | 不动 | 方法论,跨项目通用 |
| `rules/plan.md` | **骨架** | 不动 | 格式规范,跨项目通用 |
| `rules/review.md` | **骨架** | 不动 | review 循环规范 |
| `rules/verify.md` | 混合 | 5 步 Gate + 证据清单骨架保留;项目验证命令段重写 | Gate 通用;具体命令随目标变 |
| `rules/feedback.md` | **骨架** | 不动 | 两阶段规范 |
| `rules/subagent.md` | **骨架** | 不动 | subagent 规范 |
| `rules/modules.md` | **业务** | **完全重写** | 模块边界跟着项目目标走 |
| `rules/memory.md` | **骨架** | 不动 | 分层策略通用 |
| `rules/docs.md` | 混合 | 六子章骨架保留;本项目文档清单 + 同步触发器段重写 | 写作红线通用;项目文档清单是业务 |
| `rules/worktree.md` | **骨架** | 不动 | 基础设施规范 |
| `agents/worker.md` | 混合 | 流程骨架保留;模块边界描述段重写 | 执行规范通用;项目边界是业务 |
| `agents/reviewer-spec.md` | **骨架** | 不动 | spec 审查流程通用 |
| `agents/reviewer-quality.md` | 混合 | 判定规则 + 流程骨架保留;项目审查维度段重写 | CRITICAL/MAJOR/MINOR 判定不可变;项目维度是业务 |
| `agents/docs.md` | **骨架** | 不动 | 文档写作职责通用 |
| `settings.json - hooks` | 混合 | 骨架 hooks(1-5)保留;业务 hooks(6+)重写 | 骨架是通用保护;业务随边界变 |
| `.worktrees/` | 历史 | 保留 | 用户可能有进行中的 worktree |
| `.claude/plans/` | 历史 | 保留 | 进行中的 plan 不能丢 |
| `.claude/state/session.md` | 历史 | 保留 | compact 状态 |
| `docs/` 下的文档 | 历史 | 保留 | 文档是历史 |
| `README / CHANGELOG` | 历史 | 保留 | 用户文档不动 |

**升级模式的 Phase 4.5(新增)**:bitfrog 升级时先 ★ 备份一份骨架层副本到 `.claude/.bitfrog-backup-YYYYMMDD-HHMMSS/`,Phase 5 验证通过才删。

---

## 11. Hooks(5 骨架 + N 业务)

Hook 1-5 是骨架层(升级时保留不动),Hook 6 及以后是业务层(升级时重写)。

### 11.1 Hook 1: 敏感文件拦截(v3 保留)

```json
{
  "matcher": "Write|Edit",
  "hooks": [{
    "type": "command",
    "command": "FILE=\"$CLAUDE_TOOL_INPUT_FILE_PATH\"\nif echo \"$FILE\" | grep -qE '\\.(env|pem|key|secret|credentials)$'; then\n  echo '[BLOCKED] 敏感文件禁止 AI 修改' >&2\n  echo '原因: 防止泄露密钥和凭证' >&2\n  echo '修复: 手动编辑该文件' >&2\n  exit 1\nfi",
    "timeout": 5
  }]
}
```

### 11.2 Hook 2: git commit 前 verify(v3 保留)

```json
{
  "matcher": "Bash",
  "hooks": [{
    "type": "command",
    "if": "Bash(git commit*)",
    "command": "cd {项目绝对路径} && {verify 命令} 2>&1",
    "timeout": 120,
    "statusMessage": "提交前验证"
  }]
}
```

### 11.3 Hook 3: git commit 前 review 通过检查(★ 新增)

```json
{
  "matcher": "Bash",
  "hooks": [{
    "type": "command",
    "if": "Bash(git commit*)",
    "command": "SESSION='.claude/state/session.md'\nif [ ! -f \"$SESSION\" ]; then exit 0; fi\nTASK=$(grep -E '^Current task:' \"$SESSION\" | head -1 | sed 's/.*: //')\nif [ -z \"$TASK\" ] || [ \"$TASK\" = '(none)' ]; then exit 0; fi\nif ! grep -qE \"task ${TASK}.*spec: approved\" \"$SESSION\"; then\n  echo '[BLOCKED] 当前 task 的 spec compliance review 未通过' >&2\n  echo '原因: 必须先 dispatch reviewer-spec 通过才能 commit' >&2\n  echo '修复: 按 rules/review.md 跑 two-stage review' >&2\n  exit 1\nfi\nif ! grep -qE \"task ${TASK}.*quality: approved\" \"$SESSION\"; then\n  echo '[BLOCKED] 当前 task 的 code quality review 未通过' >&2\n  echo '原因: spec 通过后必须 dispatch reviewer-quality' >&2\n  exit 1\nfi",
    "timeout": 5
  }]
}
```

**意图**:两个 reviewer 都 approved 才能 commit。`session.md` 没有 active task 时放行(简单场景不阻塞)。

### 11.4 Hook 4: git worktree add 前 ignore 验证(★ 新增)

```json
{
  "matcher": "Bash",
  "hooks": [{
    "type": "command",
    "if": "Bash(git worktree add*)",
    "command": "CMD=\"$CLAUDE_TOOL_INPUT\"\nTARGET=$(echo \"$CMD\" | grep -oE 'git worktree add [^ ]+' | awk '{print $4}' | xargs dirname)\nif [ -z \"$TARGET\" ] || [ \"$TARGET\" = '.' ]; then exit 0; fi\nif ! git check-ignore -q \"$TARGET\" 2>/dev/null; then\n  echo \"[BLOCKED] worktree 目标目录 $TARGET 未被 .gitignore ignore\" >&2\n  echo '原因: 防止 worktree 内容污染主仓库' >&2\n  echo \"修复: echo '$TARGET/' >> .gitignore && git add .gitignore && git commit -m 'chore: ignore $TARGET'\" >&2\n  exit 1\nfi",
    "timeout": 5
  }]
}
```

### 11.5 Hook 5: Write `.claude/plans/*.md` 后 plan 格式自检(★ 新增)

```json
{
  "matcher": "Write|Edit",
  "hooks": [{
    "type": "command",
    "if": "matches(.claude/plans/*.md)",
    "command": "FILE=\"$CLAUDE_TOOL_INPUT_FILE_PATH\"\nif ! grep -qE '^\\*\\*Goal:\\*\\*' \"$FILE\"; then\n  echo '[WARN] plan 缺少 **Goal:** 段' >&2\n  exit 0\nfi\nif grep -qE '(TBD|TODO|implement later|add appropriate error handling|similar to Task [0-9])' \"$FILE\"; then\n  echo '[BLOCKED] plan 含 No Placeholder 红线违规' >&2\n  echo '原因: rules/plan.md 禁止 TBD/TODO/implement later/similar to Task N/add appropriate error handling' >&2\n  echo '修复: 把 placeholder 展开成具体内容或删除该 task' >&2\n  exit 1\nfi",
    "timeout": 5
  }]
}
```

### 11.6 Hook 6: 文档同步警告(★ 新增,业务层,Q11 可选)

```json
{
  "matcher": "Write|Edit",
  "hooks": [{
    "type": "command",
    "if": "matches({project_source_glob})",
    "command": "FILE=\"$CLAUDE_TOOL_INPUT_FILE_PATH\"\nif grep -qE '^(export |public |def |func )' \"$FILE\" 2>/dev/null; then\n  MOD=$(basename \"$(dirname \"$FILE\")\")\n  if [ ! -f \"docs/modules/$MOD.md\" ]; then\n    echo \"[WARN] 文件含公开接口但 docs/modules/$MOD.md 不存在\" >&2\n    echo '提醒: rules/docs.md#B 要求新增公开接口时同步文档' >&2\n  fi\nfi\nexit 0",
    "timeout": 5
  }]
}
```

**注意**:这个 hook 只警告(exit 0),不阻断。Q11 答"不要"时不生成。

### 11.7 所有 hook 的 stderr 约定

所有骨架 hook 的 stderr 输出遵循 **三要素**:
1. **违反了什么**
2. **为什么**
3. **怎么改**

---

## 12. bitfrog SKILL.md 本身的 Phase 增量

### 12.1 Phase 1: 探索项目上下文(扩展)

**保留 v3**:
- 读 package.json / pyproject.toml 等
- 读现有 CLAUDE.md / .claude/
- `git log --oneline -20`
- 扫目录结构
- 读 README
- 检测用户熟练度

**新增 v4**:
- 扫 `.worktrees/` 是否存在;`git check-ignore .worktrees/` 验证
- 扫 `.claude/rules/` 是否存在(升级模式检测)
- 扫 `.claude/plans/` 和 `.claude/state/` 是否存在
- 扫 `docs/` 和 `docs/modules/`
- 扫 `README.md` 和 `CHANGELOG.md`
- **升级模式额外**:读取 `.claude/rules/*.md` 现有内容,按 B.3 映射表识别骨架/业务段
- **升级模式额外**:备份骨架层到 `.claude/.bitfrog-backup-YYYYMMDD-HHMMSS/`

### 12.2 Phase 2: Brainstorm(问题扩展)

**保留 v3 的 Q1-Q10**。

**新增 v4 问题**:

- **Q11 文档习惯**(rules/docs.md 的业务段输入):
  - 项目有 README / CHANGELOG 吗?需要生成 skeleton 吗?
  - 模块接口文档要按 `docs/modules/<module>.md` 拆还是集中在 ARCHITECTURE.md?
  - 有没有特定 commit 触发条件必须同步文档?
- **Q12 Worktree 偏好**:
  - 本项目要启用 worktree 吗?(默认启用 + 条件触发 / 询问后启用 / 显式 disable)
- **Q13 初始 architectural decisions**(memory.md 初始填充):
  - 项目有哪些已经定下来的架构选择值得记到 `decisions.md#architectural`?
- **Q14 Project-specific triggers**(rules/brainstorm.md 的项目专属触发器 + rules/modules.md):
  - 哪几个目录/模块改动时必须先读某些上下文或必须问你?

**升级模式下跳过 Q1** 如果用户说"项目目标不变"。

### 12.3 Phase 3: 确认设计(分段扩展)

**保留 v3 的 4 段**(CLAUDE.md 结构 / Hooks 设计 / Agent 定义 / 初始决策)。

**新增 v4 段**:
- 段 5:Rules 文件清单(11 个,骨架/业务标注)
- 段 6:Plan / Session state / Worktree 基础设施
- 段 7:文档 skeleton(README / CHANGELOG / docs/modules/)
- 段 8:Two-stage review 循环(reviewer 拆分)

### 12.4 Phase 4: 生成配置(大扩展)

**保留 v3**(4.1-4.7):CLAUDE.md / settings.json / reviewer-quality.md / docs.md / worker.md / feedback-log.md / decisions.md

**新增 v4**:

**4.8 生成 rules/*.md(11 个)**
- 骨架部分 copy 模板
- 业务部分从 brainstorm 结果拼装

**4.9 生成 reviewer-spec.md**

**4.10 生成目录骨架**
```bash
mkdir -p .claude/plans
mkdir -p .claude/state
mkdir -p .worktrees          # 只在 Q12 答启用时
mkdir -p docs/modules
```

**4.11 生成 session.md 初始内容**(空模板)

**4.12 更新 .gitignore**
```
.worktrees/
.claude/state/
.claude/.bitfrog-backup-*/
```
**不** ignore `.claude/plans/`(plan 是可移交合约,要 commit)。

**4.13 生成 README / CHANGELOG skeleton**(仅 Q11 答"要"且文件不存在时)

**4.14 CLAUDE.md 模板大改**(Section 7.1 的精简指针表风格)

### 12.5 Phase 5: 验证(大扩展)

**保留 5.1-5.6**(v3 的结构 / hooks 分层 / hook 可用性 / 交叉引用 / reviewer 判定规则 / 升级兼容性)。

**新增 5.7-5.11**:

**5.7 Rules 文件验证**:
- [ ] `.claude/rules/` 下 11 个文件都存在
- [ ] 每个 rule 文件含定义的关键章节
- [ ] 骨架 rule 文件内容和 bitfrog 模板一致
- [ ] 业务 rule 文件(modules.md / verify.md 业务段 / docs.md 业务段)含 brainstorm 产出
- [ ] CLAUDE.md 骨架指针表引用的 rules/*.md 都真实存在

**5.8 Two-stage review 拆分验证**:
- [ ] `.claude/agents/reviewer-spec.md` 存在
- [ ] `.claude/agents/reviewer-quality.md` 存在
- [ ] reviewer-quality.md 判定规则段未被修改
- [ ] plan 模板的 task 结构含 Step 6-9 的 two-stage review loop
- [ ] Hook 3 存在且 if 条件正确

**5.9 基础设施验证**:
- [ ] `.claude/plans/` 目录存在
- [ ] `.claude/state/session.md` 存在且有模板内容
- [ ] `.worktrees/` 存在(Q12 启用时)
- [ ] `.gitignore` 含 `.worktrees/` `.claude/state/` `.claude/.bitfrog-backup-*/`
- [ ] `.gitignore` **不**含 `.claude/plans/`
- [ ] `docs/modules/` 目录存在

**5.10 文档 skeleton 验证**(Q11 答"要"时):
- [ ] `README.md` 存在
- [ ] `CHANGELOG.md` 存在
- [ ] 如果是 bitfrog 生成的,原文件不存在才生成(不覆盖)

**5.11 Memory 分层验证**:
- [ ] `decisions.md` 含 `## Architectural` / `## Operational` / `## Learned` 三个段
- [ ] brainstorm Q13 的初始决策已写入对应段
- [ ] `rules/memory.md` 存在且定义了三层

---

## 13. Upgrade Mode

对应 Section 10 的映射表,升级模式 7 步:

### Step U1: 检测和备份

```bash
if [ -f CLAUDE.md ] && grep -q "项目目标" CLAUDE.md; then
  UPGRADE_MODE=true
fi

BACKUP_DIR=".claude/.bitfrog-backup-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$BACKUP_DIR"
cp -r .claude/rules/ "$BACKUP_DIR/rules/" 2>/dev/null || true
cp -r .claude/agents/ "$BACKUP_DIR/agents/" 2>/dev/null || true
cp CLAUDE.md "$BACKUP_DIR/" 2>/dev/null || true
cp .claude/settings.json "$BACKUP_DIR/" 2>/dev/null || true
```

### Step U2: 识别骨架 vs 业务层

按 Section 10 映射表分类,输出给用户 diff 报告(骨架保留 / 业务重写 / 历史追加)。

### Step U3: 重新 brainstorm(只跑业务相关问题)

跳过 Q1 如果用户说目标不变;跑 Q2-Q14 里和业务层相关的。

开场白:
> "检测到已有 harness 配置。我会保留骨架层(workflow / brainstorm / plan / review / feedback / subagent / memory / worktree / reviewer-spec / docs agent / 骨架 hooks),只更新业务层(模块边界 / 验证命令 / 文档清单 / reviewer-quality 的项目审查维度 / 业务 hooks)。先问几个问题。"

### Step U4: 重写业务层(只动业务部分)

- `rules/modules.md` 完全重写
- `rules/verify.md` 只替换业务段
- `rules/docs.md` 只替换业务段
- `agents/worker.md` 只替换模块边界段
- `agents/reviewer-quality.md` 只替换项目审查维度段,**判定规则段绝对不动**
- `settings.json` 只替换业务 hook,骨架 hook 原位不动

### Step U5: 追加新 decisions(标注"目标变更")

```markdown
## [Arch-XXX] 项目目标变更
- **日期**: YYYY-MM-DD
- **触发**: 运行 bitfrog 升级模式
- **原目标**: (备份里的 CLAUDE.md 项目目标)
- **新目标**: (brainstorm Q1 新回答)
- **影响**: 更新的业务层文件列表
- **目标变更**: 是
```

### Step U6: 跑 Phase 5 验证

全部 5.1-5.11 跑一遍,特别是 5.6 升级兼容性:
- [ ] 骨架层内容未被修改(和备份 diff)
- [ ] 历史 decisions 未被删除
- [ ] 历史 feedback-log 未被删除
- [ ] 新决策标注了"目标变更"

### Step U7: 清理备份

Phase 5 全部通过才删 `$BACKUP_DIR`。失败 → 保留备份,报告失败项,让用户决定是 rollback 还是修复。

**Rollback 命令**(打印给用户,不自动跑):
```
⚠ 升级验证失败,骨架层已备份。

如需回滚:
  rm -rf .claude/rules .claude/agents CLAUDE.md .claude/settings.json
  cp -r {BACKUP_DIR}/* .
  rm -rf {BACKUP_DIR}

或修复失败项后手动删除备份:
  rm -rf {BACKUP_DIR}
```

---

## 14. 从 superpowers 吸收了什么、没吸收什么

| Superpowers Skill | 吸收? | 吸收内容 |
|------------------|-------|---------|
| **brainstorming** | 部分 | 动作池(探索 context / 一次一个问题 / MCQ 优先 / 2-3 方案 / 分段确认 / 拆子项目)+ "不继承 session history" 原则。**不吸收**固定问题清单和 "design approval gate"(换成退出条件)。 |
| **writing-plans** | 大部分 | Plan 文件格式(header / checkbox / bite-sized / no placeholder / type consistency / self-review)。**不吸收**强制 TDD 前两步(改成"如适用")。 |
| **executing-plans** | 部分 | Load plan 后的 critical review + stop-when-blocked 原则。**不用**它作为 runtime skill。 |
| **subagent-driven-development** | 大部分 | Fresh subagent per task / two-stage review / 4 种状态处理 / model selection / parallel 规则 / "不让 subagent 读 plan,provide full text" 原则。**不吸收**强制 TDD 默认。 |
| **requesting-code-review** | 全部 | 时机 / request template / push back 权利 / reviewer 拿 precisely crafted context。 |
| **receiving-code-review** | 全部 | 6 步 pattern / 禁止性能性同意 / YAGNI check / unclear 优先澄清。 |
| **verification-before-completion** | 全部 | 5 步 gate / evidence 清单 / agent success ≠ success。 |
| **finishing-a-development-branch** | 全部 | 4 options / test verify gate / cleanup rules。 |
| **using-git-worktrees** | 全部 | Directory 选择优先级 / check-ignore 验证 / 5 步创建 / baseline 验证。 |
| **dispatching-parallel-agents** | 全部 | 独立 domain 规则 / implementer 永远不并行 / debug 调查可并行 / prompt 结构。 |
| **systematic-debugging** | 2 个动作 | "沿 data flow 追" + "找同仓库 working examples 对比"。**不吸收** Four Phases 仪式 / TDD 强制 / Red Flags 清单 / Iron Law 术语。 |
| test-driven-development | ❌ | bitfrog 的测试策略更灵活,hook + verify + reviewer 已覆盖更好。 |
| writing-skills | ❌ | bitfrog 不生成 wrapper skill。 |
| using-superpowers | ❌ | 红旗表/反例表,用户说没用。 |

---

## 15. v3 → v4 的产出物 diff

### 15.1 新增文件

- `.claude/rules/workflow.md`
- `.claude/rules/brainstorm.md`
- `.claude/rules/plan.md`
- `.claude/rules/review.md`
- `.claude/rules/verify.md`
- `.claude/rules/feedback.md`
- `.claude/rules/subagent.md`
- `.claude/rules/modules.md`
- `.claude/rules/memory.md`
- `.claude/rules/docs.md`
- `.claude/rules/worktree.md`
- `.claude/agents/reviewer-spec.md`
- `.claude/state/session.md`
- `.claude/plans/` (目录)
- `.worktrees/` (目录)
- `docs/modules/` (目录)
- `README.md`(skeleton,如果没有)
- `CHANGELOG.md`(skeleton,如果没有)

### 15.2 改名

- `.claude/agents/reviewer.md` → `.claude/agents/reviewer-quality.md`

### 15.3 内容大改

- `CLAUDE.md` — 从"大而全"改为"项目目标 + 骨架指针表"(Section 7.1)
- `.claude/agents/worker.md` — 扩展,引用 `rules/subagent.md`
- `.claude/memory/decisions.md` — 从单层改为三层(Section 9)
- `.claude/settings.json` — hooks 从 2 条扩到 6+ 条(Section 11)
- `.gitignore` — 追加 `.worktrees/` `.claude/state/` `.claude/.bitfrog-backup-*/`

### 15.4 内容微调

- `feedback-log.md` — 保留格式,文案提示升级到两阶段

### 15.5 删除

- 无(v3 所有文件都保留或改名)

---

## 16. Migration Plan(v3 → v4)

1. bitfrog repo 升级 `SKILL.md` 到 v4(按 Section 12 的 Phase 增量)
2. 在 bitfrog repo 自己上测试 `/bitfrog-harness-init` 升级模式(dogfood)
3. 修复发现的问题
4. 在一个其他项目上测试初始化模式
5. 发布 v4(tag + CHANGELOG)

---

## 17. Success Criteria

1. 在一个新项目上跑 `/bitfrog-harness-init`,生成所有 v4 的产出物,Phase 5 验证全通过
2. 在一个已有 v3 harness 的项目上跑升级模式,骨架层保留不变,业务层重写,历史保留,rollback 路径可用
3. 在实际使用中,AI 的行为变化(可观察):
   - 收到任务第一动作是**复述**(而不是直接看 code)
   - 小任务**不走 brainstorm 的仪式**(30 秒退出)
   - 大任务**写 plan 文件**,compact 后能 resume
   - 每 task 走 **two-stage review**(不自己决定跳过)
   - **微 feedback 每 task 必写**(session.md 有记录)
   - **文档和代码同 commit**(reviewer-spec 兜底)
4. feedback-log.md 驱动的进化机制继续工作(和 v3 一致)

---

## 18. Open Questions and Risks

### 18.1 Risk: Hook 3 的 active task 标记机制

Hook 3 检查 session.md 是否有 "task N | spec: approved | quality: approved"。如果 session.md 被手动清空或没更新,hook 会放行(因为 Current task 为 none)—— 这是故意的,避免误伤简单场景。但也意味着"有意绕过 review"是可能的。

**判断**:接受这个 trade-off。bitfrog 的哲学是"提供兜底,不剥夺用户自由"。用户显式绕过 ≠ AI 忘记。

### 18.2 Risk: 11 个 rule 文件 AI 真的会按需读吗

CLAUDE.md 的指针表是否足够明确让 AI 正确选择读哪个?

**缓解**:
- 初始版本通过 dogfood 观察
- feedback-log 记录"AI 读错 rule / 漏读 rule"的情况
- 同类 3 次进化出 hook 或 CLAUDE.md 硬规则

### 18.3 Risk: 升级模式的骨架/业务边界误判

升级时 bitfrog 需要准确识别"这一段是骨架 / 那一段是业务"。如果用户手动改了骨架层,升级会覆盖用户的改动。

**缓解**:
- 备份骨架层到 `.bitfrog-backup-YYYYMMDD-HHMMSS`
- Phase 5 验证失败不删备份
- 提供 rollback 命令

### 18.4 Risk: Plan 文件膨胀

每 task 11 个 step,大 feature 可能有几十个 task,plan 文件变大。

**缓解**:
- plan 按 feature 拆分(大 feature 拆子 plan)
- `.claude/plans/` 是 flat 目录,按日期排序,归档不删

### 18.5 Risk: Two-stage review 的 overhead

每 task 都走 reviewer-spec + reviewer-quality,subagent 调用 × 2,成本和时间翻倍。

**缓解**:
- Model selection 指南(reviewer-spec 可以用较 cheap 的 model,reviewer-quality 用 most capable)
- 只在 task 级别 review,不在 step 级别
- 小 bug 修复可以跳过 plan → 跳过 review loop(适用于 S 级任务)

### 18.6 Open Question: docs agent 的 dispatch 时机量化

rules/docs.md#D 说"小修 worker 自修,大修 dispatch docs agent",但"多大算大"没量化。

**提议**:留给 feedback-log 驱动 —— 初版用"涉及 3+ 文档 or 整段重构"作为模糊判断,踩坑后细化。

### 18.7 Open Question: Worktree disable 的 UX

Q12 答"disable"时,workflow 流程图里的 worktree 节点怎么处理?

**提议**:生成的 workflow.md 里 worktree 段用 conditional markdown:
```markdown
## Worktree(本项目已 disable)
项目在 brainstorm Phase 2 Q12 选择了不启用 worktree。
所有 task 直接在当前 branch 执行。
```

---

## 19. Out of Scope(明确不做)

- **跨 harness 支持**(Cursor / Codex / Windsurf):推后到 v5
- **自动化的 plan → task 拆分工具**:留给 writing-plans skill 做
- **文档质量的 LLM 评分**:reviewer-quality 按红线 grep 判断即可,不需要 LLM 评分
- **Plan 归档自动压缩**:plans/ 目录自然增长,不做压缩
- **Session state 多分支支持**:一次只支持一个 active plan,多 worktree 时每个 worktree 独立 session.md
- **性能 benchmark**:v4 是方法论补强,不涉及 runtime performance
- **国际化**:CLAUDE.md 模板固定中文(bitfrog 用户是中文使用者),英文内容保留用于技术标识符

---

## 20. Appendix: bitfrog v3 → v4 的核心 mental model 变化

### 20.1 v3 的 mental model

> "bitfrog 生成一套**规则文件**(CLAUDE.md + hooks + agents),AI 做事时遵守这些规则。"

规则是静态的,AI 是被动的。

### 20.2 v4 的 mental model

> "bitfrog 生成一套**规则 + 方法论 + 基础设施**,AI 做事时有明确的**工作流**(第一动作 → 动态 brainstorm → 条件 plan → 条件 worktree → 写代码 → verify → two-stage review → feedback → finishing),方法论嵌在**按需加载的 rules 文件**里,状态在**compact-safe 的 plan + session 文件**里,兜底在**硬 hooks**里。"

规则 + 方法论 + 基础设施 三件事,AI 是**有工作流的**协作者。

---

**End of spec.**
