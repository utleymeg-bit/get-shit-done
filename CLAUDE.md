# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

GSD ("Get Shit Done") is a meta-prompting / context-engineering / spec-driven development system distributed as the npm package `get-shit-done-cc`. It installs as commands, agents, hooks, skills, and CLI tools into many AI coding runtimes (Claude Code, OpenCode, Gemini CLI, Kilo, Codex, Copilot, Cursor, Windsurf, Antigravity, Augment, Trae, Qwen Code, CodeBuddy, Cline). Most "code" in this repo is prompt content (`.md` files) that ships verbatim to those runtimes; the JS/TS code is the installer, runtime hooks, the CLI helper that workflows call, and the SDK that drives autonomous runs.

## Common Commands

Node 22+ required. Node 24 is the primary CI target.

```bash
# Install repo dev deps (runs no postinstall hooks for end users)
npm install

# Run the full test suite — wraps `node --test tests/*.test.cjs` with parallel workers.
# `pretest` builds the SDK first; CI runs the same flow on Ubuntu+macOS / Node 22+24.
npm test

# Run a single test file directly (skip the SDK build if it's already built)
node --test tests/<name>.test.cjs

# Limit test concurrency (default 4)
TEST_CONCURRENCY=1 npm test

# Coverage gate — fails if lib/*.cjs line coverage drops below 70%
npm run test:coverage

# Lint tests for the source-grep anti-pattern (CI gate `lint-tests`)
npm run lint:tests

# Build the SDK (TypeScript → sdk/dist) — required before running SDK or full test suite
npm run build:sdk

# Build/copy hooks into hooks/dist for npm packaging (runs on prepublishOnly)
npm run build:hooks

# Run the SDK's own vitest suites directly
cd sdk && npm test                    # all (unit + integration)
cd sdk && npm run test:unit           # unit only
cd sdk && npm run test:integration    # integration (120s timeout)
cd sdk && npx vitest run path/to/file.test.ts   # single SDK test

# Smoke-run the installer locally
node bin/install.js --claude --local
```

There is no project-wide linter or formatter beyond `lint:tests`. Conventional commits (`feat:`/`fix:`/`docs:`/`refactor:`/`test:`/`ci:`) are expected.

## Architecture (the big picture)

GSD is a layered prompt-orchestration system. A user invokes a slash command, which loads a workflow, which spawns specialized agents into fresh context windows, which call CLI helpers to read/write file-based state under `.planning/`. Read `docs/ARCHITECTURE.md` for the full diagram — the layers below are what you must keep straight when editing.

```
commands/gsd/*.md   →   get-shit-done/workflows/*.md   →   agents/gsd-*.md
   (entry point)         (thin orchestrator,                (specialized agent
                          loads context, spawns agents)      with fresh 200K context)
                                    │
                                    ▼
                         gsd-sdk query <handler>     +     get-shit-done/bin/gsd-tools.cjs
                         (sdk/src — TS, primary)           (legacy CLI, still used)
                                    │
                                    ▼
                              .planning/  (state, plans, phases — file-based, no DB)
```

Key invariants when working in any layer:

- **Fresh context per agent.** Workflows are *thin* — they load context via `gsd-sdk query init.<workflow>` and spawn agents; they do not do heavy work themselves. Don't push logic from `bin/lib/` up into workflow `.md` files.
- **Workflow size budgets** (`tests/workflow-size-budget.test.cjs`): XL ≤ 1700 lines, LARGE ≤ 1500, DEFAULT ≤ 1000, `discuss-phase.md` < 500. When a workflow grows past its tier, decompose into `workflows/<name>/modes/*.md` + `workflows/<name>/templates/*` and turn the parent into a dispatcher (`workflows/discuss-phase/` is the canonical example).
- **Agent size budgets** also exist (`tests/agent-size-budget.test.cjs`): XL 1600 / Large 1000 / Default 500 lines. Extract shared prose into `get-shit-done/references/*.md` and `@-reference` it from agents/workflows.
- **`agents/` is canonical, period.** Only `agents/` at repo root is tracked by git. `.claude/agents/`, `.cursor/agents/`, and `.github/agents/gsd-*` are install-sync outputs and are gitignored — never edit them. If they look stale, re-run `node bin/install.js`.
- **Two parallel CLI surfaces.** `gsd-sdk query` (TS, in `sdk/src/`) is the supported programmatic surface. `get-shit-done/bin/gsd-tools.cjs` is the legacy compatibility CLI still wired into many workflows. New handlers go in the SDK; keep `gsd-tools.cjs` in lockstep when behavior must be identical.
- **`.planning/` is the entire database.** All state is human-readable Markdown + JSON written under `.planning/`. No server, no DB. `STATE.md`/`config.json`/`phases/`/`research/` survive `/clear` and are inspectable by humans and agents.
- **Absent = enabled.** Workflow feature flags in `config.json` default to `true` when missing. Don't add code that requires a key to be explicitly set to enable a default.

### What lives where

- `bin/install.js` — multi-runtime installer (Claude Code, Codex, Copilot, Cursor, etc.). Knows the on-disk layout for every runtime, including Codex sandbox modes per agent and Copilot tool-name remapping.
- `bin/gsd-sdk.js` — thin shim that execs the built SDK CLI (`sdk/dist/cli.js`).
- `get-shit-done/bin/gsd-tools.cjs` + `get-shit-done/bin/lib/*.cjs` — the legacy CLI workflows shell out to. CommonJS, no external deps, one module per concern (`state`, `phase`, `roadmap`, `config`, `verify`, `template`, `frontmatter`, `init`, `milestone`, `security`, `uat`, `docs`, `workstream`, `schema-detect`, `profile-pipeline`, `profile-output`, …). Top-level `init.cjs` is the compound context-loader called by every workflow's bootstrap step.
- `sdk/src/` — TypeScript SDK. `cli.ts` is the entry, `query.ts`/`gsd-tools.ts` host registered query handlers, `cli-transport.ts`/`ws-transport.ts` are the two transports for autonomous runs, `init-runner.ts`/`phase-runner.ts`/`milestone-runner.ts` orchestrate `auto`/`init`/phase/milestone modes, `context-engine.ts` + `context-truncation.ts` manage the SDK's own context budgeting. Tests are colocated; `*.integration.test.ts` files run in the `integration` vitest project with a 120s timeout.
- `commands/gsd/*.md` — slash command entry points (frontmatter + body). One file per `/gsd-*` command.
- `get-shit-done/workflows/` — orchestrator prompts the commands resolve to.
- `get-shit-done/references/` — shared prose `@-referenced` from workflows/agents (gates, checkpoints, model profiles, anti-patterns, thinking-model patterns, TDD, etc.). When agents/workflows would duplicate >a few paragraphs, extract here.
- `get-shit-done/templates/` — Markdown templates used by `template fill` / `phase scaffold` for new planning artifacts.
- `hooks/` — runtime hooks. JS hooks (`gsd-statusline`, `gsd-context-monitor`, `gsd-check-update[-worker]`, `gsd-prompt-guard`, `gsd-read-guard`, `gsd-read-injection-scanner`, `gsd-workflow-guard`) are pure Node, copied to `hooks/dist/` by `scripts/build-hooks.js` for npm. Bash hooks (`gsd-session-state.sh`, `gsd-validate-commit.sh`, `gsd-phase-boundary.sh`) are community/opt-in via `hooks.community` in `.planning/config.json`.
- `tests/` — `node:test` regression suites. `helpers.cjs` is the shared utility module (use it; don't inline temp dirs). The `bug-NNNN-*.test.cjs` naming pattern marks regression tests for specific issues — add one when fixing a bug.
- `scripts/` — build + CI helpers (`run-tests.cjs`, `build-hooks.js`, `lint-no-source-grep.cjs`, `verify-tarball-sdk-dist.sh`, `secret-scan.sh`, `prompt-injection-scan.sh`, `gen-inventory-manifest.cjs`).
- `docs/` — contributor docs. `INVENTORY.md` + `INVENTORY-MANIFEST.json` are the authoritative roster of commands/workflows/agents/hooks/CLI modules; `ARCHITECTURE.md` is the deep dive.

## Conventions That Will Trip You Up

- **CommonJS only in `get-shit-done/bin/lib/`, hooks, and `tests/`** — `require()`, `module.exports`, `.cjs` extensions. Never `import`. **No external deps in core** — only Node built-ins. The SDK (`sdk/src/`) is ESM TypeScript; that's the *only* place `import` belongs.
- **Tests use `node:test` + `node:assert/strict` only.** No Jest, Mocha, Chai, vitest in the root suite. (vitest is for the SDK suite under `sdk/`.) Use `helpers.cjs` (`createTempProject`, `createTempGitProject`, `createTempDir`, `cleanup`, `runGsdTools`) — don't inline `mkdtempSync`. Cleanup pattern is `beforeEach`/`afterEach` or `t.after()`; **never `try/finally` inside a test body** (it masks failures and is rejected in review).
- **No source-grep tests.** A test that calls `readFileSync` on a `.cjs` source path to assert a string is present is rejected and CI-blocked by `npm run lint:tests`. Write a behavioral test through `runGsdTools()` instead. The narrow exceptions (`source-text-is-the-product`, `architectural-invariant`, `structural-regression-guard`, `docs-parity`, `integration-test-input`, `structural-implementation-guard`) require a standalone `// allow-test-rule: <reason>` comment immediately above the file's opening block — see `CONTRIBUTING.md` for the full policy and examples.
- **For multi-line fixture strings, `array.join('\n')`** — template literals inside test bodies inherit surrounding indentation and break regex anchors.
- **Security primitives, always.** Use `execFileSync` (array args), never `execSync` (string interpolation). Validate any user-provided path with `validatePath()` from `get-shit-done/bin/lib/security.cjs`. In GitHub Actions `run:` blocks, never inline `${{ … }}` — bind to an `env:` mapping first.
- **Issue-first contribution rule.** Every PR must link an issue with the right label (`confirmed-bug` / `approved-enhancement` / `approved-feature`) and use the matching template under `.github/PULL_REQUEST_TEMPLATE/`. Draft PRs are auto-closed. Bug fixes require a regression test that fails without the fix.
- **Don't edit outside a GSD workflow when one applies.** Per `.clinerules`, real changes should go through `/gsd:plan-phase` → `/gsd:execute-phase` → `/gsd:verify-work`. Quick fixes: `/gsd-quick`. Investigation: `/gsd-debug`. Direct repo edits are reserved for cases where the user explicitly bypasses the workflow.

## When You Modify Specific Things

- **Adding a CLI handler:** add to `sdk/src/` (preferred) and register in the query map; if a workflow needs it via the legacy CLI, also wire it into `get-shit-done/bin/gsd-tools.cjs` and the matching module under `bin/lib/`. Behavioral test goes in `tests/` calling `runGsdTools()`.
- **Adding/changing an agent:** edit only `agents/<name>.md`. Re-run `bin/install.js` locally if you need to test the deployed copy. Mind the size budget (`tests/agent-size-budget.test.cjs`); push shared prose into `get-shit-done/references/`.
- **Adding/changing a workflow:** edit `get-shit-done/workflows/<name>.md` (or its `modes/`/`templates/` subtree). Mind the size budget; decompose into `<name>/modes/*.md` if you cross it. Workflows must stay thin orchestrators.
- **Adding a slash command:** create `commands/gsd/<name>.md` with proper frontmatter (name, description, allowed-tools) and a body that delegates to a workflow.
- **Adding a hook:** put it in `hooks/`, register it in `scripts/build-hooks.js`'s `HOOKS_TO_COPY` (JS hooks are validated for syntax before copying), and document the event/purpose in `docs/ARCHITECTURE.md`.
- **Touching `.github/workflows/*.yml`:** never inline `${{ … }}` inside `run:` — bind via `env:`.

## Reference

- `CONTRIBUTING.md` — issue-first process, full testing standards, source-grep policy, exemption categories.
- `docs/ARCHITECTURE.md` — full layered diagram, agent contracts, hook responsibilities, runtime abstraction.
- `docs/INVENTORY.md` (+ `INVENTORY-MANIFEST.json`) — authoritative roster of commands, workflows, agents, hooks, CLI modules.
- `.clinerules` — short-form core rule list (kept consistent with this file).
