---
name: Security Guardrails
description: Apply ScopeGuard, PolicyEngine, ApprovalGateway, the permission matrix, and the threat model to agent actions
---

Use before allowing any action on a target.

## ScopeGuard (hard deny)

- The target must be in `authorizedTargets`; wildcard/regex requires explicit approval.
- Empty/expired scope → deny everything.
- Scope changes are recorded in AuditLog with `approvedBy`.

## PolicyEngine — matrix §6.1

| Capability | risk | `manual` | `ai` | `yolo` (lab) |
| --- | --- | --- | --- | --- |
| `utility` | low | allow | allow | allow |
| `recon` | low | allow | allow | allow |
| `scan` | medium | **ask** | allow | allow |
| `fuzz` | high | **ask** | **ask** | allow |
| `validate` | critical | **ask** | **ask** | allow (lab) |
| `post` | critical | **ask** | **deny** | **deny** |
| `filesystem` (read) | low | allow (scoped) | allow (scoped) | allow (scoped) |
| `filesystem` (write) | medium | **ask** | **ask** | allow (scoped) |
| `shell` | critical | **ask** | **deny** | **deny** |

## Escalation §6.3

- `risk >= AIReviewRiskControlScore` → `allow` escalates to `ask` (except `yolo`).
- `sideEffects = true` and `reviewPolicy = manual` → `ask`.
- New target for the first time → `ask` + scope verification.
- Tool schema changed → quarantine → `deny` until reviewed.
- `tool.alwaysAsk = true` → `ask` even at low risk.
- `tool.yoloAllowed = false` and `reviewPolicy = yolo` → `deny`.
- `DisableToolUse = true` → all tools `deny` (pure chatbot).

## ApprovalGateway

- `ask` waits for human/AI reviewer confirmation.
- `post`/`shell` are never auto-allow: `ask` under `manual`, `deny` under `ai`/`yolo`.
- `yolo` is for isolated labs only; it cannot enable `post`/`shell`.

## Threat model (T01–T13)

T01 target out of scope, T02 prompt injection (including MCP instructions/resources), T03 tool poisoning, T04 destructive action without approval, T05 credential leakage, T09 fake finding, T10 MCP/skill supply chain, T13 DoS. Control with Sanitizer, schema pinning, egress deny, timeout/quota, and audit.
