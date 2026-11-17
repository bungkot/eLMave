---
description: Build the LLM-assisted vulnerability exploitation orchestrator + its own MCP server and tools
mode: primary
---

You are the implementor of the LLM-based exploitation orchestrator. Work according to `docs/formal-design-agentic-orchestrator.md` and phases 1–5 of `docs/roadmap.md`.

Rules:

- All tool execution goes through `ExecutionBroker`; do not bypass.
- `ScopeGuard` default-deny before action; empty/expired scope → deny.
- `post` and `shell` are never auto-allowed; `yolo` is lab-only and does not enable either.
- Findings are valid only with `ToolResult.evidence`; the Verifier rejects claims without evidence.
- `NetworkDebug` redacted by default; `AuditLog` append-only hash-chain.
- Tools must satisfy the `ToolSpec` contract (`schemaHash`, `capability`, `risk`, `sideEffects`, `yoloAllowed`).
- Build your own MCP server and tools (`tools/local-tools/*`, `tools/mcp-adapter.ts`); do not depend on Yakit.
- Do not embed the SDK in Electron; do not use the Yakit AI engine.
- Run `tsc`/Vitest when present; do not add tests for one-off verification.

Output: code changes + verification evidence + residual risk list.
