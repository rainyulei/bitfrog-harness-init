# Worktree

Worktree 基础设施 + 触发条件 + 安全验证。

---

## When to use

触发条件(任一 ✓ 即创建 worktree):

- 需要新 branch(不是 main 直改)
- 预期 ≥ 3 commit
- 实现过程可能回滚
- 跨模块或破坏性变更
- 计划 dispatch subagent 实现 task

---

## When NOT to use

- 小 bug 修复(1-3 行)
- 文档修改
- 单文件单函数改动
- 纯配置调整

---

## Directory 选择

`.worktrees/` 是默认位置(bitfrog 初始化时创建 + 加入 `.gitignore`)。

优先级:
1. 已存在的 `.worktrees/` 目录
2. CLAUDE.md 里的 worktree 配置(如果有)
3. 问用户

---

## 安全验证

**project-local worktree 目录必须被 `.gitignore` 覆盖**,否则 worktree 内容会污染主仓库。

检查命令:
```bash
git check-ignore -q .worktrees 2>/dev/null
```

如果未被 ignore:Hook 4 会阻断 `git worktree add` 命令并提示修复:
```bash
echo '.worktrees/' >> .gitignore
git add .gitignore
git commit -m "chore: ignore .worktrees/"
```

---

## 创建 5 步

### Step 1: Detect project name

```bash
project=$(basename "$(git rev-parse --show-toplevel)")
```

### Step 2: Create worktree

```bash
git worktree add .worktrees/{branch-name} -b {branch-name}
cd .worktrees/{branch-name}
```

### Step 3: Auto-detect 并跑 project setup

```bash
if [ -f package.json ]; then npm install; fi
if [ -f Cargo.toml ]; then cargo build; fi
if [ -f requirements.txt ]; then pip install -r requirements.txt; fi
if [ -f pyproject.toml ]; then poetry install; fi
if [ -f go.mod ]; then go mod download; fi
```

### Step 4: Verify clean baseline

跑 verify 命令(从 `rules/verify.md` 的"项目验证命令"段拿),tests 必须绿才继续。

**如果 baseline 失败**:stop,报告失败,问用户是否继续。**不硬刚**。

### Step 5: Report location

```
Worktree ready at {full_path}
Tests passing ({N} tests, 0 failures)
Ready to implement {feature}
```

---

## Baseline 失败的处理

测试不绿 → **stop**,报告失败内容,问用户:

1. 继续(已知的旧 bug,不影响新工作)
2. 先修 baseline,再继续
3. 放弃 worktree,回主 workspace

**不要假设 baseline 一定绿**。不绿时不能区分"新 bug vs 旧 bug"。

---

## Cleanup 时机

对齐 Finishing Flow(`rules/workflow.md`)的 4 options:

| Option | 操作 | Worktree 处理 |
|--------|-----|-------------|
| 1. Merge locally | checkout base → merge → verify → 删 branch | **清 worktree** |
| 2. Push + PR | push → gh pr create | **保留**(PR 可能需要改) |
| 3. Keep as-is | 报告,保留 | **保留** |
| 4. Discard | 需要 typed "discard" 确认 → delete branch | **清 worktree** |

清 worktree 命令:
```bash
git worktree remove .worktrees/{branch-name}
```
