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
