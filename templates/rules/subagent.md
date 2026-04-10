# Subagent

Subagent 使用的完整规范。

---

## Why Subagents

- **隔离 context**:subagent 有自己的 context,不污染主 session
- **保护主 session 的 context window**:主 session 留给协调工作
- **Never inherit session history**:subagent 拿到精心设计的 prompt,不是对话 history
- **Precisely craft instructions + context**:给 subagent 正好够用的 full text,不多不少

---

## Two-stage Review(硬约束)

每个 task 都走:
1. dispatch **reviewer-spec** → 通过
2. dispatch **reviewer-quality** → 通过
3. 才能勾 task 的 checkbox

**顺序不可颠倒**。详见 `rules/review.md`。

---

## Dispatch Prompt 原则

- **Full text provision**:把 task 的全文 + 相关 context 直接塞进 prompt,**不要让 subagent 自己读 plan 文件**
- **自包含**:所有需要的 context 都在 prompt 里
- **明确 output 格式**:告诉 subagent 返回什么(状态 + 具体内容)
- **明确约束**:"don't change X", "fix tests only", "don't refactor production code"
- **Specific scope**:一个 subagent 一个清晰的 problem domain,不是 "fix all bugs"

---

## Implementer 的 4 种状态处理

Subagent 完成工作后返回四种状态之一:

**DONE**:进 spec review。

**DONE_WITH_CONCERNS**:读 concerns 再决定。
- 关于正确性 / scope 的 concerns → 先解决再 review
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

---

## Model Selection 指南

用能 handle the task 的最便宜 model 来节省成本:

| 任务类型 | Model |
|---------|-------|
| 机械实现(1-2 文件,清晰 spec,isolated function) | **Cheap model** |
| 整合任务(多文件,pattern matching,debug) | **Standard model** |
| 架构、设计、**review(quality)** | **Most capable model** |

**核心**:
- **reviewer-spec** 的工作是机械对照 plan + diff + docs.md#B 触发条件 → 可以用 **cheaper model**(结构化 checking,不需要深度判断)
- **reviewer-quality** 的工作是代码质量 + 模块边界 + 文档红线判定 → 必须 **most capable model**(FAIL 判定的准确性直接影响整个 review loop 的质量)

---

## Parallel Dispatch 规则

**可以 parallel**:
- 3+ 独立 test file failure(不同 root cause)
- 多个独立 subsystem 的 bug
- 独立的 debug 调查

**不可以 parallel**:
- 多个 **implementer** subagent(**永远不行**,会 file conflict)
- Related failures(修一个可能修其他)
- 需要全系统视角的 investigation
- Shared state / 会编辑同一文件

---

## Red Flags

- 并行 dispatch 多个 implementer
- 让 subagent 自己读 plan 文件(应 provide full text)
- 跳过 scene-setting context
- 先 code quality 后 spec compliance(顺序错)
- 让 implementer 的 self-review 代替真正的 review
- Move to next task while review has open issues

---

## Dispatch Prompt 模板

### worker dispatch prompt

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

### reviewer-spec dispatch prompt

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

### reviewer-quality dispatch prompt

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
