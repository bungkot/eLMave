# Development Roadmap — Independent Exploitation Harness

- Date: 2026-11-17
- Status: plan (expanded from `docs/formal-design-agentic-orchestrator.md` §12)
- Implementation guide: `docs/development-guide.md`
- How to read: each phase has a **deliverable**, a **gate**, and **gate evidence**;
  status is only changed after the gate passes and is proven.

> **Boundary.** This roadmap is for building the harness on **own assets /
> authorized environments**. `post` and `shell` are never auto-allowed; `yolo`
> is for isolated labs only.

---

## 1. Phase overview

| Phase | Name | Purpose | Key deliverable | Estimate | Status |
| --- | --- | --- | --- | --- | --- |
| 0 | Foundation | project scaffolding, Node 22, TS strict, Zod, Vitest, CI | `package.json`, `tsconfig`, `src/` structure | 1–2 weeks | Not started |
| 1 | Core & Planner | basic Gateway, SDK host, ToolRegistry, read-only Planner | `gateway/auth.ts`, `sdk/host.ts`, `tools/registry.ts`, `agent/planner.ts` | 2–3 weeks | Not started |
| 2 | Guardrail | ScopeGuard, PolicyEngine, ApprovalGateway, AuditLog | `guard/scope-guard.ts`, `policy/*`, `runtime/audit-log.ts` | 2–3 weeks | Not started |
| 3 | Execution | ExecutionBroker, sandbox, sanitizer, Verifier | `runtime/execution-broker.ts`, `guard/sanitizer.ts`, `guard/verifier.ts` | 3–4 weeks | Not started |
| 4 | MCP & Tool | MCP adapter, schema pinning, catalog, open-source wrapping | `tools/mcp-adapter.ts`, `guard/schema-pinning.ts` | 3–4 weeks | Not started |
| 5 | Memory & Autonomy | Memory/RAG, sub-agent, lab `yolo` mode | `memory/*`, `agent/sub-agent.ts` | 2–3 weeks | Not started |
| 6 | Operations | Observability, release, hardening | structured logs, runbook, export redaction | 2–3 weeks | Not started |

Estimates are **relative** for 1–2 part-time developers; adjust after Phase 0
is done and team velocity is measured.

---

## 2. Timeline

```text
Fase 0  ├────┤
Fase 1      ├──────┤
Fase 2            ├──────┤
Fase 3                  ├──────┤
Fase 4              ├────────────────┤   (starts after Phase 1; ends after Phase 3)
Fase 5                          ├──────┤
Fase 6                                    ├────────┤
        M0   M1   M2   M3   M4   M5   M6   M7   M8   M9   M10
```

- Phases 0–3 are sequential (hard dependencies).
- Phase 4 can start after Phase 1 (adapter), then proceeds alongside Phases 2–3.
- Phase 5 requires Phase 3.
- Phase 6 proceeds incrementally; supply chain hardening can start earlier.

Milestones (M) = end of the calendar month since Phase 0 starts. Exact dates are
set once team capacity and priorities are known.

---

## 3. Planning principles

1. **Gate first, status later.** No phase is marked done without gate evidence.
2. **Single execution gate.** All tools go through `ExecutionBroker`; there is no
   bypass path, including direct MCP.
3. **Threat model closed per phase.** Each phase lists the threat IDs it
   controls; remaining risk is recorded as residual.
4. **Evidence before claims.** A finding is valid only with `ToolResult.evidence`.
5. **Default `manual`.** `yolo` is for isolated labs only and cannot enable
   `post`/`shell`.
6. **Independent.** No Yakit runtime dependency; open-source tools are wrapped
   with pinned version/hash and their provenance is recorded.

---

## 4. Phase details

### Phase 0 — Foundation

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

### Phase 1 — Core & Planner

**Purpose**: SDK host runs, tools are registered under contract, the planner
produces structured plans — **without dangerous execution**.

**Deliverables**

- Basic `gateway/auth.ts` (operator auth + Web/CLI ctl → SDK session).
- `sdk/host.ts` (`OpenCode.create()` + session), basic `sdk/plugins.ts`.
- `tools/tool-spec.ts` + `tools/registry.ts` (`schemaHash` validation).
- `agent/planner.ts` with structured output (Zod).
- `sdk/permissions.ts` deny-by-default.

**Tasks**

- [ ] `gateway/auth.ts`: operator auth + minimal HTTP/WS ctl toward the SDK session.
- [ ] In-process SDK host + session creation.
- [ ] `ToolSpec` + `defineTool` + `ToolRegistry.register`.
- [ ] Rejection of tools without `schemaHash`.
- [ ] Planner: `plan()` and `selectAction()` with Zod-validated output.
- [ ] `ctx.tool.transform` plugin installed (basic wrapper).
- [ ] SDK permissions: deny everything except a coarse allowlist.

**Gate**

- Structured output valid for task plans.
- Tools without `schemaHash` rejected at registration.
- No dangerous execution yet; read-only tools.

**Gate evidence**: registry unit tests + planner output; captured `permissions`.

**Dependencies**: Phase 0

**Threat model**: T01 (initial surface), T03 (tool contract)

**Rollback**: disable the plugin; the SDK host executes nothing.

---

### Phase 2 — Guardrail

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
- [ ] `ApprovalGateway` (human/AI) + `approver` recording.
- [ ] `AuditLog`: `sha256(seq|at|event|payload|prevHash)`, startup/export verification.
- [ ] `ReviewPolicy` + `AIReviewRiskControlScore`.
- [ ] Negative matrix tests.

**Gate (negative tests)**

- Out-of-scope target → `deny`.
- Empty/expired scope → `deny` everything.
- `post`/`shell` → `deny` under `ai`/`yolo`.
- Audit chain verified at startup and export.

**Gate evidence**: negative test output; hash chain verification result.

**Dependencies**: Phase 1

**Threat model**: T01, T04, T06

**Rollback**: disable the plugin; no execution is released.

---

### Phase 3 — Execution

**Purpose**: a single execution gate with sandboxing, sanitization, and evidence
verification.

**Deliverables**

- `runtime/execution-broker.ts` (single execution gate).
- `runtime/sandbox.ts` (non-root, resource limits, egress deny).
- `guard/sanitizer.ts` (complete), `guard/verifier.ts`.
- `runtime/audit-log.ts` integrated into execution.

**Tasks**

- [ ] `ExecutionBroker.execute` with timeout + quota.
- [ ] Non-root sandbox + default-deny egress + per-job allowlist.
- [ ] Sanitizer: observation = data; redact secrets.
- [ ] Verifier: `ToolResult.evidence` required.
- [ ] Audit integration for `allowed/asked/approved/rejected/denied/executed`.
- [ ] DoS tests: slow tools stopped; queue bounded.

**Gate**

- Default-deny egress works.
- Findings without `ToolResult.evidence` rejected.
- Timeout and quota stop long jobs.

**Gate evidence**: sandbox/egress tests; timeout logs; example rejected finding.

**Dependencies**: Phase 2

**Threat model**: T02, T05, T07, T08, T09, T13

**Rollback**: disable `ExecutionBroker` → all tools stop; safe.

---

### Phase 4 — MCP & Tool

**Purpose**: exposed as an MCP server and starting to have real tools (read-only →
offensive in stages), including open-source wrapping.

**Deliverables**

- `tools/mcp-adapter.ts` (stdio + Streamable HTTP).
- `guard/schema-pinning.ts` + quarantine.
- `tools/local-tools/*`: read-only `recon`/`scan`, then `fuzz`/`validate`.
- `tools/adapters/*`: 1–2 open-source tools pinned by version/hash + license.
- Open-source tool provenance in AuditLog.

**Tasks**

- [ ] Streamable HTTP server on `127.0.0.1`; discovery endpoint.
- [ ] Normalization `<server>_<tool>`; tool list.
- [ ] Schema pinning + quarantine on `ForceSync`.
- [ ] `recon` and `scan` tools with `evidence`.
- [ ] Open-source wrapping: license, version, hash, risk scan, provenance.
- [ ] `opencode mcp add lab --url ...`; discovery test.
- [ ] Test one read-only tool before offensive ones.

**Gate**

- `lab_<tool>` discovery succeeds in OpenCode.
- `description`/`params` change → quarantine.
- Complete audit (open-source tool provenance recorded).

**Gate evidence**: captured `opencode mcp list`; quarantine tests; audit entries.

**Dependencies**: Phase 1 (adapter), Phase 3 (real execution)

**Threat model**: T03, T10, T08

**Rollback**: withdraw the MCP registration; tools stop in OpenCode.

---

### Phase 5 — Memory & Autonomy

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

**Dependencies**: Phase 3

**Threat model**: T02, T11, T04

**Rollback**: disable sub-agent/yolo; return to `manual`.

---

### Phase 6 — Operations

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

**Dependencies**: All phases

**Threat model**: T05, T06, T10, T12

**Rollback**: release held; harness stays in the lab.

---

## 5. Parallel workstreams

| Workstream | Main phase | Dependencies | Start |
| --- | --- | --- | --- |
| Core harness (gateway, SDK, registry, planner) | 0–1 | — | Phase 0 |
| Guardrail & policy | 2 | Phase 1 | Phase 1 |
| Execution & sandbox | 3 | Phase 2 | Phase 2 |
| MCP transport & tool catalog | 4 | Phase 1, 3 | Phase 1 |
| Open-source wrapping | 4–6 | Phase 3 | Phase 3 |
| Memory & sub-agent | 5 | Phase 3 | Phase 3 |
| Security hardening & release | 6 | All | Incremental |

```text
Fase 0 ─► Fase 1 ─► Fase 2 ─► Fase 3 ─► Fase 4 ─► Fase 5 ─► Fase 6
              │                     │           │
              └── MCP transport ────┘           │
                                    └─ wrapping ┘
```

---

## 6. Dependency matrix

| Phase | Requires | Produces for other phases |
| --- | --- | --- |
| 0 | — | repo, CI, scripts |
| 1 | 0 | gateway, registry, planner, SDK host |
| 2 | 1 | scope/policy/approval/audit |
| 3 | 2 | broker, sandbox, sanitizer, verifier |
| 4 | 1, 3 | MCP adapter, catalog, open-source wrapping |
| 5 | 3 | memory, sub-agent, yolo |
| 6 | 4, 5 | release, observability, runbook |

---

## 7. Risks and mitigations

| ID | Risk | Mitigation | Phase |
| --- | --- | --- | --- |
| R01 | Copyleft-licensed open-source tool | check license before wrapping; document | 4 |
| R02 | Tool poisoning / malicious description | schema pin + sanitization + quarantine | 2, 4 |
| R03 | Prompt injection from observations | Sanitizer; observation = data | 3, 5 |
| R04 | Egress to internal/metadata | default-deny + per-job allowlist | 3, 4 |
| R05 | False findings | Verifier requires `evidence` | 3 |
| R06 | Ambiguous scope | `ScopeGuard` hard deny; expiry; audit | 2 |
| R07 | Sandbox escape / RCE | non-root, resource limits, egress deny | 3 |
| R08 | Long-job DoS | timeout + per-operator quota | 3 |
| R09 | Credential leak via log/export | default redaction; `NetworkDebug` redacted | 3, 6 |
| R10 | In-process SDK host compromise | Node 22+, dependency pin, minimal permission | 1, 6 |
| R11 | Phase 3 delay blocking 4–5 | start MCP transport earlier (after Phase 1) | 1, 4 |
| R12 | Design change during implementation | ADR revision + threat model + matrix; record here | All |

---

## 8. Cross-phase acceptance criteria

- [ ] No tool calls outside `ExecutionBroker`.
- [ ] `ScopeGuard` default-deny and empty/expired scope reject everything.
- [ ] `post`/`shell` never auto-allowed; `yolo` cannot enable them.
- [ ] All tools have `schemaHash`, `capability`, `risk`, `sideEffects`,
  `yoloAllowed`.
- [ ] All executions produce `ToolResult.evidence`.
- [ ] Append-only audit + hash chain verified.
- [ ] `NetworkDebug` default redacted; exports redacted.
- [ ] MCP server on loopback; not exposed without authentication.
- [ ] SDK built-in tools wrapped/disabled.
- [ ] No Yakit runtime dependency.

---

## 9. Definition of done

`tsc` clean, Vitest green, related threat model closed, audit chain verified,
tool discovery proven, and test findings carry `evidence`.

For Phase 6, additionally: E2E green in the lab, incident runbook present,
redaction verified, and supply chain hardening recorded.

---

## 10. Progress tracking

Entry template (fill in after the gate passes):

| Phase | Status | Gate passed | Evidence (output/hash) | Threat model closed | Date |
| --- | --- | --- | --- | --- | --- |
| 0 | Not started | — | — | — | — |
| 1 | Not started | — | — | — | — |
| 2 | Not started | — | — | — | — |
| 3 | Not started | — | — | — | — |
| 4 | Not started | — | — | — | — |
| 5 | Not started | — | — | — | — |
| 6 | Not started | — | — | — | — |

Update rules:

1. Change phase status **only** after the gate passes and is proven.
2. Record gate evidence (test output, artifact hash) in the related entry.
3. Update the affected threat model/permission matrix.
4. If the design changes, add an ADR revision and reference it from here.

---

## 11. Revision history

| Date | Revision | Changes |
| --- | --- | --- |
| 2026-11-17 | 0.1 | Initial roadmap: phase 0–6 overview, deliverables, gates, workstreams, risks. |
| 2026-11-17 | 0.2 | Added: timeline, estimates, per-phase tasks, gate evidence, dependency matrix, ID-tagged risks, acceptance criteria, progress tracking. |

---

## 12. References

- `docs/formal-design-agentic-orchestrator.md` — binding design (ADRs, matrices, threat model).
- `docs/development-guide.md` — stack, structure, code contracts, configuration.
- `docs/tool-authoring.md` — the `ToolSpec` contract and tool wrapping.
- `docs/testing-and-verification.md` — test strategy and evidence.
- `docs/refs/opencode-mcp-re-tool-catalog.md` — open-source tool/MCP catalog.
- `docs/refs/opencode-sdk-yakit-tool-bridge-analysis.md` — SDK/MCP integration patterns (reference).
