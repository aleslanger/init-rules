---
name: init-rules
description: Initialize, audit, or improve the CURRENT repository's Claude Code rules (CLAUDE.md and .claude/rules/*.md) from verified repository evidence. Explicit invocation only via /init-rules.
disable-model-invocation: true
---

# /init-rules: evidence-based project rules

You are creating, auditing, or improving the Claude Code configuration of the **current repository only**. Optional user focus: $ARGUMENTS

## 0. Scope and hard limits

- Write ONLY to the current project's files: `CLAUDE.md` (root or `.claude/CLAUDE.md`, whichever the project already uses; default root), and `.claude/rules/*.md`. The only exception is the conditional `AGENTS.md`/nested `CLAUDE.md` edit described below.
- NEVER modify `~/.claude/CLAUDE.md`, `~/.claude/rules/`, `~/.claude/settings*.json`, user skills, or any other global or user-level configuration.
- NEVER modify application source, tests, CI, manifests, or dependencies. NEVER commit, push, or change git state.
- Do NOT edit `.claude/settings*.json` or `.claude/hooks/` automatically. Mechanical enforcement is RECOMMENDED in the final report, not installed, unless the user explicitly asks for it afterwards.
- Do NOT edit `AGENTS.md` or nested `CLAUDE.md` files unless they are clearly Claude config for this repo AND the change is needed to remove a conflict; if so, say so in the report.
- Determine the repository root first (`git rev-parse --show-toplevel`, else the working directory). If the working directory is your home directory or not a project, stop and ask.

## 1. Core philosophy (applies to analysis AND to the rules you write)

**Evidence over assumptions.** Repository evidence is the source of truth. Never invent commands, technologies, versions, architecture, deployment, test procedures, security mechanisms, production behavior, ownership, or operational requirements. Classify every finding as:
- **VERIFIED**: directly observed in executable config, manifests, CI, or code.
- **INFERRED**: plausible from evidence but not directly confirmed. Never state as fact in a rule.
- **UNKNOWN**: not established. Never becomes a rule. Goes to the report's Unknowns.

**Accuracy beats comprehensiveness.** A small, precise configuration beats a large, plausible-looking one. Every rule must change behavior; delete anything that would not.

The generated rules must instill these agent behaviors, phrased concretely for THIS repository (not as slogans):
- **Think before coding:** inspect the implementation, nearby context, callers/consumers, and relevant tests; understand expected behavior and blast radius before editing. The first plausible location is not proof the full change has been found.
- **Prefer simplicity:** smallest coherent solution; established repo patterns; no speculative abstractions, premature generalization, unnecessary dependencies, or hypothetical architecture.
- **Surgical changes:** every changed line traces to the task, a necessary supporting change, a correctness issue the task exposed, or required validation. No opportunistic refactors, unrelated formatting/renaming, dependency churn, or drive-by cleanup.
- **Own the outcome:** understand → implement → validate → investigate failures → fix introduced failures → completeness sweep → inspect final diff → report remaining uncertainty accurately. Editing files is never "done".
- **Verify before reporting:** distinguish implemented / statically checked / unit tested / integration tested / manually exercised / fully validated / unverified. Never claim tests pass, build succeeds, the app works, a bug is fixed, or a migration is safe unless that verification actually ran and succeeded. State exactly what remains unverified.
- **Condition → action → verification:** write rules as `WHEN <condition> → DO <action> → VERIFY <observable result>` wherever the shape fits. Never write "write clean code", "follow best practices", "be careful", "write secure code", or similar filler.

## 2. Workflow

Track these phases with a todo list. Do not skip phases; keep each one proportional to the repo's size.

### Phase A: Inventory existing agent configuration (always first)

Read, if present: root and nested `CLAUDE.md`, `.claude/CLAUDE.md`, `CLAUDE.local.md` (read-only; it is personal, never edit it), `AGENTS.md`, `.claude/rules/**`, `.claude/settings.json`, `.claude/settings.local.json` (note only; never copy values), `.claude/hooks/`, `.claude/skills/`, `.claude/agents/`, `.claude/workflows/`, `.cursorrules`, `.github/copilot-instructions.md`, and contributor/architecture docs (`CONTRIBUTING*`, `docs/architecture*`, ADRs).

Record what already exists. This decides whether this run is a **create** or an **audit/update** (idempotency, section 9).

### Phase B: Targeted repository discovery

Use cheap evidence first: `git ls-files | head`/counts, top-level tree, manifests, CI files, then targeted reads. Do not read the repository blindly. For large repos, you may delegate read-only discovery to an Explore subagent, using a smaller/faster model where the tool allows it, and keep only conclusions.

Establish, where evidence permits, and label each item VERIFIED / INFERRED / UNKNOWN:

- **Structure:** apps, packages, libraries, modules, services, infrastructure, tests, generated output, docs.
- **Tooling:** languages, frameworks, build tooling, package manager (lockfile is evidence), build/test/lint/format/typecheck/static-analysis tools.
- **Architecture:** entry points, module/package boundaries, public interfaces, APIs, persistence, workers/queues, external integrations.
- **Data:** persistence model, schemas, migrations (and their tool), transaction boundaries, generated models, source-of-truth files.
- **Security:** input validation, authentication, authorization, secret handling, env configuration, trust boundaries, sensitive data. Treat **authentication and authorization separately**; authentication never implies authorization.
- **Operations:** CI, deployment, releases, versioning, logging, monitoring, error handling, runtime config.

**Evidence hierarchy** when sources conflict (strongest first):
1. executable configuration
2. CI configuration
3. build/package manifests
4. test/static-analysis configuration
5. source code
6. architecture docs
7. contributor docs
8. README
9. repeated repository convention
10. inference

Surface material conflicts in the report; do not silently resolve them.

### Phase C: Canonical commands

Discover the repository-supported command for each task that applies: bootstrap/install, build, focused test (single file/test), broader test, lint, format, typecheck, static analysis, dev runtime, code generation.

- Prefer commands CI runs or repository scripts define (`package.json` scripts, `Makefile`, `justfile`, `Taskfile`, `pyproject` tool config, `tox`/`nox`, `Cargo`, `go`, etc.).
- Confirm each command exists: the script/target is defined, and the tool is a declared dependency or used by CI.
- You MAY run cheap, read-only, side-effect-free commands to verify (e.g. `--help`, `--version`, listing targets, a dry run). Do NOT run installs, builds, migrations, deploys, or anything that mutates state, network resources, or databases without asking.
- Never put an unverified command in CLAUDE.md. If the focused-test syntax can't be established, say UNKNOWN rather than guessing.

### Phase D: Risk analysis

Evaluate only risks actually relevant to this repo, with evidence:

- **Correctness:** regressions, stale callers, API contract changes, compatibility, error handling, partial state.
- **Inputs/injection:** validation, normalization, malformed input, SQL/query injection, shell/command injection, path traversal, unsafe deserialization, other trust boundaries.
- **AuthN / AuthZ (separately):** where identity is established; where permissions are enforced; ownership/resource-level checks; privilege boundaries.
- **Data integrity:** migrations, destructive ops, transactions, partial writes, retry safety, duplicate processing, idempotency.
- **Concurrency:** races, atomicity, locking, check-then-act, ordering assumptions, concurrent updates.
- **Performance (only where relevant):** Big-O, hot loops, N+1 access, repeated I/O or network calls, blocking work, memory-heavy ops, unbounded collections. Do not invent performance requirements.
- **Reliability:** dependency failures, timeouts, retries, cleanup, partial availability, resource leaks.
- **Observability:** meaningful logging, sensitive-data leakage in logs, the monitoring/tracing the project already uses, useful error context. Do not impose a stack the project doesn't use.
- **Generated files:** identify generated artifacts and their source of truth and generator command (section 4).
- **Secrets:** where secrets enter (env files, secret managers, CI secrets). Never read out, copy, or quote secret values; note only the mechanism and file paths.

### Phase E: Audit existing configuration (if any)

Never overwrite existing rules blindly. Classify every existing instruction as one of:
- **KEEP**: true, valuable, correctly scoped.
- **REWRITE**: valuable but vague, unactionable, or unobservable.
- **MOVE**: belongs in a path-scoped rule, or vice versa.
- **DELETE**: obsolete, duplicate, contradicted by evidence, generic filler, or references nonexistent commands/paths.
- **ENFORCE MECHANICALLY**: a hard restriction that exists only as prose and whose violation is unacceptable.
- **UNKNOWN**: can't verify; keep unchanged and flag it, rather than delete it on a guess.

Look specifically for: obsolete facts, duplicates, conflicts, nonexistent commands, nonexistent paths, generic filler, wrong scope, and "hard restrictions" that are only prose. Preserve intentional project knowledge (team conventions, gotchas, decisions) even if you can't fully re-derive it, unless evidence contradicts it.

### Phase F: Design and write the configuration

**Architecture:** a compact root CLAUDE.md plus a few focused `.claude/rules/*.md`.

**CLAUDE.md**: only broadly needed, high-value, VERIFIED content:
- project purpose (one or two lines)
- architecture summary and key boundaries
- canonical commands (verified only)
- universal project invariants
- working method (the behaviors from section 1, made concrete for this repo, compactly)
- repository-specific **Definition of Done** (section 7)

Target about 50–150 lines; exceed about 200 only with a stated reason. Not a handbook. Don't restate what Claude can trivially read from the code (file listings, obvious conventions).

**`.claude/rules/*.md`**: one focused concern per file. Create a file only when there is enough repository-specific content to justify it. Candidate names, used ONLY where justified: `verification.md`, `change-discipline.md`, `security.md`, `testing.md`, `architecture.md`, `database.md`, `api.md`, `generated-files.md`, `orchestration.md`. Fewer strong rules beat many weak ones. Avoid duplicating content between CLAUDE.md and rules files.

**Path scoping:** when a rule applies only to part of the repo (API, migrations, frontend, backend, infra, generated files, a package), scope it with the `paths` frontmatter this Claude Code version supports for `.claude/rules/*.md`:

```markdown
---
paths:
  - "src/api/**/*.ts"
  - "migrations/**"
---
```

If a write to `.claude/rules/` is denied or blocked by a permission prompt, do NOT fold path-specific rules into CLAUDE.md as a workaround. Ask the user to approve the write; if that is impossible (headless run), leave CLAUDE.md compact and put the proposed rules files' full content in the final report.

Rules without `paths` load always. Every glob MUST match at least one existing tracked file. Check with `git ls-files '<glob>'`, or `find` when the repo is not a git repo. Do not invent other frontmatter keys.

**Rule content requirements** (include each only where applicable to this repo):

- **Sweep verification:** after a change, inspect affected callers, consumers, implementations, schemas, types, tests, fixtures, snapshots, migrations, config, generated artifacts, and directly related docs, with depth proportional to blast radius. Don't require repo-wide searches for trivial edits. Name the concrete places in THIS repo where ripple effects happen (e.g. a shared types package and its consumers).
- **Failure handling:** treat failures as evidence. Read the actual error. Classify it as introduced / pre-existing / environmental / unknown. Fix failures your change introduced. Report the rest accurately. Don't rerun an identical failing command without changing something or gaining information.
- **Validation-bypass protection:** never silently disable or skip tests, remove meaningful assertions, skip required checks, suppress relevant errors or lint/type diagnostics, bypass hooks (`--no-verify` and equivalents), weaken validation, or disable security controls to get green. If a safeguard looks wrong, investigate and explain rather than bypass it.
- **Generated files** (where verified): `WHEN generated output must change → DO edit its source of truth and run <verified generator command> → VERIFY the generated diff corresponds to the source change.` Never hand-edit generated artifacts unless repo evidence says that's correct.
- **Secrets:** never put secret values into Claude config or responses. Rules must forbid committing credentials, echoing or logging secrets, exposing them in responses, and replacing secret injection with hardcoded values. Name the repo's actual secret mechanism and files (paths only).
- **AuthN/AuthZ:** where the repo has both, a scoped rule: WHEN changing authentication or request handling → DO verify authorization/ownership checks separately at <verified enforcement points> → VERIFY with tests or code inspection that the permission path is covered.
- **Data/concurrency:** where the repo mutates shared or persistent state, retries, or processes queues: require examining transactions, idempotency, duplicate processing, and check-then-act races, scoped to the relevant paths.
- **Performance:** only for verified hot paths or query layers: consider complexity, N+1 access, and repeated I/O when changing loops or data access there.
- **External calls:** where integrations exist: consider timeout, retry, and failure behavior using the patterns the repo already uses.
- **Orchestration (subagents / parallel agents):** generate only when the repo is large enough, or has independent verified modules, for delegation to matter; or when the project already defines agents/workflows in `.claude/agents/` or `.claude/workflows/`. A small single-module repo gets at most a line or two in CLAUDE.md, or nothing. Ground every rule in the repo's verified boundaries:
  - WHEN a task needs broad read-only exploration across many files → DO delegate it to a read-only subagent and keep only conclusions → VERIFY key claims by reading the cited files before acting on them. Do not delegate small or single-file tasks.
  - WHEN splitting work across parallel agents → DO partition by verified module or package boundaries (name them) so that no two agents edit the same file; a shared contract (schema, generated client, shared types, migrations) is changed by one agent first, and consumers follow → VERIFY the agents' file sets are disjoint before launching them.
  - WHEN delegating → DO give each agent a self-contained brief: goal, files in scope, files out of scope, constraints from this config, the verified validation commands, and the expected output → VERIFY the brief doesn't rely on conversation context the agent can't see.
  - WHEN an agent reports results → DO treat them as claims, not evidence: inspect its diff and re-run the relevant validation yourself → VERIFY before reporting; never relay "tests pass" or "fixed" from an agent without observed output.
  - WHEN parallel work is merged → DO run broader validation on the combined result and review the full final diff yourself; integration failures belong to the orchestrator → VERIFY the combined result, not each part in isolation.
  - Prefer project-defined agents and workflows when they exist; reference them by their actual names. Keep the number of agents proportional to the task. Destructive or outward-facing actions (deploys, migrations, pushes) stay with the orchestrator and need the user's confirmation; never delegate them.
  - **Model and effort proportionality.** WHEN delegating → DO pick the cheapest model and lowest reasoning effort that can do the job: a smaller, faster model and low effort for search, lookups, summarizing, and mechanical edits; the largest model and high/xhigh/max effort only for genuinely hard reasoning (cross-module design, subtle security or concurrency analysis, a bug that cheaper attempts failed on) → VERIFY that any elevated choice has a stated reason in the brief. Escalate on evidence (insufficient output, a failed attempt), never preemptively. Only use mechanisms the installed version supports: the per-call `model` override on the Agent tool, and the `model` / `effort` frontmatter in `.claude/agents/*.md`. Prefer model aliases over pinned model IDs unless the repo pins them. WHEN an existing project agent definition sets the largest model or high+ effort for routine work → report it as a REWRITE recommendation (agent definitions are outside this skill's write scope).
- **Herdr (herdr.dev), only if used:** generate these rules only when there's evidence the repo or its contributors use Herdr (references in agent config, docs, or scripts, e.g. `git grep -il herdr`; a `herdr` skill in `.claude/skills/`) or the user asks for them via the arguments. Otherwise omit them. Don't copy CLI syntax into the rules; point to `herdr --help` and the herdr skill, which are authoritative. Rules:
  - WHEN about to use Herdr → DO check `test "${HERDR_ENV:-}" = 1` and use Herdr only when the user explicitly asked for it → VERIFY the check passed; otherwise use the built-in subagents, and never control a Herdr session from outside a Herdr-managed pane.
  - WHEN starting an agent or command through Herdr → DO use a sibling pane in the current tab and cwd, with background (no-focus) mode and a unique agent name; create workspaces, tabs, or worktrees only on explicit request → VERIFY by taking pane and agent IDs from the JSON responses, never by guessing them.
  - The partitioning, brief, and claims-not-evidence rules above apply to Herdr agents too: one owner per file, and read the agent's output and inspect its diff yourself before reporting. A Herdr `unknown` state doesn't prove completion. WHEN an agent is `blocked` on an approval or question → DO inspect it and ask the user, never answer it on your own.
  - Model and effort proportionality applies to agents started through Herdr; pass native agent options only as listed by the installed `herdr agent` help.
  - Never close workspaces, tabs, or panes you didn't create, never run `herdr server stop`, and never kill the Herdr process unless the user explicitly asks.

### Phase G: Rule quality gate

For every instruction, ask:
1. **True?** Backed by repository evidence, or genuinely universal agent behavior?
2. **Behavior-changing?** If removing it changes nothing, remove it.
3. **Actionable?** Does Claude know what to do?
4. **Observable?** Can compliance be checked?
5. **Scope?** Should it be path-scoped?
6. **Duplicated?** If so, consolidate.
7. **Filler?** If so, delete.
8. **Is prose enough?** If failure is unacceptable, flag it for mechanical enforcement (section 8).

### Phase H: Adversarial self-review

Mentally run the resulting rules against these scenarios, keeping only the ones applicable to this repo, and fix the rules if any scenario slips through:
- A small change is turning into a giant refactor. Do the surgical-change rules stop it?
- A shared API changed without inspecting consumers. Is impact analysis required?
- A generated file was edited directly. Would Claude find the source of truth?
- A test fails. Would Claude investigate rather than disable it?
- Authentication code changes. Would authorization be examined separately?
- Validation can't run. Would Claude avoid claiming success?
- A secret is encountered. Is disclosure prevented?
- Retryable or concurrent state mutation changes. Are races and idempotency examined?
- A hot path or query loop changes. Are algorithmic and I/O costs considered?
- An external dependency call changes. Are timeout and failure behavior considered?
- Two parallel agents both need to edit a shared file or contract. Do the rules force a single owner and an ordering?
- A subagent reports "all tests pass". Would the orchestrator re-verify before reporting it?
- A trivial lookup is delegated to the largest model at max effort. Do the rules steer it to a cheaper model and lower effort?

### Phase I: Validate the configuration

Using only capabilities that actually exist:
- Files exist at the discovered locations: `CLAUDE.md` (root or `.claude/`) and `.claude/rules/*.md`.
- Frontmatter parses as YAML and uses only `paths` (rules files). Check by inspection or a quick parse (e.g. `python3 -c 'import yaml,sys; ...'` if available; otherwise state that it was inspected manually).
- Every `paths` glob matches existing files (`git ls-files` / `find`).
- Every command in the config is backed by a script, target, or CI step you observed. List each with its evidence.
- Every referenced file or directory path exists.
- No rules contradict each other or the remaining existing config.
- No secret values are present: grep the written files for key/token/password-like patterns and high-entropy strings.
- Optional, only if it works in this environment: `/memory` (interactive) lists loaded memory files. Mention it as a manual check for the user; do not claim it ran if you could not run it. Do not invent other diagnostic commands.

### Phase J: Final diff gate

Inspect the complete diff of everything you changed: `git diff` and `git status --porcelain` for tracked and untracked files, or a before/after comparison of copies saved in Phase A when the repo is not a git repo. Confirm:
- only CLAUDE.md and `.claude/rules/*.md` changed (plus any explicitly justified agent-config file); no application code, tests, CI, or manifests
- no secret values added
- all referenced paths exist, and all commands are repository-backed
- generated files are handled via their source of truth
- no contradictory, duplicate, or filler rules remain
- local rules are path-scoped appropriately
- no unsupported assumptions (INFERRED stated as fact, or UNKNOWN turned into a rule)
- nothing in the rules or the report implies validation that wasn't performed

Fix any issues, then re-inspect the diff.

## 7. Definition of Done (generated for the repo)

Generate a repository-specific DoD in CLAUDE.md containing only the items that apply, each naming verified commands:
- the requested behavior is implemented
- focused validation is run (name the verified focused-test command)
- broader validation is proportional to blast radius (name the verified commands)
- generated artifacts are refreshed via their generator, where applicable
- every failure is investigated and classified, and introduced failures are fixed
- the final diff is reviewed for unrelated changes
- the report states exactly what was and was not verified

Never require a command that couldn't be verified. If, for example, there's no verified lint command, omit lint rather than invent one.

## 8. Rules vs mechanical enforcement

Prose rules are guidance, not security boundaries. For each critical restriction, ask: *what happens if Claude ignores this?* If the answer includes secret exposure, a destructive production action, irreversible data modification, bypassing mandatory validation, a dangerous deployment, or corruption of protected generated files, list it as a mechanical-enforcement candidate. Possible enforcement points include Claude Code permissions (`permissions.deny` in `.claude/settings.json`), hooks (PreToolUse), CI checks, linters, branch protection, and filesystem permissions.

Do NOT create these automatically. Recommend them in the report. The one exception is when the user explicitly asked for it AND the behavior is unambiguous, safe, supported, and clearly appropriate.

## 9. Idempotency

This skill must be safe to rerun.
- Always start from Phase A: inspect what exists.
- Preserve intentional project knowledge and the existing file structure, headings, and wording when they're still correct. Do not reformat, reorder, or rephrase valid rules.
- Update only stale or incorrect facts, consolidate genuine duplication, and add rules only for newly verified risks.
- Don't recreate a rule that already exists in equivalent form, even under a different name.
- If nothing meaningful changed in the repo or the config, make little or no diff, and say so.

## 10. Final report (concise)

Return:

**Repository findings**: only important VERIFIED findings on architecture, tooling, boundaries, validation, and operations.

**Configuration changed**: for each file: `path` · purpose · scope (always-loaded or the `paths` globs) · created/updated.

**Highest-impact rules**: which rules most reduce realistic failure risk in this repo, and why.

**Existing knowledge preserved**: important pre-existing instructions that were kept.

**Rewritten or removed rules**: meaningful REWRITE/MOVE/DELETE decisions and the reason for each.

**Unknowns**: relevant facts that couldn't be established, plus conflicts found between sources.

**Risks**: only the relevant categories (correctness, security, data integrity, concurrency, performance, reliability, operations, maintainability).

**Mechanical enforcement candidates**: for each:
- Risk:
- Current guidance:
- Why prose may be insufficient:
- Potential enforcement point:
- Recommendation:

**Validation**: exactly which commands were executed, which checks were performed, what passed, what failed, what wasn't run, and what remains unverified. Never claim validation that didn't occur.
