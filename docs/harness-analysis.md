# In-depth Analysis of the Yakit Project Harness

- Date: 2026-11-17
- Status: architecture analysis, no product code changes
- Scope: the harness layers (tooling, agent, verification, release, and runtime) surrounding the Yakit product code
- Method: static analysis — direct reading of configuration, scripts, skills, workflows, and documentation in a local checkout
- Code snapshot: branch `master`, commit `f813c64de`, package version `1.4.9-1113`

> This document complements two earlier analysis artifacts:
> `docs/refs/opencode-sdk-yakit-tool-bridge-analysis.md` (OpenCode SDK integration), and
> `docs/refs/opencode-mcp-re-tool-catalog.md` (reverse engineering MCP catalog).

> **Scope note (eLMave).** This analysis is about the Yakit product harness. The recommendation
> to "connect OpenCode as an external MCP client" in §6 applies to the Yakit context;
> the eLMave architecture follows ADR-001 (in-process SDK + own MCP server).

---

## Table of Contents

1. [Working Definition](#1-working-definition)
2. [Harness Layer Map](#2-harness-layer-map)
3. [Per-Layer Analysis](#3-per-layer-analysis)
4. [Cross-Layer Design Patterns](#4-cross-layer-design-patterns)
5. [Gaps and Points of Concern](#5-gaps-and-points-of-concern)
6. [Conclusion](#6-conclusion)
7. [Internal References](#7-internal-references)

---

## 1. Working Definition

In this document, *harness* refers to the entire **scaffolding** around the Yakit product code — configuration, CLI, scripts, agents, workflows, and gates — that governs how that code is installed, run, verified, secured, and released. This scope covers two categories that need to be distinguished conceptually:

- **Development harness**: the structure governing how developers and AI agents work on the code (OpenCode, skills, CLI, CI/CD, git hooks).
- **Runtime/agentic harness**: the structure that is part of the product itself and governs how AI executes security tools (Electron main, preload bridge, gRPC, MCP server, ReviewPolicy).

The two are connected through the MCP bridge documented in `docs/refs/opencode-sdk-yakit-tool-bridge-analysis.md`.

---

## 2. Harness Layer Map

| # | Layer | Primary location | Core function |
| --- | --- | --- | --- |
| 1 | Agent / AI | `AGENTS.md`, `.agents/skills/`, `.github/agents/`, `.github/copilot-instructions.md`, OpenCode documentation | Governs agent behavior, the commit → PR workflow, SDK/MCP integration |
| 2 | Build & development | `cli/`, `scripts/check-deps.js`, `scripts/build-electron.mjs`, `scripts/start-electron.mjs` | Single install/start/build/pack entry point, environment injection, readiness checks |
| 3 | Verification | `vitest.config.ts`, `scripts/ci-select-vitest-tests.js`, `e2e/`, 19 CI workflows | Test selection, real-engine E2E, quality gates |
| 4 | Git & commit | `.husky/pre-commit`, `.lintstagedrc.json`, `.prettierrc.js`, `.editorconfig`, `commit-msg` / `create-pr` skills | Formatting, commit messages, mandatory review, PR creation |
| 5 | Release | `docs/release-pipeline.md`, `scripts/release/`, `packageScript/electron-builder.config.js` | Two-phase prepare/finalize, signing, OSS upload |
| 6 | Runtime (in product) | `app/main`, preload, `app/protos/grpc.proto` (670 RPCs), MCP server, ReviewPolicy | Yaklang engine, agentic tool bridge |

---

## 3. Per-Layer Analysis

### 3.1 Agent / AI Layer

`AGENTS.md` serves as more than a startup guide. This document contains explicit coding behavior rules — think before coding, prioritize simplicity, precise changes, and target-based execution — plus a hard rule that every decision requiring user choice must be delivered through an interactive mechanism (`AskUserQuestion`), not a list of options in plain text.

The three core skills in `.agents/skills/` form a connected workflow chain:

| Skill | Responsibility | Distinguishing characteristic |
| --- | --- | --- |
| `code-review` | Six-dimension review with three-block output | tsc/vitest **must be run**; PR mode uses two OID worktrees (base/head) for baseline attribution; report writing requires permission |
| `commit-msg` | Range determination → per-type summary → confirmation → commit | `type: subject`, one line per type; `git commit -F` (explicitly forbids `git commit -m` because of shell interpolation risk) |
| `create-pr` | Commit → review → push → create/update PR | `gh` is required with a manual fallback; explicit push `HEAD:refs/heads/$branch`; review conclusions injected into the PR body; automatic-mode merge rules |

In addition, `.github/agents/yakit-frontend-maintainer.agent.md` establishes a frontend maintainer agent with a *smallest change set* constraint and `disable-model-invocation: true`, while `.github/copilot-instructions.md` aligns Copilot's code style rules with `.prettierrc.js`.

The OpenCode integration documentation asserts the architectural position:

- `@opencode/sdk` is the **new agent host**, not a provider/model replacement nor the Yakit AI engine.
- The Yakit AI engine lives in the external (Go) Yaklang process accessed via gRPC; the TypeScript repo only holds IPC, types, and catalogs.
- The most realistic integration path is OpenCode as an **MCP client** of Yakit's `StreamableHttpUrl` endpoint.
- Embedding the SDK in Electron main is risky because Electron 27 uses Node 18.17.1, while the modern SDK is ESM/Effect-based with `Promise.withResolvers` and `Symbol.asyncDispose`.

> The defining character of this layer: a **human checkpoint** on hard-to-reverse actions, and **anti-handwaving** — agents are forbidden from marking ✅ without real execution evidence.

### 3.2 Build & Development Layer

`cli/cli.mjs` is the single entry point (`yarn cli <command>`). Its flow: parse arguments → inject `YAKIT_*` variables → spawn local binaries in `node_modules/.bin`, with no yarn/npm/pnpm intermediary for execution.

The standout two-phase design:

1. The `install`, `add`, `remove` commands **are intercepted before loading commander**, using only Node built-in modules. As a result, a fresh clone can run `yarn cli install` directly.
2. The `start`, `build`, `pack`, `electron`, `dev` commands load `commander`, `inquirer`, `execa`, and `concurrently`.

The CLI behavior is **TTY-aware**: on a TTY, missing arguments trigger interactive prompts; in non-TTY environments (CI), missing arguments immediately produce `exit 1`. `cli/env.mjs` makes `YAKIT_EDITION` the single source of truth for the six editions, with legacy name translation only via `toEngineHandshakeName()`.

`scripts/check-deps.js` checks three conditions: `node_modules` is missing, declarative dependencies are missing or fixed versions do not match, and `yarn.lock` changed relative to `git HEAD`. The check is fast and requires no network, but it has a documented blind spot: changes to a lockfile that is already committed (for example after `git pull`) are not detected by the `git diff HEAD` comparison.

Render readiness is treated strictly: **a LISTEN status on a port is not the same as ready**. The client must receive HTTP 200 with valid HTML; `scripts/start-electron.mjs` waits at most 120 seconds before starting Electron. The main process is built with esbuild in CJS format (target `node16.17` / `chrome108`), type checking is done separately (`typecheck:electron`), IPC modules are registered via lazy-load through `app/main/ipcModules.js`, and runtime resources are located through `runtimePaths.js`.

### 3.3 Verification Layer

Vitest 4 is configured in `vitest.config.ts` to cover three subprojects with the `jsdom` environment. The configuration loads custom stub plugins for stylesheets, i18n, `electronBridge`, Monaco, and React 19 shims (`react-resize-detector`, `xterm-for-react`), plus the `@renderer`, `@engine`, `@app` aliases. Test coverage is limited to the `__test__/` directory, with an explicit exclusion for icon migration tests.

`scripts/ci-select-vitest-tests.js` selects tests deterministically based on three rules:

1. A changed file is `*.test.*` / `*.spec.*` → included.
2. A business file has a parallel test `__test__/<stem>.(test|spec).*` → included.
3. A test under `__test__/` imports the changed module (via alias or relative path) → included.

Selection results are written to `GITHUB_OUTPUT`; there are `WORKFLOW_INPUT_TESTS` and `CI_VITEST_MANUAL_TESTS` overrides, plus special handling that adds `electron-build.test.mjs` and `cli-build.test.mjs` when build/CLI scripts change.

The E2E subsystem is the most complex part. The runner (`scripts/run-electron-e2e.mjs`, WDIO 9 + electron-service) **owns all processes and temporary files**; developers and CI are forbidden from starting Electron or Yak earlier. Four suites are available: smoke, real-engine, web-fuzzer MCP, and MITM performance. Its distinguishing characteristics:

- Build cache is **content-addressed** keyed on Git HEAD, the full diff, Go version, GOOS/GOARCH, and the fixture contract version.
- Engine readiness must be proven by a **real Echo RPC**; log lines or an open port are not considered ready.
- **No fixed sleep** as the determinant of business readiness; the PR suite does not retry; a cleanup failure changes the run status to failed.
- The MITM matrix uses harness versions 5/9 with `smoke` / `standard` / `stress` profiles, a `diagnosticOnly` mode rejected by the comparator, and a 15% regression threshold plus a unit-specific noise floor.

19 GitHub workflows cover per-edition builds, multi-platform matrices, the release pipeline, and PR gates (`ci:eslint`, `ci:tsc`, `ci:renderer-boundary`, `ci:media-size`, `i18n:check`, `ci:electron`).

### 3.4 Git & Commit Layer

Husky `pre-commit` runs `lint-staged`, which applies `prettier --write` to JS/TS/JSON/CSS/SCSS/MD files. The conventions set by `.prettierrc.js` and `.editorconfig` are consistent: 120-column width, 2-space indentation, no semicolons, single quotes, trailing commas, and LF.

The `commit-msg` skill produces a `<type>: <subject>` message in Chinese with one line per change type, without `(#PR号)`, and saves the message to a temporary file before `git commit -F`. The `create-pr` skill forbids the `master` branch and forks, requires `code-review` before push, and uses `--body-file` for dynamic content so that shell interpolation cannot occur.

### 3.5 Release Layer

The two-phase release pipeline (`docs/release-pipeline.md`) separates **prepare** (frontend build, per-platform pre-assembly, draft candidate, no engine download) from **finalize** (engine download, SHA-256 verification, injection, signing/notarization, installer creation, OSS/GitHub upload, upgrade pointer update). Protections include non-overwritable candidates, tags that must match the edition, a lock during publish, and ETag/Content-MD5 verification.

`electron-builder` runs `beforePack` to rebuild production artifacts. The installer does not include the main process source, tests, or source maps; source maps and product hashes are archived separately in `reports/electron-build` as diagnostic artifacts.

### 3.6 Runtime Layer (in Product)

At this layer the harness becomes part of the product. Electron main together with the `window.yakitBridge` preload bridge serves as the gateway to the gRPC client and the external Yaklang engine through 670 RPC methods. Yakit also provides its own MCP server (`StartMcpServer` → `StreamableHttpUrl`), a tool catalog (`GetMCPToolList`), and **ReviewPolicy** with three modes: `manual`, `ai`, and `yolo`.

This is both the connection point with OpenCode and a risk point: calling a tool directly through MCP **bypasses Yakit's ReviewPolicy**, so permissions must be explicitly mapped to OpenCode's `permissions` mechanism. The document `docs/refs/opencode-mcp-re-tool-catalog.md` provides a curation of external MCP servers (IDA, Ghidra, Binary Ninja, Frida, HexStrike, and others) that can complement Yakit tools, with the note that most are stdio-based and copyleft-licensed.

---

## 4. Cross-Layer Design Patterns

| Pattern | Manifestation | Value delivered |
| --- | --- | --- |
| Single entry point, single source of truth | `yarn cli` + `YAKIT_EDITION` | Eliminates behavioral differences between editions and package managers |
| Two-phase execution | Zero-dependency intercept → full runtime | A fresh clone can install without a bootstrap; CI only needs a dependency subset |
| Fail explicitly in CI | Non-TTY → `exit 1`, no prompt | No silent interaction hangs the pipeline |
| Human checkpoint | Before commit, push, PR, and report writing | Hard-to-reverse actions always get confirmation |
| Evidence before claims | real tsc/vitest, real Echo, comparator rejects `diagnosticOnly` | Prevents false test results and misleading comparisons |
| Everything bounded | Timeout, memory limit, loopback-only, content-hash, allowlist | Failures are local and diagnosable |

---

## 5. Gaps and Points of Concern

| # | Finding | Impact | Suggested mitigation |
| --- | --- | --- | --- |
| 1 | `check-deps` does not detect an already-committed lockfile | Startup with stale `node_modules` passes the check | Ask for an explicit reinstall after `git pull` |
| 2 | Electron 27 / Node 18.17.1 versus `@opencode/sdk` ESM/Effect | Embedding the SDK is risky at runtime | Run OpenCode as an external MCP client |
| 3 | Direct MCP calls bypass Yakit's `ReviewPolicy` | Offensive tools can run without human-in-the-loop | Map `ReviewPolicy` to OpenCode `permissions` |
| 4 | MCP server defaults to `127.0.0.1:11432`; remote mode `0.0.0.0` | Exposure to the network without authentication is dangerous | Do not expose without protection |
| 5 | Main window `nodeIntegration: true` / `contextIsolation: false` | XSS can potentially escalate to Node RCE | Migrate gradually to the Link window pattern |
| 6 | AI engine and tools live in the external yaklang repo | The TypeScript repo only holds the contract, not the implementation | Treat it as a stable gRPC contract |

Note: finding number 5 and several other offensive surfaces are *by design* for a security testing tool. The problem is not the capability itself, but providing a GUI that displays untrusted content (traffic, AI, Markdown) to a window with full Node power.

---

## 6. Conclusion

The Yakit harness is a mature layered system with a consistent philosophy: **a single entry point, a single source of truth, evidence-based verification, and human confirmation at dangerous points**. Its main strengths lie in three things:

1. A zero-dependency CLI that is TTY-aware and can run on a fresh clone.
2. An E2E runner with a content-addressed cache and readiness checks without fixed sleeps.
3. Agent skills that are anti-handwaving and checkpoint-based.

Two harnesses must be distinguished when making architectural decisions: the **development** harness around the repo, which governs how code is worked on, and the **agentic harness inside the product**, which governs how AI executes security tools. The existing OpenCode integration documentation bridges the two with the right recommendation: do not replace the Yakit AI engine, do not embed the SDK in Electron, but connect OpenCode as an external MCP client and then map ReviewPolicy to permission *(Yakit context; eLMave chooses an in-process SDK + own MCP server).*

The most fragile point is not the offensive features — which are intentional — but the **trust boundary**: a missed lockfile, a Main window with full Node power, and MCP tools that can be called without policy. All three are relatively localized to fix and provide the largest security impact.

---

## 7. Internal References

| File | Relevance |
| --- | --- |
| `AGENTS.md` | Startup guide, agent behavior rules, interactive decision mechanism |
| `cli/README.md`, `cli/cli.mjs`, `cli/env.mjs` | CLI contract and `YAKIT_*` variables |
| `scripts/check-deps.js` | Dependency check logic and its blind spot |
| `docs/electron-main-engineering.md` | Main process engineering, preload, and resources |
| `e2e/README.md` | Electron E2E runbook, cache, performance profiles, failure policy |
| `scripts/ci-select-vitest-tests.js`, `vitest.config.ts` | Test selection and configuration |
| `.agents/skills/*/SKILL.md` | review → commit → PR workflow chain |
| `.github/agents/yakit-frontend-maintainer.agent.md` | Frontend maintainer agent constraints |
| `.husky/pre-commit`, `.lintstagedrc.json`, `.prettierrc.js` | Formatting and commit harness |
| `docs/release-pipeline.md` | Two-phase release pipeline |
| `docs/refs/opencode-sdk-yakit-tool-bridge-analysis.md` | OpenCode SDK bridge to Yakit tools/MCP |
| `docs/refs/opencode-mcp-re-tool-catalog.md` | Reverse-engineering MCP server catalog |
