# Memory

decisions.md 三层分层 + 写入/归档规则。

---

## 三层定义

### Layer 1: Architectural(长期,不归档)

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

---

### Layer 2: Operational(中期,季度 review)

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

---

### Layer 3: Learned(短期,定期归档)

**内容**:
- feedback-log 同类 3 次后沉淀的短规则
- "这个地方 AI 容易踩的坑"
- 具体的"别这么写"经验

**写入时机**:
- feedback-log 同类 3 次 → 升级为 learned
- 进化机制自动触发(或 reviewer-quality 发现重复问题提出)

**格式**:
```markdown
## [Learn-007] auth 模块 token refresh 不要嵌套 try-catch
- **日期**: 2026-04-05
- **规则**: auth/refresh.ts 的 try-catch 只包 network call,不包 parse
- **原因**: 嵌套 catch 会吃掉真实错误,改了 3 次才发现
- **对应 feedback-log 条目**: [2026-04-01] [2026-04-03] [2026-04-05]
- **状态**: active
```

**归档规则**:
- 每月归档
- 超过 3 个月没被触发的 `active` 降为 `archived`
- learned 归档到 `feedback-log.md` 的 `## Archived` 段
- 规则够普适 → 进 CLAUDE.md 或 hook

---

## 读取时机表

| 时机 | 读哪层 |
|-----|-------|
| brainstorm 前(建立约束) | **architectural**(全部)+ **operational**(相关模块) |
| load context 阶段 | **operational**(相关模块)+ **learned**(相关模块) |
| 写代码卡住时 | **learned**(相关模块)— "之前踩过什么坑" |
| reviewer-quality 审查时 | **operational**(约定)+ **learned**(重犯检查) |
| compact 后 resume | **architectural**(不变约束) |

---

## 归档规则

| 层 | 归档策略 |
|---|---------|
| **Architectural** | 不归档,永远保留全部历史 |
| **Operational** | 每季度 review,过时标 deprecated,不删 |
| **Learned** | 每月归档,超 3 月未触发 → archived → feedback-log#Archived |

---

## 写入的硬动作(嵌入 workflow)

在 `rules/workflow.md` 的"任务收尾 checklist"里有两条:

- [ ] 本 task 是否产生新的 architectural 决策?(有 → `decisions.md#Architectural`)
- [ ] 本 task 是否产生新的 operational 约定?(有 → `decisions.md#Operational`)

**learned 层不在 task 收尾时写**,由 feedback-log 同类 3 次自动触发升级。

---

## 三层的文件结构

`decisions.md` 使用三个二级标题分层:

```markdown
# 架构决策记录

## Architectural
(长期决策)

## Operational
(中期约定)

## Learned
(短期沉淀)
```

详细模板见 `templates/memory/decisions.md`。
