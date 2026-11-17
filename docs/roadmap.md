# Development Roadmap — Three Tracks: Agentic Code · Agentic Scanning · MCP

- Date: 2026-11-17
- Revision: 0.4 — restructured into three tracks (see §11)
- Status: plan (expanded from `docs/formal-design-agentic-orchestrator.md` §12)
- Binding design: `docs/formal-design-agentic-orchestrator.md` (design rev 0.3 — SDK-as-agent)
- Implementation guide: `docs/development-guide.md`
- How to read: each sub-phase has a **deliverable**, a **gate**, and **gate evidence**;
  status is only changed after the gate passes and is proven.
- Track IDs: `A<n>` = agentic code (TypeScript + OpenCode SDK + Ollama), `B<n>` = agentic
  scanning (tools), `C<n>` = MCP. Old Phase 0–6 tags are kept in parentheses so existing
  cross-references in other docs remain resolvable.

> **Boundary.** This roadmap is for building the harness on **own assets /
> authorized environments**. `post` and `shell` are never auto-allowed; `yolo`
> is for isolated labs only.

---

## 1. Track overview

| Track | Name | Scope | Old phases | Estimate | Status |
| --- | --- | --- | --- | --- | --- |
| **A** | **Agentic Code Development** (TypeScript + OpenCode SDK + Ollama) | Harness/agent itself: gateway, SDK host, Ollama provider, ToolRegistry, thin planner, guardrail, broker, sandbox, memory, operations | 0, 1, 2, 3, 5, 6 | 13–19 weeks | Not started |
| **B** | **Agentic Scanning** (tools) | Tool catalog, local `recon`/`scan` → `fuzz`/`validate` tools, scanning session, evidence/verification, open-source wrapping | 4 (tool part) | 5–7 weeks | Not started |
| **C** | **MCP Development** | MCP server, schema pinning/quarantine, `<server>_<tool>` discovery, `callTool → broker.execute` routing | 4 (adapter part) | 4–6 weeks | Not started |

Estimates are **relative** for 1–2 part-time developers; adjust after A0 is done and team
velocity is measured.

**How the three tracks relate**

- Track A sub-phases are **sequential hard dependencies** (A0 → A1 → A2 → A3 → A5 → A6).
- Track B and C are **views on shared infrastructure, not independent stacks**. Both depend on
  A3 `ExecutionBroker`. They may start early (B1/C1 after A1) but **no tool may execute outside
  A3**, including tools reached through MCP.
- Old Phase 4 is split: tool work → Track B, MCP transport work → Track C. All security gates
  from the original phase plan are preserved in the sub-phase gates below.

---

## 2. Timeline

```text
Track A   A0 ── A1 ── A2 ── A3 ─────────────── A5 ── A6
Track B             └── B1 ── B2 ── B3
Track C             └── C1 ── C2 ── C3 ── C4
          M0   M1   M2   M3   M4   M5   M6   M7   M8   M9   M10
```

- A0–A3 are sequential (hard dependencies).
- B1 and C1 may start after A1; B2/B3 and C3/C4 require A2/A3.
- A5 requires A3. A6 proceeds incrementally; supply-chain hardening can start earlier.

Milestones (M) = end of the calendar month since A0 starts. Exact dates are set once team
capacity and priorities are known.

---

## 3. Planning principles

1. **Gate first, status later.** No sub-phase is marked done without gate evidence.
2. **Single execution gate.** All tools go through `ExecutionBroker`; there is no bypass path,
   including direct MCP `callTool`.
3. **Threat model closed per sub-phase.** Each sub-phase lists the threat IDs it controls;
   remaining risk is recorded as residual.
4. **Evidence before claims.** A finding is valid only with `ToolResult.evidence`.
5. **Default `manual`.** `yolo` is for isolated labs only and cannot enable `post`/`shell`.
6. **Independent.** No Yakit runtime dependency; open-source tools are wrapped with pinned
   version/hash and their provenance is recorded.
7. **SDK owns the loop (design rev 0.3).** OpenCode SDK is the agent; there is no second
   `react-loop.ts`. The only execution gate is `ExecutionBroker.execute`, called from
   `ctx.tool.transform` → `wrapTool`. Ollama is a local LLM provider, pinned by digest.
8. **Tracks are views, not bypasses.** Track B/C never call a tool outside A3's broker. A track
   may start early, but its gate cannot pass before the broker gate (A3) passes.

---

## 4. Track details

### Track A — Agentic Code Development (TypeScript + OpenCode SDK + Ollama)

Build the harness itself. Per design rev 0.3, OpenCode SDK is the agent and owns the ReAct loop;
Ollama is a local provider (OpenAI-compatible `/v1`, model pinned by digest); `agent/*` contains
only thin helpers, and `wrapTool` in `ctx.tool.transform` is the sole caller of the broker.

#### A0 — Foundation (Phase 0)

**Purpose**: prepare a healthy code repo before harness logic is written.

**Deliverables**

- `package.json`, strict `tsconfig.json`, `vitest.config.ts`, basic CI.
- `src/` and `tests/` structure per `docs/development-guide.md` §3.
- `dev`, `build`, `start`, `typecheck`, `test`, `test:e2e` scripts.
- Short `README` + `config/default.json` + `config/lab.json` (no secrets).

**Tasks**

- [ ] Initialize package manager + lockfile; lock Node 22+.
- [ ] Strict `tsconfig`; lint/format rules.
- [ ] Vitest: unit, integration, negative, e2e (separated).
- [ ] CI: `typecheck` → `test` → (e2e when available).
- [ ] `fixtures/` directory and MCP stub.
- [ ] `EngagementScope` template + empty `ToolSpec`.

**Gate**

- `pnpm typecheck` and `pnpm test` green in a clean repo.
- Lockfile locked; Node 22+ documented.

**Gate evidence**: captured CI output + commit hash.

**Dependencies**: —

**Threat model**: — (pre-harness)

**Rollback**: remove the scaffolding; no target impact.

---

#### A1 — Agent Core & SDK Host (Phase 1)

**Purpose**: the SDK host runs with Ollama as provider, tools are registered under contract, and
guardrails are installed — **without dangerous execution**.

**Deliverables**

- Basic `gateway/auth.ts` (operator auth + Web/CLI ctl → SDK session).
- `sdk/host.ts` (`OpenCode.create()` + session) with Ollama provider config.
- `sdk/plugins.ts`: `ctx.tool.transform` (`wrapTool` → `broker.execute`) and `ctx.mcp.transform`.
- `tools/tool-spec.ts` + `tools/registry.ts` (`schemaHash` validation).
- `agent/planner.ts` as a thin structured-output helper (no `react-loop.ts`).
- `sdk/permissions.ts` deny-by-default.

**Tasks**

- [ ] `gateway/auth.ts`: operator auth + minimal HTTP/WS ctl toward the SDK session.
- [ ] In-process SDK host + session creation.
- [ ] Configure Ollama as OpenAI-compatible provider (`/v1`, model pinned by digest) in the OpenCode config loaded by the host; verify `sessions.prompt()` reaches the model and tool-calling works (design rev 0.3).
- [ ] `ToolSpec` + `defineTool` + `ToolRegistry.register`.
- [ ] Rejection of tools without `schemaHash`.
- [ ] Planner (thin helper): `plan()` and `selectAction()` with Zod-validated output — no second `react-loop.ts`.
- [ ] `ctx.tool.transform` installed: `wrapTool` is the only caller of `broker.execute`.
- [ ] SDK permissions: deny everything except a coarse allowlist; remove built-in `shell`/`edit`/`write` where not needed.

**Gate**

- Structured output valid for task plans.
- Tools without `schemaHash` rejected at registration.
- Ollama provider verified: `sessions.prompt()` reaches the model and tool-calling succeeds.
- One dummy read-only tool executes through `wrapTool` → `broker.execute` with exactly one audit entry.
- No dangerous execution yet; read-only tools.

**Gate evidence**: registry unit tests + planner output; captured `permissions`; Ollama provider/tool-call trace; audit entry.

**Dependencies**: A0

**Threat model**: T01 (initial surface), T03 (tool contract), T12 (SDK host)

**Rollback**: disable the plugin; the SDK host executes nothing.

---

#### A2 — Guardrail (Phase 2)

**Purpose**: default-deny authorization before any execution.

**Deliverables**

- `guard/scope-guard.ts`, basic `guard/sanitizer.ts`.
- `policy/policy-engine.ts`, `policy/review-policy.ts`, `policy/risk-scorer.ts`,
  `policy/approval-gateway.ts`.
- `runtime/audit-log.ts` (append-only hash chain).
- `config/engagement-scope.ts`, `config/capabilities.ts`.

**Tasks**

- [ ] `ScopeGuard.assertInScope` + expiry + `approvedBy`.
- [ ] `PolicyEngine` per matrix §6.1 + escalation §6.3.
- [ ] `ApprovalGateway` (human/AI) + `approver` recording (via `ctx.permission.assert` / gateway).
- [ ] `AuditLog`: `sha256(seq|at|event|payload|prevHash)`, startup/export verification.
- [ ] `ReviewPolicy` + `AIReviewRiskControlScore`.
- [ ] Negative matrix tests.

**Gate (negative tests)**

- Out-of-scope target → `deny`.
- Empty/expired scope → `deny` everything.
- `post`/`shell` → `deny` under `ai`/`yolo`.
- Audit chain verified at startup and export.

**Gate evidence**: negative test output; hash chain verification result.

**Dependencies**: A1

**Threat model**: T01, T04, T06

**Rollback**: disable the plugin; no execution is released.

---

#### A3 — Execution (Phase 3)

**Purpose**: a single execution gate with sandboxing, sanitization, and evidence verification.

**Deliverables**

- `runtime/execution-broker.ts` (single execution gate).
- `runtime/sandbox.ts` (non-root child process/container, resource limits, egress deny).
- `guard/sanitizer.ts` (complete), `guard/verifier.ts`.
- `runtime/audit-log.ts` integrated into execution.
- `runtime/evidence-store.ts` (artifacts referenced by `artifactRef`, not copied secrets).

**Tasks**

- [ ] `ExecutionBroker.execute` with timeout + quota.
- [ ] Non-root sandbox + default-deny egress + per-job allowlist (including local TS tools).
- [ ] Sanitizer: observation = data; redact secrets; sanitize MCP instructions/resources.
- [ ] Verifier: `ToolResult.evidence` required (`hash`, `at`, `artifactRef`).
- [ ] Audit integration for `allowed/asked/approved/rejected/denied/executed` — one entry per execution.
- [ ] DoS tests: slow tools stopped; queue bounded.

**Gate**

- Default-deny egress works.
- Findings without `ToolResult.evidence` rejected.
- Timeout and quota stop long jobs.

**Gate evidence**: sandbox/egress tests; timeout logs; example rejected finding.

**Dependencies**: A2

**Threat model**: T02, T05, T07, T08, T09, T13

**Rollback**: disable `ExecutionBroker` → all tools stop; safe.

---

#### A5 — Memory & Autonomy (Phase 5)

**Purpose**: per-engagement memory, sub-agent, and lab-only `yolo` mode.

**Deliverables**

- `memory/session-memory.ts`, `memory/knowledge-base.ts`.
- `agent/sub-agent.ts`.
- `yolo` mode for isolated labs only.

**Tasks**

- [ ] Memory/knowledge base isolation per engagement.
- [ ] Sub-agent for read-only/reverse-engineering tasks.
- [ ] `yolo` locked to the lab; cannot enable `post`/`shell`.
- [ ] Sanitizer over KB/observation content.
- [ ] Memory retention and cleanup.

**Gate**

- Session/memory isolation per engagement.
- `yolo` cannot enable `post`/`shell`.
- Sanitizer blocks injection from observations/KB.

**Gate evidence**: isolation tests; yolo→post/shell rejection tests.

**Dependencies**: A3

**Threat model**: T02, T11, T04

**Rollback**: disable sub-agent/yolo; return to `manual`.

---

#### A6 — Operations (Phase 6)

**Purpose**: operable and safely releasable.

**Deliverables**

- Observability: structured logs, metrics, audit tracing.
- Packaging/runbook, automatic redaction before export.
- Supply chain hardening (dependency pinning, MCP/skill scanning).

**Tasks**

- [ ] Structured logs + metrics (tool duration, denials, quota).
- [ ] Automatic redaction before export/reports.
- [ ] Recovery/incident runbook + audit log restore.
- [ ] Advanced CI: dependency scanning + hash verification.
- [ ] Lab E2E environment.

**Gate**

- E2E green in the lab environment.
- Audit chain + redaction verified.
- Recovery/incident runbook exists.

**Gate evidence**: E2E results; incident simulation; redaction results.

**Dependencies**: All tracks

**Threat model**: T05, T06, T10, T12

**Rollback**: release held; harness stays in the lab.

---

### Track B — Agentic Scanning (tools)

Build scanning capability and its evidence discipline. Every tool is executed via A3's broker;
a *scanning* SDK session uses a restricted tool-set, separate from the code-dev session.

#### B1 — Tool Catalog & Read-only Tools (Phase 4a)

**Purpose**: define tools under contract and prove one safe, read-only execution.

**Deliverables**

- `ToolSpec` entries for `utility`/`recon`/`scan` with `schemaHash`, `capability`, `risk`,
  `sideEffects`, `yoloAllowed`, `timeoutMs`.
- `tools/local-tools/*`: read-only `recon` and `scan` tools producing `ToolResult.evidence`.
- Registry entries filtered by `enabledCapabilities`.

**Tasks**

- [ ] Author `ToolSpec` for each tool; reject any without `schemaHash`.
- [ ] Implement one `utility`/`recon` tool with sanitized observation + `evidence`.
- [ ] Implement one `scan` tool enforcing `allowedPorts` against scope.
- [ ] Register through `ToolRegistry`; confirm it reaches `wrapTool` → broker.
- [ ] Unit tests for params validation and evidence generation.

**Gate**

- Every registered tool has `schemaHash`, `capability`, `risk`, `sideEffects`, `yoloAllowed`.
- One read-only tool returns `ToolResult.evidence` with `hash`, `at`, `artifactRef`.
- Tool execution recorded as exactly one audit entry.

**Gate evidence**: registry tests; sample `ToolResult` with evidence; audit entry.

**Dependencies**: A1 (registry), A3 (real execution)

**Threat model**: T03, T09

**Rollback**: unregister the tools; no execution path remains.

---

#### B2 — Scanning Session & Staged Verification (Phase 4b)

**Purpose**: operate scanning under policy with human approval and staged rollout.

**Deliverables**

- Dedicated scanning SDK session with restricted tool-set (no filesystem/edit).
- Approval flow wired to `ApprovalGateway` (`manual` default).
- Staged verification: read-only → `scan` → (lab only) `fuzz`/`validate`.

**Tasks**

- [ ] Separate SDK session for scanning; `post`/`shell` never in its tool-set.
- [ ] `ask` decisions routed to operator approval via gateway / `ctx.permission.assert`.
- [ ] Stage 1: read-only tool (already in B1) confirmed under `manual`.
- [ ] Stage 2: `scan` tool with approval + scope + port check.
- [ ] Verifier rejects any finding without `ToolResult.evidence`.

**Gate**

- `post`/`shell` → `deny` under `ai`/`yolo`; `ask` under `manual`.
- Findings without evidence rejected.
- Staged read-only → scan test passes with approval recorded.

**Gate evidence**: approval trace; negative tests; sample rejected finding.

**Dependencies**: A2, A3, B1

**Threat model**: T01, T04, T09

**Rollback**: disable the scanning session; return to code-dev session only.

---

#### B3 — Offensive Tools & Open-source Wrapping (Phase 4c)

**Purpose**: add `fuzz`/`validate` in the lab and wrap licensed external tools safely.

**Deliverables**

- `tools/local-tools/*` for lab-only `fuzz`/`validate` with evidence.
- `tools/adapters/*`: 1–2 open-source tools pinned by version + hash + license.
- Open-source tool provenance recorded in `AuditLog`.

**Tasks**

- [ ] Lab-only `fuzz` and `validate` tools with `yoloAllowed` governed by ADR-005.
- [ ] License check before wrapping; document copyleft risk (R01).
- [ ] Pin version + hash; scan for prompt-injection/supply-chain risk.
- [ ] Record provenance in audit; never use `latest`.
- [ ] Test one wrapped tool against a lab target through the broker.

**Gate**

- License, version, and hash recorded for each wrapped tool.
- Provenance present in audit; `yolo` cannot enable `post`/`shell`.
- All executions still go through A3 broker with evidence.

**Gate evidence**: license/hash records; audit provenance entries; lab test result.

**Dependencies**: B1, B2, A3

**Threat model**: T10, T07, T08, T02

**Rollback**: withdraw the adapter registration; tools stop in OpenCode.

---

### Track C — MCP Development

Build the MCP transport, pin its catalog, and prove that MCP never becomes an execution bypass.

#### C1 — MCP Server Transport (Phase 4a)

**Purpose**: expose our own MCP server on loopback.

**Deliverables**

- `tools/mcp-adapter.ts`: Streamable HTTP for OpenCode remote, stdio for local tools.
- Bind `127.0.0.1` (default port `11432`); discovery endpoint.

**Tasks**

- [ ] Streamable HTTP server on `127.0.0.1`; discovery endpoint.
- [ ] Verify `ELM_BIND` cannot silently become `0.0.0.0` without auth.
- [ ] Health/discovery check with a stub tool catalog.

**Gate**

- Server binds loopback only; endpoint reachable at `http://127.0.0.1:11432/mcp`.
- No unauthenticated external exposure.

**Gate evidence**: server log; `curl` discovery output.

**Dependencies**: A1

**Threat model**: T08

**Rollback**: stop the server; no agent impact.

---

#### C2 — Schema Pinning & Quarantine (Phase 4b)

**Purpose**: detect tool poisoning on catalog refresh.

**Deliverables**

- `guard/schema-pinning.ts`.
- `ForceSync` comparison of `schemaHash`; quarantine on change.

**Tasks**

- [ ] Compute/store `schemaHash` for `description`, `params`, `capability`, `risk`.
- [ ] `ForceSync` → compare → quarantine on mismatch; `deny` until reviewed.
- [ ] Negative test: modified `description`/`params` → quarantine.

**Gate**

- `description`/`params` change → quarantine.
- Quarantined tool cannot be executed.

**Gate evidence**: quarantine test output; audit entry.

**Dependencies**: A1, C1

**Threat model**: T03, T10

**Rollback**: reset the pin; tool returns to normal after review.

---

#### C3 — Discovery & OpenCode Connection (Phase 4c)

**Purpose**: make our tools visible to OpenCode under normalized names.

**Deliverables**

- `<server>_<tool>` normalization (e.g. `lab_port_scan`).
- `ctx.mcp.transform` registration + `opencode mcp add lab --url ...`.

**Tasks**

- [ ] Tool list normalization to `<server>_<tool>`.
- [ ] `opencode mcp add lab --url http://127.0.0.1:11432/mcp`.
- [ ] `opencode mcp list` discovery test.
- [ ] Test one read-only tool before any offensive tool.

**Gate**

- `lab_<tool>` discovery succeeds in OpenCode.
- Read-only tool test passes; offensive tools not yet enabled.

**Gate evidence**: captured `opencode mcp list`; read-only test result.

**Dependencies**: C1, C2

**Threat model**: T03

**Rollback**: withdraw MCP registration; tools stop in OpenCode.

---

#### C4 — Execution Routing & Sanitization (Phase 4d)

**Purpose**: prove MCP is a transport, not a bypass.

**Deliverables**

- MCP `callTool` translated into `broker.execute` (never direct `tool.execute`).
- Exactly one audit entry per execution, including MCP-reached tools.
- Sanitizer over MCP server instructions/resources and tool descriptions.

**Tasks**

- [ ] Route `callTool` → `broker.execute` through `wrapTool`.
- [ ] Negative test: direct MCP call cannot bypass ScopeGuard/PolicyEngine.
- [ ] Sanitizer filters MCP instructions/resources before system prompt (T02/T10).
- [ ] Complete audit including open-source tool provenance.

**Gate**

- No execution path from MCP bypasses A3 broker.
- Sanitizer blocks injected instructions from MCP metadata.
- Complete audit chain verified.

**Gate evidence**: bypass negative test; sanitizer test; audit chain verification.

**Dependencies**: C3, A3

**Threat model**: T02, T06, T08

**Rollback**: disable MCP registration; all execution returns to local tools only.

---

## 5. Track dependencies

| Item | Track | Requires | Produces for other tracks |
| --- | --- | --- | --- |
| A0 Foundation | A | — | repo, CI, scripts |
| A1 Agent Core | A | A0 | SDK host, Ollama config, registry, planner, wrapTool |
| A2 Guardrail | A | A1 | scope/policy/approval/audit |
| A3 Execution | A | A2 | broker, sandbox, sanitizer, verifier — **required by B and C** |
| A5 Memory | A | A3 | memory, sub-agent, lab yolo |
| A6 Operations | A | all tracks | release, observability, runbook |
| B1 Tool Catalog | B | A1, A3 | read-only tools + evidence |
| B2 Scanning Session | B | A2, A3, B1 | scanning session, staged verification |
| B3 Offensive Tools | B | B1, B2, A3 | fuzz/validate, wrapped tools, provenance |
| C1 MCP Transport | C | A1 | loopback MCP server |
| C2 Schema Pinning | C | A1, C1 | quarantine on schema change |
| C3 Discovery | C | C1, C2 | `lab_<tool>` in OpenCode |
| C4 Execution Routing | C | C3, A3 | no-bypass proof, sanitized MCP metadata |

---

## 6. Dependency matrix (track-level)

| Track | Requires | Produces |
| --- | --- | --- |
| A | A0 → A1 → A2 → A3 → A5 → A6 | the harness and the single execution gate |
| B | A1, A2, A3 | scanning tools and evidence discipline |
| C | A1, A3 | MCP transport, catalog pinning, discovery |

**Critical path**: A0 → A1 → A2 → A3 → (B2 / C4) → A5 → A6.

---

## 7. Risks and mitigations

| ID | Risk | Mitigation | Track |
| --- | --- | --- | --- |
| R01 | Copyleft-licensed open-source tool | check license before wrapping; document | B3 |
| R02 | Tool poisoning / malicious description | schema pin + sanitization + quarantine | A1, C2 |
| R03 | Prompt injection from observations | Sanitizer; observation = data | A3, A5, C4 |
| R04 | Egress to internal/metadata | default-deny + per-job allowlist | A3, C1, C4 |
| R05 | False findings | Verifier requires `evidence` | A3, B2 |
| R06 | Ambiguous scope | `ScopeGuard` hard deny; expiry; audit | A2 |
| R07 | Sandbox escape / RCE | non-root child process, resource limits, egress deny | A3 |
| R08 | Long-job DoS | timeout + per-operator quota | A3 |
| R09 | Credential leak via log/export | default redaction; `NetworkDebug` redacted | A3, A6 |
| R10 | In-process SDK host compromise | Node 22+, dependency pin, minimal permission | A1, A6 |
| R11 | A3 delay blocking B/C | start C1 and B1 after A1 (transport/catalog first) | A3, B, C |
| R12 | Design change during implementation | ADR revision + threat model + matrix; record here | all |
| R13 | Local Ollama model tool-calling too weak | verify tool-calling at A1 gate; pin model digest; avoid small models for fuzz/validate | A1, B3 |

---

## 8. Cross-track acceptance criteria

- [ ] No tool calls outside `ExecutionBroker` (A3), including MCP (C4).
- [ ] `ScopeGuard` default-deny and empty/expired scope reject everything (A2).
- [ ] `post`/`shell` never auto-allowed; `yolo` cannot enable them (A2, B2, A5).
- [ ] All tools have `schemaHash`, `capability`, `risk`, `sideEffects`, `yoloAllowed` (B1, C2).
- [ ] All executions produce `ToolResult.evidence` (A3, B1).
- [ ] Append-only audit + hash chain verified (A2, C4).
- [ ] `NetworkDebug` default redacted; exports redacted (A6).
- [ ] MCP server on loopback; not exposed without authentication (C1).
- [ ] SDK built-in tools wrapped/removed (A1); Ollama provider pinned by digest (A1).
- [ ] No Yakit runtime dependency (all tracks).

---

## 9. Definition of done

`tsc` clean, Vitest green, related threat model closed, audit chain verified, tool discovery
proven, and test findings carry `evidence`. Applies per sub-phase.

For A6, additionally: E2E green in the lab, incident runbook present, redaction verified, and
supply chain hardening recorded.

---

## 10. Progress tracking

Entry template (fill in after the gate passes):

| Track | Sub-phase | Status | Gate passed | Evidence (output/hash) | Threat model closed | Date |
| --- | --- | --- | --- | --- | --- | --- |
| A | A0 (Phase 0) | Not started | — | — | — | — |
| A | A1 (Phase 1) | Not started | — | — | — | — |
| A | A2 (Phase 2) | Not started | — | — | — | — |
| A | A3 (Phase 3) | Not started | — | — | — | — |
| A | A5 (Phase 5) | Not started | — | — | — | — |
| A | A6 (Phase 6) | Not started | — | — | — | — |
| B | B1 (Phase 4a) | Not started | — | — | — | — |
| B | B2 (Phase 4b) | Not started | — | — | — | — |
| B | B3 (Phase 4c) | Not started | — | — | — | — |
| C | C1 (Phase 4a) | Not started | — | — | — | — |
| C | C2 (Phase 4b) | Not started | — | — | — | — |
| C | C3 (Phase 4c) | Not started | — | — | — | — |
| C | C4 (Phase 4d) | Not started | — | — | — | — |

Update rules:

1. Change status **only** after the gate passes and is proven.
2. Record gate evidence (test output, artifact hash) in the related entry.
3. Update the affected threat model/permission matrix.
4. If the design changes, add an ADR revision and reference it from here.

---

## 11. Revision history

| Date | Revision | Changes |
| --- | --- | --- |
| 2026-11-17 | 0.1 | Initial roadmap: phase 0–6 overview, deliverables, gates, workstreams, risks. |
| 2026-11-17 | 0.2 | Added: timeline, estimates, per-phase tasks, gate evidence, dependency matrix, ID-tagged risks, acceptance criteria, progress tracking. |
| 2026-11-17 | 0.3 | Design rev 0.3 locked (R12): OpenCode SDK is the agent (no second `react-loop.ts`); Ollama local provider pinned by digest; guardrails in `ctx.tool.transform` → `wrapTool` → `ExecutionBroker`. Phase 1 tasks/gate updated; no phase, gate, or estimate changed. |
| 2026-11-17 | 0.4 | **Restructured into three tracks**: A = agentic code (TS + OpenCode SDK + Ollama), B = agentic scanning (tools), C = MCP. Old Phase 0–6 tags retained in parentheses. All gates, threat IDs, risks, and acceptance criteria preserved; new R13 (Ollama tool-calling), C4 (no-bypass gate), A3 evidence-store deliverable. No security gate removed. |

---

## 12. References

- `docs/formal-design-agentic-orchestrator.md` — binding design (ADRs, matrices, threat model).
- `docs/development-guide.md` — stack, structure, code contracts, configuration.
- `docs/tool-authoring.md` — the `ToolSpec` contract and tool wrapping.
- `docs/testing-and-verification.md` — test strategy and evidence.
- `docs/refs/opencode-mcp-re-tool-catalog.md` — open-source tool/MCP catalog.
- `docs/refs/opencode-sdk-yakit-tool-bridge-analysis.md` — SDK/MCP integration patterns (reference).
