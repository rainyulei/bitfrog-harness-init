# bitfrog-harness-init

One command to make any repo agent-ready.

`/bitfrog-harness-init` brainstorms your project deeply, then generates a minimal harness engineering configuration — so AI agents can write reliable code in your codebase.

## What it generates

```
your-project/
├── CLAUDE.md                      # AI workbook — project structure, module boundaries, test strategy, workflows
├── Plans.md                       # Development plan
├── feedback-log.md                # Error log — 3 similar issues → new rule
├── .claude/
│   ├── settings.json              # Hooks — auto-detect boundary violations, block sensitive files, pre-commit verify
│   ├── agents/
│   │   ├── worker.md              # Dev agent — test-first, feedback loop, forbidden actions
│   │   └── reviewer.md            # Review agent — read-only, PASS/FAIL with grep commands
│   └── memory/
│       └── decisions.md           # Architecture decisions — auto-loaded each session
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

## Principles

- **Brainstorm first** — won't generate config until it understands your project
- **Respect existing config** — extends your CLAUDE.md, merges into settings.json
- **Minimal viable harness** — no rules the project doesn't need
- **Hook errors must be useful** — what was violated → why the rule exists → how to fix
- **Evolution over perfection** — start simple, feedback-log drives improvements

## Reference implementation

See [storyflow](https://github.com/rainyulei/storyflow) for a complete example of harness engineering applied to a real project.

## License

MIT
