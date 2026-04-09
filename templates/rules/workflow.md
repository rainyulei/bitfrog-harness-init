# Workflow

开工总流程 + 第一动作 + 完整流程图 + compact 协议。AI 每次 session 的入口。

---

## 第一动作(强制,不可跳过)

**收到任何任务的第一个动作永远是"复述 + 检查理解",不是看 code,不是写 plan,不是动手。**

### 动作格式(长短不限,关键在"暴露理解"不在长度)

1. 用自己的话把任务重述 1-3 句
2. 列出你现在理解到的:
   - **目标** 是什么(尽量具体到"改哪个区域让它做什么")
   - **成功的样子** 是什么(怎么算做完)
   - **当前的模糊点**(心里不确定的地方)
3. 按任务类型决定下一步:

| 任务类型 | 复述后的自然下一步 |
|---------|---------------------|
| **Bug / 故障** | 沿症状链路追:"X 崩了" → "X 由什么触发?" → "触发点前的状态是什么?" → 直到第一个产生坏值的地方 |
| **Feature** | 先找项目里最接近的**正常工作**的同类代码,推断项目模式 |
| **重构** | 先确认"为什么改"的 root cause(用户说"难维护"时,具体是什么难?) |
| **探索性**("让它更快""让它更好") | 先问 metric("更快多少算够"),不问就是瞎优化 |
| **机械性**(改 100 个文件的 API) | 跳过复述之后的动作,直接进 plan |

### 退出第一动作

能对自己说出"我知道目标是 X,我接下来要做 Y"—— 这个动作就完成了,进入动态 brainstorm(见 `rules/brainstorm.md`)。

说不出来就继续复述 / 问问题,**不要急着看 code**。

### 为什么强制

Senior engineer 收到任务的第一反应就是"我理解你是要 X,对吗?"。AI 倾向于跳过这一步直接看 code,结果 context 看了一堆但目标理解错了 —— 这就是"过早自信"的根源。

**强制第一动作是复述,就把理解偏差在最便宜的时刻暴露。** compact 会丢对话记忆,但文件(包括 AI 写下的复述)不丢 —— 这也是 compact-safe 的第一环。

---

## 完整流程图

```
收到任务
  ↓
[强制] 复述 + 检查理解(rules/workflow.md 本文件)
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
[条件] plan(能写 + 需要写, rules/plan.md)
  写到 .claude/plans/YYYY-MM-DD-{slug}.md
  checkbox + bite-sized step + no placeholder
  ↓
[条件] 创建 worktree(触发任一, rules/worktree.md)
  新分支 / 多 commit / 回滚风险 / 跨模块 / 需要 subagent
  ↓
写代码
  ↓
verify(rules/verify.md 5 步 gate)
  ↓
dispatch reviewer-spec(rules/review.md + rules/subagent.md)
  ↓ 有 gap → 回写代码
  ↓ PASS
dispatch reviewer-quality
  ↓ 有 issue → 回写代码
  ↓ PASS
勾 checkbox + 微 feedback(rules/feedback.md)+ 文档同步检查(rules/docs.md#B)
  ↓ 下一 step
所有 checkbox 勾完
  ↓
Finishing Flow(本文件下方 + rules/worktree.md):
  跑完整测试 → Determine base → Present 4 options(merge/PR/keep/discard)
  → Execute + Cleanup worktree
  ↓
深 feedback(if 踩坑)→ session.md 清理
```

---

## 骨架指针表

按任务场景跳到对应 rule 文件:

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
| compact 前 | 本文件 `## Compact 协议` 段 | ✓ |
| compact 后 | 读 `.claude/state/session.md` → 找 active plan | ✓ |

---

## Compact 协议 {#compact-protocol}

compact 会丢对话上下文,但 `.claude/state/session.md` 和 `.claude/plans/*.md` 是文件,不会丢。**compact 前必须把状态持久化到 session.md,compact 后从 session.md 恢复**。

### Compact 前(AI 感知上下文窗口接近上限时)

1. **强制更新** `.claude/state/session.md`:
   - `Active plan`: 当前在跑的 plan 文件路径
   - `Current task`: 当前 task 编号
   - `Current step`: 当前 step 描述
   - `## 刚才在想什么`: 1-3 句,刚试了什么 / 下一秒要做什么 / 未 commit 的临时决定
   - `## Blocker / 未决定`: 如有
2. Hook 兜底:距离上次更新 > 30 min → 提醒

### Compact 后(或新 session 开工)

1. 读 `.claude/state/session.md`
2. 读 session.md 里 `Active plan` 指向的 `.claude/plans/*.md`
3. 找第一个未勾 checkbox
4. 从那里继续,不需要猜

**Compact-resume 的 source of truth 永远是文件,不是"我记得刚才在做什么"。** 如果 session.md 说 `Current step: Step 3`,就是 Step 3,哪怕你"感觉"像是 Step 5。

---

## 任务收尾 checklist

每个 task 的所有 step 勾完之后,在勾最后一个 checkbox 之前,执行以下 checklist:

- [ ] **微 feedback**:在 `.claude/state/session.md` 的"今日 log"段追加一行
      `YYYY-MM-DD HH:MM | task# | {动作} | {意外(无则"无")}`
      见 `rules/feedback.md`
- [ ] **深 feedback**(条件):如果本 task 中有"意外"字段非空、verify 连续 2 次失败、或用户纠正过做法 → 在 `feedback-log.md` 写完整条目
- [ ] **文档同步**(条件):按 `rules/docs.md#B` 检查本 task 的代码改动是否触发文档同步。触发则在**同一 commit** 内更新对应文档(worker 自修 or dispatch docs agent)
- [ ] **Architectural 决策写入**(条件):本 task 是否产生新的 architectural 决策?(选了技术栈 / 架构模式 / 不可变约束)→ 写入 `.claude/memory/decisions.md#architectural`。见 `rules/memory.md`
- [ ] **Operational 约定写入**(条件):本 task 是否产生新的 operational 约定?(某个 library / pattern / 命名约定)→ 写入 `.claude/memory/decisions.md#operational`
- [ ] **Session 清理**(整个 plan 完成时):所有 task 勾完后,清空 `.claude/state/session.md` 的 `Active plan` / `Current task` / `Current step` 字段,保留"今日 log"作为历史记录

**上述 checklist 不是建议,是硬动作**。清单不过 → 不算 task 完成 → 不能勾 checkbox。

---

## Finishing Flow

当一个 plan 的所有 task 的所有 checkbox 都勾完时,执行 finishing flow:

1. **Verify tests**: 跑完整测试套件,失败 → stop,报告,不进下一步
2. **Determine base branch**: `git merge-base HEAD main` 或 `git merge-base HEAD master`,或问用户
3. **Present 4 options**(不加解释):
   1. Merge back to base locally
   2. Push + Create PR
   3. Keep as-is(稍后处理)
   4. Discard this work
4. **Execute choice**(见 `rules/worktree.md` 的 Cleanup 时机段)
5. **Cleanup worktree**(Options 1 & 4)
6. **Deep feedback** if 有坑
7. **Clear session.md** 的 active plan 字段
