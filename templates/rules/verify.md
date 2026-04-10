# Verify

Verify 的完整规范 + 项目专属验证命令。

---

## 核心原则

> **Evidence before claims, always.** 没跑过验证命令就说"通过了"是说谎不是效率。

---

## 5 步 Gate Function

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

---

## Evidence 清单

每种 claim 需要对应的证据:

| Claim | 需要的证据 | 不足(不可接受) |
|------|----------|---------------|
| Tests pass | 测试命令输出 0 failures | "上次跑过" / "应该通过" |
| Linter clean | Linter 输出 0 errors | Partial check / extrapolation |
| Build succeeds | Build 命令 exit 0 | "Linter passed"(linter ≠ compiler) |
| Bug fixed | 测原症状通过 | "代码改了假设修好" |
| Regression test works | **Red-green cycle 验证过** | 测试通过一次 |
| **Agent completed** | **VCS diff 显示改动** | **Agent reports "success"(不可信)** |
| Requirements met | Line-by-line checklist | Tests passing alone |

---

## Red-Green Cycle(regression test 必做)

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

---

## 禁止表述

以下表述在**验证前**绝对不可使用:

- "should / probably / seems to"
- "Great / Perfect / Done / Awesome"(在验证命令**跑完之前**)
- 任何 gratitude expression
- 任何 wording imply success without having run the verification

---

## Agent success ≠ success

Subagent 自报"成功"**必须 check VCS diff 才算数**。

理由:agent 可能报告 DONE 但实际没改代码,或改的不是要求的地方。

```
正确做法:
  Agent reports success
    → git diff BASE..HEAD 看实际改动
    → 确认改动和 task 要求一致
    → ONLY THEN 说 "agent completed"

错误做法:
  Agent reports success → "好的,完成了" ← 不可接受
```

---

## 项目验证命令

<!-- 由 bitfrog Phase 2 Q9 产出填充。以下是业务层内容,升级模式下重写。 -->

- **typecheck**: `{typecheck 命令}`
- **lint**: `{lint 命令}`
- **test**: `{test 命令}`
- **build**: `{build 命令}`(如适用)
- **e2e**: `{e2e 命令}`(如适用)
- **组合 verify**: `{组合命令,如 npm run verify}`

每个命令的超时:
- typecheck / lint: 30 秒
- test: 120 秒
- build: 120 秒
- e2e: 300 秒
- 组合 verify: 120 秒
