# Development Guide — Independent Exploitation Harness

- Date: 2026-11-17
- Status: implementation guide
- Binding design: `docs/formal-design-agentic-orchestrator.md`
- Roadmap: `docs/roadmap.md`
- Reference snapshot: Yakit `master` @ `f813c64de`, `1.4.9-1113` — **architecture
  and tool-naming reference only**, not a runtime dependency.
- Scope: build our own **MCP server + TypeScript tools** for
  LLM-assisted vulnerability exploitation, with wrapping of licensed, authorized
  open-source tools.

> **Principle.** This harness is **independent**: it does not depend on the Yakit
> runtime or any other external product. Open-source tools may be wrapped, but
> the wrapping layer, registry, policy, and audit remain owned by this repo.
>
> **Ethical boundary.** All development and testing is only for **own assets /
> authorized environments**. This document contains architecture and contracts,
> not real exploit payloads.

---

## Table of Contents

1. [How to use this guide](#1-how-to-use-this-guide)
2. [Prerequisites](#2-prerequisites)
3. [Repository structure](#3-repository-structure)
4. [Configuration and secrecy](#4-configuration-and-secrecy)
5. [Per-phase implementation flow](#5-per-phase-implementation-flow)
6. [Code contract](#6-code-contract)
7. [MCP server](#7-mcp-server)
8. [Testing](#8-testing)
9. [Release gate and definition of done](#9-release-gate-and-definition-of-done)
10. [Troubleshooting](#10-troubleshooting)
11. [References](#11-references)

---

## 1. How to use this guide

Suggested reading order:

1. `docs/formal-design-agentic-orchestrator.md` — ADRs, architecture, permission
   matrix, and the **binding** threat model.
2. `docs/roadmap.md` — phases, deliverables, gates, and status tracking.
3. This document — stack, structure, code contracts, and implementation steps.
4. `docs/tool-authoring.md` — the `ToolSpec` contract and tool wrapping.
5. `docs/testing-and-verification.md` — test strategy and evidence rules.
6. `docs/refs/opencode-mcp-re-tool-catalog.md` — open-source tool/MCP catalog.

This repo currently contains **analysis documentation**; there is no `package.json`,
build, or CI yet. Phase 0 of the roadmap starts code scaffolding. This guide
serves as the reference once that phase begins.

---

## 2. Prerequisites

| Layer | Choice | Note |
| --- | --- | --- |
| Runtime | Node.js 22+ | requires `Promise.withResolvers`, `Symbol.asyncDispose` |
| Language | TypeScript strict | avoid stray `any` |
| Validation | Zod | all tool parameters are validated on the server |
| MCP | `@modelcontextprotocol/sdk` | stdio + Streamable HTTP |
| Agent SDK | `@opencode/sdk` | in-process host (reference version 2.3.0) |
| Testing | Vitest | unit, integration, negative, E2E |
| Policy | TypeScript code | not passive config |
| Sandbox | OS-level / container | non-root, egress deny |
| Package manager | pnpm / npm / yarn | lockfile is locked and committed |

### 2.1 Pinned versions

- `@opencode/sdk` — use a version proven compatible with Node 22+; do not float
  (`^` should be locked in the lockfile and reviewed on upgrades).
- `@modelcontextprotocol/sdk` — pin the version; use `protocol: 'auto'` until the
  pinned version is proven to support `2026-07-28`.
- Wrapped open-source tools — **always** pin version + hash; never use
  `latest` (see `docs/tool-authoring.md` §5).

### 2.2 Lab environment

- One or more **own servers** as test targets, with full permission.
- The lab network is isolated; egress to the internet/internal cloud is denied
  unless allowed per job.
- The build host runs Node 22+; do not embed the SDK in Electron 27 (Node 18).

---

## 3. Repository structure

Target structure (design §11), plus the directories needed for testing and
configuration:

```text
src/
  gateway/        http.ts, ws.ts, auth.ts
  sdk/            host.ts, plugins.ts, permissions.ts
  agent/          planner.ts, react-loop.ts, structured-output.ts, sub-agent.ts
  policy/         policy-engine.ts, review-policy.ts, risk-scorer.ts, approval-gateway.ts
  guard/          scope-guard.ts, sanitizer.ts, schema-pinning.ts, verifier.ts
  tools/          registry.ts, tool-spec.ts, mcp-adapter.ts, adapters/*, local-tools/*
  runtime/        execution-broker.ts, sandbox.ts, audit-log.ts
  memory/         session-memory.ts, knowledge-base.ts
  config/         capabilities.ts, engagement-scope.ts
tests/
  unit/           policy, scope, sanitizer, schema-pinning
  integration/    registry, MCP adapter, SDK plugin
  negative/       out-of-scope, post/shell, tool poisoning, egress
  e2e/            sesi OpenCode + MCP server
fixtures/
  scope.json, tool-catalog.json, mcp-stub/
config/
  default.json, lab.json
docs/
scripts/
  dev.mjs, build.mjs, start.mjs
```

### 3.1 Module boundary rules

- `gateway/*` and the renderer must not call tools directly; everything goes
  through `ExecutionBroker`.
- `tools/*` must not import policy; policy lives in `policy/*`.
- `agent/*` plans and selects actions, but does not execute.
- `runtime/*` is the only execution and audit layer.
- `guard/*` must not call tools; it is purely checks and sanitization.

### 3.2 Data directories

- Audit log: append-only, outside the build directory; path via configuration.
- Memory/knowledge base: isolated per engagement (`memory/session-memory.ts`).
- Evidence artifacts: hash + reference (`artifactRef`), not copies of secrets.

---

## 4. Configuration and secrecy

### 4.1 Environment variables (example)

| Variable | Default | Meaning |
| --- | --- | --- |
| `ELM_BIND` | `127.0.0.1` | bind for MCP/HTTP; never `0.0.0.0` without auth |
| `ELM_PORT` | `11432` | Streamable HTTP port; follows the Yakit reference — change if it conflicts with another service |
| `ELM_REVIEW_POLICY` | `manual` | `manual` \| `ai` \| `yolo` |
| `ELM_RISK_SCORE` | `high` | threshold for `allow` → `ask` escalation |
| `ELM_SCOPE_FILE` | `config/scope.json` | `EngagementScope` definition |
| `ELM_AUDIT_LOG` | `data/audit.log` | path for the append-only log |
| `ELM_MCP_URL` | `http://127.0.0.1:11432/mcp` | URL for `opencode mcp add` |
| `ELM_NETWORK_DEBUG` | `redacted` | never `full` in production |
| `ELM_DISABLE_TOOL_USE` | `false` | `true` = pure chatbot, all tools denied |

### 4.2 Secret references

- No secrets in the repo. Use `{env:NAMA}` or `{file:path}`.
- Model API keys, tokens, and lab credentials must not enter logs/audit except
  as redacted references.
- `NetworkDebug` defaults to **redacted**; exports must pass through the
  Sanitizer.

### 4.3 Suggested initial configuration

```jsonc
{
  "reviewPolicy": "manual",
  "aiReviewRiskControlScore": "high",
  "enablePlan": true,
  "enabledCapabilities": ["utility", "recon", "scan"],
  "disableToolUse": false,
  "networkDebug": "redacted",
  "mcp": {
    "lab": { "type": "remote", "url": "http://127.0.0.1:11432/mcp",
             "oauth": false, "protocol": "auto", "codemode": false }
  }
}
```

`enabledCapabilities` filters tool `capability` (equivalent to Yakit's
`AIEnabledCapability`); `post` and `shell` are never auto-allowed — `ask` under
`manual`, `deny` under `ai`/`yolo` (matrix §6.1).

---

## 5. Per-phase implementation flow

| Phase | Core modules | Brief gate | Status |
| --- | --- | --- | --- |
| 0 | scaffolding, `package.json`, `tsconfig`, Vitest, CI | typecheck + tests green | Not started |
| 1 | `sdk/host`, `tools/registry`, `agent/planner` | structured output valid; tools without `schemaHash` rejected | Not started |
| 2 | `guard/scope-guard`, `policy/*`, `runtime/audit-log` | negative tests deny; audit chain verified | Not started |
| 3 | `runtime/execution-broker`, `sandbox`, `sanitizer`, `verifier` | egress deny; findings without evidence rejected | Not started |
| 4 | `tools/mcp-adapter`, `schema-pinning`, catalog, open-source wrapping | `lab_<tool>` discovery; quarantine works | Not started |
| 5 | `memory/*`, `agent/sub-agent`, lab `yolo` mode | session isolation; yolo cannot enable post/shell | Not started |
| 6 | observability, packaging, hardening | E2E green; runbook exists | Not started |

Deliverable, task, and gate-evidence details are in `docs/roadmap.md`.

### 5.1 Task order that must not be reversed

1. `ToolSpec` + `schemaHash` before any tool is registered.
2. `ScopeGuard` before `PolicyEngine` is used.
3. `AuditLog` before `ExecutionBroker` executes.
4. `Sanitizer`/`Verifier` before observations enter the LLM context.
5. `ExecutionBroker` before the MCP adapter is connected to OpenCode.

### 5.2 Per-phase definition of done

`tsc` clean, Vitest green, related threat model IDs closed, audit chain
verified, tool discovery proven, and test findings carry `evidence`.

---

## 6. Code contract

### 6.1 `ToolSpec` and `defineTool`

See `docs/tool-authoring.md` §1 for the field table. Hard rules:

- Without `schemaHash` → registration denied.
- `description` is length-limited and sanitized; no hidden instructions.
- `params` are validated with Zod **on the server side**, not trusted from the
  model.

### 6.2 `ToolRegistry`

```ts
export class ToolRegistry {
  private tools = new Map<string, ToolSpec>()
  register(spec: ToolSpec): void {
    if (!spec.schemaHash) throw new Error('tool without schemaHash')
    if (this.tools.has(spec.name)) throw new Error('duplicate tool')
    this.tools.set(spec.name, spec)
  }
  get(name: string): ToolSpec | undefined { return this.tools.get(name) }
  list(capability?: Capability): ToolSpec[] { /* filter for enabledCapabilities */ }
}
```

- `ForceSync` compares `schemaHash`; a change → **quarantine** → `deny`
  until reviewed (design §5.4).
- `DisableToolUse = true` → the registry is effectively empty (all `deny`).

### 6.3 `ScopeGuard`

```ts
export class ScopeGuard {
  assertInScope(target: string, scope: EngagementScope): void {
    if (scope.expiresAt < new Date().toISOString()) deny('scope expired')
    if (!scope.authorizedTargets.includes(target)) deny('target out of scope')
  }
}
```

- Called **before** any action; before `PolicyEngine`.
- Wildcard/regex only with explicit approval.
- Empty / expired scope → all actions `deny`.

### 6.4 `PolicyEngine`

Applies matrix §6.1 and escalation §6.3:

```ts
export type Decision = 'allow' | 'ask' | 'deny'

policy.evaluate({
  actor, capability, risk, sideEffects,
  target, reviewPolicy, score, tool
})
```

- `post` / `shell` → `deny` under `ai`/`yolo`; `ask` under `manual`.
- `risk >= score` → `allow` escalates to `ask` (except under `yolo`).
- `sideEffects && reviewPolicy === 'manual'` → `ask`.
- `tool.alwaysAsk` → `ask` even at low risk.
- `tool.yoloAllowed === false && reviewPolicy === 'yolo'` → `deny`.
- First-time new target → `ask` + scope verification.

### 6.5 `ApprovalGateway`

- `ask` waits for human or AI-reviewer confirmation.
- For an in-process SDK host, the approval prompt may use
  `ctx.permission.assert` (OpenCode) or an operator bridge via `gateway/`.
- Records the `approver` in the audit entry.
- Rejection → `audit.rejected`; the loop stops/continues without execution.

### 6.6 `ExecutionBroker`

The single execution gate (ADR-006):

```ts
const result = await broker.execute(action)   // sandbox + timeout + audit + ToolResult
```

- Non-root sandbox, CPU/memory limits, **egress deny** + per-job allowlist.
- Timeout `action.tool.timeoutMs`; quota per operator.
- Without `ExecutionBroker` no tool runs — including direct MCP.

### 6.7 `Sanitizer` and `Verifier`

- `Sanitizer.sanitize(result.observation)`: the observation is **data**, not
  commands; drop/gray out instructions, redact secrets. The same applies to MCP
  server instructions and resources that enter the system prompt (T02/T10).
- `Verifier.isGoalReached(observation, plan, evidence)`: a finding is valid only
  when `ToolResult.evidence` exists (`hash`, `at`, `artifactRef`).

### 6.8 `AuditLog`

Append-only hash chain:

```ts
hash = sha256(seq | at | event | payload | prevHash)
```

- There is no edit/delete API.
- The chain is verified at startup and on export.
- Store `evidenceHash`, not the sensitive raw evidence.

### 6.9 SDK plugin and permissions

```ts
await ctx.tool.transform((editor) => {            // REQUIRED: all tools
  for (const tool of editor.list()) {
    editor.update(tool.id, (draft) => wrapTool(draft))
  }
})
await ctx.mcp.transform((editor) => {
  editor.set('lab', { type: 'remote', url: process.env.ELM_MCP_URL,
                      oauth: false, protocol: 'auto', codemode: false })
})
```

SDK permissions are a coarse deny-list (`resource: "*"`), so real authorization
stays in the TypeScript `PolicyEngine` (ADR-003). Built-in tools (`shell`,
`edit`, `write`) are wrapped, disabled via `permissions`/`policies`, or removed
via `editor.remove(...)`; `DisableToolUse` turns off all tools.

### 6.10 Agent loop

`planner.plan` → `selectAction` (Zod-validated) → `audit.planned` →
`scopeGuard.assertInScope` → `policy.evaluate` → `audit.allowed`/`audit.asked` →
`approval` when `ask` (`audit.approved`/`audit.rejected`) → `broker.execute` →
`sanitizer` → `memory` → `audit.executed` → `verifier.isGoalReached`. See
design §7.2.

---

## 7. MCP server

### 7.1 Transport

- **Streamable HTTP** for the OpenCode remote client (absolute URL).
- **stdio** for local/container tools.
- Default bind `127.0.0.1`; never `0.0.0.0` without authentication.

```bash
pnpm dev      # dev server (hot reload)
pnpm build
pnpm start
```

### 7.2 Discovery

Tool names are normalized `<server>_<tool>` (e.g. `lab_port_scan`). Verify the
tool list that appears in OpenCode before allowing offensive tools.

### 7.3 Connecting OpenCode

```bash
opencode mcp add lab --url http://127.0.0.1:11432/mcp
opencode mcp list
```

Two distinct paths: (1) the agent host is an in-process OpenCode SDK process that
registers the MCP server via `ctx.mcp.transform` (design §4.2); (2)
`opencode mcp add` is used by the OpenCode client (CLI/TUI) for manual discovery
verification. Both point to the same MCP server.

### 7.4 Staged verification

1. Test one read-only `utility`/`recon` tool.
2. Test a `scan` tool with approval.
3. Only then consider `fuzz`/`validate` in the lab.
4. `post`/`shell` are never auto-allowed.

---

## 8. Testing

Full strategy in `docs/testing-and-verification.md`. Summary:

```bash
pnpm typecheck
pnpm test            # unit + integration + negative
pnpm test:e2e        # requires MCP server + OpenCode
```

- Mandatory negative tests: out-of-scope → `deny`; expired scope → `deny`;
  `post`/`shell` under `ai`/`yolo` → `deny`; findings without `evidence` → `deny`.
- CI: `typecheck` → `test` → `test:e2e` → audit chain verification.
- Offensive tools are not run in CI jobs; use mocks/stubs.

---

## 9. Release gate and definition of done

Before a phase is declared done:

1. `tsc` clean and Vitest green in a clean repo.
2. The phase security gate passes and **evidence** is recorded (test output,
   artifact hash).
3. Related threat model IDs are closed or the residual is documented.
4. The permission matrix and design are updated if the contract changes.
5. Tool discovery is proven in OpenCode.
6. Test findings carry `ToolResult.evidence` and the audit chain is verified.

---

## 10. Troubleshooting

| Symptom | Likely cause | Action |
| --- | --- | --- |
| Tool does not appear in OpenCode | server not running / wrong URL / wrong bind | check `opencode mcp list`, server log, `ELM_BIND` |
| All actions `deny` | empty/expired scope | update `EngagementScope` + `approvedBy` |
| `post`/`shell` denied | intentionally hard-denied under `ai`/`yolo` | run `manual` + approval, or don't |
| Tool quarantined | `schemaHash` changed | review `description`/`params`, then reset the pin |
| Observation leaks instructions | Sanitizer not yet applied | install the Sanitizer before the LLM |
| Audit chain fails | file modified/incomplete | restore from backup, verify again |
| Egress leaking out | allowlist/egress deny not yet active | enable egress deny + per-job allowlist |

---

## 11. References

- `docs/formal-design-agentic-orchestrator.md` — binding design (ADRs, matrices, threat model).
- `docs/roadmap.md` — phase roadmap, deliverables, and gates.
- `docs/tool-authoring.md` — the `ToolSpec` contract and tool wrapping.
- `docs/testing-and-verification.md` — test strategy and evidence rules.
- `docs/refs/opencode-mcp-re-tool-catalog.md` — open-source tool/MCP references.
- `docs/refs/opencode-sdk-yakit-tool-bridge-analysis.md` — SDK/MCP integration patterns (reference).
