---
name: bitfrog-harness-init
description: "Use when setting up harness engineering for a project — generates CLAUDE.md, hooks, agents, feedback-log, and decisions.md through collaborative brainstorming to deeply understand the project first."
---

# BitFrog Harness Init

通过深度 brainstorm 理解项目后，生成一套最简可用的 harness engineering 配置。

<HARD-GATE>
在完成 brainstorm 并获得用户确认之前，不允许写任何 harness 文件。必须先彻底理解项目，再生成配置。不管项目看起来多简单，都要走完 brainstorm 流程。
</HARD-GATE>

## 输出物

最终生成以下文件（不多不少）：

```
project/
├── CLAUDE.md                      # AI工作手册
├── Plans.md                       # 开发计划
├── feedback-log.md                # 出错记录（进化机制）
├── .claude/
│   ├── settings.json              # hooks（执行层）
│   ├── agents/
│   │   ├── worker.md              # 开发agent
│   │   └── reviewer.md            # 审查agent
│   └── memory/
│       └── decisions.md           # 架构决策记录
```

## Phase 1: 探索项目上下文

在提问之前，先自己搞清楚：

1. **读项目文件** — package.json / pyproject.toml / go.mod / Cargo.toml（判断语言和依赖）
2. **读现有配置** — 如果已经有 CLAUDE.md、.claude/、.eslintrc、tsconfig 等，必须尊重不覆盖
3. **读 git log** — 最近20条commit，了解项目在做什么、开发节奏
4. **扫目录结构** — `find src/ -type f | head -50` 或 `ls -R src/`，了解代码组织方式
5. **读README** — 如果有的话

把发现总结为一段话告诉用户："我看到这是一个xxx项目，用了xxx技术栈，目前在做xxx。以下是我的几个问题。"

## Phase 2: Brainstorm（核心）

一次一个问题，理解清楚再问下一个。优先用选择题，但需要深挖时用开放题。

### 必问问题（按顺序）

**Q1: 项目目标**
"这个项目的最终目标是什么？不是功能列表，是它解决什么问题、给谁用。"

**Q2: 架构分层**
基于你在Phase 1看到的代码结构，提出你理解的模块划分，问用户：
"我看到代码大概分成了[xxx/yyy/zzz]这几层，我理解的职责是[...]。这个理解对吗？有没有我漏掉的边界？"

**Q3: 模块边界**
"这些模块之间，哪些调用关系是你想严格控制的？比如：某个模块不允许直接调用另一个模块的内部API，必须走封装接口。"

如果用户不确定，基于常见模式给出建议（参考下方"边界模式库"）。

**Q4: 外部依赖**
"项目有哪些外部依赖需要特殊处理？比如：第三方API（需要超时/重试/兜底）、数据库（需要migration规范）、文件系统（需要路径约束）。"

**Q5: 测试现状**
"现在有测试吗？用什么框架？覆盖率大概什么水平？你希望这套harness对测试有什么要求？"

选项建议：
- A. 测试先行（TDD，先写测试再写实现）
- B. 实现后补测试（每个PR必须带测试）
- C. 关键路径测试（只测核心逻辑，不要求全覆盖）
- D. 目前没测试，先不加测试要求

**Q6: 已知的坑**
"这个项目里，AI（或你自己）之前反复犯的错有哪些？比如：总是把代码写到错误的目录、忘记处理某种边界情况、生成的代码风格不对。"

这些直接写入初始的 feedback-log.md。

**Q7: 验证命令**
"项目现在跑验证用什么命令？类型检查、lint、测试分别怎么跑？"

如果没有，根据技术栈推荐并确认。

**Q8: 提交规范**
"commit message有没有格式要求？分支策略是什么？"
- A. conventional commits（feat/fix/refactor/...）
- B. 自由格式，但要求一句话说清楚改了什么
- C. 有其他规范（请说明）

### 可选追加问题（根据项目复杂度决定是否问）

**Q9: 多agent协作** — 如果项目较大
"开发过程中需要几种角色？默认是 worker（写代码）+ reviewer（审代码）。你需要额外的角色吗？比如：专门的测试agent、文档agent、安全审查agent。"

**Q10: 环境特殊要求** — 如果有
"有没有环境相关的特殊限制？比如：只能在特定目录写文件、某些环境变量必须存在、依赖特定的系统工具。"

**Q11: 已有的CLAUDE.md** — 如果探索阶段发现已有
"你已经有一个CLAUDE.md了。我应该：A. 在它基础上扩展harness配置  B. 完全重写  C. 保留它，harness配置写到单独文件"

## Phase 3: 确认设计

把brainstorm的结论整理成一份harness设计摘要，分段呈现给用户确认：

### 段1: CLAUDE.md结构
```
我计划为CLAUDE.md写入以下内容：
- 项目概述：[一句话]
- 技术栈：[表格]
- 项目结构：[目录树]
- 模块边界：[N条硬性规则]
- [外部依赖]规范：[具体要求]
- 测试策略：[选择的模式 + 具体命令]
- 工作流：[开发循环 + 反馈循环]
- 进化规则：[feedback-log使用方式]
```
"这个结构OK吗？"

### 段2: Hooks设计
```
settings.json 的 hooks：
- PreToolUse:
  - [拦截敏感文件]
  - [git commit前跑验证]（如果有验证命令）
- PostToolUse:
  - [模块边界检查1: ...]
  - [模块边界检查2: ...]
  - ...
```
每个hook说明：检测什么 → 为什么 → 报错信息怎么写

"这些hooks合理吗？有没有太严或太松的？"

### 段3: Agent定义
```
- worker: [职责 + 工具权限 + 工作流]
- reviewer: [审查维度 + 只读权限 + 判定规则]
```
"两个角色够吗？审查维度是否需要调整？"

### 段4: 初始决策
```
基于brainstorm，我会在decisions.md预写以下架构决策：
- [决策1]
- [决策2]
- ...
```

每段确认后再进入下一段。全部确认后进入Phase 4。

## Phase 4: 生成配置

按以下顺序生成文件：

### 4.1 CLAUDE.md

参考模板结构（根据brainstorm结果填充）：

```markdown
# [项目名]

[一句话描述]

## 技术栈
[表格：层 | 技术 | 版本]

## 项目结构
[目录树，标注每个目录的职责]

## 模块边界（硬性规则）
[hooks机制说明]
[规则表格：规则 | 含义 | 检测方式]

## [外部依赖名]调用规范
[具体的超时/重试/兜底要求，最好有代码示例]

## 测试策略
### 原则
[TDD / 实现后补测 / 关键路径]

### 单元测试
[框架 + 命令 + 每个模块的测试要点]

### E2E测试（如果有）
[框架 + 命令 + 测试场景]

### 视觉测试（如果有）
[框架 + 命令 + 检查项]

### 验证命令
[汇总所有命令 + npm run verify]

## 工作流
### 开始任务
[读哪些文件]

### 开发循环
[写测试→写实现→verify→通过提交/失败修复→3轮上限→记录feedback-log]

### 提交前
[verify通过 + git diff + commit message格式]

### 完成任务后
[更新Plans + 写decisions + 写feedback-log]

## 反馈循环
[具体流程：verify失败→读错误→修复→再verify→3轮→停下记录]
[禁止行为：跳过测试、注释代码、降低标准]

## 进化规则
[feedback-log使用方式]
[什么时候加新规则：同类3次→CLAUDE.md加规则+settings.json加hook]
[什么时候改规则：误报频繁→放宽]
```

### 4.2 .claude/settings.json

hooks设计原则：
- **每个hook单一职责** — 一个hook检测一件事
- **PreToolUse** — 拦截型（敏感文件、commit前验证）
- **PostToolUse** — 检查型（模块边界、代码规范）
- **错误信息三要素** — 违反了什么 → 为什么有这条规则 → 怎么改
- **timeout合理** — 轻量检查5s，验证命令120s
- **尊重已有配置** — 如果项目已有settings.json，合并而不是覆盖

commit前验证的hook格式（如果项目有verify命令）：
```json
{
  "matcher": "Bash",
  "hooks": [{
    "type": "command",
    "if": "Bash(git commit*)",
    "command": "cd [project-root] && npm run verify 2>&1",
    "timeout": 120,
    "statusMessage": "提交前检查: npm run verify"
  }]
}
```

模块边界检查的hook格式：
```json
{
  "matcher": "Write|Edit",
  "hooks": [{
    "type": "command",
    "command": "[检测脚本]",
    "timeout": 10
  }]
}
```

### 4.3 .claude/agents/worker.md

```markdown
---
name: worker
description: [基于项目定制的描述]
tools: [Read, Write, Edit, Bash, Grep, Glob, WebSearch, WebFetch]
---

# Worker

## 开工前
1. 读 CLAUDE.md
2. 读 Plans.md
3. 读 feedback-log.md
4. 读 .claude/memory/decisions.md

## 开发流程
[基于brainstorm Q5的测试策略选择]

## 反馈循环
[verify失败→读错误→修→再verify→3轮→停下记录]

## 禁止行为
[基于项目定制：跳过测试、注释代码、违反模块边界、etc.]

## 提交
[verify通过 + commit message格式]
```

### 4.4 .claude/agents/reviewer.md

```markdown
---
name: reviewer
description: [基于项目定制的描述]
tools: [Read, Grep, Glob, Bash]
---

# Reviewer

## 审查前
1. 读 CLAUDE.md
2. 跑验证命令确认机械化检查通过
3. git diff 看变更范围

## 审查维度
[基于brainstorm Q3的模块边界 → 生成具体的审查项]
[每个维度标注 CRITICAL / MAJOR / MINOR]
[附带快速检查的grep命令]

## 输出格式
Verdict: PASS / FAIL + 分级问题列表

## 判定规则
CRITICAL → FAIL / MAJOR → FAIL / 只有MINOR → PASS
```

### 4.5 feedback-log.md

```markdown
# Feedback Log

[说明：AI出错/卡住/输出不符合预期时记录。同类3次→补规则]

[格式模板]

---

[如果brainstorm Q6有已知的坑，预填在这里]
```

### 4.6 .claude/memory/decisions.md

```markdown
# 架构决策记录

[格式模板]

---

[预填brainstorm中确认的架构决策]
```

### 4.7 Plans.md

```markdown
# [项目名] 开发计划

[基于brainstorm了解到的项目状态，写出当前的开发计划]
[如果项目已有明确的下一步，写在这里]
[如果不清楚，写一个Phase 1占位，让用户后续填充]
```

## 生成后

1. 列出所有生成的文件和简要说明
2. 问用户："要不要打开看一下？有没有需要调整的？"
3. 如果用户满意，问是否需要 git commit 这些配置文件

## 边界模式库（brainstorm Q3的参考）

常见的模块边界模式，根据项目类型选择适用的：

**Web前端项目**
- UI组件不直接调用API → 通过service/store层
- 样式不使用内联style → 统一CSS模块或Tailwind
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
- 配置读取集中管理 → 不在各处读env

**数据处理**
- 数据获取/清洗/转换/输出分层
- 每层输入输出类型明确
- 副作用（文件写入、API调用）集中在最外层

## 关键原则

- **brainstorm优先** — 不理解项目就不生成配置，宁可多问不猜
- **尊重已有配置** — 有CLAUDE.md就在其基础上扩展，有settings.json就合并
- **最简可用** — 不加项目不需要的规则，不设不会被检测的规范
- **hook错误信息必须有用** — 违反了什么 + 为什么 + 怎么改，三要素缺一不可
- **进化而不是完美** — 初始配置够用就行，feedback-log驱动后续演进
