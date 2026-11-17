---
name: Audit and Redaction
description: Write the append-only hash-chain AuditLog and redact secrets before export
---

## Audit entry

`seq`, `at`, `actor` (`agent` | `operator` | `admin`), `event` (`planned` | `allowed` | `asked` | `approved` | `rejected` | `denied` | `executed`), `action`, `decision`, `approver`, `evidenceHash`, `prevHash`, `hash = sha256(seq|at|event|payload|prevHash)`.

## Rules

- Append-only; no edit/delete API.
- The hash chain is verified at startup and at export.
- Store `evidenceHash`, not raw evidence when it is sensitive.

## Redaction

- `NetworkDebug` defaults to **redacted**, never exported.
- Redact API keys, tokens, passwords, and sensitive prompts from logs/exports.
- Remind the operator before sharing a diagnostic package.
