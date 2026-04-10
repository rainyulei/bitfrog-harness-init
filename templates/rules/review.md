# Review

Review 循环规范 + 请求/接收规范 + 处理原则。

---

## 为什么要 review 层

bitfrog v3 的 reviewer agent 是 review 的**执行者**,但没有定义:
- **什么时候**请求 review(时机)
- **请求时怎么打包 context**(避免把整个对话塞给 reviewer)
- **收到 review feedback 后怎么响应**(技术严谨 vs 性能性同意)
- **推回 reviewer 的权利和方法**

v4 补上完整的 review **循环**。

---

## 强制时机

**强制**:
- 每个 task 完成后(plan 的 Step 6 和 Step 8)
- 重大 feature 完成后
- Merge 前

**可选但有价值**:
- Stuck 时(fresh perspective)
- 重构前 baseline 检查
- 复杂 bug 修完

---

## Two-stage 顺序(硬约束,不可颠倒)

1. **reviewer-spec**(第一阶段):对照 plan 检查 spec compliance + 文档同步
2. **reviewer-quality**(第二阶段,**只在 spec PASS 后**):代码质量 + 模块边界 + 文档质量

**顺序不可颠倒**:quality review 先于 spec 是 red flag。理由:quality 是对"做对的东西"评价质量,spec 不对根本不是在评价同一个东西。

---

## Request Dispatch 原则

- **Never inherit session history** — reviewer 拿到 precisely crafted context,不是 requester 的对话
- **保护主 session 的 context window** 给协调工作
- Request 包含:
  - `PLAN_PATH`:plan 文件路径
  - `TASK_NUMBER`:当前 task 编号
  - `BASE_SHA` / `HEAD_SHA`:精确 diff 边界
  - `DESCRIPTION`:简短摘要

完整 dispatch prompt 模板见 `rules/subagent.md`。

---

## Receive Feedback 的 6 步 Pattern

```
收到 review feedback 时:

1. READ:      完整读完 feedback,不要反应
2. UNDERSTAND: 用自己的话 restate 要求(或者问)
3. VERIFY:    对照 codebase 实际情况
4. EVALUATE:  对 THIS codebase 技术正确吗?
5. RESPOND:   技术确认 或 reasoned pushback
6. IMPLEMENT: 一次一条,每条测
```

---

## 禁止性能性同意

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

> 如果你要写 "Thanks",**DELETE IT**,写 "Fixed: {做了什么}"。

---

## YAGNI Check

reviewer 说 "implement X properly"(加字段 / 加 database / 加 export)时:

```
先 grep codebase 看 X 有没有被用:
  - 没用 → "This isn't called. Remove it (YAGNI)?"
  - 有用 → 按 reviewer 说的 implement
```

---

## Unclear Item 优先澄清

如果 review 有多条 feedback,部分懂部分不懂:

- **不要**先做懂的那部分,后问不懂的
- **先**问清楚不懂的,然后一起做
- **理由**:items may be related,部分理解 = 错误实现

---

## Push Back 时机和方法

### 什么时候推回

- Suggestion 破坏现有功能
- Reviewer 缺完整 context
- YAGNI 违反(unused feature)
- 对 this codebase 技术错误
- 有 legacy / 兼容性原因
- 和 user 之前的架构决策冲突

### 怎么推回

- 技术推理,不 defensive
- 引用具体 tests / code
- 如果推回错了 → "You were right, verified X does Y. Implementing now."
  **不长篇道歉,不 defending 为什么推回**

---

## Review 结果的处理原则:直接行动,不写报告

reviewer 返回的 finding 分三类,每类的处理方式不同 —— **不产生"review 报告"这种中间文档**。

### 类别 1:本 task 引入的问题(代码 or 文档)

→ worker 直接修 → re-review → 通过后勾 checkbox
→ **不写**"review 报告"文档,不开独立 task
→ 理由:问题和 context 都在手上,修比记录便宜

### 类别 2:review 发现的是本来就有的 bug(不是本 task 引入的)

→ 写到 `feedback-log.md`(深 feedback)
→ 视情况:
  - 小 + 和本 task 相关 → 在本 task 一起修
  - 大 or 和本 task 无关 → 开独立 task 到下一个 plan,不阻塞本 task
→ 理由:不让"无关 bug"污染当前 task 的 review 循环

### 类别 3:review 发现 plan 本身有问题

→ Stop → escalate 到人(AskUserQuestion 或对话)
→ **不**自作主张改 plan
→ 理由:plan 是合约,合约有问题要和用户对齐再改

### 禁止动作

- 不要把 review finding 写成独立的 `review-YYYYMMDD.md` 文件(review 是流程不是产物)
- 不要把 review 对话塞到 `decisions.md`(那是架构决策不是修复记录)
- 不要在 commit message 里详细复述 reviewer 的所有 finding(commit 只说结果不说过程)
