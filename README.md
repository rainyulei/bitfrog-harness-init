# bitfrog-harness-init

One command to make any repo agent-ready.

`/bitfrog-harness-init` brainstorms your project deeply, then generates a minimal harness engineering configuration — so AI agents can write reliable code in your codebase.

## What it generates

```
your-project/
├── CLAUDE.md                      # Root pointer table — project goal + skeleton index
├── feedback-log.md                # Deep feedback log — 3 similar → new rule/hook
├── .claude/
│   ├── settings.json              # Hooks (5 skeleton + N business)
│   ├── rules/                     # 11 rule files (methodology + constraints)
│   │   ├── workflow.md            # First action + flow + compact protocol
│   │   ├── brainstorm.md          # Dynamic action pool + exit conditions
│   │   ├── plan.md                # Plan format + 11-step task structure
│   │   ├── review.md              # Two-stage review loop
│   │   ├── verify.md              # 5-step evidence gate
│   │   └── ...                    # + feedback, subagent, modules, memory, docs, worktree
│   ├── agents/
│   │   ├── worker.md              # Implementer with 4-status reporting
│   │   ├── reviewer-spec.md       # Stage-1: spec compliance + doc sync
│   │   ├── reviewer-quality.md    # Stage-2: code quality + doc red lines
│   │   └── docs.md                # Large-scale doc refactor executor
│   ├── plans/                     # Plan files (git tracked, movable contracts)
│   ├── state/
│   │   └── session.md             # Compact-safe scratchpad (gitignored)
│   └── memory/
│       └── decisions.md           # 3-tier: Architectural / Operational / Learned
```

## How it works

1. **Explores** — reads your project files, git log, directory structure
2. **Brainstorms** — asks 8-11 questions one at a time to understand your project goals, module boundaries, test strategy, known pitfalls
3. **Confirms** — presents the harness design section by section for your approval
4. **Generates** — writes all config files tailored to your specific project

It never guesses. It asks first.

## Install — 10 seconds

Open Claude Code and paste this:

> Install bitfrog-harness-init: run **`git clone --single-branch --depth 1 https://github.com/rainyulei/bitfrog-harness-init.git ~/.claude/skills/bitfrog-harness-init`** then tell me it's ready and I can use `/bitfrog-harness-init` in any project.

That's it. Claude clones the skill and you're good to go.

### Add to your repo so teammates get it (optional)

> Add bitfrog-harness-init to this project: run **`mkdir -p .claude/skills && cp -Rf ~/.claude/skills/bitfrog-harness-init .claude/skills/bitfrog-harness-init && rm -rf .claude/skills/bitfrog-harness-init/.git`** then tell me it's ready.

Files get committed to your repo. `git clone` just works for teammates.

### Alternative: plugin marketplace

```
/plugin marketplace add rainyulei/bitfrog-harness-init
/plugin install bitfrog-harness-init@rainyulei-bitfrog-harness-init
```

## Usage

In any project directory:

```
/bitfrog-harness-init
```

Then answer the questions. That's it.

## What is Harness Engineering?

Three layers of working with AI:

| Layer | What it controls | Example |
|-------|-----------------|---------|
| Prompt Engineering | How you talk to AI | "Write a function that..." |
| Context Engineering | What AI can see | CLAUDE.md, docs, codebase |
| **Harness Engineering** | **What happens after AI acts** | **Hooks, tests, feedback loops, agent review** |

Harness Engineering is the third layer — it's not about making AI smarter, it's about making AI **reliable**.

This plugin generates the harness layer for your project.

## What's New in v4 (2026-04-10)

- **Dynamic brainstorm mode** — exit-condition driven, no fixed question lists
- **Plan as movable contract** — `.claude/plans/*.md` + compact-safe session state
- **Two-stage review loop** — reviewer-spec + reviewer-quality, mandatory order
- **Doc management layer** — code changes must sync docs, violations = MAJOR FAIL
- **Memory three-tier layering** — architectural / operational / learned
- **Worktree infrastructure** — isolation for large changes with baseline verification
- **Information architecture** — root CLAUDE.md becomes a pointer table, rules split into `.claude/rules/*.md`
- **Upgrade mode** — 7-step flow preserving skeleton layer, rewriting business layer only

See [design spec](docs/superpowers/specs/2026-04-10-bitfrog-v4-design.md) for details.

## Principles

- **Brainstorm first** — won't generate config until it understands your project
- **Respect existing config** — upgrade mode preserves skeleton layer and history
- **Minimal viable harness** — no rules the project doesn't need
- **Hook errors must be useful** — what was violated → why the rule exists → how to fix
- **Evolution over perfection** — start simple, feedback-log drives improvements
- **First action is mandatory** — always restate + check understanding before coding
- **Documents are compact-safe source of truth** — AI relies on files, not conversation memory
- **Selective integration** — absorb good parts of superpowers, skip ceremony

## Reference implementation

See [storyflow](https://github.com/rainyulei/storyflow) for a complete example of harness engineering applied to a real project.

## License

MIT
