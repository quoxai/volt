# VOLT Worked Examples (v0.1)

This document provides end-to-end examples of VOLT traces and bundles, showing how VOLT integrates with **AEE** and **AOCL** in realistic scenarios.

Each example includes:
- scenario description
- event sequence (what gets recorded)
- sample `events.ndjson` lines (abbreviated for readability)
- sample `manifest.json`
- what a verifier report looks like

> Note: Hash values are shown as placeholders (`<hash...>`) for readability. In real bundles they are full 64-hex SHA-256 digests.

---

## Example 1 — Simple "Read-only" tool call (no HITL)

### Scenario
User asks: "Check the status of the Quox API service."

- AEE receives request
- AOCL allows read-only action
- Tool executes `systemctl status quox-api`
- Results returned to user
- Bundle verified PASS

### Event sequence
1. `run.started`
2. `aee.envelope.received`
3. `aocl.policy.evaluated` → allow
4. `aocl.decision.approved`
5. `tool.call.requested` (shell read-only)
6. `tool.call.executed` (+ stdout attachment)
7. `aee.envelope.sent`
8. `run.completed`

### Sample events.ndjson (subset)

```json
{"volt_version":"0.1","event_id":"E1","run_id":"RUN1","ts":"2026-02-28T19:00:00.000Z","seq":1,"event_type":"run.started","actor":{"actor_type":"system","actor_id":"quox.core"},"context":{"correlation_id":"corr-001","aee_envelope_id":"env-001"},"payload":{"entrypoint":"api.chat"},"prev_hash":"0000000000000000000000000000000000000000000000000000000000000000","hash":"<hash1>"}
{"volt_version":"0.1","event_id":"E2","run_id":"RUN1","ts":"2026-02-28T19:00:00.010Z","seq":2,"event_type":"aee.envelope.received","actor":{"actor_type":"system","actor_id":"quox.aee.gateway"},"context":{"correlation_id":"corr-001","aee_envelope_id":"env-001"},"payload":{"channel":"web","size_bytes":612,"redacted":true},"prev_hash":"<hash1>","hash":"<hash2>"}
{"volt_version":"0.1","event_id":"E3","run_id":"RUN1","ts":"2026-02-28T19:00:00.020Z","seq":3,"event_type":"aocl.policy.evaluated","actor":{"actor_type":"system","actor_id":"quox.aocl"},"context":{"correlation_id":"corr-001","aocl_policy_id":"policy.ops.readonly","aocl_decision_id":"dec-001"},"payload":{"result":"allow","reason":"read_only_check"},"prev_hash":"<hash2>","hash":"<hash3>"}
{"volt_version":"0.1","event_id":"E4","run_id":"RUN1","ts":"2026-02-28T19:00:00.021Z","seq":4,"event_type":"aocl.decision.approved","actor":{"actor_type":"system","actor_id":"quox.aocl"},"context":{"correlation_id":"corr-001","aocl_decision_id":"dec-001"},"payload":{"result":"allow"},"prev_hash":"<hash3>","hash":"<hash4>"}
{"volt_version":"0.1","event_id":"E5","run_id":"RUN1","ts":"2026-02-28T19:00:00.030Z","seq":5,"event_type":"tool.call.requested","actor":{"actor_type":"agent","actor_id":"quox.agent.ops"},"context":{"correlation_id":"corr-001"},"payload":{"tool_name":"shell","operation":"service_status","target":"quox-api","inputs_redacted":true},"prev_hash":"<hash4>","hash":"<hash5>"}
{"volt_version":"0.1","event_id":"E6","run_id":"RUN1","ts":"2026-02-28T19:00:00.180Z","seq":6,"event_type":"tool.call.executed","actor":{"actor_type":"runner","actor_id":"runner:vm-ops-01"},"context":{"correlation_id":"corr-001"},"payload":{"tool_name":"shell","status":"success","duration_ms":150,"attachment_refs":[{"hash_alg":"sha256","hash":"<stdout_hash>","content_type":"text/plain","label":"stdout"}]},"prev_hash":"<hash5>","hash":"<hash6>"}
{"volt_version":"0.1","event_id":"E7","run_id":"RUN1","ts":"2026-02-28T19:00:00.200Z","seq":7,"event_type":"aee.envelope.sent","actor":{"actor_type":"system","actor_id":"quox.aee.gateway"},"context":{"correlation_id":"corr-001","aee_envelope_id":"env-001"},"payload":{"channel":"web","size_bytes":980},"prev_hash":"<hash6>","hash":"<hash7>"}
{"volt_version":"0.1","event_id":"E8","run_id":"RUN1","ts":"2026-02-28T19:00:00.210Z","seq":8,"event_type":"run.completed","actor":{"actor_type":"system","actor_id":"quox.core"},"context":{"correlation_id":"corr-001"},"payload":{"status":"success","summary":"quox-api is active"},"prev_hash":"<hash7>","hash":"<hash8>"}
```

### Sample manifest.json

```json
{
  "volt_version": "0.1",
  "bundle_id": "BUNDLE1",
  "run_id": "RUN1",
  "created_ts": "2026-02-28T19:00:01.000Z",
  "hash_alg": "sha256",
  "events_file": "events.ndjson",
  "event_count": 8,
  "first_event_hash": "<hash1>",
  "last_event_hash": "<hash8>",
  "bundle_mode": "final",
  "correlation_id": "corr-001",
  "producer": { "name": "quox-core", "version": "0.9.x" },
  "attachments_present": true,
  "redactions_present": true
}
```

### Verifier report (PASS)

```json
{
  "result":"PASS",
  "run_id":"RUN1",
  "bundle_id":"BUNDLE1",
  "volt_version":"0.1",
  "hash_alg":"sha256",
  "event_count":8,
  "attachments_verified":true,
  "signatures_verified":false,
  "warnings":[]
}
```

---

## Example 2 — Production write requires HITL (policy-gated)

### Scenario

User asks: "Update NGINX config on production and reload."

- AOCL policy requires HITL approval for prod writes
- HITL approval is granted by a human
- Tool writes config and reloads service
- VOLT proves the sequence (approval before execution)

### Event sequence

1. `run.started`
2. `aee.envelope.received`
3. `aocl.policy.evaluated` → hitl_required
4. `aocl.hitl.required`
5. `hitl.requested`
6. `hitl.approved`
7. `tool.call.requested` (write + reload)
8. `tool.call.executed` (+ stdout/stderr)
9. `aee.envelope.sent`
10. `run.completed`

### Sample events.ndjson (subset)

```json
{"volt_version":"0.1","event_id":"E1","run_id":"RUN2","ts":"2026-02-28T19:10:00.000Z","seq":1,"event_type":"run.started","actor":{"actor_type":"system","actor_id":"quox.core"},"context":{"correlation_id":"corr-002","aee_envelope_id":"env-002"},"payload":{"entrypoint":"api.chat"},"prev_hash":"0000000000000000000000000000000000000000000000000000000000000000","hash":"<h1>"}
{"volt_version":"0.1","event_id":"E2","run_id":"RUN2","ts":"2026-02-28T19:10:00.010Z","seq":2,"event_type":"aee.envelope.received","actor":{"actor_type":"system","actor_id":"quox.aee.gateway"},"context":{"correlation_id":"corr-002","aee_envelope_id":"env-002"},"payload":{"channel":"web","size_bytes":1200,"redacted":true},"prev_hash":"<h1>","hash":"<h2>"}
{"volt_version":"0.1","event_id":"E3","run_id":"RUN2","ts":"2026-02-28T19:10:00.050Z","seq":3,"event_type":"aocl.policy.evaluated","actor":{"actor_type":"system","actor_id":"quox.aocl"},"context":{"correlation_id":"corr-002","aocl_policy_id":"policy.prod.write.requires_hitl","aocl_decision_id":"dec-002"},"payload":{"result":"hitl_required","reason":"prod_write_operation"},"prev_hash":"<h2>","hash":"<h3>"}
{"volt_version":"0.1","event_id":"E4","run_id":"RUN2","ts":"2026-02-28T19:10:00.051Z","seq":4,"event_type":"aocl.hitl.required","actor":{"actor_type":"system","actor_id":"quox.aocl"},"context":{"correlation_id":"corr-002","aocl_decision_id":"dec-002"},"payload":{"required_approvals":1},"prev_hash":"<h3>","hash":"<h4>"}
{"volt_version":"0.1","event_id":"E5","run_id":"RUN2","ts":"2026-02-28T19:10:01.000Z","seq":5,"event_type":"hitl.requested","actor":{"actor_type":"system","actor_id":"quox.hitl"},"context":{"correlation_id":"corr-002","aocl_decision_id":"dec-002"},"payload":{"request_id":"hitl-002","prompt_summary":"Approve prod nginx config write + reload","expires_ts":"2026-02-28T19:15:01.000Z"},"prev_hash":"<h4>","hash":"<h5>"}
{"volt_version":"0.1","event_id":"E6","run_id":"RUN2","ts":"2026-02-28T19:10:20.000Z","seq":6,"event_type":"hitl.approved","actor":{"actor_type":"human","actor_id":"user:adam","display_name":"Adam"},"context":{"correlation_id":"corr-002","aocl_decision_id":"dec-002"},"payload":{"request_id":"hitl-002","comment":"Proceed"},"prev_hash":"<h5>","hash":"<h6>"}
{"volt_version":"0.1","event_id":"E7","run_id":"RUN2","ts":"2026-02-28T19:10:21.000Z","seq":7,"event_type":"tool.call.requested","actor":{"actor_type":"agent","actor_id":"quox.agent.ops"},"context":{"correlation_id":"corr-002"},"payload":{"tool_name":"ssh","operation":"write_and_reload","target":"prod-web-01","inputs_redacted":true},"prev_hash":"<h6>","hash":"<h7>"}
{"volt_version":"0.1","event_id":"E8","run_id":"RUN2","ts":"2026-02-28T19:10:22.000Z","seq":8,"event_type":"tool.call.executed","actor":{"actor_type":"runner","actor_id":"runner:prod-web-01"},"context":{"correlation_id":"corr-002"},"payload":{"tool_name":"ssh","status":"success","duration_ms":940,"attachment_refs":[{"hash_alg":"sha256","hash":"<stdout_hash>","content_type":"text/plain","label":"stdout"},{"hash_alg":"sha256","hash":"<stderr_hash>","content_type":"text/plain","label":"stderr"}]},"prev_hash":"<h7>","hash":"<h8>"}
{"volt_version":"0.1","event_id":"E9","run_id":"RUN2","ts":"2026-02-28T19:10:22.500Z","seq":9,"event_type":"aee.envelope.sent","actor":{"actor_type":"system","actor_id":"quox.aee.gateway"},"context":{"correlation_id":"corr-002","aee_envelope_id":"env-002"},"payload":{"channel":"web","size_bytes":1450},"prev_hash":"<h8>","hash":"<h9>"}
{"volt_version":"0.1","event_id":"E10","run_id":"RUN2","ts":"2026-02-28T19:10:23.000Z","seq":10,"event_type":"run.completed","actor":{"actor_type":"system","actor_id":"quox.core"},"context":{"correlation_id":"corr-002"},"payload":{"status":"success","summary":"Config updated and nginx reloaded"},"prev_hash":"<h9>","hash":"<h10>"}
```

### What VOLT proves here

- AOCL flagged HITL requirement **before** any tool execution
- Human approval occurred (with actor id) **before** tool execution
- Tool execution outputs are integrity-checked attachments

This is the "enterprise trust" story in one chain.

---

## Example 3 — Multi-agent workflow + tool failure (incident reconstruction)

### Scenario

User asks: "Deploy a new version."

- Planner agent delegates to Builder agent and Deployer agent
- Build succeeds, deploy fails on a remote tool call
- Run ends in failure
- VOLT preserves exact sequence and failure evidence

### Event sequence (simplified)

1. `run.started`
2. `aee.envelope.received`
3. `aocl.policy.evaluated` → allow (staging)
4. `aee.envelope.sent` (plan summary)
5. `tool.call.requested` (build)
6. `tool.call.executed` (build artifacts)
7. `tool.call.requested` (deploy)
8. `tool.call.failed` (stderr attachment)
9. `run.failed`

### Sample failure event

```json
{
  "volt_version":"0.1",
  "event_id":"E8",
  "run_id":"RUN3",
  "ts":"2026-02-28T19:30:12.000Z",
  "seq":8,
  "event_type":"tool.call.failed",
  "actor":{"actor_type":"runner","actor_id":"runner:stage-web-01"},
  "context":{"correlation_id":"corr-003"},
  "payload":{
    "tool_name":"ssh",
    "status":"failed",
    "duration_ms":511,
    "error_code":"REMOTE_EXIT_1",
    "attachment_refs":[
      {"hash_alg":"sha256","hash":"<stderr_hash>","content_type":"text/plain","label":"stderr"}
    ],
    "outputs_redacted": false
  },
  "prev_hash":"<hash7>",
  "hash":"<hash8>"
}
```

### Verifier report (PASS, even though run failed)

Verification PASS means the evidence is intact, not that the operation succeeded.

```json
{
  "result":"PASS",
  "run_id":"RUN3",
  "bundle_id":"BUNDLE3",
  "volt_version":"0.1",
  "hash_alg":"sha256",
  "event_count": 23,
  "attachments_verified": true,
  "signatures_verified": false,
  "warnings": []
}
```

### Why this matters

In incidents, the first question is "what happened?"
VOLT gives you a **forensically useful** timeline:

- what tool ran
- where it ran
- what failed
- with integrity you can trust

---

## Example 4 — Tampering detection (what FAIL looks like)

### Scenario

Someone edits a single event in `events.ndjson` after the run (e.g., changes `hitl.denied` to `hitl.approved`) but forgets to recompute hashes.

### Expected result

Verifier fails with `EVENT_HASH_MISMATCH` at the modified event.

Example FAIL report:

```json
{
  "result":"FAIL",
  "reason":"EVENT_HASH_MISMATCH",
  "details":{
    "seq":6,
    "event_id":"E6",
    "expected_hash":"<computed_from_modified_event>",
    "found_hash":"<original_hash>"
  }
}
```

### If they recompute hashes for the rest of the chain

Without signatures:

- a purely cryptographic chain cannot distinguish "original" from "rewritten" if the attacker can rewrite the entire chain.

With signatures/attestations:

- the rewrite breaks signature verification.

This is why signing is a high-value milestone after v0.1.

---

## Bundle folder examples (reference layouts)

### Example folder layout

```text
RUN2-bundle/
  manifest.json
  events.ndjson
  attachments/
    12/
      <stdout_hash>
    a4/
      <stderr_hash>
  redactions/
    redactions.json
```

---

## Practical takeaway

If you can show a customer Example 2 ("prod write required HITL") and the verification PASS report, you've shown something most orchestrators can't:

- not just automation
- **automation with proof**
