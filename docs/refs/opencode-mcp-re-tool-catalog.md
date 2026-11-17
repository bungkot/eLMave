# Catalog of Reverse Engineering & Security MCP Servers for OpenCode SDK

- Date: 2026-11-17
- Status: market research results (GitHub/internet), no code changes yet
- Scope: a collection of external MCP servers equivalent to or complementing Yakit tools (`port_scan`, `httpflow`, `ssa`, `yakscript`, `decompile`, `disasm`, `IDA`, `Ghidra`, `binwalk`, `protocol`, `packet`) and how to use them with the OpenCode SDK
- Source: GitHub Search API, accessed 2026-11-17. Star counts are indicative and change over time.

## Conclusion

1. **No single MCP server matches all Yakit tools**, but a combination of several professional servers can cover the same areas and even more deeply.
2. **Most RE MCPs are `stdio`-based**, so in OpenCode they are configured as `type: local` with `command`. Servers providing Streamable HTTP (e.g. the Yakit MCP server) are configured as `type: remote`.
3. **Recommended core stack**: IDA (`mrexodia/ida-pro-mcp`), Ghidra (`bethington/ghidra-mcp`), Binary Ninja (`mrphrazer/binary-ninja-headless-mcp`), Frida (`dnakov/frida-mcp`), pentest/network (`0x4m4/hexstrike-ai`), protocol/web (`Mouseww/anything-analyzer`), firmware (`c0decave/axross-mcp-hub`).
4. **Don't forget agent security**: offensive tools must be restricted via OpenCode `permissions`; scan skills/MCPs before use (e.g. `NVIDIA/SkillSpector`).

## 1. Relevant OpenCode Configuration

OpenCode uses the `mcp.servers` schema. Tool names are normalized to `<server>_<tool>` (e.g. `ida_decompile`, `hexstrike_nmap`).

- **stdio server** (majority of RE MCPs) → `type: local`:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "servers": {
      "ida": {
        "type": "local",
        "command": ["uvx", "ida-pro-mcp"],
        "codemode": false
      },
      "ghidra": {
        "type": "local",
        "command": ["java", "-jar", "ghidra-mcp.jar"],
        "codemode": false
      },
      "frida": {
        "type": "local",
        "command": ["uvx", "frida-mcp"],
        "codemode": false
      },
      "hexstrike": {
        "type": "local",
        "command": ["uvx", "hexstrike-ai"],
        "codemode": false
      }
    }
  },
  "permissions": [
    { "action": "hexstrike_*", "resource": "*", "effect": "deny" }
  ]
}
```

- **Streamable HTTP server** (e.g. the Yakit MCP server) → `type: remote`:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
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

Notes:

- `codemode: false` exposes tools directly to the model; for servers with hundreds of tools, `codemode: true` (default) is more token-efficient.
- `protocol` can be `legacy` (default), `auto`, or `2026-07-28`.
- Via CLI: `opencode mcp add <nama> -- <command>` for local servers, `opencode mcp add <nama> --url <url>` for remote servers.
- In the OpenCode SDK, dynamic registration can use `ctx.mcp.transform((editor) => editor.set(...))` (see `docs/refs/opencode-sdk-yakit-tool-bridge-analysis.md`).

## 2. Reverse engineering / disassembler / decompiler

| MCP Server | Language | License | Notes (mapping to Yakit) |
| --- | --- | --- | --- |
| [mrexodia/ida-pro-mcp](https://github.com/mrexodia/ida-pro-mcp) | Python | MIT | **IDA Pro**, ~13.5k⭐. De facto standard: 100+ tools (decompile, disasm, rename, xref, type). Equivalent of `IDA` / `decompile` / `disasm`. |
| [LaurieWired/GhidraMCP](https://github.com/LaurieWired/GhidraMCP) | Java | Apache-2.0 | **Ghidra**, ~12k⭐. GUI plugin + bridge, complete tools, large community. |
| [bethington/ghidra-mcp](https://github.com/bethington/ghidra-mcp) | Java + Python | Apache-2.0 | **Ghidra**, ~6.2k⭐. 200+ tools, GUI plugin + **headless server** + Docker + Ghidra Server; suitable for automation/CI. |
| [blacktop/ida-mcp-rs](https://github.com/blacktop/ida-mcp-rs) | Rust | MIT | **IDA headless (idalib)**, ~906⭐. Without GUI, good for automated pipelines. |
| [fosdickio/binary_ninja_mcp](https://github.com/fosdickio/binary_ninja_mcp) | Python | GPL-3.0 | **Binary Ninja** plugin, ~460⭐. |
| [mrphrazer/binary-ninja-headless-mcp](https://github.com/mrphrazer/binary-ninja-headless-mcp) | Python | GPL-2.0 | **Binary Ninja headless**, ~264⭐, 180 tools. |
| [Invoke-RE/binja-lattice-mcp](https://github.com/Invoke-RE/binja-lattice-mcp) | Python | MIT | Binary Ninja + remote DB/MCP, ~67⭐. |
| [symgraph/GhidrAssistMCP](https://github.com/symgraph/GhidrAssistMCP) | Java | MIT | Ghidra native extension, ~813⭐. |
| [LING71671/open-reverselab](https://github.com/LING71671/open-reverselab) | Python | Apache-2.0 | RE workspace (desktop app) + MCP for **Ghidra, Frida, x64dbg, Rizin**, ~1.4k⭐. |
| [oakkaya/edb-debugger-mcp](https://github.com/oakkaya/edb-debugger-mcp) | Python | MIT | EDB/GDB/pwntools, 147 debugging tools. |

## 3. Dynamic instrumentation / mobile

| MCP Server | Language | License | Notes |
| --- | --- | --- | --- |
| [dnakov/frida-mcp](https://github.com/dnakov/frida-mcp) | Python | MIT | Original stdio Frida MCP, ~468⭐. |
| [firerpa/lamda](https://github.com/firerpa/lamda) | Python | MIT | Android full-stack: Frida, MITM, 160+ API, MCP, ~8.7k⭐. Very powerful. |
| [FuzzySecurity/kahlo-mcp](https://github.com/FuzzySecurity/kahlo-mcp) | TypeScript | — | Frida MCP for Android instrumentation. |
| [index-login/MobileRE-Skill](https://github.com/index-login/MobileRE-Skill) | JS | MIT | Mobile RE skill + MCP: Frida hooks, unpacking, DEX dump, Ghidra MCP. |
| [ZSA233/frida-analykit](https://github.com/ZSA233/frida-analykit) | Python | MIT | Frida toolkit + MCP. |

## 4. Network / pentest (equivalent to `port_scan`, `httpflow`, `packet`)

| MCP Server | Language | License | Notes |
| --- | --- | --- | --- |
| [0x4m4/hexstrike-ai](https://github.com/0x4m4/hexstrike-ai) | Python | MIT | ~13.7k⭐, 150+ tools (nmap, nuclei, ffuf, sqlmap, dsu). **Closest to Yakit's tool breadth.** |
| [cyproxio/mcp-for-security](https://github.com/cyproxio/mcp-for-security) | TypeScript | MIT | Collection of SQLMap, FFUF, Nmap, Masscan. (archived) |
| [f1tz/mcp-for-security-python](https://github.com/f1tz/mcp-for-security-python) | Python | MIT | Python version of the collection above. |
| [DMontgomery40/pentest-mcp](https://github.com/DMontgomery40/pentest-mcp) | TypeScript | AGPL-3.0 | stdio/HTTP/SSE, nmap/gobuster/nikto/John/hashcat. |
| [hevnsnt/CVEngine](https://github.com/hevnsnt/CVEngine) | Python | GPL-3.0 | Attack surface + nmap/nuclei/bbot, ~738⭐. |
| [rfunix/tengu](https://github.com/rfunix/tengu) | Python | MIT | nmap + Metasploit. |
| [ridgebackinfosec/cerno](https://github.com/ridgebackinfosec/cerno) | Python | MIT | Triage Nessus/nuclei + nmap/NetExec/Metasploit. |
| [hackersatyamrastogi/pentesting-cyber-mcp](https://github.com/hackersatyamrastogi/pentesting-cyber-mcp) | JS | MIT | Curation of 50+ MCP security servers. |

## 5. Protocol / web / JS reverse engineering (equivalent to `httpflow`, `protocol`, `packet`)

| MCP Server | Language | License | Notes |
| --- | --- | --- | --- |
| [Mouseww/anything-analyzer](https://github.com/Mouseww/anything-analyzer) | TypeScript | Apache-2.0 | Browser capture + MITM proxy + JS hooks + AI analysis + MCP, ~3.9k⭐. |
| [vmoranv/jshookmcp](https://github.com/vmoranv/jshookmcp) | TypeScript | AGPL-3.0 | 735 tools / 36 domains, search-first: JS deobfuscation, CDP, Frida, network analysis, ~2.1k⭐. |
| [hangwin/mcp-chrome](https://github.com/hangwin/mcp-chrome) | TypeScript | MIT | Chrome control via MCP, ~12.5k⭐ (more toward browser automation). |

## 6. Firmware / binary analysis (equivalent to `binwalk`)

| MCP Server | Language | License | Notes |
| --- | --- | --- | --- |
| [c0decave/axross-mcp-hub](https://github.com/c0decave/axross-mcp-hub) | Python | — | MCP container: **radare2, capa, binwalk, YARA, mcpcap, nmap, nuclei, sqlmap, dnstwist**. One hub, packed. |
| [n-3-0-l-d-3-v/ultron](https://github.com/n-3-0-l-d-3-v/ultron) | Python | Apache-2.0 | Evidence-first binary/firmware analysis, Ghidra headless + binwalk, SBOM. |
| [daedalus/mcp_reverse_engineering](https://github.com/daedalus/mcp_reverse_engineering) | Python | MIT | Lightweight wrapper: angr, binwalk, frida, gdb, ghidra, radare2, objdump, strings, upx. |
| [Cossack9989/binwalk-mcp-server](https://github.com/Cossack9989/binwalk-mcp-server) | Python | — | Small, focused on binwalk. |

## 7. AI-agent security (complementary)

- [NVIDIA/SkillSpector](https://github.com/NVIDIA/SkillSpector) — Python, Apache-2.0, ~26k⭐: scan skills/MCPs before use (prompt injection, data exfil).
- [qybaihe/bla](https://github.com/qybaihe/bla) — TypeScript, MIT, ~5.9k⭐: "blue defense" MCP to protect binaries from AI RE agents.
- [AiSOC](https://github.com/beenuar/AiSOC) — Python, MIT: MCP-based SOC/triage.

## 8. Curation list

- [reacher-z/Awesome-REA](https://github.com/reacher-z/Awesome-REA) — ~3k⭐, curation of RE tools/cases/papers + MCP.
- [morluto/awesome-reverse-engineering](https://github.com/morluto/awesome-reverse-engineering) — ~2.2k⭐.

## 9. Recommended core stack

| Area | Primary choice | Alternative |
| --- | --- | --- |
| IDA Pro | `mrexodia/ida-pro-mcp` | `blacktop/ida-mcp-rs` (headless) |
| Ghidra | `bethington/ghidra-mcp` (headless, 200+ tools) | `LaurieWired/GhidraMCP` |
| Binary Ninja | `mrphrazer/binary-ninja-headless-mcp` | `fosdickio/binary_ninja_mcp` |
| Frida / dynamic | `dnakov/frida-mcp` | `firerpa/lamda` (full Android) |
| Network / pentest | `0x4m4/hexstrike-ai` | `f1tz/mcp-for-security-python` |
| Protocol / web | `Mouseww/anything-analyzer` | `vmoranv/jshookmcp` (JS/web RE) |
| Firmware / binwalk | `c0decave/axross-mcp-hub` | `daedalus/mcp_reverse_engineering` |

## 10. Limitations and Risks

- **Catalog changes**: repositories can be archived, change licenses, or stop being maintained. Re-check before using in production.
- **Copyleft licenses**: some servers (Binary Ninja, jshookmcp) are licensed GPL/AGPL — note this if tool output will be used commercially.
- **Repos with unreasonable metrics**: there are repos with very high star/fork counts that appear inflated (e.g. `morluto/rea`, `affaan-m/ECC`); not recommended.
- **Attack surface**: offensive tools (nmap, sqlmap, hashcat, Frida) can be executed by the agent inadvertently. Restrict with `permissions` (`effect: deny`) and appropriate `codemode`.
- **Do not expose the MCP server to the internet** without authentication; the local default `127.0.0.1` is safer.
- **Bypassing Yakit's ReviewPolicy**: if OpenCode calls tools directly, Yakit's policy does not automatically apply; map it to OpenCode permissions.

## Related References

- `docs/refs/opencode-sdk-yakit-tool-bridge-analysis.md` — analysis of OpenCode SDK integration with Yakit tools/MCP.
- `docs/ipc-startup-acceptance-0911.md` — Electron/Node runtime reference.
- OpenCode MCP configuration: https://opencode.ai/docs/mcp-servers/
