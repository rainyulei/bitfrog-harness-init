# Changelog

All notable changes to bitfrog-harness-init will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [2.0.0] - 2026-04-10

### Added
- Templates directory with 22 template files (CLAUDE.md + 11 rules + 4 agents + memory/state/hooks/README/CHANGELOG skeletons)
- Dynamic brainstorm mode (action pool + exit conditions, not fixed question lists)
- Two-stage review loop (reviewer-spec + reviewer-quality)
- Plan as movable contract (.claude/plans/)
- Session state for compact-safe work resumption (.claude/state/session.md)
- Worktree infrastructure (.worktrees/ with .gitignore)
- Doc management layer (rules/docs.md + role collaboration table)
- Memory three-tier layering (architectural / operational / learned)
- Upgrade mode (7-step flow with backup + rollback)
- 3 new skeleton hooks (Hook 3: commit-review gate, Hook 4: worktree ignore verify, Hook 5: plan format self-check)
- Phase 2 questions Q11-Q14 (docs habits / worktree / architectural init / project-specific triggers)
- Phase 3 sections 5-8 (rules inventory / infrastructure / docs skeleton / two-stage review)
- Phase 5 sections 5.7-5.11 (rules / review-split / infrastructure / docs / memory validation)

### Changed
- SKILL.md: Phase 1-5 extended, upgrade mode section added, templates/ referenced instead of inline
- `.claude/agents/reviewer.md` renamed to `reviewer-quality.md`
- Root CLAUDE.md template changed from monolithic to pointer-table style
- `.claude/memory/decisions.md` changed from single-layer to three-tier
- `.claude/settings.json` hooks from 2 to 5+ skeleton hooks

### Removed
- "Future support for Cursor/Codex/Windsurf" promise (deferred to v5)
- Inline template content in SKILL.md (moved to templates/ directory)

## [1.0.0] - 2026-04-04

- v3: remove Plans.md, add project goal, hooks skeleton/business layers, upgrade mode
- Move SKILL.md to root for direct skill discovery
- Update install: one-line paste in Claude Code
- Initial release: bitfrog-harness-init plugin
