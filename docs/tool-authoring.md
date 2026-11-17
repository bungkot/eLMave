# Tool Authoring — ToolSpec, Registry, and Open-Source Wrapping

- Date: 2026-11-17
- Status: implementation guide
- Basis: `docs/formal-design-agentic-orchestrator.md` §5

---

## 1. `ToolSpec` Contract

Every tool must satisfy:

| Field | Required | Meaning |
| --- | --- | --- |
| `name` | yes | normalization of `<server>_<tool>` (e.g. `lab_port_scan`) |
| `description` | yes | sanitized, length-limited, contains no hidden instructions |
| `params` | yes | Zod schema, validated server-side |
| `capability` | yes | `utility \| recon \| scan \| fuzz \| validate \| post \| filesystem \| shell` |
| `risk` | yes | `low \| medium \| high \| critical` |
| `sideEffects` | yes | whether it changes target/local state |
| `yoloAllowed` | yes | whether it may run in `yolo` mode |
| `timeoutMs` | yes | execution limit |
| `schemaHash` | yes | schema hash for tool-poisoning detection |
| `source` | yes | `mcp` \| `local` |
| `alwaysAsk` | optional | requires approval even at low risk |

---

## 2. Capability table (design §5.3)

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

\* `yolo` is for an isolated lab only; on your own server it remains `ask`.

---

## 3. Example tool

```ts
import { z } from 'zod'
import { defineTool } from '../tool-spec'

export const portScanTool = defineTool({
  name: 'lab_port_scan',
  description: 'Scans ports on an in-scope target. Read-only, light side effects.',
  params: z.object({
    target: z.string().min(1),
    ports: z.array(z.number().int().min(1).max(65535)).max(1000).default([80, 443]),
    timeoutMs: z.number().int().min(100).max(60_000).default(5_000),
  }),
  capability: 'scan',
  risk: 'medium',
  sideEffects: true,
  yoloAllowed: true,
  timeoutMs: 60_000,
  async execute(params, ctx) {
    ctx.scope.assertInScope(params.target)      // hard deny when out of scope
    const result = await scanPorts(params)     // internal / open-source adapter
    return {
      observation: result.summary,             // for the LLM, already sanitized
      evidence: {
        hash: result.hash,
        at: new Date().toISOString(),
        artifactRef: result.ref,
      },
    }
  },
})
```

`defineTool` wraps `ToolSpec` (metadata) + `execute`; `ToolSpec` itself does not
contain `execute` (design §5.1). A `scan` tool must honor
`EngagementScope.allowedPorts` when present.

Registration:

```ts
// tools/registry.ts
import { registry } from './registry'
import { portScanTool } from './local-tools/port-scan'

registry.register(portScanTool)
```

---

## 4. Contract rules

1. A tool without `schemaHash` must not be registered. `schemaHash = sha256(canonical JSON { description, params, capability, risk })`.
2. Changes to `description`/`params` during catalog refresh (`ForceSync` = forced sync of the MCP catalog) → the tool is quarantined.
3. `post` and `shell` are never automatically allowed.
4. Every execution produces `ToolResult.evidence`; without it a finding is invalid. The MCP adapter must wrap external results as `evidence`; raw output must not be thrown at the model.
5. `execute` must not throw raw data at the model; return a sanitized `observation`.

---

## 5. Wrapping open-source tools

Open-source tools/MCP may be referenced (nmap, nuclei, ffuf, sqlmap, binwalk,
radare2, IDA/Ghidra/Frida MCP, etc.) through
`docs/refs/opencode-mcp-re-tool-catalog.md`.

Wrapping steps:

1. Create an adapter `tools/adapters/<tool>.ts` that runs the pinned-version binary/MCP.
2. Check the **license** (e.g. GPL/AGPL) and the usage decision.
3. Pin the **version + hash**; never use `latest`.
4. Run it in a non-root sandbox, with **egress deny** and a timeout.
5. Sanitize the output; do not forward raw stderr/stdout to the model; wrap it as `ToolResult.evidence`.
6. Record the **provenance** (source, version, hash, license) in the AuditLog.
7. Register it as its own `ToolSpec` and pass it through the same guardrail.
8. MCP server instructions/resources are filtered by the `Sanitizer` (T02/T10).

Principle: it is better to wrap a maintained open-source tool than to rewrite it,
but the wrapping layer, registry, and policy remain owned by this repo.

---

## 6. References

- `docs/formal-design-agentic-orchestrator.md` §5 (contract & capability catalog)
- `docs/refs/opencode-mcp-re-tool-catalog.md` (open-source catalog)
- `docs/testing-and-verification.md` (tool testing)
