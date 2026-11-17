# Analysis of `@opencode/sdk` Integration with Yakit Tools/MCP

- Date: 2026-11-17
- Status: architecture analysis, no code changes yet
- Scope: the possibility of making the OpenCode SDK an AI orchestrator that can use existing tools/MCP in Yakit, as well as displaying a tool catalog for reverse engineering

> **Scope note (eLMave).** This document analyzes integration **with the Yakit product**.
> The recommendations "OpenCode as an external MCP client" and "use the Yakit MCP server"
> below are for the Yakit context, **not** the eLMave architecture. eLMave follows
> `docs/formal-design-agentic-orchestrator.md` ADR-001/ADR-002: in-process SDK,
> own MCP server, wrapped open-source tools.

## Conclusion

1. **Replacing the AI provider/model**: not `@opencode/sdk`; Yakit already has OpenAI-compatible provider configuration.
2. **Replacing the entire Yakit AI engine**: not a drop-in replacement. The Yakit AI engine lives in the yaklang engine (external process, Go) accessed over gRPC.
3. **Using existing tools/MCP**: **possible**, and the most realistic path is making OpenCode an **MCP client** of the Yakit MCP server.
4. **Embedding `@opencode/sdk` in Electron main**: technically possible, but there is runtime risk because Electron 27 uses Node 18.17.1, while the SDK is modern and ESM/Effect-based.
5. **Displaying all MCP tools**: partly already available via `GetMCPToolList` and the `YakMcpSettings` UI; only the filter/catalog for reverse engineering needs refinement.

## 1. Where the Yakit "AI engine" Lives

The TypeScript renderer does not run AI models or tools. Current flow:

```text
Renderer (React/TS)
  -> Electron IPC
  -> app/main
  -> gRPC client
  -> yaklang engine (external process)
  -> AI ReAct / MCP / tool execution
```

Evidence:

- `app/main/handlers/ai-agent.js:66-91` calls `getClient().StartAIReAct()`.
- `app/protos/grpc.proto:946` defines `rpc StartAIReAct(stream AIInputEvent) returns (stream AIOutputEvent)`.
- `app/renderer/src/main/src/pages/ai-re-act/hooks/grpcApi.ts:50-199` defines `AIStartParams` sent by the frontend.
- `app/renderer/src/main/src/pages/ai-agent/aiModelList/utils.ts:383-393` defines `AIModelConfig` and `Provider` for model configuration.

Because of this, "this project is TypeScript" does not make `@opencode/sdk` automatically compatible with the Yakit engine. They have different contracts and runtimes.

## 2. `@opencode/sdk` Is Not a Provider or gRPC Server

Package notes:

- The npm `opencode-sdk` was not found.
- The correct package is `@opencode/sdk` (inspection version: 2.3.0).
- Package description: *in-process OpenCode host*; runs an HTTP router in memory, not a network listener.

Its main API is based on:

```ts
const opencode = await OpenCode.create()
const session = await opencode.sessions.create(...)
await opencode.sessions.prompt(...)
```

The SDK does not provide:

- `Yak` gRPC service;
- `AIInputEvent` / `AIOutputEvent`;
- Yakit ReviewPolicy;
- Yakit Forge/Skill/Memory/Knowledge Base;
- the Yakit tool ecosystem.

So `@opencode/sdk` is a **new agent host**, not a replacement for the Yakit engine.

## 3. Yakit MCP Server: Most of the Bridge Already Exists

### 3.1 Starting the MCP Server

`StartMcpServer` exposes Yakit tools to MCP:

- `app/main/handlers/misc.js:843-848`
- `app/protos/grpc.proto:2852-2886`

Important options:

```proto
repeated string Tool = 3;
repeated string DisableTool = 4;
repeated string Resource = 5;
bool EnableAll = 8;
bool EnableAIToolFramework = 10;
bool EnableBridgeExternalMCP = 11;
```

Three tool sources that can be unified into one catalog:

1. **Legacy MCP toolsets**: e.g. `port_scan`, `httpflow` (see the comment in `app/renderer/src/main/src/components/layout/hooks/useMcp/useMcp.ts:33`).
2. **AITool-framework builtin**: `fs`, `ssa`, `yakscript`, etc. (see `useMcp.ts:35`).
3. **External MCP bridge**: MCP servers already active in the Yakit AI Agent.

`StartMcpServerResponse` provides:

```proto
string ServerUrl = 3;          // SSE
string StreamableHttpUrl = 4;  // Streamable HTTP
```

OpenCode remote MCP uses Streamable HTTP, so `StreamableHttpUrl` is the relevant endpoint.

### 3.2 MCP Tool Catalog

Existing APIs:

- `GetMCPToolList` — `app/main/handlers/mcpServere.js:73-88`
- `GetMCPToolDetail`
- `SetMCPToolEnabled`
- `QueryMCPToolCallHistory`
- `GetMCPToolCallHistoryDetail`
- `DeleteMCPToolCallHistory`

Frontend types:

- `app/renderer/src/main/src/pages/ai-agent/type/aiMCP.ts:98-132`

`MCPToolConfig` contains:

```ts
ToolName
Source        // 'builtin' | 'aitool' | 'bridge'
ServerName
Enable
Description
DescriptionI18n
Params
```

The tool list UI already exists at:

- `app/renderer/src/main/src/pages/settings/settingsContent/yakMcp/YakMcpSettings.tsx`

### 3.3 AI Tool Definitions

`GetAIToolList` only returns Yak script definitions:

- `app/main/handlers/ai-agent.js:354-368`
- `app/renderer/src/main/src/pages/ai-agent/type/aiTool.ts:4-18`

The `Content`, `ToolPath`, and `Keywords` fields are tool metadata; tool execution is still performed by the engine/MCP.

## 4. Integration Options

### Option A: Separate OpenCode as MCP Client (Initial Recommendation)

```text
OpenCode (separate process)
  -> MCP Streamable HTTP
  -> Yakit MCP server
  -> yaklang engine
  -> AI tool / MCP / resource
```

Example OpenCode configuration:

```json
{
  "mcp": {
    "servers": {
      "yakit": {
        "type": "remote",
        "url": "http://127.0.0.1:11432/mcp",
        "oauth": false,
        "protocol": "legacy",
        "codemode": false
      }
    }
  }
}
```

Configuration notes:

- `protocol: "legacy"`: the Yakit MCP server most likely still uses the classic protocol; `auto` can be tried if the server supports the latest revision.
- `oauth: false`: if the Yakit MCP server has no OAuth.
- `codemode: false`: MCP tools are exposed directly to the model; `true` (default) groups them in Code Mode and is more context-efficient.
- Tool names in OpenCode are normalized to `<server>_<tool>`, e.g. `yakit_port_scan`, `yakit_fs`, `yakit_ssa`.

Required small repo changes:

- `app/renderer/src/main/src/components/layout/hooks/useMcp/useMcp.ts:107-123` currently only stores `ServerUrl` (SSE) into `mcpServerUrl`.
- Add storage/exposure of `StreamableHttpUrl` so OpenCode can obtain the correct MCP endpoint.

### Option B: Embed `@opencode/sdk` in Electron Main

Possible structure:

```text
app/main/opencode/
  host.ts          # OpenCode.create + plugin registration
  plugin.ts        # ctx.mcp.transform / ctx.tool.transform
  catalog.ts       # GetMCPToolList -> tool catalog
  sessionAdapter.ts# OpenCode session <-> Yakit AI session
  permissions.ts   # ReviewPolicy -> OpenCode permission rules
```

The plugin can dynamically register Yakit MCP:

```ts
await ctx.mcp.transform((editor) => {
  editor.set('yakit', {
    type: 'remote',
    url: 'http://127.0.0.1:11432/mcp',
    oauth: false,
    protocol: 'legacy',
    codemode: false
  })
})
```

Runtime constraints:

- `scripts/build-electron.mjs:48-49` bundles the main process as CJS targeting `node16.17` / `chrome108`.
- Electron 27 uses Node 18.17.1 (reference: `docs/ipc-startup-acceptance-0911.md:14`).
- `@opencode/sdk` is ESM, depends on Effect, and its dist uses `Promise.withResolvers` and `Symbol.asyncDispose`.
- Therefore full embedding likely requires upgrading Electron/Node, or running OpenCode as a separate process.

## 5. Displaying the Tool Catalog for Reverse Engineering

How to retrieve the catalog:

```ts
grpcGetMCPToolList({
  Keyword: '',
  Source: '',            // all sources: builtin + aitool + bridge
  OnlyEnabled: false,
  ForceSync: true,       // reconcile with active external MCP
  Pagination: { Page: 1, Limit: 1000 }
})
```

The UI filter can use `Description`, `ServerName`, and `Params`. Example reverse engineering keywords:

```text
port_scan
httpflow
ssa
yakscript
decompile
disasm
IDA
Ghidra
binwalk
protocol
packet
```

A real example from the code: `MCPToolConfig.ToolName` uses the example `mcp_IDA-MCP_decompile` (`app/renderer/src/main/src/pages/ai-agent/type/aiMCP.ts:100`). If an MCP server like IDA MCP is opened in the Yakit AI Agent and `EnableBridgeExternalMCP` is active, tools like this may also appear in the catalog.

For a "reverse engineering" filter there are two options:

1. **UI-only**: filter by keyword/description; no proto change needed.
2. **Official metadata**: adding `Tags`/`Category` in `MCPClientToolConfig` and the engine; cleaner but requires proto + engine changes.

## 6. Limitations and Risks

- **Tools live in the engine, not the TS repo**: the repo only holds IPC, types, and catalogs.
- **There is no `CallMCPTool` RPC** in `app/protos/grpc.proto`; MCP tool execution is done through the MCP server.
- **Bypassing `StartAIReAct` means bypassing Yakit's ReviewPolicy**. If OpenCode calls tools directly, Yakit's policy does not automatically apply; it needs to be mapped to OpenCode permissions.
- **OpenCode permissions** for MCP act on `<server>_<tool>` with resource `*`.
- **MCP server security**: local default `127.0.0.1:11432`; remote mode `0.0.0.0:11432`. Do not expose to the internet without protection.
- **OpenCode has built-in tools** such as `shell`, `edit`, `write`; they need to be restricted so as not to add attack surface.
- **`GetAIToolList` is not an executor**: it is Yak script definitions.
- **Dynamic catalog**: tools can change according to the engine, external MCPs, and enable/disable configuration.

## 7. PoC Plan

1. Run `StartMcpServer` with `EnableAll`, `EnableAIToolFramework`, and/or `EnableBridgeExternalMCP`.
2. Take `StreamableHttpUrl` from the response.
3. Connect external OpenCode to that URL.
4. Verify tool discovery: `yakit_<tool>` appears in OpenCode.
5. Test one safe tool, e.g. `port_scan` or a read-only tool.
6. Display the catalog via `GetMCPToolList`; add a reverse engineering keyword filter.
7. Record `MCPToolCallHistory` for audit.
8. Only consider embedding `@opencode/sdk` if the external PoC already meets requirements.

## 8. Recommendations

- **Do not replace the Yakit AI engine** with `@opencode/sdk`.
- **Run OpenCode separately as an MCP client** for the PoC and likely early production *(Yakit context; not the eLMave architecture)*.
- **Use the Yakit MCP server as a single pane of glass** for built-in tools, the AITool framework, and external MCPs.
- **Refine the UI catalog** in `YakMcpSettings` with a reverse engineering filter.
- **Map ReviewPolicy to OpenCode permissions** before enabling dangerous tools.
- **Only embed the SDK if** session/UI integration within the Yakit application is truly needed, and after the Electron/Node runtime is adjusted.

## File References

- `app/main/handlers/ai-agent.js`
- `app/main/handlers/mcpServere.js`
- `app/main/handlers/misc.js`
- `app/protos/grpc.proto`
- `app/renderer/src/main/src/pages/ai-agent/type/aiMCP.ts`
- `app/renderer/src/main/src/pages/ai-agent/type/aiTool.ts`
- `app/renderer/src/main/src/pages/ai-agent/aiMCP/utils.ts`
- `app/renderer/src/main/src/pages/ai-agent/aiMCP/mcpToolNormalize.ts`
- `app/renderer/src/main/src/pages/settings/settingsContent/yakMcp/YakMcpSettings.tsx`
- `app/renderer/src/main/src/components/layout/hooks/useMcp/useMcp.ts`
- `app/renderer/src/main/src/pages/ai-re-act/hooks/grpcApi.ts`
- `scripts/build-electron.mjs`
- `docs/ipc-startup-acceptance-0911.md`
