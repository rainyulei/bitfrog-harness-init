---
name: bitfrog-harness-init
description: "Use when setting up harness engineering for a project — generates CLAUDE.md (pointer-table style), .claude/rules/ (11 rule files), .claude/agents/ (4 agents with two-stage review), hooks, feedback-log, decisions (3-tier), session state, and plans infrastructure through collaborative brainstorming."
---

# BitFrog Harness Init

通过深度 brainstorm 理解项目后，生成一套完整的 harness engineering 配置：**动态方法论**（第一动作 / brainstorm / plan / review / verify / feedback）+ **静态规则**（模块边界 / hooks）+ **基础设施**（worktree / session state / plans / memory 三层）+ **文档管理层**。

<HARD-GATE>
在完成 brainstorm 并获得用户确认之前，不允许写任何 harness 文件。必须先彻底理解项目，再生成配置。不管项目看起来多简单，都要走完 brainstorm 流程。
</HARD-GATE>

## 运行模式

### 初始化模式（新项目）
项目没有 `.claude/rules/` 和 `CLAUDE.md` → 完整走 Phase 1-5，生成全部文件。

### 升级模式（已有 harness，项目目标变化）
检测到已有 `.claude/rules/` 或 CLAUDE.md 含"项目目标"章节 → 进入升级模式。
详见本文件底部"升级模式"段。

## 输出物

```
project/
├── CLAUDE.md                          # 根：项目目标 + 骨架指针表（精简）
├── feedback-log.md                    # 深 feedback 日志（进化机制）
├── README.md                          # 用户文档 skeleton（可选生成）
├── CHANGELOG.md                       # changelog skeleton（可选生成）
├── .gitignore                         # 追加 .worktrees/ .claude/state/ .claude/.bitfrog-backup-*/
├── .worktrees/                        # worktree 隔离区（默认启用，Q12 可 disable）
├── docs/
│   └── modules/                       # 模块接口文档
├── .claude/
│   ├── settings.json                  # hooks（5 骨架 + N 业务）
│   ├── agents/
│   │   ├── worker.md                  # 实现 agent（可能多个）
│   │   ├── reviewer-spec.md           # spec compliance reviewer（必有）
│   │   ├── reviewer-quality.md        # code quality reviewer（必有）
│   │   └── docs.md                    # 文档 agent（必有）
│   ├── rules/                         # 分层规则（11 个）
│   │   ├── workflow.md                # 第一动作 + 流程图 + compact 协议
│   │   ├── brainstorm.md              # 动作池 + 退出条件
│   │   ├── plan.md                    # plan 格式 + 11-step task 结构
│   │   ├── review.md                  # review loop + 6-step pattern
│   │   ├── verify.md                  # 5 步 gate + evidence 清单
│   │   ├── feedback.md               # 微/深 feedback 两阶段
│   │   ├── subagent.md               # subagent 规范 + dispatch 模板
│   │   ├── modules.md                # 模块边界规则（业务层）
│   │   ├── memory.md                 # decisions 三层分层
│   │   ├── docs.md                   # 文档管理完整规范
│   │   └── worktree.md              # worktree 基础设施
│   ├── plans/                         # plan 文件（git tracked，可移交合约）
│   │   └── YYYY-MM-DD-{slug}.md
│   ├── state/                         # session 短期便签（gitignored）
│   │   └── session.md
│   └── memory/
│       └── decisions.md               # 三层：Architectural / Operational / Learned
```

**不生成 Plans.md** — harness 不做项目管理，只管代码质量和方法论。

**reviewer-spec.md、reviewer-quality.md 和 docs.md 是所有项目的基础层，必须生成。** worker 的数量和分工由 brainstorm 决定。

---

## Phase 1: 探索项目上下文

在提问之前，先自己搞清楚：

1. **读项目文件** — package.json / pyproject.toml / go.mod / Cargo.toml（判断语言和依赖）
2. **读现有配置** — 如果已经有 CLAUDE.md、.claude/，判断是初始化模式还是升级模式
3. **读 git log** — `git log --oneline -20`，了解项目在做什么、开发节奏
4. **扫目录结构** — `ls -R src/` 或 `find . -type f -name '*.ts' -o -name '*.py' | head -50`
5. **读 README** — 如果有的话
6. **检测用户熟练度** — 根据用户的用词、提问深度、技术术语使用情况，调整后续沟通的技术深度。不要问用户"你是什么水平"，而是从对话中推断

### v4 新增扫描

7. **扫 `.worktrees/`** — 是否存在？`git check-ignore .worktrees/` 验证是否被 ignore
8. **扫 `.claude/rules/`** — 是否存在？（升级模式检测）
9. **扫 `.claude/plans/` 和 `.claude/state/`** — 是否存在？
10. **扫 `docs/` 和 `docs/modules/`** — 是否存在？
11. **扫 `README.md` 和 `CHANGELOG.md`** — 是否存在？

### 升级模式额外探索

**如果检测到升级模式**：
- 读取 `.claude/rules/*.md` 的现有内容，按"升级模式"段的骨架/业务映射表识别每个文件的骨架段和业务段
- **备份骨架层**到 `.claude/.bitfrog-backup-YYYYMMDD-HHMMSS/`
- 明确告诉用户："检测到已有 harness 配置。我会保留骨架层（workflow / brainstorm / plan / review / feedback / subagent / memory / worktree / reviewer-spec / docs agent / 骨架 hooks），只更新业务层（模块边界 / 验证命令 / 文档清单 / reviewer-quality 的项目审查维度 / 业务 hooks）。"

把发现总结为一段话告诉用户。

---

## Phase 2: Brainstorm（核心）

一次一个问题，理解清楚再问下一个。优先用选择题，但需要深挖时用开放题。

### 必问问题（按顺序）

**Q1: 项目目标**
"这个项目的最终目标是什么？不是功能列表，是它解决什么问题、给谁用。"

这个回答会写入 CLAUDE.md 的"项目目标"章节，作为所有业务规则的依据。

**Q2: 架构分层**
基于 Phase 1 看到的代码结构，提出你理解的模块划分：
"我看到代码大概分成了[xxx/yyy/zzz]这几层，我理解的职责是[...]。这个理解对吗？有没有我漏掉的边界？"

**Q3: 模块边界**
"这些模块之间，哪些调用关系是你想严格控制的？"

如果用户不确定，基于项目类型给出具体建议（参考下方"边界模式库"）。

**Q4: 数据流方向**
"数据在这些模块之间是怎么流动的？有没有单向要求？比如 A→B→C，C 不能反向调用 A。"

**Q5: 外部依赖**
"项目有哪些外部依赖需要特殊处理？比如：第三方 API（需要超时/重试/兜底）、数据库（需要 migration 规范）、文件系统（需要路径约束）。"

**Q6: 测试现状**
"现在有测试吗？用什么框架？你希望这套 harness 对测试有什么要求？"

选项：
- A. 测试先行（TDD，先写测试再写实现）
- B. 实现后补测试（每个 PR 必须带测试）
- C. 关键路径测试（只测核心逻辑，不要求全覆盖）
- D. 目前没测试，先不加测试要求

**Q7: 工作方式**
"你平时写这个项目，代码从开始到提交，中间经过几步？比如：一个人从头写到尾？前后端可以同时写？必须先做 A 再做 B？"

根据回答推断协作模式：
- "一个人写完提交" → 单 worker
- "前端后端可以同时写" → 多 worker 并行（fan-out）
- "必须先写 API 再写前端" → 流水线（pipeline）
- "不同模块不同人" → 专家池（expert-pool）

**不要向用户暴露模式名称。** 用他们自己的语言确认。

**Q8: 已知的坑**
"这个项目里，AI（或你自己）之前反复犯的错有哪些？"

这些直接写入初始的 feedback-log.md。

**Q9: 验证命令**
"项目现在跑验证用什么命令？类型检查、lint、测试分别怎么跑？"

如果没有，根据技术栈推荐并确认。

**Q10: 提交规范**
"commit message 有没有格式要求？分支策略是什么？"
- A. conventional commits（feat/fix/refactor/...）
- B. 自由格式，但要求一句话说清楚改了什么
- C. 有其他规范

### v4 新增问题

**Q11: 文档习惯**
"关于文档：
1. 项目有 README / CHANGELOG 吗？要我生成 skeleton 吗？
2. 模块接口文档按 `docs/modules/<module>.md` 拆，还是集中在 ARCHITECTURE.md？
3. 有没有特定的文件路径模式（如 `src/**/*.ts`），写入时应该提醒文档同步？"

**Q12: Worktree 偏好**
"本项目要启用 worktree 吗？
A. 默认启用 + 条件触发（推荐）
B. 询问后启用
C. 显式 disable（项目小，不需要隔离）"

**Q13: 初始 architectural decisions**
"项目有哪些已经定下来的架构选择值得记到 `decisions.md#Architectural`？（技术栈 / 架构模式 / 不可变约束）"

**Q14: 项目专属触发器**
"哪几个目录/模块改动时，**必须先读某些上下文**或**必须问你**？（如 '改 auth/* 必须先读 session.ts 的并发注释'；'改 migration 必须问回滚策略'）"

### 可选追加问题

**Q15: 环境特殊要求** — 如果有
"有没有环境相关的特殊限制？"

**Q16: 已有的 CLAUDE.md** — 如果探索阶段发现已有（非升级模式）
"你已经有一个 CLAUDE.md 了。我应该：A. 在它基础上扩展  B. 完全重写  C. 保留它，harness 写到单独文件"

**升级模式**：如果用户说"项目目标不变"，跳过 Q1；Q2-Q14 按需跑（只跑业务层相关问题）。

---

## Phase 3: 确认设计

把 brainstorm 的结论整理成 harness 设计摘要，**分段**呈现给用户确认。每段确认后再进入下一段。

### 段 1: CLAUDE.md 结构
列出 CLAUDE.md 的章节清单（v4 精简指针表风格），明确标注"项目目标"章节的内容和骨架指针表的 13 行。

### 段 2: Hooks 设计
分两层列出：
- **骨架 hooks（永远不变）**：敏感文件拦截、git commit 前 verify、commit 前 review 通过检查、worktree ignore 验证、plan 格式自检
- **业务 hooks（跟着目标变）**：模块边界检测、文档同步警告（可选）

每个 hook 说明：检测什么 → 为什么 → 报错信息怎么写。
问："这些 hooks 合理吗？有没有太严或太松的？"

### 段 3: Agent 定义
- 基础层（必有）：reviewer-spec + reviewer-quality + docs
- 协作层：N 个 worker，各自负责什么
- 说明 two-stage review 循环：reviewer-spec（spec compliance + 文档同步）→ reviewer-quality（代码质量 + 文档红线）
问："这个分工合理吗？"

### 段 4: 初始决策
列出预填到 decisions.md#Architectural 的初始架构决策（来自 Q13）。

### 段 5: Rules 文件清单（v4 新增）
列出 11 个 rule 文件，每个标注骨架/业务：
- 骨架（跨项目通用）：workflow / brainstorm / plan / review / feedback / subagent / memory / worktree
- 混合（骨架 + 业务 placeholder）：verify / docs
- 业务（完全由 brainstorm 填充）：modules

问："这些规则文件的分工清楚吗？"

### 段 6: 基础设施（v4 新增）
展示 Plan / Session state / Worktree 的初始化计划：
- `.claude/plans/`（git tracked，可移交合约）
- `.claude/state/session.md`（gitignored，compact-safe 便签）
- `.worktrees/`（Q12 启用时）
- `.gitignore` 更新

问："基础设施的设置 OK 吗？"

### 段 7: 文档 skeleton（v4 新增）
按 Q11 的答案展示将生成的 README / CHANGELOG / docs/modules/ skeleton。如果用户已有就不生成。

### 段 8: Two-stage review 循环（v4 新增）
明确告诉用户 reviewer 从 1 个（v3）拆成 2 个，v4 的 review loop 是 spec → quality 顺序。文档违规 = MAJOR → FAIL。

问："这个 review 流程能接受吗？"

全部确认后进入 Phase 4。

---

## Phase 4: 生成配置

### 4.1 CLAUDE.md

从 `templates/CLAUDE.md` 复制模板，填入 brainstorm 产出：
- `{项目名}` ← 项目名
- `{一句话描述}` ← Phase 1 探索总结
- `{brainstorm Q1 的完整回答}` ← Q1 产出
- `{技术栈表格}` ← Phase 1 + Q2 产出
- `{项目结构}` ← Phase 1 扫描 + Q2 产出

**v4 的 CLAUDE.md 是精简指针表风格**，详见 `templates/CLAUDE.md`。所有"怎么做"的细节在 `.claude/rules/*.md` 里。

### 4.2 .claude/settings.json

#### Claude Code Hooks API 参考（硬性规范）

```
环境变量：
- PreToolUse/Write|Edit: $CLAUDE_TOOL_INPUT_FILE_PATH — 即将写入的文件路径
- PostToolUse/Write|Edit: $CLAUDE_TOOL_INPUT_FILE_PATH — 刚写入的文件路径
- PreToolUse/Bash: $CLAUDE_TOOL_INPUT — bash 命令内容

timeout 单位：秒（不是毫秒）
- 轻量检查（grep/文件检测）: 5
- 验证命令（npm run verify 等）: 120

退出码：
- exit 0 — 通过
- exit 1 — 阻断（配合 stderr 输出错误信息）
- stderr 输出会显示给 AI，必须包含三要素：违反了什么 → 为什么 → 怎么改
```

#### hooks 生成流程

1. **骨架 hooks**：从 `templates/hooks/skeleton-hooks.json` 复制 5 条骨架 hooks
   - Hook 1: 敏感文件拦截
   - Hook 2: git commit 前 verify（填入 Q9 的验证命令）
   - Hook 3: git commit 前 review 通过检查
   - Hook 4: git worktree add 前 ignore 验证
   - Hook 5: Write plan 后格式自检

2. **业务 hooks**：根据 Q3 的模块边界规则生成
   - 每条标注"hook 自动检测"的边界规则生成一个独立 hook
   - 业务 hooks 遵循 v3 的格式规范（每个 hook 检测 file path pattern + grep 违规 pattern → stderr 三要素）

3. **可选业务 hooks**：
   - Hook 6: 文档同步警告（Q11 答"需要"时，根据 Q11 的文件路径模式生成）

4. 合并骨架和业务 hooks 为最终的 `settings.json`

#### hooks 生成后自检（必须执行）

1. **边界↔hook 一一对应**：每条标注"hook 自动检测"的边界规则，settings.json 里必须有对应的 hook
2. **去重**：检测范围重叠的合并
3. **grep pattern 验证**：对每个 hook 构造心理测试用例
4. **timeout 合理性**：grep 检查用 5 秒，verify 命令用 120 秒
5. **骨架/业务分层清晰**：用 `_name` 字段区分

### 4.3 .claude/agents/reviewer-quality.md

从 `templates/agents/reviewer-quality.md` 复制，填入项目专属审查维度（来自 Q3 模块边界 + Q6 测试要求）。

**判定规则段不可修改**：CRITICAL→FAIL / MAJOR→FAIL / 只有 MINOR→PASS / Doc red line = MAJOR。

### 4.4 .claude/agents/docs.md

从 `templates/agents/docs.md` 复制，填入项目专属描述。

### 4.5 .claude/agents/worker.md

从 `templates/agents/worker.md` 复制，填入：
- 项目描述
- 模块边界段（从 rules/modules.md 复制该 worker 负责区域）
- 数据流方向

根据 Q7 工作方式决定生成几个 worker。

### 4.6 feedback-log.md

从 `templates/feedback-log.md` 复制，预填 Q8（已知的坑）到初始条目区。

### 4.7 .claude/memory/decisions.md

从 `templates/memory/decisions.md` 复制（三层骨架），预填 Q13 的初始 architectural decisions 到 `## Architectural` 段。

### 4.8 生成 .claude/rules/*.md（11 个文件）

对每个 rule 文件：
- **骨架层**（workflow / brainstorm / plan / review / feedback / subagent / memory / worktree）：从 `templates/rules/` 直接复制
- **混合层**（verify / docs）：从 `templates/rules/` 复制，替换业务 placeholder 段：
  - `verify.md`：填入 Q9 的验证命令到"项目验证命令"段
  - `docs.md`：填入 Q11 的文档清单和同步触发器到"本项目文档清单""本项目文档同步触发器"段
- **业务层**（modules）：从 `templates/rules/modules.md` 的 placeholder 结构出发，用 Q2-Q5 + Q14 的产出**完整填充**每个段

### 4.9 生成 .claude/agents/reviewer-spec.md

从 `templates/agents/reviewer-spec.md` 复制，填入项目专属描述。

### 4.10 创建目录骨架

```bash
mkdir -p .claude/plans
mkdir -p .claude/state
mkdir -p .worktrees          # 只在 Q12 答"启用"时
mkdir -p docs/modules
```

### 4.11 生成 .claude/state/session.md

从 `templates/state/session.md` 复制。

### 4.12 更新 .gitignore

追加（如果还没有）：
```
.worktrees/
.claude/state/
.claude/.bitfrog-backup-*/
```

**不 ignore** `.claude/plans/` — plans 是可移交合约，必须 commit 进版本控制。

### 4.13 生成 README / CHANGELOG skeleton

**仅在 Q11 答"要"且文件不存在时**生成：
- README.md：从 `templates/README.md` 复制，填入项目名 + 项目目标
- CHANGELOG.md：从 `templates/CHANGELOG.md` 复制

**不覆盖用户已有的 README / CHANGELOG。**

### 4.14 CLAUDE.md 指针表风格

v4 的 CLAUDE.md 从 v3 的"大而全"改为"精简指针表"。详细内容在 `rules/*.md` 里，CLAUDE.md 只保留：项目目标 + 技术栈 + 项目结构 + 第一动作（一句话 + 指向 rules/workflow.md）+ 骨架指针表（13 行）+ 进化规则。

---

## Phase 5: 生成后验证（必须执行）

生成所有文件后，执行以下验证。任何一项不通过就修复后再检查。

### 5.1 结构验证
- [ ] 所有输出物文件都已创建
- [ ] agent 的 frontmatter 格式正确（name、description、tools 字段）
- [ ] settings.json 是合法 JSON
- [ ] CLAUDE.md 包含"项目目标"章节和骨架指针表

### 5.2 hooks 分层验证
- [ ] 骨架 hooks 存在（5 条：敏感文件拦截、commit 前 verify、commit 前 review 检查、worktree ignore、plan 格式自检）
- [ ] 业务 hooks 与 rules/modules.md 中标注"hook 自动检测"的规则一一对应
- [ ] 没有重复的 hook
- [ ] CLAUDE.md 中清楚标注了每条规则是骨架层还是业务层

### 5.3 Hook 可用性
对每个 PostToolUse hook，构造心理测试用例：
- "如果 AI 在[文件 A]里写了[违规代码 B]，这个 hook 能拦住吗？"
- 拦不住就修复 grep pattern

### 5.4 交叉引用
- [ ] CLAUDE.md 中的验证命令和 settings.json 中 git commit hook 的 verify 命令一致
- [ ] worker.md 中的模块边界描述和 rules/modules.md 一致
- [ ] reviewer-quality.md 中的审查维度覆盖了 rules/modules.md 中所有标注"reviewer 审查"的规则

### 5.5 reviewer 判定规则检查
- [ ] reviewer-quality.md 中判定规则含 "CRITICAL → FAIL" "MAJOR → FAIL" "Doc red line violation = MAJOR"
- [ ] 没有被改成其他宽松变体

### 5.6 升级兼容性检查（仅升级模式）
- [ ] 骨架层内容未被修改（和备份 diff）
- [ ] 历史 decisions 未被删除
- [ ] 历史 feedback-log 未被删除
- [ ] 新决策标注了"目标变更"

### 5.7 Rules 文件验证（v4 新增）
- [ ] `.claude/rules/` 下 11 个文件都存在（workflow / brainstorm / plan / review / verify / feedback / subagent / modules / memory / docs / worktree）
- [ ] 每个 rule 文件含定义的关键章节
- [ ] 骨架 rule 文件内容和 `templates/rules/*` 模板一致
- [ ] 业务 rule 文件（modules.md / verify.md 业务段 / docs.md 业务段）含 brainstorm 产出
- [ ] CLAUDE.md 骨架指针表引用的 rules/*.md 都真实存在

### 5.8 Two-stage review 拆分验证（v4 新增）
- [ ] `.claude/agents/reviewer-spec.md` 存在
- [ ] `.claude/agents/reviewer-quality.md` 存在
- [ ] reviewer-quality.md 判定规则含 "CRITICAL → FAIL" "MAJOR → FAIL" "Doc red line violation = MAJOR"
- [ ] settings.json 含 Hook 3（commit 前 review 通过检查）

### 5.9 基础设施验证（v4 新增）
- [ ] `.claude/plans/` 目录存在
- [ ] `.claude/state/session.md` 存在且有模板内容
- [ ] `.worktrees/` 存在（Q12 启用时）
- [ ] `.gitignore` 含 `.worktrees/` `.claude/state/` `.claude/.bitfrog-backup-*/`
- [ ] `.gitignore` **不**含 `.claude/plans/`
- [ ] `docs/modules/` 目录存在

### 5.10 文档 skeleton 验证（v4 新增，Q11 答"要"时）
- [ ] `README.md` 存在
- [ ] `CHANGELOG.md` 存在
- [ ] bitfrog 生成的 skeleton 只在原文件不存在时生成（不覆盖）

### 5.11 Memory 分层验证（v4 新增）
- [ ] `decisions.md` 含 `## Architectural` / `## Operational` / `## Learned` 三个段
- [ ] brainstorm Q13 的初始 architectural decisions 已写入对应段
- [ ] `rules/memory.md` 存在且定义了三层

---

## 升级模式（Upgrade Mode）

当 Phase 1 检测到已有 harness 配置时，进入升级模式。

### 骨架/业务层映射表

| 文件 | 层级 | 升级时处理 |
|-----|-----|----------|
| CLAUDE.md | 混合 | 骨架指针表 + 进化规则保留；项目目标 + 技术栈 + 项目结构可能重写 |
| feedback-log.md | 历史 | **全部保留**，只追加 |
| decisions.md | 历史 | **全部保留**，追加新决策（标注"目标变更"） |
| rules/workflow.md | **骨架** | 不动 |
| rules/brainstorm.md | **骨架** | 不动 |
| rules/plan.md | **骨架** | 不动 |
| rules/review.md | **骨架** | 不动 |
| rules/verify.md | 混合 | 5 步 Gate 骨架保留；项目验证命令段重写 |
| rules/feedback.md | **骨架** | 不动 |
| rules/subagent.md | **骨架** | 不动 |
| rules/modules.md | **业务** | **完全重写** |
| rules/memory.md | **骨架** | 不动 |
| rules/docs.md | 混合 | 六子章骨架保留；本项目文档清单 + 同步触发器段重写 |
| rules/worktree.md | **骨架** | 不动 |
| agents/worker.md | 混合 | 流程骨架保留；模块边界描述段重写 |
| agents/reviewer-spec.md | **骨架** | 不动 |
| agents/reviewer-quality.md | 混合 | 判定规则骨架保留；项目审查维度段重写 |
| agents/docs.md | **骨架** | 不动 |
| settings.json hooks | 混合 | 骨架 hooks(1-5)保留；业务 hooks 重写 |
| .worktrees/ / .claude/plans/ / .claude/state/ | 历史 | 保留 |
| docs/ 下的文档 | 历史 | 保留 |

### Step U1: 检测和备份

```bash
BACKUP_DIR=".claude/.bitfrog-backup-$(date +%Y%m%d-%H%M%S)"
mkdir -p "$BACKUP_DIR"
cp -r .claude/rules/ "$BACKUP_DIR/rules/" 2>/dev/null || true
cp -r .claude/agents/ "$BACKUP_DIR/agents/" 2>/dev/null || true
cp CLAUDE.md "$BACKUP_DIR/" 2>/dev/null || true
cp .claude/settings.json "$BACKUP_DIR/" 2>/dev/null || true
```

### Step U2: 识别骨架 vs 业务层

按上方映射表分类，输出给用户一个 diff 报告：骨架保留 / 业务重写 / 历史追加。

### Step U3: 重新 brainstorm（只跑业务相关问题）

跳过 Q1 如果用户说"项目目标不变"；跑 Q2-Q14 里和业务层相关的。

开场白："检测到已有 harness 配置。我会保留骨架层，只更新业务层。先问几个问题。"

### Step U4: 重写业务层（只动业务部分）

- `rules/modules.md` 完全重写
- `rules/verify.md` 只替换业务段（项目验证命令）
- `rules/docs.md` 只替换业务段（项目文档清单 + 同步触发器）
- `agents/worker.md` 只替换模块边界段
- `agents/reviewer-quality.md` 只替换项目审查维度段，**判定规则段绝对不动**
- `settings.json` 只替换业务 hook，骨架 hook 原位不动

### Step U5: 追加新 decisions

在 `decisions.md#Architectural` 追加一条，标注"目标变更"：
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

全部 5.1-5.11 验证清单跑一遍，特别是 5.6 升级兼容性。

### Step U7: 清理备份

Phase 5 全部通过才删 `$BACKUP_DIR`。失败 → 保留备份，报告失败项，让用户决定是 rollback 还是修复。

**Rollback 命令**（打印给用户，不自动跑）：
```
如需回滚：
  rm -rf .claude/rules .claude/agents CLAUDE.md .claude/settings.json
  cp -r {BACKUP_DIR}/* .
  rm -rf {BACKUP_DIR}
```

---

## 生成后

1. 列出所有生成的文件和简要说明
2. 报告 Phase 5 验证结果
3. 问用户："要不要打开看一下？有没有需要调整的？"
4. 如果用户满意，问是否需要 git commit

---

## 边界模式库（brainstorm Q3 的参考）

根据项目类型给出具体建议：

**Web 前端项目**
- UI 组件不直接调用 API → 通过 service/store 层
- 路由定义集中管理 → 不在组件里写路由逻辑

**全栈项目**
- 前端不直接访问数据库 → 通过 API 层
- API 层不包含业务逻辑 → 逻辑在 service 层
- 数据库操作只在 repository/model 层

**AI 应用**
- AI API 调用集中在一个模块 → 统一超时/重试/兜底
- prompt 模板与代码分离 → 模板文件不硬编码
- AI 输出必须校验 → 不直接信任 AI 返回值

**CLI 工具**
- 命令定义与业务逻辑分离
- I/O 操作集中在入口 → 核心逻辑纯函数
- 配置读取集中管理

**数据处理**
- 数据获取/清洗/转换/输出分层
- 每层输入输出类型明确
- 副作用集中在最外层

---

## 工具适配

当前版本 **深度绑定 Claude Code**。hooks 格式、环境变量、agent 定义均为 Claude Code 原生格式。

跨 harness 支持（Cursor / Codex / Windsurf）推后到 v5。

---

## 关键原则

### v3 原则（保留）

- **brainstorm 优先** — 不理解项目就不生成配置，宁可多问不猜
- **reviewer 和 docs 是基础层** — 所有项目必有，不需要用户选择
- **hooks 分骨架/业务两层** — 骨架永远不变，业务跟着项目目标变
- **项目目标写进 CLAUDE.md** — 目标变了重跑 harness-init 升级模式
- **不生成 Plans.md** — harness 不做项目管理
- **尊重已有配置** — 升级模式保留骨架层和历史记录
- **最简可用** — 不加项目不需要的规则
- **hook 错误信息三要素** — 违反了什么 + 为什么 + 怎么改
- **生成后必须验证** — Phase 5 不可跳过
- **reviewer 判定规则不可修改** — CRITICAL→FAIL / MAJOR→FAIL
- **"你的第一版代码很少是对的"** — 写进每个 worker 的 DNA
- **进化而不是完美** — feedback-log 驱动后续演进

### v4 新增原则

- **第一动作强制** — 收到任务先复述 + 检查理解，不跳过
- **brainstorm 是动态模式** — 不是 phase，是"明确目标 ↔ load context"耦合螺旋，退出条件驱动
- **plan 是可移交合约** — 和 session 解耦，compact-safe
- **Two-stage review 硬约束** — spec first → quality after，不可颠倒
- **Verify 5 步 gate** — Evidence before claims；Agent success ≠ success（必须 check VCS diff）
- **微 feedback 每 task 必写** — 嵌入 plan Step 10，不靠自觉
- **文档是 compact-safe source of truth** — 违规 = MAJOR → FAIL
- **Review 直接行动，不写报告** — 三类 finding 分流（本 task / 本来就有 / plan 错）
- **选择性吸收 superpowers** — brainstorm 哲学 / plan 格式 / review 循环 / verify evidence / worktree 隔离；**不吸收** TDD 原教旨 / debug 四阶段 / 红旗清单
- **骨架/业务分层** — 升级模式保骨架重写业务
