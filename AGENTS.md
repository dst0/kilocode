# AGENTS.md

Kilo CLI is an open source AI coding agent that generates code from natural language, automates tasks, and supports 500+ AI models.

- ALWAYS USE PARALLEL TOOLS WHEN APPLICABLE.
- The default branch in this repo is `main`.
- Prefer automation: execute requested actions without confirmation unless blocked by missing info or safety/irreversibility.
- You may be running in a git worktree. All changes must be made in your current working directory — never modify files in the main repo checkout.

## Build and Dev

- **Dev**: `bun run dev` (runs from root) or `bun run --cwd packages/opencode --conditions=browser src/index.ts`
- **Dev with params**: `bun dev -- help`
- **Extension**: `bun run extension` (build + launch VS Code with the extension in dev mode). Pass `--no-build` to skip the build.
- **Typecheck**: `bun turbo typecheck` (uses `tsgo`, not `tsc`). Includes the JetBrains plugin and requires Java 21; do not run `java -version` as a routine preflight. Only check Java when a Gradle/Java command fails with a Java-version or missing-Java error. If missing, install via SDKMAN: `sdk install java 21-tem && sdk use java 21-tem`. If SDKMAN is not installed, see https://sdkman.io/install.
- **Test**: `bun test` from `packages/opencode/` (NOT from root -- root blocks tests)
- **Single test**: `bun test ./test/tool/tool-define.test.ts` from `packages/opencode/`
- **CLI build artifact size check**: after `bun run script/build.ts --single --skip-install` in `packages/opencode/`, use `du -h dist/*/*/bin/kilo` (scoped package output lives under `dist/@kilocode/`)
- **SDK regen**: After changing server endpoints in `packages/opencode/src/server/`, run `./script/generate.ts` from root to regenerate `packages/sdk/js/`
- **Knip** (unused exports): `bun run knip` from `packages/kilo-vscode/`. CI runs this — all exported types/functions must be imported somewhere. Remove or unexport unused exports before pushing.
- **Source links**: After adding or changing URLs in `packages/kilo-vscode/`, `packages/kilo-vscode/webview-ui/`, or `packages/opencode/src/`, run `bun run script/extract-source-links.ts` from the repo root and commit the updated `packages/kilo-docs/source-links.md`. CI runs this check — the build fails if the file is stale.
- **kilocode_change check**: `bun run check-kilocode-change` from `packages/kilo-vscode/`. CI runs this — `kilocode_change` is a marker for upstream merge conflicts and must not appear in `packages/kilo-vscode/` or `packages/kilo-ui/` (these are entirely Kilo Code additions). Remove the markers before pushing.
- **opencode annotation check**: `bun run script/check-opencode-annotations.ts` from repo root. CI runs this on PRs touching `packages/opencode/` — every Kilo-specific change in shared opencode files must be annotated with `kilocode_change` markers. Exempt paths (no markers needed): `packages/opencode/src/kilocode/`, `packages/opencode/test/kilocode/`, and any path containing `kilocode` in the name.
- **Effect facade ratchet**: Do not add runtime-backed Promise facades to shared `packages/opencode/src` Effect services; use service dependencies, `AppRuntime`, or Kilo-owned boundaries. Run `bun run script/check-opencode-promise-facades.ts` when touching service adapters.
- **workflow allowlist**: `bun run script/check-workflows.ts` from repo root. CI runs this as part of the annotations workflow — any `.yml` / `.yaml` file added to or removed from `.github/workflows/` must be reflected in the hardcoded list in `script/check-workflows.ts`. Prevents upstream-merged workflows from silently starting to run in our CI.
- **Backend/SDK programmatic testing**: see [TESTING.md](./TESTING.md) for spawning the local main-branch backend (`bun dev serve`) and driving it via `curl` — use this instead of `kilo serve` (prod binary) when testing backend fixes.

## Quality Checks

Before saying an implementation is ready, run the smallest relevant checks that can catch lint, typecheck, and test failures for the touched package. Do not rely on manual extension launch to discover build problems. Fix failures you introduced before the final response, or state exactly which check is still failing or could not be run.

| Area | Checks |
|---|---|
| Root / cross-package | `bun run lint`, `bun run typecheck` |
| CLI | From `packages/opencode/`: `bun run typecheck`, `bun test` or targeted `bun test ./path/to/file.test.ts` |
| VS Code extension | From `packages/kilo-vscode/`: `bun run typecheck`, `bun run lint`, `bun run test:unit` or `bun run test` |
| Extension build/package | From `packages/kilo-vscode/`: `bun run compile` or `bun run package` when touching build, packaging, SDK, or webview integration paths |
| JetBrains plugin | From `packages/kilo-jetbrains/`: `./gradlew typecheck`, `./gradlew test`. Requires Java 21; do not run `java -version` as a routine preflight. Check Java only after a Java-version or missing-Java failure. |
| CI-only guards | Run affected guards documented above, such as `bun run knip`, `bun run check-kilocode-change`, `bun run script/check-opencode-annotations.ts`, or source link extraction |

Never run root `bun test`; the root script prints `do not run tests from root` and exits with code 1. Use package-level tests instead.

## Products

All products are clients of the **CLI** (`packages/opencode/`), which contains the AI agent runtime, HTTP server, and session management. Each client spawns or connects to a `kilo serve` process and communicates via HTTP + SSE using `@kilocode/sdk`.

| Product | Package | Description |
|---|---|---|
| Kilo CLI | `packages/opencode/` | Core engine. TUI, `kilo run`, `kilo serve`. Fork of upstream OpenCode. |
| Kilo VS Code Extension | `packages/kilo-vscode/` | VS Code extension. Bundles the CLI binary, spawns `kilo serve` as a child process. Includes the **Agent Manager** — a multi-session orchestration panel with git worktree isolation. |

**Agent Manager** refers to a feature inside `packages/kilo-vscode/` (extension code in `src/agent-manager/`, webview in `webview-ui/agent-manager/`). It is not a standalone product. See the extension's `AGENTS.md` for details.

In each VS Code extension host, one `KiloConnectionService` is created for the sidebar, every Kilo editor tab, and Agent Manager; it lazily starts and reuses one current `kilo serve` backend at a time. Agent Manager worktree sessions pass a directory context to this shared backend rather than starting one per worktree. State captured by the active service layer, such as Snapshot `trackState`, is shared across those requests; only directory-keyed `InstanceState` data is isolated.

Extension-specific settings should live in the Kilo extension settings, not default VS Code settings, unless they are intentionally VS Code-wide. Experimental flags should follow existing flag patterns, not VS Code settings; they usually belong in the Kilo Experimental settings section.

## Package Instructions

- When a task primarily touches `packages/kilo-jetbrains/`, read `packages/kilo-jetbrains/AGENTS.md` before planning or editing. It covers split-mode architecture, IntelliJ source lookup, threading fundamentals, UI guidelines, and session component architecture.

## Monorepo Structure

Turborepo + Bun workspaces. The packages you'll work with most:

| Package | Name | Purpose |
|---|---|---|
| `packages/opencode/` | `@kilocode/cli` | Core CLI -- agents, tools, sessions, server, TUI. This is where most work happens. |
| `packages/sdk/js/` | `@kilocode/sdk` | Auto-generated TypeScript SDK (client for the server API). Do not edit `src/gen/` by hand. |
| `packages/kilo-vscode/` | `kilo-code` | VS Code extension with sidebar chat + Agent Manager. See its own `AGENTS.md` for details. |
| `packages/kilo-gateway/` | `@kilocode/kilo-gateway` | Kilo auth, provider routing, API integration |
| `packages/kilo-telemetry/` | `@kilocode/kilo-telemetry` | PostHog analytics + OpenTelemetry |
| `packages/kilo-i18n/` | `@kilocode/kilo-i18n` | Internationalization / translations |
| `packages/kilo-ui/` | `@kilocode/kilo-ui` | SolidJS component library shared by the extension webview and docs screenshot stories |
| `packages/util/` | `@opencode-ai/util` | Shared utilities (error, path, retry, slug, etc.) |
| `packages/plugin/` | `@kilocode/plugin` | Plugin/tool interface definitions |

## Commits and PR Titles

Use conventional commit-style messages and PR titles: `type(scope): summary`.

Valid types are `feat`, `fix`, `docs`, `chore`, `refactor`, and `test`. Scopes are optional; use the affected package or area when helpful, e.g. `core`, `opencode`, `tui`, `app`, `desktop`, `sdk`, or `plugin`.

Examples: `fix(tui): simplify thinking toggle styling`, `docs: update contributing guide`, `chore(sdk): regenerate types`.

## Style Guide

- Keep things in one function unless composable or reusable
- Avoid unnecessary destructuring. Instead of `const { a, b } = obj`, use `obj.a` and `obj.b` to preserve context
- Avoid `try`/`catch` where possible
- Avoid using the `any` type
- Prefer single word variable names where possible
- Use Bun APIs when possible, like `Bun.file()`
- Rely on type inference when possible; avoid explicit type annotations or interfaces unless necessary for exports or clarity

### Avoid let statements

Prefer `const`. Replace `let` + if/else assignment with a ternary or an IIFE. Reassignment is the only legitimate reason to reach for `let`.

### Naming Enforcement (Read This)

THIS RULE IS MANDATORY FOR AGENT WRITTEN CODE.

- Use single word names by default for new locals, params, and helper functions.
- Multi-word names are allowed only when a single word would be unclear or ambiguous.
- Do not introduce new camelCase compounds when a short single-word alternative is clear.
- Before finishing edits, review touched lines and shorten newly introduced identifiers where possible.
- Good short names to prefer: `pid`, `cfg`, `err`, `opts`, `dir`, `root`, `child`, `state`, `timeout`.
- Examples to avoid unless truly required: `inputPID`, `existingClient`, `connectTimeout`, `workerPath`.

### Avoid else statements

Prefer early returns (or an IIFE) over `else`. After an `if` that returns/throws, the `else` is redundant.

### No empty catch blocks

Never leave a `catch` block empty. An empty `catch` silently swallows errors and hides bugs. If you're tempted to write one, ask yourself:

1. Is the `try`/`catch` even needed? (prefer removing it)
2. Should the error be handled explicitly? (recover, retry, rethrow)
3. At minimum, log it via `log.error("...", { err })` so failures are visible — never `catch {}` or `catch (e) {}` with no body.

### Prefer single word naming

Default to a single-word name for variables, parameters, and helper functions. Reach for a multi-word name only when a single word would be genuinely ambiguous in context — not just because the longer name "reads nicer". The rule is about meaning, not character count: don't introduce camelCase compounds like `inputPID`, `existingClient`, `connectTimeout`, or `workerPath` when `pid`, `client`, `timeout`, or `path` is already clear from the surrounding code. See the "Naming Enforcement" section above for the preferred vocabulary.

## Test Quality & Adversarial Review

- Tests must never be added solely as mechanical line-fillers to pass coverage gates. Tests must meaningfully verify domain logic, invariant preservation, realistic crash recovery, positive cases, negative cases, and edge cases.
- Bug fixes must start with a reproducible failing regression test before writing the fix.
- For non-trivial features, bug fixes, or test additions, automatically spawn an adversarial test-critic subagent to review the tests. The critic must evaluate whether the suite verifies real behavior vs artificial line coverage, identifies missing edge cases, and flags fragile/vacuous tests before work is completed.
- Never use coverage bypass comments (e.g. `/* v8 ignore */`, `#[cfg(not(coverage))]`, `# pragma: no cover`) to bypass coverage gates. All code in the repository must be reachable and exercised by tests; dead or unreachable code must be deleted rather than kept or suppressed (except rare compiler/type-exhaustiveness edge cases where a branch is syntactically required but provably unreachable at runtime).
- If you create or modify a test file, run it and iterate on test or implementation until it passes.

## Markdown Tables

Do not pad markdown table cells for column alignment. Use the compact form with single-space-padded content cells and a minimal separator row:

```
| Command | What it runs |
|---|---|
| `kilo serve` | The prod CLI on `$PATH`. |
```

Do **not** right-pad cells to line up columns:

```
| Command                       | What it runs             |
| ----------------------------- | ------------------------ |
| `kilo serve`                  | The prod CLI on `$PATH`. |
```

Padding makes every content change rewrite the entire table, which blows up diffs on untouched rows. Markdown files are excluded from prettier (see `.prettierignore`) so running the formatter won't re-pad them, and `script/check-md-table-padding.ts` enforces the rule in CI. Run `bun run script/check-md-table-padding.ts --fix` to auto-rewrite padded tables.

## Commit Conventions

[Conventional Commits](https://www.conventionalcommits.org/) with scopes matching packages: `vscode`, `cli`, `agent-manager`, `sdk`, `ui`, `i18n`, `kilo-docs`, `gateway`, `telemetry`, `desktop`. Omit scope when spanning multiple packages.

## Changesets

User-facing changes (features, fixes, breaking changes) require a changeset file for release notes. Prefer one concise changeset per PR, grouping related changes when possible. Run `bunx changeset add` or manually create `.changeset/<slug>.md`. Use `patch` for bug fixes, `minor` for new features, `major` for breaking changes. See `.changeset/README.md` for details.

Changeset descriptions appear directly in release notes and are read by end users. Keep them concise and feature-oriented — describe **what changed from the user's perspective**, not implementation details. Write in imperative mood (e.g. "Support exporting conversations as markdown" not "Add a new export handler that serializes session messages to .md files").

## Pull Requests

PR descriptions should explain **what** changed, **why** the change is needed, and the intent or constraints a reviewer cannot infer from the diff alone. Keep simple PRs brief, but give non-trivial changes enough context to stand on their own. Skip file-by-file inventories, test result summaries, and anything obvious from the code itself.

## GitHub Issues

When creating or managing GitHub issues for the VS Code extension or JetBrains plugin via `gh`, load `.kilo/skills/gh-issues/SKILL.md`. It covers templates, project boards (`VS Code Extension`, `Jetbrains Plugin`), title conventions, and the `gh auth refresh -s project` recovery path.

## Fork Merge Process

Kilo CLI is a fork of [opencode](https://github.com/anomalyco/opencode).

**Very important**: when planning or coding, update shared files with OpenCode as last resort! Everything is shared code from OpenCode, except folders that contain `kilo` in the name or have a parent directory that contains `kilo` in the name. Example of kilo specific folders: `packages/opencode/src/kilocode/` and `packages/kilo-docs/`. Always look for ways to implement your feature or fix in a way that minimizes changes to shared code.

### Minimizing Merge Conflicts

We regularly merge upstream changes from opencode. To minimize merge conflicts and keep the sync process smooth:

1. **Prefer `kilocode` directories** - Place Kilo-specific code in dedicated directories whenever possible:
   - `packages/opencode/src/kilocode/` - Kilo-specific source code
   - `packages/opencode/test/kilocode/` - Kilo-specific tests
   - `packages/kilo-gateway/` - The Kilo Gateway package

2. **Minimize changes to shared files** - When you must modify files that exist in upstream opencode, keep changes as small and isolated as possible.

3. **Use `kilocode_change` markers** - When modifying shared code, mark your changes with `kilocode_change` comments so they can be easily identified during merges.
   Do not use these markers in files within directories with kilo in the name

4. **Avoid restructuring upstream code** - Don't refactor or reorganize code that comes from opencode unless absolutely necessary.

5. **Mirror new config keys to the cloud schema** - When adding a `kilocode_change` key to `Config.Info` in `packages/opencode/src/config/config.ts`, also add the matching JSON Schema entry in `apps/web/src/app/config.json/extras.ts` in the [cloud repo](https://github.com/Kilo-Org/cloud). See [CLI Config Schema](packages/kilo-docs/pages/contributing/architecture/config-schema.md) for the step-by-step.

The goal is to keep our diff from upstream as small as possible, making regular merges straightforward and reducing the risk of conflicts.

### Git conflict style

`bun install` sets `merge.conflictStyle=zdiff3` repo-locally via `script/setup-git.ts` (wired into `postinstall`). Conflicts include the common ancestor between `|||||||` and `=======`, which is what `script/upstream/` and `mergiraf` rely on for structural resolution and what makes manual resolution on shared opencode files tractable. If you've overridden it in your user config, the repo-local setting takes precedence — don't override it back.

### Kilocode Change Markers

When editing shared upstream files, mark Kilo-specific lines with `kilocode_change` comments so future merges can find them. The basic forms are:

- Single line: `const value = 42 // kilocode_change`
- Multi-line block: wrap with `// kilocode_change start` / `// kilocode_change end`
- New file in a shared path: `// kilocode_change - new file` at the top
- JSX/TSX: use `{/* kilocode_change */}` (and `{/* kilocode_change start */}` / `end`)

Markers are NOT needed in paths that contain `kilocode` in the name (e.g. `packages/opencode/src/kilocode/`, `packages/opencode/test/kilocode/`) — these are entirely Kilo Code additions and won't conflict with upstream.

For decision rules on when to keep changes inline vs. extract Kilo logic, marker placement guidance, and verification commands, load `.kilo/skills/kilocode-merge-minimizer/SKILL.md`.

## Durable Learning Capture

- Treat every resolved bug, regression, setup trap, operator mistake, failed experiment, and unexpected behavior as a learning opportunity, not only as a code change.
- Before or while fixing an issue, preserve the observable symptom and decisive evidence. Once understood, record the root cause rather than only the final patch.
- Record enough detail to make the learning reusable:
  - what went wrong and why;
  - which approaches were tried, including what worked, what did not work, and why;
  - any unexpected constraints, side effects, or environmental differences;
  - the correct path and how it was verified;
  - the regression test, prevention rule, cleanup, or reset procedure that prevents recurrence.
- Put durable guidance in the appropriate canonical repository document in the same change: use `AGENTS.md` for agent behavior, `README.md` for user or setup paths, and the canonical architecture or product documentation for design and runtime contracts.
- When a cross-session memory tool such as `memory_save` is available, save resolved bugs, architectural decisions, durable facts, and learned patterns so future sessions can retrieve them.
- Do not leave important learnings only in chat, temporary notes, commit history, or a pull-request discussion.
- If an issue exposes repeated agent friction, add the shortest durable instruction here that would have prevented it.
- Keep learning records safe: never store credentials, tokens, private keys, customer data, or sensitive payloads; sanitize examples and evidence.

## Mandatory Learning Log

- Maintain the repository-wide append-only journal at `docs/learnings.md`.
- Add an entry in the same change whenever work reveals a resolved bug or regression, failed or misleading experiment, unexpected behavior, setup or environment trap, non-obvious constraint, important workaround, or rejected approach with reusable rationale.
- Routine successful work does not need an entry unless it produces a reusable insight.
- Use the exact entry structure documented in `docs/learnings.md`. Include the task/context, observation or failure, evidence, approaches tried and their outcomes, root cause, resolution, verification, prevention or follow-up, and the reusable learning.
- Mark uncertainty honestly. If root cause or resolution is incomplete, record the entry as `Partial` or `Open` and state what evidence is still missing.
- Keep the journal append-only by default: do not delete or rewrite older entries merely to make the history cleaner.
- Exception for confirmed falsehoods: when authoritative evidence proves that an entry itself was fabricated, hallucinated, or factually false, correct or remove the false content so future agents do not reuse it.
- A confirmed-falsehood correction must never be silent. Mark the entry `Corrected` and add a dated correction note stating what was wrong, the authoritative evidence used, and what was changed. Do not repeat removed sensitive content.
- If the evidence is incomplete or disputed, do not rewrite the original entry; add a dated `Partial` or `Open` follow-up instead.
- Link relevant issues, commits, logs, or regression tests when safe and useful.
- Never place credentials, tokens, private keys, customer data, sensitive payloads, or unsanitized production evidence in the journal.

<!-- destinationworks-universal-agent-baseline:v1 -->
## Universal Delivery Baseline (v1)

These rules are the portable minimum for Destination Works repositories. Repository-specific instructions may strengthen them or name concrete commands, but must not silently weaken them.

### Evidence, scope, and decisions

- Read the repository instructions and relevant canonical docs before changing files. Check available cross-session memory when prior decisions or recurring failures may affect the work.
- While actively working, reread a repository-root `user_updates.md` at least once per minute when it exists. Treat new entries as user instructions, handle them before continuing, remove only entries that were fully handled, and never delete the file itself.
- Establish the live baseline before diagnosing or claiming completion. Prefer direct evidence from current code, tests, CI, deployed artifacts, or authenticated system state over comments, stale reports, or agent summaries.
- Preserve unrelated and user-owned changes. Use an isolated branch/worktree for broad work, stage intentionally, and never reset, clean, delete, or rewrite unrelated state to simplify a task.
- For non-trivial changes, compare 2-3 viable approaches and record the decisive tradeoffs. Proof-test material assumptions with a focused reproduction or authoritative source before committing to the design.
- Test scripted replacements and bulk mechanical edits on a disposable copy of one representative file before applying them broadly; inspect the result for collateral changes.
- Keep implementation, user/setup documentation, architecture/runtime contracts, and operator guidance synchronized in the same change.
- Store closed, well-compressible logs and temporary evidence with Brotli quality 6 when practical. Never compress an actively appended log as one stream: rotate or close it into chunks first, then compress each completed chunk. Use a format better suited to append, random access, or unsupported tooling when required, and record the reason for that exception.

### Durable learning capture

- Maintain `docs/learnings.md` as the repository-wide learning journal. Add an entry in the same work that reveals a material resolved bug/regression, failed or misleading experiment, unexpected behavior, setup/environment trap, non-obvious constraint, important workaround, or rejected approach with reusable rationale; routine successful work needs no entry.
- Record the task/context, observable symptom, sanitized decisive evidence, approaches tried and why each worked or failed, root cause or honest uncertainty, resolution, verification, prevention/follow-up, reusable rule, and safe references. Use `Resolved`, `Partial`, or `Open` status truthfully.
- Keep entries append-only by default. Correct prior understanding with a new linked entry rather than rewriting history.
- Exception: when authoritative evidence proves an existing statement was fabricated, hallucinated, or factually false, correct or remove the false content so it cannot mislead future work. Mark the entry `Corrected` and add a dated note stating what was wrong, the authoritative evidence, and what changed; never use this exception for disputed interpretation, ordinary staleness, or changed external conditions.
- Promote the shortest prevention rule into the appropriate canonical instructions, setup guide, architecture contract, or operator runbook in the same change. Do not leave durable knowledge only in chat, commit history, a PR, or the journal.
- Never record secrets, credentials, private keys, customer data, sensitive payloads, device codes, or unsanitized production evidence.

### Validation and test quality

- Discover and use the repository's canonical commands; do not invent shared command names where the project does not define them.
- Use a validation ladder: fast targeted feedback while iterating, the repository pre-commit gate before commit, and the full pre-push/release-relevant gate before push. If a named gate does not exist, run the closest repository-native equivalent and document the exact evidence.
- A hook is developer feedback, not the authoritative merge gate. CI must rerun required checks from a clean checkout.
- Never weaken, skip, or replace a failing check merely to make it green. Read the failure, fix the cause, rerun the narrowest relevant test, then rerun the containing gate.
- Validate generated artifacts against their source and canonical generator. Do not hand-edit generated output or accept drift.
- Tests must cover meaningful behavior, negative/error paths, and important boundaries. Coverage is a regression signal, not a reason to add vacuous line-fillers or bypass comments.
- For non-trivial or high-risk changes, obtain an independent adversarial review of assumptions, tests, failure handling, and rollback before publication.
- Process-timeout tests must prove that descendants and inherited pipes are gone, not merely that the direct child received a signal. When an external Unix `kill` command receives a negative process-group operand, terminate option parsing with `--` and cover the Linux path.
- For user-visible UI changes, exercise the changed path in the real browser or installed application after automated tests pass; record the nearest honest evidence if UI automation is unavailable.

### Git, pull requests, and CI enforcement

- Start from current remote truth, keep commits scoped and reviewable, and verify the exact staged diff before committing. Do not mix unrelated work into one PR.
- A local pass, push, or successful agent report is not proof that remote CI passed. Confirm the remote PR head SHA and every required check on that exact revision.
- Self-merge only when branch/ruleset protection actually enforces the required checks and they all pass. If protection is unavailable, checks cannot start, or the head changed after validation, leave the PR open for owner approval.
- CI workflows must use least-privilege permissions, pinned third-party actions, explicit timeouts/concurrency, and repository-owned validation commands.
- Self-hosted workflows must target verified organization runner labels, check prerequisites early, and prefer runner-local/preinstalled toolchains and caches over dynamic marketplace installers or billing-dependent artifact/cache services.
- Prove self-hosted readiness as the runner service account with its real non-interactive `HOME`, `PATH`, permissions, working directory, and any runner-managed persisted environment snapshot; an administrator's shell or manually constructed environment is not equivalent to a real workflow job.
- Prerequisite probes must exercise the concrete subcommands and capabilities the job invokes, not infer support only from a parent runtime's major version.
- Runner services must restart after unexpected failure and terminate the complete job process group; for systemd, use `Restart=on-failure` and `KillMode=control-group`. Bound build/test parallelism to the shared host's measured memory budget and provision recovery swap without treating swap as permission for unbounded concurrency.
- Run unrelated repository or organization runner services under distinct Unix service accounts so user-scoped signals and cleanup cannot cross repository boundaries. After a runner migration, disable superseded services and watchdogs immediately; never leave a deleted registration in an automatic restart loop.
- Containers that bind-mount a reusable self-hosted worktree must write generated files as the runner UID/GID, or normalize ownership before exit even on failure. Prove a subsequent clean checkout can remove prior outputs.
- Scope runner prerequisites to the job's actual contract: native test jobs must not require release-only cross-platform emulation, while every published platform must fail closed unless its build and execution prerequisites are verified.
- PR descriptions must explain why the change was needed, what changed, approaches rejected, exact validation, bugs found/fixed with regression evidence, learning-log entries, risk, and rollback.

### Security and supply chain

- Never store or expose credentials, tokens, private keys, customer data, sensitive payloads, device codes, or unsanitized production evidence in source, logs, fixtures, PRs, or learning records.
- Treat dependency lifecycle scripts, lockfile changes, generated code, binary downloads, workflow actions, and base images as reviewed supply-chain inputs. Pin immutable versions/digests where supported and fail on unreviewed drift.
- When JavaScript is used, prefer `.js` filenames and migrate `.mjs` references unless the user explicitly requires another extension.
- Run repository-appropriate dependency, secret, and static security checks before publication. Waive only a specific reviewed false positive with narrow evidence; never use broad exclusions that hide future findings.
- Security-sensitive configuration and deployment paths must fail closed when required identity, authorization, signing, backup, or runtime prerequisites are missing.

### Release and deployment integrity

- When the repository publishes a deployable artifact, build it once, identify it by immutable digest, and test the exact bytes that will be promoted on every published platform.
- Generate provenance/SBOM, scan, sign, and verify the same immutable artifact before promotion. Promote by digest without rebuilding.
- Separate immutable provenance tags from mutable environment pointers. Publish and verify evidence first, move the smallest mutable production pointer last, verify the live promoted state, and define an exact rollback to the previously recorded digest.
- Do not describe registry publication as runtime deployment. If no external runtime target and verification contract are configured, state that boundary and fail closed rather than claiming production delivery.
- Rehearse backup/restore and rollback through safe isolated commands that produce inspectable evidence; documentation-string checks alone are not operational proof.

<!-- /destinationworks-universal-agent-baseline:v1 -->
