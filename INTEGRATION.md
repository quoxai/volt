# VOLT Integration with AEE + AOCL (v0.1)

This document explains **exactly where VOLT records evidence** within a Quox run, and how VOLT integrates cleanly with:

- **AEE** (Agent Envelope Exchange) — messaging/envelopes/correlation
- **AOCL** (Agent Orchestration Control Layers) — policy, permissions, HITL gates, decisions
- **Tool execution** — shell/ssh/http/db/files/etc. middleware

VOLT is the evidence substrate. It does not replace AEE or AOCL. It observes and records.

---

## 1) Integration goals

A VOLT integration MUST ensure:

- Every important action creates a VOLT event
- Events are chained correctly (`prev_hash`)
- Correlation and IDs link back to AEE + AOCL objects
- Sensitive data is excluded by default
- Tool execution is recorded with "requested → executed/failed" lifecycle events
- HITL checkpoints are recorded as verifiable approvals/denials

---

## 2) Where VOLT sits (runtime view)

```text
             ┌─────────────────────────────────────┐
User / UI -->│ AEE Gateway (envelopes in/out)      │
             └───────────────┬─────────────────────┘
                             │
                             v
                     ┌───────────────┐
                     │ AOCL Engine    │  (policy checks, HITL gates)
                     └───────┬───────┘
                             │ allow / deny / require HITL
                             v
                     ┌───────────────┐
                     │ Orchestrator   │  (agent routing, planning)
                     └───────┬───────┘
                             │ tool calls
                             v
                     ┌───────────────┐
                     │ Tool Middleware│  (shell, ssh, http, file, etc.)
                     └───────┬───────┘
                             │
                             v
                        Tools / Runners

VOLT Recorder is invoked at each boundary to emit evidence events.
```

---

## 3) The "evidence points" (mandatory hook points)

VOLT v0.1 defines **minimum evidence points** you MUST hook to get meaningful integrity:

### 3.1 Run lifecycle

Emit:

- `run.started` (as early as possible)
- `run.completed` OR `run.failed` OR `run.cancelled`

### 3.2 AEE boundaries (envelopes/messages)

Emit:

- `aee.envelope.received`
- `aee.envelope.sent`

Optionally:

- `aee.message.parsed`
- `aee.message.rejected`

### 3.3 AOCL policy/decision boundaries

Emit:

- `aocl.policy.evaluated` (every time AOCL checks a policy)
- `aocl.decision.approved` / `aocl.decision.denied` (for allow/deny)
- `aocl.hitl.required` (when HITL is required before continuing)

### 3.4 HITL lifecycle (if used)

Emit:

- `hitl.requested`
- `hitl.approved` / `hitl.denied` / `hitl.timed_out`

### 3.5 Tool execution boundaries (most important)

Emit:

- `tool.call.requested`
- `tool.call.executed` OR `tool.call.failed`

This is your "flight recorder" for automation actions.

---

## 4) Correlation and ID mapping rules (AEE/AOCL → VOLT)

### 4.1 Required mapping

VOLT `context.correlation_id` SHOULD be the **AEE correlation id** for the run.

If AEE does not provide one, the orchestrator MUST generate one and propagate it.

### 4.2 AEE mapping

If present, populate:

- `context.aee_envelope_id`
- `context.aee_message_id` (optional, if AEE uses message IDs)

### 4.3 AOCL mapping

If present, populate:

- `context.aocl_policy_id`
- `context.aocl_decision_id`

### 4.4 Parent/child mapping (optional but recommended)

For nested flows, use:

- `context.parent_event_id` to reference the event that caused this one

Example:

- `tool.call.requested` has `parent_event_id` pointing to an `aocl.decision.approved` event.

---

## 5) Suggested middleware architecture

VOLT becomes easy if you treat it like **middleware**:

- **AEE middleware**: emits message events
- **AOCL middleware**: emits decision events
- **Tool middleware**: emits tool call events
- **HITL middleware**: emits approval events

Everything writes to a shared `VoltRecorder` for the run.

---

## 6) Minimal interfaces (pseudocode)

Below is a minimal, implementation-friendly pseudocode interface set.

### 6.1 VoltRecorder

```pseudo
interface VoltRecorder:
  start_run(run_meta) -> run_id
  record_event(event_type, actor, context, payload) -> event_id
  attach_bytes(label, content_type, bytes) -> attachment_ref
  finalize_run(status, summary) -> void
```

**Notes:**

- `record_event` handles:
  - seq increment
  - prev_hash linkage
  - hash calculation
  - NDJSON append
- `attach_bytes` stores content-addressed blob and returns `{hash_alg, hash, content_type, label}`

### 6.2 VoltBundler

```pseudo
interface VoltBundler:
  finalize_bundle(run_id, mode="final") -> bundle_path
```

Responsibilities:

- write `manifest.json`
- ensure `events.ndjson` complete
- copy/link attachments into bundle layout

### 6.3 VoltVerifier

```pseudo
interface VoltVerifier:
  verify(bundle_path, strict=true) -> VerificationReport
```

---

## 7) Exactly when to emit events (lifecycle map)

This section is the "do this, then this" order.

### 7.1 Run start

When a user request enters Quox orchestration:

1. Emit `run.started`
2. Emit `aee.envelope.received` (if via AEE)
3. Emit `aee.message.parsed` (optional)

**Why:** establishes genesis event and correlation.

### 7.2 AOCL policy checks

Every time AOCL evaluates permission/policy:

- Emit `aocl.policy.evaluated` with:
  - policy id
  - decision id
  - outcome (`allow`, `deny`, `hitl_required`)
  - reason code (non-sensitive)

Then:

- If allow: emit `aocl.decision.approved`
- If deny: emit `aocl.decision.denied` and end run
- If HITL: emit `aocl.hitl.required`

### 7.3 HITL gating

When HITL required:

- Emit `hitl.requested` with:
  - request_id
  - summary string
  - expiry timestamp

On user action:

- Emit `hitl.approved` or `hitl.denied`
- If denied, end run with `run.cancelled` or `run.failed` (implementation-defined)

### 7.4 Tool call lifecycle (mandatory)

Before tool execution:

- Emit `tool.call.requested` with:
  - tool name
  - operation
  - target metadata (safe)
  - inputs redacted flag

After execution:

- On success: `tool.call.executed`
- On failure: `tool.call.failed`

Attach outputs as content-addressed attachments where allowed:

- stdout/stderr
- JSON results
- artifact files (reports)

**Important:** do not attach secrets; sanitize/truncate.

### 7.5 Response out

When sending results back via AEE:

- Emit `aee.envelope.sent`
- Emit `run.completed` (or failed)

---

## 8) Payload guidance for each integration (privacy-first)

### 8.1 AEE events payload

Safe fields:

- channel (`web`, `api`, `cli`)
- size in bytes
- envelope type or intent tags

Avoid:

- raw message body unless explicitly allowed and sanitized

Example:

```json
"payload": {
  "channel": "web",
  "size_bytes": 1842,
  "intent": "deploy",
  "redacted": true
}
```

### 8.2 AOCL decision payload

Safe fields:

- `result`: `allow | deny | hitl_required`
- `reason`: policy reason code (not raw secrets)
- `required_approvals`: count (if relevant)

Example:

```json
"payload": {
  "result": "hitl_required",
  "reason": "prod_write_operation",
  "required_approvals": 1
}
```

### 8.3 Tool call payload

Safe fields:

- tool name, operation name
- target identifiers (non-sensitive)
- duration, status
- attachment refs

Avoid:

- raw command lines with secrets
- raw HTTP headers/cookies
- entire file contents by default

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

## 9) End-to-end worked flow (AEE + AOCL + tool + HITL)

Scenario: user asks to update a production config, AOCL requires HITL, tool writes file.

Event sequence:

1. `run.started`
2. `aee.envelope.received`
3. `aocl.policy.evaluated` → `hitl_required`
4. `aocl.hitl.required`
5. `hitl.requested`
6. `hitl.approved`
7. `tool.call.requested`
8. `tool.call.executed` (+ attachments stdout/stderr)
9. `aee.envelope.sent`
10. `run.completed`

This creates:

- a verifiable chain of custody
- a proof that approval happened before execution
- a tool execution record tied to the same correlation id

---

## 10) Minimal integration checklist (QuoxCORE)

To integrate VOLT into QuoxCORE with minimal work:

- [ ] Create a `VoltRecorder` instance per run (run_id scoped)
- [ ] Emit `run.started` immediately
- [ ] Hook AEE in/out to emit envelope events
- [ ] Hook AOCL evaluation to emit policy/decision events
- [ ] Wrap all tools in middleware that emits requested/executed/failed events
- [ ] Hook HITL approvals to emit approval/denial events
- [ ] Finalize run and write bundle manifest at completion
- [ ] Add `volt-verify` to CI or internal checks for bundles

---

## 11) Recommended next improvements (roadmap pointers)

(Not required for v0.1, but strongly recommended soon)

- Remote runner signatures (attestations)
- Strict redaction library + linting (prevent accidental secret logging)
- Trace viewer UI (developer ergonomics)
- Replay fixtures (record tool outputs for re-run simulation)
