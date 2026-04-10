# Feedback

微 feedback + 深 feedback 两阶段规范。

---

## 微 feedback(每 task 必写)

**定位**:task 级状态 log,便宜动作,不是复盘。

**格式**:
```
YYYY-MM-DD HH:MM | task# | {动作} | {意外 (无则"无")}
```

**位置**:`.claude/state/session.md` 的"今日 log"段。

**强制方式**:嵌入 plan 的 **Step 10**(不靠自觉)。每完成一个 task 必须在 session.md 追加一行。

**时机**:每完成一个 task 的 Step 10。

**理由**:便宜动作一定写,累积成低成本的活动 trace,既给未来 session 参考,也是深 feedback 的触发器。

---

## 深 feedback(踩坑才写)

**定位**:踩坑后的复盘,为了进化规则。信息密度高。

**位置**:`feedback-log.md`(保留 v3 格式)。

**格式**:
```markdown
### [YYYY-MM-DD] 问题简述
- **场景**: 在做什么时出的问题
- **错误**: 具体错了什么
- **原因**: 为什么会出错
- **修复**: 怎么修的
- **分类**: [模块边界|测试|类型|AI 调用|其他]
- **规则更新**: 无 / CLAUDE.md 加了 xxx / hook 加了 xxx
```

**触发条件**(任一):
- verify 连续 2 次失败
- 同类问题 2 次出现
- 用户纠正做法("stop doing X")
- 微 feedback 的"意外"字段非空且严重

**Hook 兜底**:verify 失败 ≥ 2 次 && session 未写 deep feedback → 提醒。

---

## 两阶段的升级和归档

### 升级规则

微 feedback 的"意外"字段非空 → 自动触发升级:应该在 `feedback-log.md` 写完整的深 feedback 条目。

### 归档规则(保留 v3 进化机制)

- `feedback-log.md` **同类 3 次** → 沉淀到 `rules/` 的对应文件或 `settings.json` 加 hook
- 沉淀后的 feedback 归档到 `feedback-log.md` 的 `## Archived` 段
- 规则够普适(不只是本项目特有)→ 升级到 CLAUDE.md 的进化规则段

### 进化机制图

```
微 feedback(每 task 必写)
  ↓ "意外"非空
深 feedback(写入 feedback-log.md)
  ↓ 同类 3 次
沉淀为 rule 或 hook(写入 rules/*.md 或 settings.json)
  ↓ 规则够普适
升级到 CLAUDE.md 进化规则段
```

---

## 和其他 rules 的关系

- **rules/workflow.md** — 任务收尾 checklist 引用本文件(微 + 深 feedback)
- **rules/plan.md** — plan 的 Step 10 引用微 feedback
- **rules/review.md** — review 发现"本来就有的 bug" → 写深 feedback(类别 2 finding)
- **rules/verify.md** — verify 连续失败 → 触发深 feedback
