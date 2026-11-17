---
name: MCP and Own Tools
description: Build your own MCP server and TypeScript tools for the exploitation orchestrator (not a Yakit integration)
---

Use when creating/developing the MCP server and the orchestrator's tool registry. Yakit is only an architecture/tool-name reference, not a dependency.

## Principles

- ADR-002: MCP-first + local TS tools.
- All tools go through `ToolRegistry` and `ExecutionBroker`; no direct calls.
- OpenCode MCP tools are normalized `<server>_<tool>`.
- The agent host is the OpenCode SDK in-process; `opencode mcp add` is only for the OpenCode client (CLI/TUI) when verifying discovery.

## Modules (docs §11)

- `sdk/host.ts` — `OpenCode.create()` + session.
- `sdk/plugins.ts` — `ctx.tool.transform` (wraps ScopeGuard → PolicyEngine → ExecutionBroker) and `ctx.mcp.transform`.
- `sdk/permissions.ts` — deny by default; `resource: "*"` is coarse.
- `tools/registry.ts` + `tools/tool-spec.ts` — tool list, Zod validation, `schemaHash`.
- `tools/mcp-adapter.ts` — MCP adapter (stdio / Streamable HTTP).
- `tools/local-tools/*` — recon/scan/fuzz/validate/utility/filesystem.

## MCP transport

- Stdio for local/container tools.
- Streamable HTTP for OpenCode remote client: absolute URL, `oauth: false` when there is no auth.
- Default loopback; do not expose to the network without authentication.
- MCP server instructions/resources also enter the system prompt; filter/sanitize (T02/T10).

## Flow

1. Define the `ToolSpec` (`capability`, `risk`, `sideEffects`, `yoloAllowed`, `timeoutMs`, `schemaHash`).
2. Validate `params` with Zod server-side.
3. Register it in the registry; wrap it with guardrails.
4. Start the server; test discovery `<server>_<tool>`.
5. Connect OpenCode: `opencode mcp add <name> --url <streamable-http-url>`.
6. Test one read-only tool; then allow offensive tools via permissions.

## Open-source tools (reference, wrapped)

- Open-source tools/MCP may be referenced (nmap, nuclei, ffuf, sqlmap, binwalk,
  radare2, IDA/Ghidra/Frida MCP, etc.) via `docs/refs/opencode-mcp-re-tool-catalog.md`.
- Wrap them as our own `ToolSpec`: check the license, pin version/hash, scan for
  prompt-injection/supply-chain risk, record provenance in AuditLog.
- The wrapping layer, registry, and guardrails remain those of this harness.

## References (not dependencies)

- `docs/refs/opencode-sdk-yakit-tool-bridge-analysis.md` — MCP/SDK patterns.
- `docs/refs/opencode-mcp-re-tool-catalog.md` — external tool catalog reference.
- `docs/formal-design-agentic-orchestrator.md` §4–§5.
