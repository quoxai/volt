# VOLT Privacy & Redaction (v0.1)

VOLT is an evidence protocol. Evidence is powerful — and dangerous — if it captures secrets, personal data, or sensitive operational content by accident.

This document defines **privacy-first logging rules** and **redaction strategies** for VOLT v0.1.

**Core principle:**
VOLT should provide integrity and accountability **without becoming a leak vector**.

---

## 1) Privacy goals

A VOLT implementation MUST aim to:

- Prevent secrets from entering events/attachments
- Minimize personal data (PII) exposure
- Avoid storing raw content unless explicitly required
- Make redactions obvious and auditable
- Support "exportable evidence" without exporting private information

---

## 2) Non-negotiables (normative)

### 2.1 Never log secrets (MUST NOT)
VOLT events and attachments MUST NOT contain:

- API keys, access tokens, session cookies
- passwords, password hashes
- private keys, seed phrases, recovery codes
- database connection strings containing credentials
- full authorization headers (e.g., `Authorization: Bearer ...`)
- raw secrets from environment variables

**Rule of thumb:** If it can grant access, it cannot be recorded.

### 2.2 Default to metadata + references (MUST)
- Events SHOULD contain metadata (tool name, operation, timing, status)
- Full outputs should be attachments only when safe and useful
- When in doubt: record a summary + content hash, not raw content

### 2.3 Explicit redaction flags (MUST)
When anything is omitted or sanitized, events MUST indicate it clearly via:
- `payload.redacted: true` and/or
- `payload.inputs_redacted: true` / `payload.outputs_redacted: true`

This prevents "silent censorship" and keeps audits honest.

---

## 3) Data classification (recommended)

Implementations SHOULD classify data into at least these categories:

- **PUBLIC**: safe to store and export (rare)
- **INTERNAL**: safe within org, not for public export
- **SENSITIVE**: requires strict redaction controls
- **SECRET**: must never be stored
- **PII**: personal data (may be regulated)

This classification can be enforced by AOCL policies, but VOLT must assume the worst and stay safe by default.

---

## 4) What VOLT SHOULD log (safe defaults)

### 4.1 AEE events
Log:
- envelope IDs
- correlation IDs
- channel (`web`, `api`, `cli`)
- size bytes
- intent tags (non-sensitive)

Avoid:
- full user message body unless explicitly approved and sanitized

Example safe payload:

```json
"payload": {
  "channel": "web",
  "size_bytes": 1842,
  "intent": "infrastructure_change",
  "redacted": true
}
```

### 4.2 AOCL policy events

Log:

- policy id
- decision id
- result (allow/deny/hitl_required)
- reason codes (not raw content)

Avoid:

- dumping full policy expressions if they contain sensitive matching patterns

Example:

```json
"payload": {
  "result": "hitl_required",
  "reason": "prod_write_operation"
}
```

### 4.3 Tool call events (the big one)

Log:

- tool name + operation
- target identifier (host alias, not raw IP if sensitive)
- status, duration
- whether inputs/outputs were redacted
- attachment references for safe outputs

Avoid:

- raw shell commands with secrets
- full file contents
- raw HTTP headers/cookies

Example:

```json
"payload": {
  "tool_name": "ssh",
  "operation": "exec",
  "target": "vm-prod-01",
  "inputs_redacted": true
}
```

---

## 5) What VOLT should NOT log (by default)

### 5.1 Shell commands

Do NOT log raw command strings unless sanitized.

Instead log:

- command class/category (`restart_service`, `write_file`, `run_migration`)
- safe target metadata
- hash of the command string if needed (not the command)

### 5.2 HTTP requests

Do NOT log:

- raw headers
- raw bodies
- cookies
- bearer tokens

Log:

- destination host (optional, may be redacted)
- method
- status code
- duration
- response size (optional)

### 5.3 File contents

Do NOT log file contents by default.

Log:

- path category (e.g., "nginx config", "app env file")
- operation (read/write)
- bytes written (optional)
- content hash (optional)
- approval linkage (AOCL/HITL)

### 5.4 User input and prompts

Do NOT log raw prompts by default.

Log:

- prompt length
- prompt classification tags
- model name/version
- token usage metadata
- hash of prompt text (optional) if you need later matching without storing content

---

## 6) Redaction strategies (v0.1)

VOLT v0.1 supports simple redaction by omission and marking.

### 6.1 Omit sensitive fields

Replace sensitive sections with:

- `"<redacted>"` placeholders OR
- boolean flags indicating omission

Example:

```json
"payload": {
  "inputs_redacted": true,
  "inputs_summary": "Write config file to prod"
}
```

### 6.2 Sanitized attachments

If outputs are useful but risky, sanitize them before attaching:

- remove secrets using regex detectors
- truncate long outputs
- replace IPs, emails, tokens with placeholders (configurable)

Then attach the sanitized output, not the raw.

### 6.3 Redaction log (optional)

If you redact, include `redactions/redactions.json`:

- which event ids were affected
- what fields were removed
- why (secret/pii/policy)

This makes redaction auditable without revealing the content.

---

## 7) Recommended "secret scanning" enforcement (strongly recommended)

Implementations SHOULD include a guard that runs before events/attachments are written:

### 7.1 Event payload scanner

- Detect token-like strings (JWT patterns, API key prefixes)
- Detect common credential keys (`password=`, `api_key`, `secret`)
- Detect PEM blocks (`-----BEGIN PRIVATE KEY-----`)
- Detect AWS keys (`AKIA...`)
- Detect common bearer/cookie patterns

If detected:

- strip the value
- set `payload.redacted: true`
- optionally raise an AOCL policy alert

### 7.2 Attachment scanner

Before writing any attachment:

- scan and sanitize
- if unsafe, either:
  - do not store attachment, or
  - store a truncated/sanitized version only

---

## 8) User data / PII guidance

If user data might appear (support tickets, emails, customer info):

- Prefer storing references/ids rather than raw content
- Avoid email addresses, phone numbers, addresses by default
- If unavoidable, store only the minimal subset and classify it `PII`
- Ensure export controls are strong (who can export evidence bundles)

---

## 9) Export safety rules (enterprise-friendly)

When exporting bundles outside the system (zip/share):

Export SHOULD:

- remove INTERNAL/SENSITIVE attachments unless explicitly included
- include a summary report of removed items
- include verification report
- preserve chain integrity (do not rewrite events; export is about packaging)

If something must be removed after the fact:

- export a "redacted bundle" and clearly label it
- never pretend it is the original full-fidelity record

---

## 10) Recommended AOCL policies for VOLT safety (informative)

AOCL is the enforcement brain. VOLT is the recorder.

Recommended AOCL policies:

- Require HITL for any tool action touching production
- Block tools that attempt to write to secret locations
- Require "sanitized outputs only" for network tools
- Prevent exporting evidence bundles containing PII without approval
- Require a "VOLT bundle verification PASS" before marking a run as complete (optional)

---

## 11) Minimal compliance checklist

A VOLT v0.1 integration should satisfy:

- [ ] No secrets in events or attachments (scanned + sanitized)
- [ ] Redactions are explicit (flags and/or redactions.json)
- [ ] Tool inputs are never stored raw by default
- [ ] HTTP bodies/headers are never stored raw by default
- [ ] File contents are never stored raw by default
- [ ] Exports are controlled and can be "export-safe"

---

## 12) Roadmap (future improvements)

Future versions may introduce:

- cryptographic field commitments (prove integrity while redacting fields)
- "redaction proofs" (provably removed content)
- configurable privacy policies per workspace/project
- differential access controls for evidence viewers

v0.1 keeps it simple: **omit + mark + sanitize**.
