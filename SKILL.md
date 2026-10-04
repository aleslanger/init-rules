---
name: init-rules
description: Initialize, audit, or improve the CURRENT repository's Claude Code rules (CLAUDE.md and .claude/rules/*.md) from verified repository evidence. Explicit invocation only via /init-rules.
disable-model-invocation: true
argument-hint: "[help | coordinator | herdr | <focus area>]"
---

# /init-rules: evidence-based project rules

You are creating, auditing, or improving the Claude Code configuration of the **current repository only**. Optional user focus: $ARGUMENTS

## Help mode

If the arguments are `help`, `--help`, `-h`, or `?`, output ONLY the help below, in the user's conversation language, and stop. Don't read, run, or write anything else.

```text
/init-rules: create, audit, or improve this repository's Claude Code rules
(CLAUDE.md + .claude/rules/*.md) from verified repository evidence.

Usage
  /init-rules                  analyze the repo; create the rules or audit/update existing ones
  /init-rules <focus area>     same, with extra attention to an area (e.g. "API", "migrations")
  /init-rules coordinator      also generate the coordinator model: the main agent doesn't write code;
                               it delegates, reviews (accept/return/reassign/discard/escalate),
                               merges, and supervises agent state
  /init-rules herdr            also generate Herdr (herdr.dev) rules, even without repo evidence of Herdr
  /init-rules help             show this help
  Arguments can be combined, e.g. /init-rules coordinator herdr API

Writes   only CLAUDE.md and .claude/rules/*.md of the current repo
         (Claude Code asks you to approve writes to .claude/)
Never    global ~/.claude config, application code, settings, hooks, commits, pushes
Rules    built only from VERIFIED evidence; commands must exist in the repo; path-scoped where possible;
         include verification, surgical-change, failure-handling, security, architecture (boundaries,
         dependency direction, exemplars), performance (budgets/benchmarks when found, hot paths),
         code quality (coding standard, conventions, no duplicate code), verified language/library
         versions (no invented APIs, no newer syntax, no deprecated APIs), docs/comments kept in sync
         with changes, and orchestration rules where they apply, plus a Definition of Done
Rerun    safe and idempotent; an unchanged repo gives little or no diff
Report   findings, files changed, highest-impact rules, preserved/rewritten rules, unknowns, risks,
         mechanical-enforcement candidates (recommended, not installed), exact validation performed
```

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
- **Never invent what doesn't exist:** no fabricated functions, methods, classes, modules, packages, config keys, CLI flags, env vars, files, endpoints, or library options. Anything used in code or cited in a report must first be confirmed to exist in the repo, in the installed or locked dependency version, or in that version's documentation. If its existence can't be confirmed, say so instead of guessing.
- **Version-correct code:** write code for the language, runtime, and dependency versions the repo actually targets. Use no syntax or API newer than the minimum supported version, and no deprecated API.
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
- **Versions:** the language/runtime version and the supported range, from:
  - version files: `.python-version`, `.nvmrc`/`.node-version`, `.tool-versions`, `mise.toml`, `rust-toolchain.toml`;
  - manifest fields: `requires-python`, `engines`, the `go` directive in `go.mod`, `edition`/`rust-version`, `composer.json` `php`, Gradle/Maven toolchain or `release`;
  - TS `target`/`lib`, Dockerfile base images, the CI matrix;
  - for frameworks and key libraries, the **locked** versions from the lockfile (not the manifest range).

  Derive the **minimum** version code must run on. When the CI matrix covers several versions, the lowest one counts. Surface conflicts between sources (e.g. `.python-version` 3.12 but `requires-python >=3.10`).
- **Deprecation tooling:** what already flags deprecated or outdated usage, with its command. Examples: ruff `UP` (pyupgrade) and its `target-version`, pyright/mypy deprecation reporting, `-W error::DeprecationWarning` in pytest config, eslint deprecation rules, TS `@deprecated` diagnostics, staticcheck `SA1019`, `go vet`, rustc/clippy deprecation lints, `-Xlint:deprecation`/`-Werror`, PHPStan deprecation rules. Also record where deprecation warnings show up (test and build output).
- **Architecture:** entry points, module/package boundaries, public interfaces, APIs, persistence, workers/queues, external integrations. Look for explicit architecture evidence:
  - ADRs and `docs/architecture*`;
  - dependency-rule tooling: import-linter, dependency-cruiser, eslint boundary/import rules, ArchUnit, Nx/Turborepo project constraints, Bazel/Gradle module visibility, Go `internal/`, TS project references or path aliases;
  - layering visible in the code (e.g. routes → services → repositories), and where it is violated;
  - the dependency direction between packages, and what is public versus internal per package.

  Record which rules are **enforced by tooling** (cite the config) versus **convention only** (cite 2–3 exemplars).
- **Performance:** look for explicit performance evidence before writing any performance rule:
  - benchmarks (pytest-benchmark, Go `Benchmark*`, criterion, JMH, vitest/jest bench);
  - load tests (k6, Locust, Gatling, JMeter);
  - budgets and thresholds (size-limit, bundlesize, Lighthouse CI budgets, perf assertions in tests);
  - SLO/latency targets in docs;
  - documented hot paths;
  - caching layers and their invalidation;
  - query patterns (eager loading, batching, pagination limits, indexes in migrations);
  - timeouts and pool sizes in config;
  - profiling or perf CI jobs.

  Note which evidence is executable (has a command) and which is documented only.
- **Code quality conventions:**
  - **Linting and types:** the enabled rule sets and severity (not just the linter's presence), type strictness, and suppression conventions (`noqa`, `eslint-disable`, `@ts-expect-error`, `#[allow]`) and how often they're used.
  - **Thresholds:** complexity and length limits, coverage thresholds.
  - **Errors and logging:** error-handling conventions (custom error types, result types, error wrapping, where errors get translated at boundaries) and the logging conventions.
  - **Tests:** test conventions (layout, fixtures, factories, mocking style, naming).
  - **Contribution process:** PR templates, CODEOWNERS, review checklists, `.editorconfig`.
  - **Coding standard:** the standard the repo actually adopts, and where it's enforced. Examples: PEP 8 via ruff/black/flake8, PSR-12 via phpcs, gofmt/golangci-lint, rustfmt/clippy, an eslint shared config (airbnb, standard), Google Java Style via checkstyle/spotless. Also look for style sections in `CONTRIBUTING*`, `STYLEGUIDE*`, or `docs/*standards*`.
  - **Conventions:** naming (file and directory case, test file names, identifiers, modules), import ordering, and module layout. Each must be confirmed by linter config or at least 3 consistent examples. Commit/PR conventions only when enforced (commitlint, conventional commits, changelog tooling).
  - **Reuse and duplication:** where the shared helpers, utils, types, constants, validators, and query helpers live (name the paths). Also any duplication tooling (jscpd, PMD CPD, pylint `duplicate-code`, Sonar) and its threshold.
  - **Exemplars:** 1–3 files that best represent "how code is written here" for each major kind of change (endpoint, service, component, migration, test).
- **Documentation:**
  - **Inventory:** READMEs (root and per package), `docs/`, `.env.example`, and where the commands, config/env vars, CLI flags, and public APIs are documented.
  - **Code-level docs:** docstring/JSDoc/rustdoc/godoc conventions and their linting (pydocstyle/ruff `D`, eslint-plugin-jsdoc, `missing_docs`).
  - **Changelog:** keep-a-changelog, changesets, release-please, or towncrier, and whether an entry is required per change.
  - **Doc tooling:** build tooling (MkDocs, Sphinx, Docusaurus, TypeDoc), link checks, doctests or tested examples, with their commands.
  - **Generated docs:** which docs are generated (OpenAPI, API reference) and from what.
  - **Drift:** existing doc drift, i.e. docs that contradict the code, including commands or paths in the README that don't exist.
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
- **Performance (only where relevant):** Big-O, hot loops, N+1 access, repeated I/O or network calls, blocking work, memory-heavy ops, unbounded collections. Do not invent performance requirements. Turn found budgets, benchmarks, and SLOs into rules; without such evidence, limit performance rules to the verified hot paths and data-access layers.
- **Architecture erosion:** boundary violations that already exist, circular dependencies, god modules, and layers the code bypasses. Rules must say how to treat existing violations: don't copy them, don't fix them unasked.
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

**`.claude/rules/*.md`**: one focused concern per file. Create a file only when there is enough repository-specific content to justify it. Candidate names, used ONLY where justified: `verification.md`, `change-discipline.md`, `security.md`, `testing.md`, `architecture.md`, `database.md`, `api.md`, `generated-files.md`, `orchestration.md`, `performance.md`, `code-quality.md`. Fewer strong rules beat many weak ones. Avoid duplicating content between CLAUDE.md and rules files.

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
- **Architecture** (when boundaries are verified; scope the rules to the affected paths). Rules come straight from the evidence: name the actual layers, packages, and allowed dependency directions.
  - WHEN adding an import or call across a module/package/layer boundary → DO check it against the verified dependency direction and use the target's public interface, not its internals → VERIFY with the enforcing tool's command where one exists; otherwise by inspecting the imports in the diff.
  - WHEN adding new code → DO place it where the closest existing equivalent lives, following the named exemplar (e.g. "new endpoint: like `<path>`") → VERIFY no new top-level module, layer, or pattern was introduced without an explicit reason in the report.
  - WHEN a change contradicts an ADR or architecture doc → DO stop and surface the conflict instead of silently deviating → VERIFY the report names the ADR.
  - WHEN changing a public interface (exported API, shared types, HTTP/RPC contract, event schema) → DO find all its consumers first and follow the repo's compatibility/versioning convention → VERIFY the consumers are updated or confirmed unaffected.
  - Existing violations (cycles, bypassed layers): don't replicate them in new code, and don't refactor them unasked; mention them in the report.
- **Performance** (budgets and benchmarks as strict rules when found; otherwise only verified hot paths and data-access layers):
  - WHEN changing code covered by a benchmark, load test, or budget → DO run the verified command before and after → VERIFY that the result stays within the budget, or report the regression with the numbers. Never claim "faster" or "no regression" without a measurement.
  - WHEN changing a loop or data access whose size grows with input, users, or rows on a hot path → DO state its complexity and the number of queries or I/O calls per item, and use the repo's existing batching, eager-loading, pagination, or caching pattern (name it) → VERIFY there's no N+1 access and no unbounded collection or result set in the diff.
  - WHEN adding or changing a cache → DO define its key, TTL, and invalidation on writes using the existing cache layer → VERIFY that a write path invalidates it.
  - Don't add speculative optimization (caches, async, concurrency) without a measured or documented need; optimize only where there is evidence.
- **Code quality** (`code-quality.md`, or CLAUDE.md if short). Only rules that change behavior, each grounded in the repo's actual config and exemplars:
  - Follow the named exemplars and the surrounding code: naming, structure, error handling, logging, and comment density. Don't import patterns from other ecosystems.
  - Error handling per the repo convention (name its error types and where errors are translated). Never swallow errors (empty `catch`/`except`, ignored return values). Error messages carry context without leaking secrets or PII.
  - WHEN adding a lint or type suppression (`noqa`, `eslint-disable`, `@ts-ignore`, `# type: ignore`, `#[allow]`, `any`) → DO fix the cause instead; if a suppression is truly required, make it narrow and add a reason comment → VERIFY the diff has no unexplained suppressions, and no weakened lint, type, or coverage config.
  - WHEN behavior changes → DO add or update a test that follows the repo's test conventions (name the fixtures/factories) and covers the failure or edge path, not just the happy path → VERIFY that the test fails without the change when practical, and is deterministic (no real time, randomness, network, or sleep, unless the repo's pattern controls them).
  - No dead code, commented-out code, debug prints, or orphan TODOs (a TODO needs context, or the repo's issue reference convention). Comments explain *why*, not *what*.
  - WHEN considering a new dependency → DO check whether the stdlib or an existing dependency covers it, and add it through the package manager so the lockfile updates → VERIFY the report justifies the new dependency.
  - Respect enforced limits (complexity, length, coverage thresholds) by restructuring, never by raising the limit.
- **Versions, real APIs, no deprecations** (always generated when versions are verified; name the actual versions and sources):
  - State the verified versions in CLAUDE.md: language/runtime minimum (and the CI matrix if any) plus the key framework/library versions from the lockfile. Example: "Python ≥3.10 (CI 3.10–3.12), FastAPI 0.115.x, SQLAlchemy 2.0.x, Pydantic v2".
  - WHEN writing code → DO use only syntax and stdlib features available in the **minimum** supported version → VERIFY against the version's docs or the configured target tool (ruff `target-version`, TS `target`, `rust-version`), not against whatever interpreter happens to be installed locally.
  - WHEN using any library/framework API, function, method, option, config key, or CLI flag → DO confirm it exists in the **locked** version: in the repo's existing usage, the installed package source (`site-packages`, `node_modules`, the module cache), or that version's official docs → VERIFY before using it. If it can't be confirmed, say so and don't guess. Never mix API generations (e.g. Pydantic v1 vs v2, SQLAlchemy 1.x vs 2.0 style, React class vs hooks) against what the repo uses.
  - WHEN choosing between APIs → DO use the non-deprecated one for that version, matching the repo's existing modern usage → VERIFY there are no new deprecation warnings in the test/build output and the configured deprecation lint passes.
  - Existing deprecated usage: don't spread it to new code, and migrate it only when it's inside the task's scope. Otherwise report it.
  - Don't upgrade the language version, dependencies, or the lockfile unasked. WHEN the task genuinely needs a newer version → DO stop and ask, naming the reason and the blast radius.
  - **Coding standard:** WHEN writing or changing code → DO follow the standard the repo adopts (name it and its config) and run the repo's formatter/linter on the changed files → VERIFY the check passes on them. Format only the code you touched, unless the repo formats whole files by convention (e.g. a pre-commit formatter); never reformat unrelated code.
  - **Conventions:** state each verified convention concretely (e.g. "files `kebab-case.ts`, components `PascalCase`, tests `test_<module>.py` next to `tests/<area>/`"). WHEN creating a file, identifier, or module → DO match these → VERIFY that the new names in the diff follow them. Commit/PR conventions only when enforced and Claude is asked to commit.
  - **No duplicate code:** WHEN about to write a helper, util, type, constant, validation, query, or UI component → DO first search the repo's shared locations (name them) and the nearby code for an existing equivalent, and reuse or extend it → VERIFY the diff doesn't re-implement something that already exists (run the duplication tool if configured).
    - WHEN the change itself would repeat the same logic → DO extract it once, following the repo's pattern for shared code.
    - Don't merge code that only looks similar but changes for different reasons, and don't build abstractions for a single use.
    - No duplicated magic values: use the existing constants/config modules.
    - Pre-existing duplication outside the task is reported, not refactored unasked.
- **Documentation and comments** (keep docs in sync with every change; name the repo's actual doc locations):
  - WHEN a change alters behavior users or developers rely on (public API, CLI flags, config/env vars, commands, setup or deploy steps, architecture or boundaries) → DO update the directly related docs in the same change: the README section, the `docs/` page, `.env.example`, and docstrings of changed public symbols → VERIFY that a grep of the docs for the old names, flags, env vars, or paths finds no stale references, and that the doc build or link check passes where one is configured.
  - WHEN changing code that has comments or docstrings → DO update or delete the ones that no longer match the code; comments explain *why* or non-obvious constraints, not what the code plainly does → VERIFY there are no stale or contradictory comments in the touched hunks.
  - Public symbols get docstrings only where the repo's convention or linter requires them, in the repo's format.
  - Changelog: WHEN the repo requires an entry per change (changesets, an `Unreleased` section, towncrier fragments) → DO add it in the repo's format → VERIFY it exists in the diff. Otherwise don't touch the changelog.
  - Generated docs (OpenAPI, API reference) are refreshed via their generator, never edited by hand.
  - Rewrite only the docs related to the change. Doc drift found elsewhere is reported, not fixed unasked.
  - **Keeping these rules current:** WHEN a change makes a fact in CLAUDE.md or `.claude/rules/` untrue (a command, path, convention, or boundary) → DO update that rule in the same change, or flag it in the report → VERIFY the rules don't reference removed paths or commands. After structural changes (new packages or modules, a tooling switch, a new CI pipeline), recommend rerunning `/init-rules`.
- **External calls:** where integrations exist: consider timeout, retry, and failure behavior using the patterns the repo already uses.
- **Orchestration (subagents / parallel agents):** generate only when the repo is large enough, or has independent verified modules, for delegation to matter; or when the project already defines agents/workflows in `.claude/agents/` or `.claude/workflows/`. A small single-module repo gets at most a line or two in CLAUDE.md, or nothing. Ground every rule in the repo's verified boundaries:
  - WHEN a task needs broad read-only exploration across many files → DO delegate it to a read-only subagent and keep only conclusions → VERIFY key claims by reading the cited files before acting on them. Do not delegate small or single-file tasks.
  - WHEN splitting work across parallel agents → DO partition by verified module or package boundaries (name them) so that no two agents edit the same file; a shared contract (schema, generated client, shared types, migrations) is changed by one agent first, and consumers follow → VERIFY the agents' file sets are disjoint before launching them.
  - WHEN delegating → DO give each agent a self-contained brief: goal, files in scope, files out of scope, constraints from this config, the verified validation commands, and the expected output → VERIFY the brief doesn't rely on conversation context the agent can't see.
  - WHEN an agent reports results → DO treat them as claims, not evidence: inspect its diff and re-run the relevant validation yourself → VERIFY before reporting; never relay "tests pass" or "fixed" from an agent without observed output.
  - WHEN parallel work is merged → DO run broader validation on the combined result and review the full final diff yourself; the orchestrator owns diagnosing integration failures and assigns the fix (coordinator model below) → VERIFY the combined result, not each part in isolation.
  - Prefer project-defined agents and workflows when they exist; reference them by their actual names. Keep the number of agents proportional to the task. Destructive or outward-facing actions (deploys, migrations, pushes) stay with the orchestrator and need the user's confirmation; never delegate them.
  - **Model and effort proportionality.** WHEN delegating → DO pick the cheapest model and lowest reasoning effort that can do the job: a smaller, faster model and low effort for search, lookups, summarizing, and mechanical edits; the largest model and high/xhigh/max effort only for genuinely hard reasoning (cross-module design, subtle security or concurrency analysis, a bug that cheaper attempts failed on) → VERIFY that any elevated choice has a stated reason in the brief. Escalate on evidence (insufficient output, a failed attempt), never preemptively. Only use mechanisms the installed version supports: the per-call `model` override on the Agent tool, and the `model` / `effort` frontmatter in `.claude/agents/*.md`. Prefer model aliases over pinned model IDs unless the repo pins them. WHEN an existing project agent definition sets the largest model or high+ effort for routine work → report it as a REWRITE recommendation (agent definitions are outside this skill's write scope).
  - **Coordinator model (main agent does not write code).** Whenever orchestration rules are generated, or the user requests it via the arguments, the rules make the main agent a coordinator:
    - **Allowed for the main agent:** reading and searching, running validation and other read-only commands, writing briefs, git branch/worktree/merge operations, and reporting. **Not allowed:** editing source, tests, config, or generated files itself, including "quick fixes" spotted during review and semantic conflict resolution. Every change, however small, is delegated (small ones to a cheap model per the proportionality rule).
    - WHEN delegating parallel or non-trivial work → DO give each agent an isolated workspace (agent `isolation: worktree`, or a branch per agent) → VERIFY each delivery arrives as its own diff or branch.
    - WHEN work is delivered → DO review it against the brief: scope (no changes outside its files), surgical-change and project rules, the agent's own validation output, and the verified validation commands re-run by the main agent. Then make exactly one decision:
      - **ACCEPT**: merge it.
      - **RETURN**: send it back to the same agent with concrete findings: `file:line`, the failing output, and the expected behavior.
      - **REASSIGN**: give it to another agent, or escalate model/effort after a failed attempt, or split the task.
      - **DISCARD**: drop the work (wrong approach, scope creep) and re-brief from scratch.
      - **ESCALATE**: ask the user when requirements are unclear or a decision is theirs.
    - → VERIFY the decision and its reason are recorded in the final report.
    - Bound the loop: after two RETURNs without progress, REASSIGN or ESCALATE instead of returning again.
    - WHEN merging → DO merge one delivery at a time, shared contracts before their consumers, and run focused validation after each merge. WHEN a merge conflicts or validation fails after a merge → DO return the conflict or failure to the owning agent on an updated base; never resolve it by editing code yourself → VERIFY the re-delivered work goes through review again.
    - **Supervising agent state.** WHEN agents are running → DO keep a status ledger for each one: task, agent name/ID, worktree/branch, model/effort, state (briefed → running → delivered → in review → returned / accepted → merged, or discarded / blocked / failed / stopped), and the last observed evidence (a notification, an output excerpt, or a diff) → VERIFY that every state change rests on observed evidence, never on an assumption.
      - Rely on the harness's completion notifications, not tight polling loops. When an agent goes quiet past a reasonable time, or is blocked or failed, inspect its actual output/transcript, then stop it and REASSIGN or ESCALATE.
      - Never report a pending agent's result or state as known. Say it is still running.
      - Under Herdr, read the state from `herdr agent list` / `herdr agent get`. `unknown` is not `done`.
      - Before finishing: no agent is still running unaccounted for, and every worktree/branch is either merged, discarded, or listed in the report → VERIFY the final report lists each agent's final state.
    - Final step: broader validation on the merged result, a full final diff review, and a report attributing each change and each validation result to the agent and the command that produced it.
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
- A new feature imports another module's internals or bypasses a layer. Do the architecture rules name the allowed direction and the public interface?
- New code invents its own structure instead of following the existing equivalent. Is there a named exemplar to follow?
- A change to benchmarked or budgeted code is reported as "faster" or "fine" without numbers. Do the rules require a before/after measurement?
- A new helper re-implements a util that already exists two directories away. Do the rules force a search of the named shared locations first?
- An env var or CLI flag is renamed, but the README, `.env.example`, and docstrings still show the old name. Do the doc rules force a grep for stale references?
- A function's behavior changes, and its docstring and comments still describe the old behavior. Would Claude update them?
- A refactor moves a module that CLAUDE.md references. Would the rules be updated, or flagged?
- A change ignores the file and identifier naming conventions, or reformats untouched code. Do the convention and standard rules prevent it?
- Code calls a library method or config option that doesn't exist in the locked version (a hallucinated or newer API). Do the rules force checking the installed source or that version's docs first?
- Code uses syntax newer than the minimum supported version (e.g. 3.12 syntax under `requires-python >=3.10`), or a deprecated API. Do the version rules catch it?
- A lint or type error is "fixed" with a suppression or a weakened config. Do the quality rules require fixing the cause?
- A change ships without a test of its failure path, or with a flaky time- or network-dependent test. Do the test rules catch it?
- Two parallel agents both need to edit a shared file or contract. Do the rules force a single owner and an ordering?
- A subagent reports "all tests pass". Would the orchestrator re-verify before reporting it?
- During review, the coordinator spots a one-line bug in a delivery. Do the rules make it RETURN the work instead of fixing it itself?
- An agent goes silent or gets stuck, and the coordinator finishes anyway. Do the rules force a ledger check and an explicit stop / reassign / report?
- A trivial lookup is delegated to the largest model at max effort. Do the rules steer it to a cheaper model and lower effort?

### Phase I: Validate the configuration

Using only capabilities that actually exist:
- Files exist at the discovered locations: `CLAUDE.md` (root or `.claude/`) and `.claude/rules/*.md`.
- Frontmatter parses as YAML and uses only `paths` (rules files). Check by inspection or a quick parse (e.g. `python3 -c 'import yaml,sys; ...'` if available; otherwise state that it was inspected manually).
- Every `paths` glob matches existing files (`git ls-files` / `find`).
- Every command in the config is backed by a script, target, or CI step you observed. List each with its evidence.
- Every referenced file or directory path exists, including the named exemplar files. Each exemplar actually represents the convention it's cited for.
- Every version stated in the rules matches its source (version file, manifest, lockfile, CI matrix), and the minimum version is derived correctly.
- Every architecture or performance tool cited (import-linter, dependency-cruiser, a benchmark/budget command, etc.) is configured in the repo, and its command is verified like any other command.
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
- architecture checks pass where tooling enforces boundaries (name the command)
- benchmarks or budgets are run when the change touches covered code (name the command), with the numbers reported
- the diff has no unexplained suppressions, dead code, debug output, or unjustified new dependencies
- the formatter/linter pass on the changed files (name the commands); new names follow the verified conventions; no logic duplicates existing code
- every API, option, and flag used exists in the locked versions; the code runs on the minimum supported version; no new deprecation warnings or deprecated APIs
- the related docs, comments, `.env.example`, and changelog (when required) are updated; the doc build or link check passes where configured; CLAUDE.md and the rules are still true after the change
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

**Repository findings**: only important VERIFIED findings on architecture, tooling, boundaries, validation, and operations. Include the architecture rules found (tool-enforced vs convention-only), the performance evidence found (budgets, benchmarks, SLOs, hot paths; executable vs documented only), the code-quality conventions and exemplars chosen, the adopted coding standard with its enforcing config, the shared-code locations used for reuse, the documentation locations, tooling, and drift found, and the verified language/runtime/framework versions (the minimum, its source, any conflicts) plus the deprecation tooling.

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
