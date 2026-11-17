# Formal Design — LLM-Assisted Exploitation Orchestrator

- Date: 2026-11-17
- Status: architecture design (no product implementation yet)
- Role: System Architect + Cybersecurity Analyst
- Derived from: `docs/harness-analysis.md`,
  `docs/refs/opencode-sdk-yakit-tool-bridge-analysis.md`, `docs/refs/opencode-mcp-re-tool-catalog.md`,
  and advanced `docs/analysis` (chat)
- Reference snapshot: Yakit `master` @ `f813c64de`, `1.4.9-1113`

---

> **Ethical note (mandatory).** This system is a security testing tool for **own assets /
> authorized environments**. The development and testing target is a **server owned by us** with
> full permission. This design deliberately places `ScopeGuard`, `PolicyEngine`, and `ApprovalGateway`
> as core components so that offensive capability cannot be directed at third-party systems.
> This document contains architecture, policy, and contracts — **not real exploitation payloads**.

---

## Table of Contents

1. [Goals and Non-Goals](#1-goals-and-non-goals)
2. [Architecture Decisions (ADR)](#2-architecture-decisions-adr)
3. [System Architecture](#3-system-architecture)
4. [In-Process OpenCode SDK Integration](#4-in-process-opencode-sdk-integration)
5. [Tool Contract](#5-tool-contract)
6. [Permission Matrix](#6-permission-matrix)
7. [ReviewPolicy and Agent Loop](#7-reviewpolicy-and-agent-loop)
8. [Final Threat Model](#8-final-threat-model)
9. [ScopeGuard and Target Authorization](#9-scopeguard-and-target-authorization)
10. [Audit and Verification](#10-audit-and-verification)
11. [TypeScript Module Structure](#11-typescript-module-structure)
12. [Roadmap and Gates](#12-roadmap-and-gates)
13. [Appendix](#13-appendix)

---

## 1. Goals and Non-Goals

### Goals

1. Build an LLM-based security agent that performs **plan & reasoning** to call
   existing tools and MCPs.
2. Special function: **vulnerability validation/exploitation** on our own server, with real
   execution evidence before a finding is declared valid.
3. Inherit the Yakit harness philosophy: *one entry point, one source of truth, evidence before
   claims, human checkpoint, everything constrained*.
4. Make policy and security **code**, not passive documentation.

### Non-Goals

- Not a replacement for the Yaklang engine / Yakit AI machine.
- Not an "LLM that writes exploits from scratch" (the LLM is the orchestrator).
- Not a tool for attacking systems without permission.
- Not an Electron application (Web/CLI-first).
- Not a complete replacement for OpenCode `permissions` — that is only a coarse deny-list.

---

## 2. Architecture Decisions (ADR)

| ADR | Decision | Rationale | Trade-off |
| --- | --- | --- | --- |
| ADR-001 | OpenCode SDK **in-process** on Node 22+ | Node 22 provides `Promise.withResolvers` & `Symbol.asyncDispose`, removing the SDK blocker on Electron 27 (Node 18) | Requires our own HTTP/WS gateway; SDK is only an in-memory router |
| ADR-002 | **MCP-first** + local TS tools | Fast access to the tool ecosystem; full control in the control plane | Most MCPs are stdio → requires containerization & limits |
| ADR-003 | **TypeScript PolicyEngine** as the primary authorization | SDK `permissions` uses `resource: "*"`, too coarse for targets/tenants | Requires our own implementation & testing |
| ADR-004 | Web/CLI-first, target = **our own server** | Compliant with permission; removes dual-renderer & Electron hardening complexity | Server RCE risk rises → requires sandbox & egress deny |
| ADR-005 | `yolo` **only** in an isolated lab | The proto document states yolo is "only for controlled environments" | Default mode remains `manual` |
| ADR-006 | All tools go through `ExecutionBroker` | Patches the document finding: direct MCP calls bypass ReviewPolicy | A single choke point → must be reliable & audited |

---

## 3. System Architecture

```text
   Web UI / CLI
        │  HTTP/WS (auth)
        ▼
   ┌──────────────────────────────────────────────────────────┐
   │  Gateway (own)                                            │
   │  - operator authn/authz                                  │
   │  - forwards prompts to the OpenCode session               │
   └───────────────────────┬──────────────────────────────────┘
                           ▼
   ┌──────────────────────────────────────────────────────────┐
   │  OpenCode SDK (in-process host)                           │
   │  OpenCode.create() → sessions.create() → prompt()         │
   │  plugin: ctx.tool.transform + ctx.mcp.transform           │
   └───────────────────────┬──────────────────────────────────┘
                           │ Planner/Reasoner (structured output)
                           ▼
   ┌──────────────────────────────────────────────────────────┐
   │  Agent Core                                               │
   │  Planner · ReAct loop · Sub-agent · Verifier              │
   └───────────────────────┬──────────────────────────────────┘
                           ▼
   ┌──────────────────────────────────────────────────────────┐
   │  ScopeGuard  →  PolicyEngine  →  ApprovalGateway          │
   │  (hard deny)   (allow/ask/deny) (manual/ai)               │
   └───────────────────────┬──────────────────────────────────┘
                           ▼
   ┌──────────────────────────────────────────────────────────┐
   │  ExecutionBroker  (THE SINGLE execution gate)             │
   │  sandbox · timeout · egress-deny · audit · result         │
   └──────────┬───────────────────────────┬───────────────────┘
              ▼                           ▼
   ┌────────────────────┐      ┌────────────────────────────┐
   │ MCP Adapters       │      │ Local TS Tools             │
   │ <server>_<tool>    │      │ recon/scan/fuzz/validate   │
   │ stdio→container    │      │ utility/filesystem/shell   │
   └────────────────────┘      └────────────────────────────┘
              │                           │
              └──────────────┬────────────┘
                             ▼
   ┌──────────────────────────────────────────────────────────┐
   │  Observation Sanitizer → Memory/RAG → LLM                │
   │  AuditLog (append-only, hash-chain)                      │
   └──────────────────────────────────────────────────────────┘
```

### Core components

| Component | Responsibility | Why mandatory |
| --- | --- | --- |
| `Gateway` | Operator auth, translates web/CLI requests → SDK session | SDK is not a network listener |
| `AgentCore` | Plan, reason, select action, reflect, sub-agent | Matches `EnablePlan`, `AIExecutionStrategy` |
| `ScopeGuard` | Ensures the target is on the official list | Hard deny before anything |
| `PolicyEngine` | `allow / ask / deny` based on actor, risk, sideEffects | `ReviewPolicy` + `AIReviewRiskControlScore` |
| `ApprovalGateway` | Human/AI review for dangerous actions | Human checkpoint |
| `ExecutionBroker` | The single tool execution; sandbox + audit | Patches the ReviewPolicy bypass |
| `Sanitizer` | Cleans tool results & MCP descriptions | Anti prompt-injection & tool poisoning |
| `Verifier` | A finding is valid only if there is a `tool_call_result` | Evidence before claims |
| `AuditLog` | Append-only + hash chain | Accountability & anti-tamper |

---

## 4. In-Process OpenCode SDK Integration

### 4.1 Host

```ts
import { OpenCode } from '@opencode/sdk'   // reference version: 2.3.0

export async function createHost() {
  const opencode = await OpenCode.create()
  const session = await opencode.sessions.create({
    // system prompt: "authorized security tester", tools restricted by the registry
  })
  return { opencode, session }
}
```

Note: the SDK is an **in-process host with an in-memory HTTP router**, not a listener. Our own
Gateway is still required to accept Web/CLI requests.

### 4.2 Plugin — main choke point

```ts
await ctx.tool.transform((editor) => {            // MANDATORY: all tools
  for (const tool of editor.list()) {
    editor.update(tool.id, (draft) => wrapTool(draft))
  }
})
await ctx.mcp.transform((editor) => {
  editor.set('lab', { type: 'remote', url: 'http://127.0.0.1:11432/mcp',
                      oauth: false, protocol: 'auto', codemode: false })
})
```

- `ctx.tool.transform` is the point where **every tool** (SDK built-ins *and* MCP) is wrapped in
  `ScopeGuard → PolicyEngine → ExecutionBroker`. The callback receives a `ToolEditor`; wrapping a
  tool means `editor.list()` then `editor.update(id, …)`. The transform is replayed after the
  MCP catalog is refreshed.
- `ctx.mcp.transform` registers the MCP server. Use `protocol: 'auto'` (probes
  `2026-07-28`, falls back to `legacy`) until a pinned `@modelcontextprotocol/sdk` is proven to
  support `2026-07-28`; do not force `2026-07-28` before verification.
- MCP server instructions and resources also enter the OpenCode system prompt; both must be
  filtered/sanitized (`Sanitizer`, T02/T10), not just tool results.
- `codemode: false` for small & curated catalogs; `true` for large catalogs.

### 4.3 Permission SDK — deny by default

```jsonc
{
  "permissions": [
    { "action": "*", "resource": "*", "effect": "deny" },
    { "action": "lab_recon_*", "resource": "*", "effect": "allow" },
    { "action": "lab_scan_*",  "resource": "*", "effect": "allow" },
    { "action": "lab_fuzz_*",  "resource": "*", "effect": "allow" },
    { "action": "lab_validate_*", "resource": "*", "effect": "allow" }
  ]
}
```

⚠️ `resource: "*"` means the SDK **cannot** distinguish targets/tenants. This is only a coarse
deny-list. Real authorization remains in the TypeScript `PolicyEngine`.

### 4.4 Recorded SDK limitations

- Built-in tools (`shell`, `edit`, `write`) can enlarge the attack surface → they must be wrapped,
  disabled via `permissions`/`policies`, or removed via `editor.remove(...)`.
  `DisableToolUse`/`AIEnabledCapability` are Yakit terms; in OpenCode the equivalents are
  `permissions`/`policies`/`ctx.tool.transform`.
- Tool names are normalized to `<server>_<tool>` (e.g. `lab_port_scan`).
- SDK behavior against custom transports must be verified against the actual package before
  choosing full in-process (see ADR-001).

---

## 5. Tool Contract

### 5.1 `ToolSpec` schema

```ts
import { z } from 'zod'

export type Capability =
  | 'utility' | 'recon' | 'scan' | 'fuzz' | 'validate'
  | 'post' | 'filesystem' | 'shell'

export type Risk = 'low' | 'medium' | 'high' | 'critical'

export interface ToolSpec {
  /** SDK normalization name: <server>_<tool>, e.g. lab_http_fuzz */
  name: string
  /** Description already sanitized, length-limited, schema-hash pinned */
  description: string
  /** Strict parameter validation on the server side */
  params: z.ZodTypeAny
  /** Capability category (used by the permission matrix) */
  capability: Capability
  /** Risk level -> determines whether approval is needed */
  risk: Risk
  /** Whether the tool changes target/local state */
  sideEffects: boolean
  /** Whether the tool may run in yolo mode */
  yoloAllowed: boolean
  /** Execution time limit (ms) */
  timeoutMs: number
  /** Requires explicit approval even for low risk? */
  alwaysAsk?: boolean
  /** Schema hash for tool-poisoning detection */
  schemaHash: string
  /** Source: 'mcp' | 'local' */
  source: 'mcp' | 'local'
}
```

Note: `ToolSpec` is metadata; `defineTool` adds `execute` (and `ctx`) outside the metadata
contract. `schemaHash = sha256(canonical JSON { description, params, capability, risk })`;
`params` is encoded as the schema shape (Zod/JSON Schema), not runtime values.

### 5.2 Agent action contract

```ts
export interface AgentAction {
  thought: string
  tool: string                 // must be registered in ToolRegistry
  args: Record<string, unknown> // validated against ToolSpec.params
  target: string               // MUST be in ScopeGuard
  rationale: string
}

export interface ToolResult {
  tool: string
  ok: boolean
  /** Sanitized summary for LLM consumption */
  observation: string
  /** Raw evidence (hash, timestamp) for the verifier & audit */
  evidence: { hash: string; at: string; artifactRef?: string }
  error?: string
}
```

### 5.3 Capability catalog (example, per the reference documents)

| Capability | Example tool (abstract) | risk | sideEffects | yoloAllowed |
| --- | --- | --- | --- | --- |
| `utility` | codec, hash, notes | low | no | yes |
| `recon` | DNS/whois/header lookup | low | no | yes |
| `scan` | port scan, fingerprint, vuln scan | medium | yes | yes |
| `fuzz` | web fuzzer, parameter fuzz | high | yes | yes* |
| `validate` | PoC execution, finding validation | critical | yes | lab only |
| `post` | webshell, reverse shell | critical | yes | **no** |
| `filesystem` | read/write lab files | low–medium | partly | scoped |
| `shell` | arbitrary commands | critical | yes | **no** |

\* `yolo` is only in an isolated lab (ADR-005); on our own server it remains `ask`.

### 5.4 Contract rules

1. A tool without a `schemaHash` must not be registered.
2. A change to `description`/`params` during catalog refresh (`ForceSync` = forced
   MCP catalog synchronization) → the tool is *quarantined*.
3. `post` and `shell` tools are **never** allowed automatically.
4. Every execution produces `ToolResult.evidence`; without it a finding is invalid. The MCP
   adapter must wrap external results into `evidence` (response hash + `at` +
   `artifactRef`); raw stdout/stderr is not returned to the model.

---

## 6. Permission Matrix

### 6.1 Matrix A — Capability × ReviewPolicy (default effect)

| Capability | risk | `manual` | `ai` | `yolo` (lab only) |
| --- | --- | --- | --- | --- |
| `utility` | low | allow | allow | allow |
| `recon` | low | allow | allow | allow |
| `scan` | medium | **ask** | allow | allow |
| `fuzz` | high | **ask** | **ask** | allow |
| `validate` | critical | **ask** | **ask** | allow (lab) |
| `post` | critical | **ask** | **deny** | **deny** |
| `filesystem` (read) | low | allow (scoped) | allow (scoped) | allow (scoped) |
| `filesystem` (write) | medium | **ask** | **ask** | allow (scoped) |
| `shell` | critical | **ask** | **deny** | **deny** |

Notes:
- `ask` = ApprovalGateway requests confirmation (human or AI reviewer).
- Default production mode on our own server: **`manual`**.
- `post`/`shell` are never auto-allowed: `ask` under `manual`, `deny` under `ai`/`yolo`.

### 6.2 Matrix B — Actor × Resource

| Actor | In-scope target | Lab files | Secrets | AuditLog | MCP configuration |
| --- | --- | --- | --- | --- | --- |
| Agent (LLM) | via PolicyEngine | via PolicyEngine | **never** | append-only | **never** (pin) |
| Operator (human) | yes | yes | own only | read | yes |
| Admin | yes | yes | yes | read + verify | yes |

### 6.3 Matrix C — Risk escalation rules

| Condition | Effect |
| --- | --- |
| `risk >= AIReviewRiskControlScore` (except `yolo`) | escalates from `allow` → `ask` |
| `sideEffects = true` and `reviewPolicy = manual` | `ask` |
| `tool.alwaysAsk = true` | `ask` (even for low risk) |
| `tool.yoloAllowed = false` and `reviewPolicy = yolo` | `deny` |
| First time for a new target | `ask` + scope verification |
| Tool schema changed | quarantine → `deny` until reviewed |
| `DisableToolUse = true` | all tools → `deny` (pure chatbot) |

Under `yolo`, the §6.1 column decides; risk escalation does not override `allow`.

---

## 7. ReviewPolicy and Agent Loop

### 7.1 Mapping `AIStartParams`

| Proto field | Implementation |
| --- | --- |
| `ReviewPolicy` | `PolicyEngine` mode: manual / ai / yolo |
| `AIReviewRiskControlScore` | escalation threshold to `ask` (Matrix C) |
| `EnablePlan` / `EnableDetachedPlan` | `planner.plan()` before an action |
| `AIExecutionStrategy` | sub-agent `dispatch_sub_re_agents` |
| `EnableSystemFileSystemOperator` | filesystem capability flag |
| `DisableToolUse` | ToolRegistry emptied |
| `AIEnabledCapability` | filter of active tool categories |
| `NetworkDebug` | default **redacted**, never exported |

Note: the left column is the Yakit proto field (`AIStartParams`) mapped to our mechanisms;
`DisableToolUse`/`AIEnabledCapability` are not OpenCode fields.

### 7.2 Loop

```ts
for (let i = 0; i < MAX_ITERATIONS; i++) {
  const observation = lastResult?.observation ?? task       // not trusted
  const plan   = await planner.plan(context, observation)
  const action = await planner.selectAction(plan)           // Zod-validated
  audit.planned(action)

  scopeGuard.assertInScope(action.target)                   // hard deny

  const decision = policy.evaluate({
    actor, capability: action.tool.capability,
    risk: action.tool.risk, sideEffects: action.tool.sideEffects,
    target: action.target, reviewPolicy, score: action.tool.risk
  })
  if (decision === 'allow') audit.allowed(action)
  if (decision === 'ask') audit.asked(action)

  if (decision === 'deny') { audit.denied(action); break }
  if (decision === 'ask') {
    const review = await approval.request(action, plan)
    if (!review.approved) { audit.rejected(action); continue }
    audit.approved(action, review.approver)
  }

  const result  = await broker.execute(action)              // the only execution
  const clean   = sanitizer.sanitize(result.observation)
  memory.record({ action, decision, clean })
  audit.executed(action, result)

  if (verifier.isGoalReached(clean, plan, result.evidence)) break
}
```

---

## 8. Final Threat Model

Scale: **L** (low), **M** (medium), **H** (high). Residual = risk after controls.

| ID | Threat (STRIDE) | L | D | Control | Residual |
| --- | --- | --- | --- | --- | --- |
| T01 | Target outside permission (Spoofing/Elevation) | M | H | ScopeGuard allowlist, default deny | L |
| T02 | Prompt injection from tool results/KB/MCP server instructions (Tampering) | H | H | Sanitizer; observations as data; separate system prompt; MCP instructions/resources filtered | M |
| T03 | Tool poisoning via `Description`/`Params` (Tampering) | M | H | Schema pin, sanitization, quarantine | L |
| T04 | Destructive action without approval (Elevation) | M | H | PolicyEngine + ApprovalGateway; post/shell never auto-allowed | L |
| T05 | Credential leakage via logs/export (Info disclosure) | M | H | Default redaction; `NetworkDebug` redacted | L |
| T06 | Audit log modified (Repudiation) | L | H | Append-only + hash chain + verification | L |
| T07 | RCE / sandbox escape from a tool (Elevation) | M | H | Non-root sandbox, resource limits, egress deny | M |
| T08 | Egress to cloud internal/metadata (Info disclosure) | M | H | Default-deny egress + per-job allowlist | L |
| T09 | False finding without evidence (Tampering) | H | M | Verifier requires `ToolResult.evidence` | L |
| T10 | Malicious MCP/skill supply chain (Tampering) | M | H | Scan + hash pin + license check + approval; MCP instructions/resources filtered | M |
| T11 | State leakage across sessions (Info disclosure) | L | M | Session & memory isolation per engagement | L |
| T12 | Compromise of the in-process SDK host (Elevation) | L | H | Node 22+, dependency pins, minimal permissions | M |
| T13 | DoS via long-running tool jobs (Denial of service) | M | M | Timeout, quota, rate limit per operator | L |

### Threat model notes

- **T02/T03/T10** are *gaps* not discussed in the original document; they are added here.
- **T07** is *by design* for offensive tools; what is controlled is the **impact**, not the
  capability.
- **T12** is specific to the in-process SDK choice (ADR-001); its risk is more manageable than
  embedding the SDK in Electron 27.

---

## 9. ScopeGuard and Target Authorization

```ts
export interface EngagementScope {
  id: string
  authorizedTargets: string[]   // e.g. ['server-internal.owned.example']
  allowedPorts?: number[]
  expiresAt: string
  approvedBy: string
}

export class ScopeGuard {
  assertInScope(target: string, scope: EngagementScope, port?: number): void {
    if (scope.expiresAt < new Date().toISOString()) deny('scope expired')
    if (!scope.authorizedTargets.includes(target)) deny('target out of scope')
    if (port !== undefined && scope.allowedPorts && !scope.allowedPorts.includes(port))
      deny('port out of scope')
  }
}
```

Rules:
1. Empty / expired scope → **all actions denied**.
2. Wildcard/regex targets are not allowed without explicit approval.
3. `allowedPorts` is optional; if present, the `scan` tool must enforce ports against this list.
4. Every scope change is recorded in the AuditLog with `approvedBy`.

---

## 10. Audit and Verification

### 10.1 Audit entry

```ts
export interface AuditEntry {
  seq: number
  at: string
  actor: 'agent' | 'operator' | 'admin'
  event: 'planned' | 'allowed' | 'asked' | 'approved' | 'rejected' | 'denied' | 'executed'
  action?: AgentAction
  decision?: Decision
  approver?: string
  evidenceHash?: string
  prevHash: string
  hash: string            // sha256(seq|at|event|payload|prevHash)
}
```

### 10.2 Rules

- Append-only; no edit/delete API.
- The hash chain is verified at startup & at export.
- Export for operators **must be redacted** (API keys, tokens, sensitive prompts).

---

## 11. TypeScript Module Structure

```text
src/
  gateway/        http.ts, ws.ts, auth.ts
  sdk/            host.ts, plugins.ts, permissions.ts
  agent/          planner.ts, react-loop.ts, structured-output.ts, sub-agent.ts
  policy/         policy-engine.ts, review-policy.ts, risk-scorer.ts, approval-gateway.ts
  guard/          scope-guard.ts, sanitizer.ts, schema-pinning.ts, verifier.ts
  tools/          registry.ts, tool-spec.ts, mcp-adapter.ts, local-tools/*
  runtime/        execution-broker.ts, sandbox.ts, audit-log.ts
  memory/         session-memory.ts, knowledge-base.ts
  config/         capabilities.ts, engagement-scope.ts
```

---

## 12. Roadmap and Gates

The full operational roadmap (Phase 0–6) is in `docs/roadmap.md`; the table below summarizes
the binding core phases 1–5.

| Phase | Deliverable | Security gate |
| --- | --- | --- |
| 1 | Gateway + SDK host + ToolRegistry + Planner (read-only tools) | Valid structured output; tools without schemaHash denied |
| 2 | ScopeGuard + PolicyEngine + ApprovalGateway + AuditLog | Negative tests: out-of-scope → deny; post/shell → deny under ai/yolo |
| 3 | ExecutionBroker + sandbox + sanitizer + verifier | Egress deny; findings without evidence denied |
| 4 | MCP adapter + schema pinning + catalog (force sync) | Changed tool → quarantine; complete audit |
| 5 | Memory/RAG + sub-agent + yolo (lab only) | Session isolation; yolo cannot enable post/shell |

Definition of done for each phase: `tsc` + green Vitest, the related threat model IDs closed, and
the audit chain verified.

---

## 13. Appendix

### 13.1 Suggested initial configuration

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

### 13.2 Relationship to source documents

| This design | Source document |
| --- | --- |
| LLM = orchestrator | `docs/harness-analysis.md` §3 (Agent/AI layer) |
| ReviewPolicy manual/ai/yolo | `AGENTS.md` (product facts) |
| Evidence before claims | `docs/harness-analysis.md` §4 |
| Everything constrained | `docs/harness-analysis.md` §4 |
| MCP bypasses ReviewPolicy | `docs/refs/opencode-sdk-yakit-tool-bridge-analysis.md` §6; `docs/refs/opencode-mcp-re-tool-catalog.md` §10 |
| `ctx.tool.transform` as the choke point | Official OpenCode V2 documentation `build/plugins/server` (Tools/Transforms), verified 2026-11-17 |
| Tool poisoning & egress | Official OpenCode V2 documentation `build/plugins/server` + `mcp-servers`, verified 2026-11-17 |
| In-process SDK is viable on Node 22+ | Official OpenCode V2 documentation `build/sdk` (`await using`), verified 2026-11-17 |

> The earlier "advanced analysis (chat)" reference was not persistent; the claims above have been
> re-verified against the official OpenCode V2 documentation on 2026-11-17.

---

*This design document binds the architecture decisions. Changes to an ADR must be recorded as a
revision and update the affected threat model and permission matrix.*
