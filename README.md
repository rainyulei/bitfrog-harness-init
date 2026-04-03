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

## Install

Add the marketplace:

```
/plugin marketplace add rainyulei/bitfrog-harness-init
```

Install the plugin:

```
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
