---
name: Tool Catalog
description: Define and register tools into ToolRegistry according to ToolSpec (capability, risk, schemaHash, yoloAllowed)
---

Use when adding/changing tools, MCP adapters, or open-source tool catalogs.

## ToolSpec contract

Every tool must have:

- `name` — `<server>_<tool>` normalization (e.g. `lab_port_scan`).
- `description` — sanitized, length-limited.
- `params` — Zod schema, validated server-side.
- `capability` — `utility | recon | scan | fuzz | validate | post | filesystem | shell`.
- `risk` — `low | medium | high | critical`.
- `sideEffects` — whether it changes target/local state.
- `yoloAllowed` — whether it is allowed in `yolo` mode.
- `timeoutMs`, `alwaysAsk?`, `schemaHash`, `source` (`mcp` | `local`).

## Capability table (docs §5.3)

| capability | risk | sideEffects | yoloAllowed |
| --- | --- | --- | --- |
| `utility` | low | no | yes |
| `recon` | low | no | yes |
| `scan` | medium | yes | yes |
| `fuzz` | high | yes | yes* |
| `validate` | critical | yes | lab only |
| `post` | critical | yes | **no** |
| `filesystem` | low–medium | partly | scoped |
| `shell` | critical | yes | **no** |

\* `yolo` is for isolated labs only; on your own server it stays `ask`.

## Rules

1. A tool without `schemaHash` must not be registered.
2. `description`/`params` changes between `ForceSync` → tool is quarantined.
3. `post` and `shell` are never automatically allowed.
4. Every execution produces `ToolResult.evidence`.

## Naming reference (not a dependency)

- Yakit is only a tool name/category reference: `port_scan`, `httpflow`, `ssa`,
  `yakscript`, `decompile`, `disasm`, `protocol`, `packet`, `binwalk`, `fs`.
- Orchestrator tools are defined by us in `tools/local-tools/*` and/or our own MCP
  server.
- Open-source tools/MCP (nmap, nuclei, ffuf, sqlmap, binwalk, radare2,
  IDA/Ghidra/Frida) may be referenced/wrapped, but become our own `ToolSpec`.
- Do not call the Yakit server/engine; do not add a Yakit runtime dependency.
