---
name: bitfrog-harness-init
description: "Use when setting up harness engineering for a project — generates CLAUDE.md, hooks, agents, feedback-log, and decisions.md through collaborative brainstorming to deeply understand the project first."
---

# BitFrog Harness Init

通过深度 brainstorm 理解项目后，生成一套最简可用的 harness engineering 配置。

<HARD-GATE>
在完成 brainstorm 并获得用户确认之前，不允许写任何 harness 文件。必须先彻底理解项目，再生成配置。不管项目看起来多简单，都要走完 brainstorm 流程。
</HARD-GATE>

## 运行模式

### 初始化模式（新项目）
项目没有 `.claude/` 和 `CLAUDE.md` → 完整走 Phase 1-5，生成全部文件。

### 升级模式（已有harness，项目目标变化）
检测到已有 CLAUDE.md 且其中有"项目目标"章节 → 进入升级模式：
1. 读取现有 CLAUDE.md，识别骨架层和业务层
2. **保留骨架层**：工作流、反馈循环、进化规则、骨架hooks（敏感文件拦截、commit前verify）、reviewer判定规则、docs agent
3. **重新brainstorm业务层**：项目目标、模块边界、业务hooks、reviewer审查维度、worker分工
4. 只重写业务层相关内容，骨架层原样保留
5. decisions.md 追加新决策（标注"目标变更"），不删除历史决策
6. feedback-log.md 保留全部历史记录

## 输出物

```
project/
├── CLAUDE.md                      # AI工作手册（含项目目标声明）
├── feedback-log.md                # 出错记录（进化机制）
├── .claude/
│   ├── settings.json              # hooks（骨架hooks + 业务hooks）
│   ├── agents/
│   │   ├── worker.md              # 开发agent（可能多个）
│   │   ├── reviewer.md            # 审查agent（必有）
│   │   └── docs.md                # 文档agent（必有）
│   └── memory/
│       └── decisions.md           # 架构决策记录
```

**不生成 Plans.md** — harness不做项目管理，只管代码质量。

**reviewer.md 和 docs.md 是所有项目的基础层，必须生成。** worker的数量和分工由brainstorm决定。

---

## Phase 1: 探索项目上下文

在提问之前，先自己搞清楚：

1. **读项目文件** — package.json / pyproject.toml / go.mod / Cargo.toml（判断语言和依赖）
2. **读现有配置** — 如果已经有 CLAUDE.md、.claude/，判断是初始化模式还是升级模式
3. **读 git log** — `git log --oneline -20`，了解项目在做什么、开发节奏
4. **扫目录结构** — `ls -R src/` 或 `find . -type f -name '*.ts' -o -name '*.py' | head -50`
5. **读README** — 如果有的话
6. **检测用户熟练度** — 根据用户的用词、提问深度、技术术语使用情况，调整后续沟通的技术深度。不要问用户"你是什么水平"，而是从对话中推断

**如果是升级模式**：额外读取现有 CLAUDE.md、settings.json、agents/，识别哪些是骨架层、哪些是业务层。

把发现总结为一段话告诉用户。如果是升级模式，明确说："检测到已有harness配置，我会保留骨架层（工作流、反馈循环、基础hooks），只更新业务层（模块边界、业务hooks、审查维度）。"

---

## Phase 2: Brainstorm（核心）

一次一个问题，理解清楚再问下一个。优先用选择题，但需要深挖时用开放题。

### 必问问题（按顺序）

**Q1: 项目目标**
"这个项目的最终目标是什么？不是功能列表，是它解决什么问题、给谁用。"

这个回答会写入CLAUDE.md的"项目目标"章节，作为所有业务规则的依据。

**Q2: 架构分层**
基于Phase 1看到的代码结构，提出你理解的模块划分：
"我看到代码大概分成了[xxx/yyy/zzz]这几层，我理解的职责是[...]。这个理解对吗？有没有我漏掉的边界？"

**Q3: 模块边界**
"这些模块之间，哪些调用关系是你想严格控制的？"

如果用户不确定，基于项目类型给出具体建议（参考下方"边界模式库"）。

**Q4: 数据流方向**
"数据在这些模块之间是怎么流动的？有没有单向要求？比如A→B→C，C不能反向调用A。"

**Q5: 外部依赖**
"项目有哪些外部依赖需要特殊处理？比如：第三方API（需要超时/重试/兜底）、数据库（需要migration规范）、文件系统（需要路径约束）。"

**Q6: 测试现状**
"现在有测试吗？用什么框架？你希望这套harness对测试有什么要求？"

选项：
- A. 测试先行（TDD，先写测试再写实现）
- B. 实现后补测试（每个PR必须带测试）
- C. 关键路径测试（只测核心逻辑，不要求全覆盖）
- D. 目前没测试，先不加测试要求

**Q7: 工作方式**
"你平时写这个项目，代码从开始到提交，中间经过几步？比如：一个人从头写到尾？前后端可以同时写？必须先做A再做B？"

根据回答推断协作模式：
- "一个人写完提交" → 单worker
- "前端后端可以同时写" → 多worker并行（fan-out）
- "必须先写API再写前端" → 流水线（pipeline）
- "不同模块不同人" → 专家池（expert-pool）

**不要向用户暴露模式名称。** 用他们自己的语言确认。

**Q8: 已知的坑**
"这个项目里，AI（或你自己）之前反复犯的错有哪些？"

这些直接写入初始的 feedback-log.md。

**Q9: 验证命令**
"项目现在跑验证用什么命令？类型检查、lint、测试分别怎么跑？"

如果没有，根据技术栈推荐并确认。

**Q10: 提交规范**
"commit message有没有格式要求？分支策略是什么？"
- A. conventional commits（feat/fix/refactor/...）
- B. 自由格式，但要求一句话说清楚改了什么
- C. 有其他规范

### 可选追加问题

**Q11: 环境特殊要求** — 如果有
"有没有环境相关的特殊限制？"

**Q12: 已有的CLAUDE.md** — 如果探索阶段发现已有（非升级模式）
"你已经有一个CLAUDE.md了。我应该：A. 在它基础上扩展  B. 完全重写  C. 保留它，harness写到单独文件"

---

## Phase 3: 确认设计

把brainstorm的结论整理成harness设计摘要，**分段**呈现给用户确认。每段确认后再进入下一段。

### 段1: CLAUDE.md结构
列出计划写入的每个章节和关键内容。**明确标注"项目目标"章节的内容。**

### 段2: Hooks设计
分两层列出：
- **骨架hooks（永远不变）**：敏感文件拦截、git commit前verify
- **业务hooks（跟着目标变）**：模块边界检测、代码规范检测

每个hook说明：检测什么 → 为什么 → 报错信息怎么写。
问："这些hooks合理吗？有没有太严或太松的？"

### 段3: Agent定义
- 基础层（必有）：reviewer + docs
- 协作层：N个worker，各自负责什么
问："这个分工合理吗？"

### 段4: 初始决策
列出预填到decisions.md的架构决策。

全部确认后进入Phase 4。

---

## Phase 4: 生成配置

### 4.1 CLAUDE.md

**硬性模板结构（不可省略任何章节）：**

```markdown
# [项目名]

[一句话描述]

## 项目目标

[brainstorm Q1的回答，完整记录]
[这是所有业务规则的依据。目标变了，业务层的规则就要跟着变。]

## 技术栈
[表格：层 | 技术 | 版本]

## 项目结构
[目录树，标注每个目录的职责和边界]

## 模块边界（硬性规则）

hooks分两层：
- **骨架hooks（永远不变）** — 敏感文件拦截、git commit前验证
- **业务hooks（跟着项目目标变）** — 模块边界检测、代码规范检测

hooks机制说明：
- **PreToolUse** — 工具调用前触发。用于拦截敏感文件写入、git commit前跑验证
- **PostToolUse** — 工具调用后触发。用于检查刚写入的文件是否违反模块边界

[规则表格：规则 | 含义 | 检测方式（hook自动检测 / reviewer审查） | 层级（骨架/业务）]

## [外部依赖名]调用规范（如果有）
[超时/重试/兜底要求，带代码示例]

## 测试策略
### 原则
[brainstorm Q6选择的模式]

### 单元测试
[框架 + 命令 + 每个模块的测试要点]

### E2E测试（如果有）
[框架 + 命令 + 测试场景]

### 验证命令
[汇总所有命令 + npm run verify / 等效命令]

## 工作流

### 开始任务
1. 读 CLAUDE.md
2. 读 feedback-log.md
3. 读 .claude/memory/decisions.md

### 开发循环
[测试策略对应的流程]
verify失败 → 读错误 → 修复 → 再verify → 最多3轮 → 停下记录feedback-log
**你的第一版代码很少是对的，不要急着提交，先verify。**

### 提交前
verify通过 + git diff检查 + commit message格式

### 完成任务后
写decisions(如有) + 写feedback-log(如有)

## 反馈循环
[具体流程图]
**禁止行为**：跳过测试、注释掉报错代码、降低检查标准、@ts-ignore

## 进化规则
- feedback-log同类3次 → CLAUDE.md加规则 + settings.json加hook
- hook误报频繁 → 放宽规则
- **项目目标发生重大变化时** → 重新运行 `/bitfrog-harness-init`，进入升级模式，保留骨架层，刷新业务层
- 初始配置够用就行，feedback-log驱动后续演进
```

### 4.2 .claude/settings.json

#### Claude Code Hooks API 参考（硬性规范）

```
环境变量：
- PreToolUse/Write|Edit: $CLAUDE_TOOL_INPUT_FILE_PATH — 即将写入的文件路径
- PostToolUse/Write|Edit: $CLAUDE_TOOL_INPUT_FILE_PATH — 刚写入的文件路径
- PreToolUse/Bash: $CLAUDE_TOOL_INPUT — bash命令内容

timeout单位：秒（不是毫秒）
- 轻量检查（grep/文件检测）: 5
- 验证命令（npm run verify等）: 120

退出码：
- exit 0 — 通过
- exit 1 — 阻断（配合stderr输出错误信息）
- stderr输出会显示给AI，必须包含三要素：违反了什么 → 为什么 → 怎么改
```

#### hooks分层

**骨架hooks（所有项目必有，升级时保留）：**

1. 敏感文件拦截：
```json
{
  "matcher": "Write|Edit",
  "hooks": [{
    "type": "command",
    "command": "FILE=\"$CLAUDE_TOOL_INPUT_FILE_PATH\"\nif echo \"$FILE\" | grep -qE '\\.(env|pem|key|secret|credentials)$'; then\n  echo '[BLOCKED] 敏感文件禁止AI修改' >&2\n  echo '原因: 防止泄露密钥和凭证' >&2\n  echo '修复: 手动编辑该文件' >&2\n  exit 1\nfi",
    "timeout": 5
  }]
}
```

2. git commit前验证（如果项目有verify命令）：
```json
{
  "matcher": "Bash",
  "hooks": [{
    "type": "command",
    "if": "Bash(git commit*)",
    "command": "cd [项目绝对路径] && [verify命令] 2>&1",
    "timeout": 120,
    "statusMessage": "提交前检查"
  }]
}
```

**业务hooks（根据brainstorm生成，升级时重写）：**

每条模块边界规则生成一个独立的hook。格式：

```bash
FILE="$CLAUDE_TOOL_INPUT_FILE_PATH"
if echo "$FILE" | grep -qE '[目录匹配pattern]'; then
  if grep -qE '[违规pattern]' "$FILE" 2>/dev/null; then
    echo '[BLOCKED] [哪条规则被违反]' >&2
    echo '原因: [为什么有这条规则]' >&2
    echo '修复: [具体怎么改]' >&2
    exit 1
  fi
fi
```

**settings.json结构必须用注释区分两层：**
```json
{
  "hooks": {
    "PreToolUse": [
      // --- 骨架hooks（不随项目目标变化） ---
      { "敏感文件拦截" },
      { "git commit前验证" },
      // --- 业务hooks（跟随项目目标变化） ---
      ...
    ],
    "PostToolUse": [
      // --- 业务hooks（跟随项目目标变化） ---
      { "模块边界检查..." }
    ]
  }
}
```

注意：JSON不支持注释。用 `"_comment"` 字段或在CLAUDE.md中说明哪些是骨架哪些是业务。实际settings.json保持合法JSON。

#### hooks生成后自检（必须执行）

1. **边界↔hook一一对应**：每条标注"hook自动检测"的边界规则，settings.json里必须有对应的hook。缺了就补。
2. **去重**：检测范围重叠的合并为一个。
3. **grep pattern验证**：对每个hook构造心理测试用例——"如果AI在[文件A]里写了[违规代码B]，这个hook能拦住吗？"拦不住就修。
4. **timeout合理性**：grep检查用5秒，verify命令用120秒。
5. **骨架/业务分层清晰**：确认骨架hooks和业务hooks没有混淆。

### 4.3 .claude/agents/reviewer.md（基础层，必有）

**硬性模板（不可修改判定规则）：**

```markdown
---
name: reviewer
description: [基于项目定制的描述]
tools: [Read, Grep, Glob, Bash]
---

# Reviewer

## 审查前
1. 读 CLAUDE.md
2. 跑验证命令确认机械化检查通过（如果没通过直接FAIL）
3. git diff 看变更范围

## 审查维度
[基于brainstorm Q3的模块边界 → 生成具体审查项]
[每个维度标注 CRITICAL / MAJOR / MINOR]

每个审查项必须附带可执行的grep检查命令。

## 输出格式
### Verdict: PASS / FAIL
- [CRITICAL] 问题 + 文件:行号 + 修复建议
- [MAJOR] 问题 + 文件:行号 + 修复建议
- [MINOR] 建议

## 判定规则（不可修改）
- 有 CRITICAL → FAIL
- 有 MAJOR → FAIL
- 只有 MINOR → PASS
```

### 4.4 .claude/agents/docs.md（基础层，必有）

```markdown
---
name: docs
description: [基于项目定制的描述]
tools: [Read, Write, Edit, Grep, Glob]
---

# Docs

## 职责
维护项目文档，确保文档与代码同步。

## 工作前
1. 读 CLAUDE.md
2. 读 .claude/memory/decisions.md
3. git diff 了解最近变更

## 文档范围
- 模块接口文档（公开API、输入输出类型、使用示例）
- 架构文档（模块间数据流、设计决策）
- 使用指南（开发环境、常用命令）

## 触发时机
- 模块公开接口变更时
- 新增模块或重大重构后
- 架构决策更新时

## 原则
- 文档跟着代码走，不提前写未实现的接口
- 代码示例必须可运行
- 简洁，不写废话
```

### 4.5 .claude/agents/worker.md

根据brainstorm Q7决定生成几个worker和各自职责。

**worker模板（每个worker都必须包含）：**

```markdown
---
name: [worker名]
description: [基于项目定制的描述]
tools: [Read, Write, Edit, Bash, Grep, Glob, WebSearch, WebFetch]
---

# [Worker名]

## 开工前
1. 读 CLAUDE.md
2. 读 feedback-log.md
3. 读 .claude/memory/decisions.md

## 开发流程
[基于brainstorm Q6的测试策略]

**你的第一版代码很少是对的。** 写完先跑verify，不要急着说"完成了"。

## 反馈循环
verify失败 → 读完整错误输出 → 定位问题 → 修复 → 再verify
- 最多3轮
- 3轮仍失败 → 停下，记录feedback-log.md，不要硬刚

## 模块边界
[从CLAUDE.md复制该worker负责区域的边界规则]
[标注数据流方向]

## 禁止行为
- 跳过测试（test.skip / describe.skip）
- 注释掉报错代码
- @ts-ignore / any 类型
- 违反模块边界（hooks会拦截，但你应该主动遵守）
- 3轮verify失败后继续硬刚

## 提交
verify通过 + git diff检查 + commit message格式
```

### 4.6 feedback-log.md

```markdown
# Feedback Log

AI出错、卡住、输出不符合预期时记录。同类3次 → CLAUDE.md加规则 + settings.json加hook。

## 格式
### [日期] 问题简述
- **场景**: 在做什么时出的问题
- **错误**: 具体错了什么
- **原因**: 为什么会出错
- **修复**: 怎么修的
- **分类**: [模块边界|测试|类型|AI调用|其他]
- **规则更新**: 无 / CLAUDE.md加了xxx / hook加了xxx

---

[brainstorm Q8收集到的已知坑，预填在这里]
```

### 4.7 .claude/memory/decisions.md

```markdown
# 架构决策记录

## 格式
### [编号] 决策标题
- **日期**: YYYY-MM-DD
- **决策**: 选了什么
- **原因**: 为什么
- **替代方案**: 考虑过但放弃的
- **影响**: 影响哪些模块
- **目标变更**: 无 / 因[xxx目标变化]而新增

---

[预填brainstorm中确认的架构决策]
```

---

## Phase 5: 生成后验证（必须执行）

生成所有文件后，执行以下验证。任何一项不通过就修复后再检查。

### 5.1 结构验证
- [ ] 所有输出物文件都已创建（不含Plans.md）
- [ ] agent的frontmatter格式正确（name、description、tools字段）
- [ ] settings.json是合法JSON
- [ ] CLAUDE.md包含"项目目标"章节

### 5.2 hooks分层验证
- [ ] 骨架hooks存在（敏感文件拦截、commit前verify）
- [ ] 业务hooks与CLAUDE.md模块边界表格中标注"hook自动检测"的规则一一对应
- [ ] 没有重复的hook
- [ ] CLAUDE.md中清楚标注了每条规则是骨架层还是业务层

### 5.3 Hook可用性
对每个PostToolUse hook，构造心理测试用例：
- "如果AI在[文件A]里写了[违规代码B]，这个hook能拦住吗？"
- 拦不住就修复grep pattern

### 5.4 交叉引用
- [ ] CLAUDE.md中的验证命令和settings.json中git commit hook的verify命令一致
- [ ] worker.md中的模块边界描述和CLAUDE.md一致
- [ ] reviewer.md中的审查维度覆盖了CLAUDE.md中所有标注"reviewer审查"的规则

### 5.5 reviewer判定规则检查
- [ ] reviewer.md中判定规则是"CRITICAL→FAIL / MAJOR→FAIL / 只有MINOR→PASS"
- [ ] 没有被改成其他宽松变体

### 5.6 升级兼容性检查（仅升级模式）
- [ ] 骨架层内容未被修改
- [ ] 历史decisions未被删除
- [ ] 历史feedback-log未被删除
- [ ] 新决策标注了"目标变更"

---

## 生成后

1. 列出所有生成的文件和简要说明
2. 报告Phase 5验证结果
3. 问用户："要不要打开看一下？有没有需要调整的？"
4. 如果用户满意，问是否需要 git commit

---

## 边界模式库（brainstorm Q3的参考）

根据项目类型给出具体建议：

**Web前端项目**
- UI组件不直接调用API → 通过service/store层
- 路由定义集中管理 → 不在组件里写路由逻辑

**全栈项目**
- 前端不直接访问数据库 → 通过API层
- API层不包含业务逻辑 → 逻辑在service层
- 数据库操作只在repository/model层

**AI应用**
- AI API调用集中在一个模块 → 统一超时/重试/兜底
- prompt模板与代码分离 → 模板文件不硬编码
- AI输出必须校验 → 不直接信任AI返回值

**CLI工具**
- 命令定义与业务逻辑分离
- I/O操作集中在入口 → 核心逻辑纯函数
- 配置读取集中管理

**数据处理**
- 数据获取/清洗/转换/输出分层
- 每层输入输出类型明确
- 副作用集中在最外层

---

## 工具适配（当前仅支持Claude Code）

当前版本针对 **Claude Code** 生成配置。hooks格式、环境变量、agent定义均为Claude Code原生格式。

未来版本将支持：
- Cursor（.cursorrules）
- Codex（AGENTS.md + .codex/）
- Windsurf（.windsurfrules）
- 通用 Git hooks（.husky/pre-commit）

适配其他工具时，规则层（CLAUDE.md定义的边界和流程）不变，只有hooks配置层需要转换格式。

---

## 关键原则

- **brainstorm优先** — 不理解项目就不生成配置，宁可多问不猜
- **reviewer和docs是基础层** — 所有项目必有，不需要用户选择
- **hooks分骨架/业务两层** — 骨架永远不变，业务跟着项目目标变
- **项目目标写进CLAUDE.md** — 目标变了重跑harness-init升级模式
- **不生成Plans.md** — harness不做项目管理
- **尊重已有配置** — 升级模式保留骨架层和历史记录
- **最简可用** — 不加项目不需要的规则
- **hook错误信息三要素** — 违反了什么 + 为什么 + 怎么改
- **生成后必须验证** — Phase 5不可跳过
- **reviewer判定规则不可修改** — CRITICAL→FAIL / MAJOR→FAIL
- **"你的第一版代码很少是对的"** — 写进每个worker的DNA
- **进化而不是完美** — feedback-log驱动后续演进
