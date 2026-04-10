# Docs

完整的文档管理规范 + 工作方式。

---

## 为什么这么严

> **AI 在这个项目里工作的唯一信息源是文件,不是对话。** 对话会被 compact 掉,文件不会。
>
> 所以:
> - AI 下一次 session 能知道这个项目当前状态,**全靠读文档**
> - 代码改了文档没改 → 下一次 session 的 AI 读到过期文档 → 做错误的判断 → 埋新 bug
> - 文档必须是 **source of truth**,不是"有时间再写"的副产品
>
> 因此:任何代码变动,只要触发了 `#B` 的条件,**文档必须在同一 commit 内同步**。文档违规 = **MAJOR → FAIL**,和代码 bug 同级,不降级不放水。

---

## 三层读者

| 层 | 读者 | 文档 | 位置 |
|---|-----|-----|-----|
| **给用户的** | 跑这个项目的人 | README / 使用示例 / CHANGELOG | 根目录 |
| **给开发者的** | 改这个项目的人 | dev setup / CONTRIBUTING / 运行验证 | 根目录或 `docs/` |
| **给 AI 的** | 下一次 session 的 AI | CLAUDE.md / `.claude/rules/*.md` / `decisions.md` / 模块接口 | `.claude/` 和 `docs/modules/` |

**每类文档严禁混内容**:README 不写 AI-load 的指针表,CLAUDE.md 不写用户跑项目的命令。

---

## B. 触发时机

触发条件 → 必须同步的文档:

| 代码里发生的事 | 必须同步的文档 |
|---|---|
| 新增/修改公开接口(exported function, public API, command flag) | `docs/modules/<module>.md` 或 README 的命令段 |
| 架构决策(选了 X 而不是 Y,且有 tradeoff) | `.claude/memory/decisions.md` 追加一条 |
| 破坏性变更 | `CHANGELOG.md` + 迁移说明 |
| 环境/依赖变更 | dev setup 段 |
| 模块边界调整 | `.claude/rules/modules.md` + 相关模块的接口文档 |
| 发现"这个项目 AI 容易踩的坑" | `feedback-log.md`(见 `rules/feedback.md`) |
| 本次 session 的 task 没动任何公开面 | **不写文档**(不强制) |

---

## C. 写作红线

AI 写文档时容易踩的坑,一条条禁掉:

1. **禁止 restate 代码**:不写 `"This function parses JSON"`,代码自己说。写"We parse JSON here because the upstream API only supports text/plain and gives us a JSON-shaped string; we need to re-interpret it"
2. **禁止提前文档**:不写未实现的接口、"将来会支持 Y"、占位
3. **代码示例必须可运行**:写了示例就要能 copy-paste 跑起来。不能运行的 → 删掉或改成可运行的
4. **禁止重复**:一个事实只在一处说,其他地方引用,不 copy paste(避免漂移)
5. **禁止装腔作势**:`comprehensive / robust / nuanced / leveraging` 这类词直接删
6. **禁止无署名决定**:`decisions.md` 每条决定必须有日期和触发原因
7. **禁止文档独立于代码的 commit**:文档更新必须和对应代码在同一个 commit

---

## D. Review 融合

### reviewer-spec 的文档维度

对照 plan 检查 spec compliance 时,**同时检查**:
- 本 task 的代码改动是否触发了 `#B` 的任一条件
- 如触发,对应文档是否在**同一 commit 内**更新

文档缺失 = spec gap = **不通过**。

### reviewer-quality 的文档维度

审查代码质量时,**同时审查**本 commit 涉及的文档是否违反 `#C` 的红线。

文档违规 = **MAJOR → FAIL**(和代码 bug 同级)。

---

## E. AI-load 友好性

### 根 CLAUDE.md

永远保持精简(项目目标 + 骨架指针表),不写细节(细节在 `rules/*.md`)。

### 模块接口文档

放在 `docs/modules/<module>.md`,每个模块一个文件,结构固定:

```markdown
# Module: <name>

## 职责(一句话)
## 对外接口(函数签名 + 用途 + 示例)
## 依赖(上游 / 下游)
## 边界(不允许做什么 — 对应 rules/modules.md)
## 已知陷阱(来自 feedback-log.md 沉淀)
```

### docs agent 被 dispatch 时,只读

1. 根 `CLAUDE.md`
2. `.claude/memory/decisions.md`(最近 10 条)
3. 当前 task 的 `git diff`
4. 对应的 `docs/modules/<module>.md`(如果存在)

**不读** `.claude/rules/` 其他文件(那是 worker 用的开发规则,不是文档写作所需)。

---

## F. Evolution 规则

保持 bitfrog 现有的 feedback-log 驱动 evolution:

- **某个文档被 AI 多次误读或漂移超过 3 次** → 这个文档的结构有问题 → 进入升级模式 brainstorm 重写该文档

---

## 工作方式(角色协作表)

| 角色 | 在文档维度上做什么 | 不做什么 |
|-----|-----------------|---------|
| **worker** | 写代码时同步维护本 task 涉及的文档(plan Step 11 触发)。只做**小范围同步改动** — 改一两行、追加一个接口说明、追加 CHANGELOG 一行 | 不做整段重写,不动别的 task 的文档 |
| **docs agent** | 被 worker 或 workflow **主动 dispatch** 时出场,做**整段/整个文档的重构或重写** | 不参与日常小修,不自动跑(必须被显式 dispatch) |
| **reviewer-spec** | 按 `#B` 检查:代码改了触发条件的东西,对应文档是否在**同一 commit 内**更新 | 不审文档质量,只审"漏没漏" |
| **reviewer-quality** | 按 `#C` 红线检查文档质量 | 不重复 reviewer-spec 的"漏没漏"检查 |

### worker 自修 vs dispatch docs agent

| 情况 | 选哪个 | 理由 |
|-----|-------|-----|
| 改动只涉及"加一行""改一个参数说明""追加 CHANGELOG 一条" | worker 自修 | 上下文在手上,dispatch overhead 不值 |
| 新增整个模块 → 写全新 `docs/modules/<module>.md` | dispatch docs agent | docs agent 有专门模板和职责 |
| 重构影响 3+ 个文档 | dispatch docs agent | 跨文档一致性需要独立 context |
| 只是 typo / 格式调整 | worker 自修 | 不值得 dispatch |

---

## 文档维度在 two-stage review 里的触发顺序

```
worker 写代码
  ↓ 改动命中 #B 任一触发条件?
    是 → 决定:小修 or 大改
           小修 → worker 自己改(Step 11)
           大改 → dispatch docs agent(Step 11 替代路径)
    否 → 跳过 Step 11
  ↓
commit(代码 + 文档 同一 commit,强制)
  ↓
reviewer-spec:文档同步了吗?(漏 = spec gap)
  ↓ 通过
reviewer-quality:文档红线违反了吗?(违反 = MAJOR → FAIL)
  ↓ 通过
勾 checkbox
```

---

## 本项目文档清单

<!-- 由 bitfrog Phase 2 Q11 产出填充。以下是业务层内容,升级模式下重写。 -->

{本项目需要维护的文档列表:README / CHANGELOG / docs/modules/ / ARCHITECTURE.md 等}

---

## 本项目文档同步触发器

<!-- 由 bitfrog Phase 2 Q11 产出填充。以下是业务层内容。 -->

{本项目特有的"改了 X 必须同步 Y 文档"的规则,如"改 public API 必须更新 OpenAPI schema"}
