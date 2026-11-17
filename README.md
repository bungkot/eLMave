# eLMave — LLM-assisted vulnerability exploitation

> Workspace for designing and building a **self-hosted security testing
> harness** — our own MCP server and TypeScript tools — that uses an LLM as an
> **orchestrator** for vulnerability validation/exploitation on **own assets /
> authorized environments**.

- **Current status:** a **documentation & analysis** workspace. There is no
  `package.json`, build, or CI yet. Phase 0 in [`docs/roadmap.md`](docs/roadmap.md)
  starts the code scaffolding.
- **Harness nature:** independent — it does not use the Yakit runtime/engine.
  Library dependencies (`@opencode/sdk`, `@modelcontextprotocol/sdk`) are pinned
  by version + hash. Open-source tools may be wrapped, but the wrapping layer,
  registry, policy, and audit remain owned by this repo.
- **Reference:** Yakit (`master` @ `f813c64de`, `1.4.9-1113`) is used only as an
  architecture and tool-naming reference, not a dependency.

> **Ethical note (mandatory).** All development and testing is only for **own
> assets or systems that explicitly grant permission**. The documents here
> contain architecture, policy, and contracts — **not real exploitation
> payloads**. Do not direct these capabilities at third-party systems.

---

## What is built

A harness with a single binding execution gate: the LLM agent plans actions, but
every tool must pass through the guardrail chain before running.

```text
   Web UI / CLI
        │
        ▼
   Gateway (operator auth, HTTP/WS)
        │
        ▼
   OpenCode SDK (in-process, Node 22+)
        │
        ▼
   Agent Core (Planner · ReAct · Sub-agent · Verifier)
        │
        ▼
   ScopeGuard → PolicyEngine → ApprovalGateway
        │
        ▼
   ExecutionBroker  (single execution gate)
        │
        ├─► MCP Adapters (stdio / Streamable HTTP)
        └─► Local TS Tools (recon / scan / fuzz / validate / utility)
```

The core components and their rationale are in
[`docs/formal-design-agentic-orchestrator.md`](docs/formal-design-agentic-orchestrator.md).

---

## Design principles

1. **Independent.** The MCP server, registry, policy, sandbox, and audit are
   written ourselves. Yakit is only an architecture/tool-naming reference.
2. **Single execution gate.** All tools — including direct MCP — go through
   `ExecutionBroker`. There is no bypass path.
3. **Default-deny.** `ScopeGuard` rejects targets outside the permission scope;
   an empty/expired scope rejects everything.
4. **Permission matrix.** `post` and `shell` are never auto-allowed; `yolo` is
   for isolated labs only; the production default is `manual`.
5. **Evidence before claims.** A finding is valid only with
   `ToolResult.evidence`; without evidence, the Verifier rejects it.
6. **Append-only audit.** A hash-chained log, verified at startup and export;
   `NetworkDebug` is redacted by default.

---

## Repository structure

```text
README.md            — this document
LICENSE              — MIT
opencode.json        — OpenCode configuration (permissions, instructions)
docs/
  formal-design-agentic-orchestrator.md — binding design (ADRs, matrices, threat model)
  development-guide.md                  — implementation guide
  roadmap.md                            — phases, gates, progress tracking
  tool-authoring.md                     — ToolSpec contract & tool wrapping
  testing-and-verification.md           — test strategy and evidence rules
  harness-analysis.md                   — harness layer analysis (Yakit reference)
  refs/
    opencode-sdk-yakit-tool-bridge-analysis.md — SDK/MCP integration patterns (reference)
    opencode-mcp-re-tool-catalog.md            — open-source tool/MCP catalog
.opencode/
  agents/             — orchestrator designer, mission planner, tool operator, verifier, etc.
  skills/             — exploitation-flow, security-guardrails, tool-catalog, etc.
```

---

## Documentation

| Document | Contents |
| --- | --- |
| [`docs/formal-design-agentic-orchestrator.md`](docs/formal-design-agentic-orchestrator.md) | The **binding** architectural design: ADR-001..006, permission matrix §6, threat model T01–T13, module structure |
| [`docs/development-guide.md`](docs/development-guide.md) | Implementation guide: stack, structure, code contracts, configuration, MCP server, troubleshooting |
| [`docs/roadmap.md`](docs/roadmap.md) | Phases 0–6 with deliverables, gates, gate evidence, timeline, risks, progress tracking |
| [`docs/tool-authoring.md`](docs/tool-authoring.md) | `ToolSpec` contract, registration, and how to wrap open-source tools |
| [`docs/testing-and-verification.md`](docs/testing-and-verification.md) | Testing levels, mandatory negative tests, evidence rules, CI gates |
| [`docs/harness-analysis.md`](docs/harness-analysis.md) | Harness layer analysis of the reference product (Yakit) |
| [`docs/refs/opencode-mcp-re-tool-catalog.md`](docs/refs/opencode-mcp-re-tool-catalog.md) | Curation of open-source RE/security MCP servers |
| [`docs/refs/opencode-sdk-yakit-tool-bridge-analysis.md`](docs/refs/opencode-sdk-yakit-tool-bridge-analysis.md) | OpenCode SDK ↔ tool/MCP integration patterns |

---

## Roadmap summary

| Phase | Name | Purpose | Status |
| --- | --- | --- | --- |
| 0 | Foundation | scaffolding, Node 22, strict TS, Vitest, CI | Not started |
| 1 | Core & Planner | basic Gateway, SDK host, ToolRegistry, read-only Planner | Not started |
| 2 | Guardrails | ScopeGuard, PolicyEngine, ApprovalGateway, AuditLog | Not started |
| 3 | Execution | ExecutionBroker, sandbox, sanitizer, Verifier | Not started |
| 4 | MCP & Tools | MCP adapter, schema pinning, open-source wrapping | Not started |
| 5 | Memory & Autonomy | Memory/RAG, sub-agent, lab `yolo` mode | Not started |
| 6 | Operations | Observability, release, hardening | Not started |

Details, gates, and gate evidence per phase: [`docs/roadmap.md`](docs/roadmap.md).

---

## Getting started

Currently:

1. Read [`docs/formal-design-agentic-orchestrator.md`](docs/formal-design-agentic-orchestrator.md)
   for the binding ADRs and permission matrix.
2. Read [`docs/development-guide.md`](docs/development-guide.md) and
   [`docs/roadmap.md`](docs/roadmap.md) to start Phase 0 implementation.
3. The OpenCode configuration is available in [`opencode.json`](opencode.json);
   project skills and agents are in `.opencode/`.

Once Phase 0 is complete, the daily workflow (MCP server + tools) is documented
in [`docs/development-guide.md`](docs/development-guide.md) §7.

---

## Project agents and skills

Skills (`.opencode/skills/`): `exploitation-flow`, `tool-catalog`,
`security-guardrails`, `finding-verification`, `audit-redaction`,
`mcp-and-tools`, `orchestrator-design`.

Agents (`.opencode/agents/`): `orchestrator-designer`, `mission-planner`,
`tool-operator`, `finding-verifier`, `security-reviewer`,
`mcp-tools-builder`, `orchestrator-implementor`.

---

## License

Released under the [MIT License](LICENSE).
