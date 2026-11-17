# AGENTS.md — eLMave workspace

> Source docs are in English. This file is the agent entry point; the
> evidence base is `docs/harness-analysis.md` and `docs/*.md`.

## Mission: independent exploitation harness

Build a **self-hosted harness — our own MCP server and our own tools** — for
LLM-assisted vulnerability exploitation against authorized targets.

- **Independent by default**: the harness must run without Yakit or any other
  external product runtime. Yakit is an architecture/tool-naming reference only.
- **Own implementation**: the tool registry, MCP server/adapter, `ExecutionBroker`,
  `PolicyEngine`, `ScopeGuard`, `Verifier`, and `AuditLog` are implemented in this
  workspace, not borrowed from Yakit.
- **Open-source references allowed**: existing open-source tools and MCP servers
  (nmap, nuclei, ffuf, sqlmap, binwalk, radare2, IDA/Ghidra/Frida MCP, etc.) may
  be referenced or wrapped, but each must be registered as our own `ToolSpec`,
  pass the same ScopeGuard/PolicyEngine, and be version/hash pinned and audited.

## What this repository is

- A **documentation / analysis workspace**, not the Yakit product source. There
  is no `package.json`, build, lint, test, or CI configuration here. Do not
  invent product commands or assume a toolchain.
- Eight Markdown documents cover the upstream snapshot and the harness build:
  - `docs/harness-analysis.md` — harness layers (agent, build, verification,
    git, release, runtime).
  - `docs/formal-design-agentic-orchestrator.md` — binding design for an
    LLM-assisted exploitation orchestrator (ADRs, permission matrix, threat
    model, roadmap).
  - `docs/refs/opencode-sdk-yakit-tool-bridge-analysis.md` — OpenCode SDK ↔ Yakit
    tool/MCP bridge.
  - `docs/refs/opencode-mcp-re-tool-catalog.md` — curated external MCP RE/security
    servers.
  - `docs/development-guide.md` — build guide: stack, layout, contracts, config.
  - `docs/roadmap.md` — phases, deliverables, gates, progress tracking.
  - `docs/tool-authoring.md` — `ToolSpec`, registry, open-source wrapping.
  - `docs/testing-and-verification.md` — test levels and evidence rules.

- **Yakit is a reference only** (architecture, tool naming, MCP/SDK patterns).
  This workspace builds its **own MCP server and tools**; do not add a Yakit
  runtime dependency or call a Yakit server. Open-source tools may be wrapped,
  but the harness stays self-hosted and independent.

## Snapshot and path discipline

- Product facts are static analysis of **Yakit `master` @ `f813c64de`,
  package `1.4.9-1113`**, dated 2026-11-17. Cite the snapshot when stating them.
- Paths such as `app/main/...`, `app/renderer/...`, `cli/...`, and
  `app/protos/grpc.proto` refer to that upstream checkout. **They are not in
  this directory.** Locate the upstream checkout before reading or editing them.
- `docs/harness-analysis.md` cites an upstream `AGENTS.md`,
  `.agents/skills/`, `.github/`, and `cli/` — those belong to the Yakit repo,
  not this workspace. This file is workspace-level and distinct.
- Preserve each doc's status labels (e.g. "no code changes yet") and keep
  verified findings separate from recommendations or speculation.

## Product facts that are easy to get wrong

- The renderer never executes scan/fuzz/AI logic. Electron main + a gRPC client
  call the external **Yaklang engine** (`yak`/`yak.exe`); this TS side holds
  IPC, types, and tool catalogs only. The AI ReAct loop lives in the engine.
- `app/protos/grpc.proto` (reference only) defines one `Yak` service with
  **670 RPCs**, mostly streaming. Its `StartMcpServerResponse` exposes
  `ServerUrl` (SSE) and `StreamableHttpUrl` (Streamable HTTP) — the pattern to
  follow for your own MCP server.
- Six editions share one codebase via `YAKIT_EDITION` (`yakit`, `yakitEE`,
  `yakitSE`, `irify`, `irifyEE`, `memfit`); engine ports and DB names differ.
- Yakit `ReviewPolicy` is `manual` | `ai` | `yolo`. **Direct MCP tool calls
  bypass ReviewPolicy**, so map it explicitly to OpenCode `permissions`.
- Electron 27 runs Node 18.17.1 while `@opencode/sdk` is ESM/Effect and needs
  Node 22+. Do **not** embed the SDK in Electron main; run OpenCode as a
  separate MCP client process.
- Reference only: Yakit MCP binds `127.0.0.1:11432` (remote `0.0.0.0`). Your own
  server must default to loopback and never be exposed without authentication.
  Port `11432` is reused as the default for our MCP server (see
  `docs/development-guide.md` §4.1); change it if Yakit/other services use the
  same port.

## Orchestrator design is binding

`docs/formal-design-agentic-orchestrator.md` is the source of truth for the
agentic orchestrator. Honor at least:

- ADR-001..006: in-process SDK on Node 22+, MCP-first, TS `PolicyEngine`,
  Web/CLI-first, `yolo` lab-only, single `ExecutionBroker`.
- Default-deny `ScopeGuard` before any action; empty or expired scope denies all.
- Permission matrix §6: `post` and `shell` are never auto-allowed, and `yolo`
  cannot enable them; production default is `manual`.
- Evidence before claims: a finding is valid only with `ToolResult.evidence`.
- `NetworkDebug` defaults to redacted; the audit log is append-only and
  hash-chained.

## Build your own MCP server and tools

Yakit is a reference only — do not integrate with or depend on its server/engine.

1. Implement local TypeScript tools under `tools/local-tools/*` and register
   them via `tools/registry.ts` (`ToolSpec`, Zod `params`, `schemaHash`).
2. Expose them through your own MCP server (`tools/mcp-adapter.ts`); follow the
   reference pattern: `StreamableHttpUrl` for OpenCode remote MCP, `ServerUrl`
   only for legacy SSE.
3. Wrap every tool with `ScopeGuard → PolicyEngine → ExecutionBroker` via
   `ctx.tool.transform` (`sdk/plugins.ts`).
4. Run the server, verify discovery as `<server>_<tool>`, then connect OpenCode:
   `opencode mcp add <name> --url <streamable-http-url>`.
5. Map `ReviewPolicy` to OpenCode `permissions`; `resource: "*"` cannot enforce
   per-target authorization. Keep the server on loopback unless authenticated.
6. Wrap open-source tools/MCP servers (see `docs/refs/opencode-mcp-re-tool-catalog.md`)
   behind the same `ToolSpec` + adapter: check license, pin version/hash, scan for
   prompt-injection/supply-chain risk, and record provenance in the audit log.
   Prefer wrapping a maintained open-source tool over reinventing it — but the
   wrapping layer, registry entry, and guardrail stay ours.

## Skills and agents

Focused on building **LLM-assisted vulnerability exploitation** (see
`docs/formal-design-agentic-orchestrator.md`).

Project skills in `.opencode/skills/` (load via the `skill` tool or `@id`):

- `exploitation-flow` — mission flow `recon → scan → fuzz → validate`, policy gates, evidence.
- `tool-catalog` — own `ToolSpec` registry, capability/risk table, schema pinning; Yakit only a naming reference.
- `security-guardrails` — ScopeGuard, PolicyEngine, ApprovalGateway, permission matrix, threat model.
- `finding-verification` — Verifier: findings need real `ToolResult.evidence`.
- `audit-redaction` — append-only hash-chain audit log and secret redaction.
- `mcp-and-tools` — build your own MCP server + local TypeScript tools.
- `orchestrator-design` — binding ADR / permission-matrix checklist.

Project agents in `.opencode/agents/`:

- `mission-planner` (subagent, read-only) — mission planning and capability selection.
- `tool-operator` (subagent) — executes tools via `ExecutionBroker` under policy.
- `finding-verifier` (subagent, read-only) — validates findings against evidence.
- `security-reviewer` (subagent, read-only) — threat-model / injection / policy review.
- `mcp-tools-builder` (subagent) — implements the MCP server, registry, and local tools.
- `orchestrator-designer` (`all`) — edits `docs/**` only.
- `orchestrator-implementor` (primary) — builds the orchestrator and its own MCP server/tools.

## Conventions

- New prose stays in English, matching the existing docs. Keep date, status,
  snapshot, and cross-reference headers accurate.
- No test suite exists here. Verify doc changes by cross-checking the cited
  source docs and the upstream snapshot; do not add a test framework for
  one-off checks.
- Ethical boundary: all work targets owned or explicitly authorized systems
  only.
