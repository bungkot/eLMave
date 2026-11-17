---
name: Orchestrator Design
description: Change or review the LLM-assisted exploitation orchestrator design according to the binding ADRs and permission matrix
---

Use this skill when writing or changing `docs/formal-design-agentic-orchestrator.md` and when implementing the orchestrator.

## Binding contract

- ADR-001..006: in-process SDK on Node 22+, MCP-first, TypeScript `PolicyEngine`, Web/CLI-first, `yolo` lab-only, a single `ExecutionBroker`.
- `ScopeGuard` default-deny before any action; empty/expired scope → everything denied.
- Permission matrix §6: `post` and `shell` are never auto-allow; `yolo` cannot enable them; `manual` by default.
- `Verifier`: a finding is valid only with `ToolResult.evidence`.
- `NetworkDebug` defaults to redacted; `AuditLog` is append-only hash-chain.

## Design change checklist

1. Did an ADR change? Record the revision + rationale.
2. Is the threat model affected? Update the related IDs.
3. Did the permission matrix change? Update the table + escalation rules.
4. A new tool must have `schemaHash`, `capability`, `risk`, `sideEffects`, `yoloAllowed`.
5. Execution still goes through `ExecutionBroker` (single execution gate).
6. Cross-references to `docs/harness-analysis.md` and `AGENTS.md` are still valid.

## Non-goals

- Not a replacement for the Yakit AI engine.
- Not "LLM writes exploit from scratch".
- Not embedding `@opencode/sdk` in Electron 27.
- Not using the Yakit server/engine as a dependency; Yakit is only an architecture reference.
- Not attacking systems without authorization.
