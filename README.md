# init-rules

User-level Claude Code skill that initializes, audits, or improves the **current repository's** Claude Code rules (`CLAUDE.md` and path-scoped `.claude/rules/*.md`) from verified repository evidence.

- Invocation: `/init-rules [help | enforce | coordinator | herdr | <focus area>]` (`/init-rules help` shows usage)
- Explicit-only: `disable-model-invocation: true`, so Claude never runs it on its own.
- Writes only the current project's `CLAUDE.md` / `.claude/rules/`; never global config or application code. With `enforce`, it also installs the mechanical guardrails you confirm item by item (permission deny/ask rules, a fast per-file lint/format feedback hook), merged into `.claude/settings.json`.

## Layout

| File | Loaded |
|---|---|
| `SKILL.md` | on `/init-rules`: scope, principles, workflow phases, Definition of Done, report |
| `references/discovery.md` | Phases B–D: what to discover (versions, architecture, performance, conventions, docs, …) and the risk list |
| `references/rule-catalog.md` | Phase F: the rule families to generate |
| `references/self-review.md` | Phase H: adversarial scenarios |
| `references/enforcement.md` | section 8 / `enforce`: verified settings and hook formats, feedback hook template |
| `tests/` | regression test (not loaded by the skill) |

## Install

```bash
git clone git@github.com:aleslanger/init-rules.git ~/projects/init-rules
ln -sfn ~/projects/init-rules ~/.claude/skills/init-rules   # -n: safe to rerun
```

## Test

```bash
tests/run.sh          # runs the skill headless on a trap-filled fixture repo and checks the result (several minutes)
tests/run.sh --keep   # keep the temp repo and report for inspection
```

The fixture (`tests/fixtures/demo`) contains deliberate traps: a nonexistent `npm test`, a nonexistent `./deploy.sh`, generic filler, a valid ownership rule that must survive, generated protobuf code, and a secret placeholder that must not be copied.

Verified with Claude Code 2.1.289.
