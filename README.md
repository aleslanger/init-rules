# init-rules

User-level Claude Code skill that initializes, audits, or improves the **current repository's** Claude Code rules (`CLAUDE.md` and path-scoped `.claude/rules/*.md`) from verified repository evidence.

- Invocation: `/init-rules [optional focus]`
- Explicit-only: `disable-model-invocation: true`, so Claude never runs it on its own.
- Writes only the current project's `CLAUDE.md` / `.claude/rules/`; never global config, application code, or settings. Mechanical enforcement (permissions, hooks) is recommended in the report, not installed.

## Install

```bash
git clone git@github.com:aleslanger/init-rules.git ~/projects/init-rules
ln -s ~/projects/init-rules ~/.claude/skills/init-rules
```

Verified with Claude Code 2.1.289.
