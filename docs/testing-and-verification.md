# Testing and Verification

- Date: 2026-11-17
- Status: implementation guide
- Principle: **evidence before claims** — no false greens.

---

## 1. Test levels

| Level | Coverage | Example |
| --- | --- | --- |
| Unit | Zod schemas, handlers, policy, scope | params validation; matrix §6; `ScopeGuard` |
| Integration | registry + MCP server + adapter | registered tool, `lab_<tool>` discovery, mock adapter |
| Negative | rejections that must occur | out-of-scope → deny; expired scope → deny |
| E2E | OpenCode session with MCP | read-only first, then tools with approval |
| Performance/DoS | timeout, quota, slow consumer | slow tool stopped; queue bounded |

---

## 2. Mandatory negative tests

| Scenario | Expectation |
| --- | --- |
| Target outside `authorizedTargets` | `deny` (T01) |
| Expired / empty scope | `deny` all actions |
| `post` in `ai`/`yolo` mode | `deny` |
| `shell` in `ai`/`yolo` mode | `deny` |
| `post`/`shell` tool with `yolo` | never auto-allow |
| Tool schema changed | quarantine → `deny` until review |
| `DisableToolUse = true` | all tools `deny` |
| Observation contains instructions | treated as data, not commands (T02) |
| MCP server instructions/resources contain commands | filtered/sanitized, do not change actions (T02/T10) |
| Dangerous tool description | sanitized/quarantined (T03) |
| Egress to an internal host | denied (T08) |
| Finding without `evidence` | denied (T09) |

---

## 3. Evidence

Every test that declares a "finding" must include `ToolResult.evidence`
(`hash`, `at`, `artifactRef`). Logs or an open port alone are **not** evidence;
a reproducible response/execution result is required.

Anti-handwaving rules:

- Do not mark ✅ / "valid" / "verified" without real output.
- Do not conclude from logs or an open port alone.
- Redact credentials in evidence that is displayed/exported.

---

## 4. Commands

```bash
pnpm typecheck
pnpm test            # unit + integration + negative
pnpm test:e2e        # requires MCP server + OpenCode
```

The `pnpm` spelling is an example; adapt to the project's package manager.

---

## 5. Threat model → test matrix

| ID | Test |
| --- | --- |
| T01 | out-of-scope target denied |
| T02 | prompt injection in observation/MCP instructions does not change actions |
| T03 | change to `description`/`params` → quarantine |
| T04 | destructive action without approval denied |
| T05 | no credentials in logs/export |
| T06 | audit chain verified at startup/export |
| T07 | non-root sandbox + resource limits |
| T08 | default-deny egress |
| T09 | finding without `evidence` denied |
| T10 | open-source tool pinned + verified |
| T11 | memory/knowledge base isolated per engagement |
| T12 | SDK host on Node 22+, dependencies pinned |
| T13 | timeout + quota per operator |

---

## 6. Fixtures

- `fixtures/scope.json` — example `EngagementScope` (targets, expiry, approver); runtime uses `config/scope.json` (`ELM_SCOPE_FILE`).
- `fixtures/tool-catalog.json` — example tool catalog for discovery.
- `fixtures/mcp-stub` — MCP server stub for integration/E2E.

---

## 7. CI gate

`typecheck` → `test` → `test:e2e` → audit chain verification. A failure at any
stage blocks merge. Offensive tools are not allowed in CI jobs; use mocks.
