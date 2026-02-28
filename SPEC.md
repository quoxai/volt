# VOLT SPEC — Verifiable Operations Ledger & Trace (v0.1)

**Status:** Draft
**Scope (v0.1):** Record → Bundle → Verify
**Non-goals (v0.1):** Deterministic replay, TraceQL, remote hardware attestation, blockchain anchoring (these are roadmap items).

VOLT defines a minimal, interoperable way to produce **tamper-evident execution traces** for agentic workflows and export them as **portable Evidence Bundles** that can be verified independently.

---

## 1. Terminology

- **Run**: A single end-to-end execution instance (user request → outcome).
- **Trace**: The ordered sequence of events for a run.
- **Event**: One atomic record describing something that happened (message, decision, tool call, approval, etc.).
- **Ledger chain**: Events linked by `prev_hash`, forming an append-only integrity chain.
- **Evidence Bundle**: A portable package containing the trace and optional attachments.
- **Attachment**: A content-addressed blob referenced by events (e.g., tool output, artifact).
- **Commitment**: A hash representing content (event hash, attachment hash, run root hash).
- **Attestation**: A signature over commitments asserting "this actor/runner observed/executed this".

---

## 2. Design constraints (normative)

### 2.1 Privacy by default
- VOLT events **MUST NOT** include secrets (API keys, passwords, raw tokens, private keys).
- VOLT events **SHOULD** store metadata and references (hashes) instead of raw content.
- Sensitive payload fields **SHOULD** be omitted or redacted at source.

### 2.2 Minimal schema, extensible evolution
- Every event **MUST** include `volt_version`.
- Unknown fields **MUST** be ignored by verifiers (forward compatibility).
- Breaking changes **MUST** increment `volt_version` (e.g., `0.2`, `1.0`).

### 2.3 Tamper evidence, not "truth"
- VOLT guarantees that recorded traces are **tamper-evident**.
- VOLT does not guarantee correctness if the host is fully compromised.

---

## 3. Transport & encoding

### 3.1 Events format
- Events are stored as **NDJSON** (newline-delimited JSON): one JSON object per line.
- Encoding **MUST** be UTF-8.
- Each line **MUST** be a complete JSON object.

### 3.2 Canonicalization (for hashing)
To ensure consistent hashing across implementations, VOLT defines **Canonical JSON**:

An implementation **MUST** produce a canonical byte representation of an event object as follows:

1. Serialize as JSON with:
   - UTF-8 encoding
   - No insignificant whitespace
2. Object keys **MUST** be sorted lexicographically (byte-wise) at every nesting level.
3. Numbers **MUST** be represented without exponent notation where possible and without trailing `.0` when integral.
4. Strings **MUST** be Unicode NFC normalized.

**Note:** If your language lacks a canonical JSON library, implement a deterministic key sort and normalized serialization.

---

## 4. Cryptographic primitives

### 4.1 Hash algorithm
- Event and attachment hashing **MUST** use `SHA-256` in v0.1.

### 4.2 Hash encoding
- Hash values **MUST** be encoded as lowercase hex strings of 64 characters.

Example:
- `sha256:2cf24dba5fb0a30e26e83b2ac5...`

VOLT stores hashes as:
- `hash_alg`: `"sha256"`
- `hash`: `"2cf24d..."`

(Optionally include a prefixed form in UI, but the stored value is pure hex.)

### 4.3 Signatures (optional in v0.1)
Signatures/attestations are optional in v0.1 but reserved in schema.
If used:
- Signature format **SHOULD** be `ed25519`.
- Public key identifiers **SHOULD** be stable (DID or key fingerprint).
- Signatures **MUST** be over a clearly defined message (see §9).

---

## 5. Identifiers & time

### 5.1 IDs
- `run_id` **MUST** be unique per run.
- `event_id` **MUST** be unique per run.
- IDs **SHOULD** be UUIDv4 or ULID.

### 5.2 Time
- `ts` **MUST** be ISO-8601 with UTC `Z`, e.g. `2026-02-28T19:11:02.123Z`.

---

## 6. Event schema (v0.1)

All events are JSON objects with these REQUIRED top-level fields:

| Field | Type | Notes |
|------|------|------|
| `volt_version` | string | e.g. `"0.1"` |
| `event_id` | string | unique per run |
| `run_id` | string | unique per run |
| `ts` | string | ISO-8601 UTC |
| `seq` | integer | monotonically increasing starting at 1 |
| `event_type` | string | dotted path, e.g. `tool.call.executed` |
| `actor` | object | who caused/observed event |
| `context` | object | correlation + linkage |
| `payload` | object | event data (privacy-safe) |
| `prev_hash` | string | hex; `"0"*64` for genesis |
| `hash` | string | hex; computed over canonical form excluding `hash` |

### 6.1 Actor object (REQUIRED)
`actor` describes who emitted the event.

REQUIRED fields:
- `actor_type`: `"agent" | "human" | "system" | "tool" | "runner"`
- `actor_id`: stable identifier (e.g., agent name/id)
OPTIONAL fields:
- `display_name`
- `org_id`, `team_id`
- `runner_id` (machine/host identity if known)

Example:
```json
"actor": {
  "actor_type": "agent",
  "actor_id": "quox.agent.routeros",
  "display_name": "RouterOS Agent"
}
```

### 6.2 Context object (REQUIRED)

`context` links this event to AEE/AOCL and other systems.

REQUIRED fields:
- `correlation_id`: stable ID spanning the run (prefer AEE correlation ID)

OPTIONAL fields:
- `parent_event_id`: for span-like linkage
- `aee_envelope_id`
- `aee_message_id`
- `aocl_policy_id`
- `aocl_decision_id`
- `workspace_id`, `project_id`
- `tags`: array of strings

Example:
```json
"context": {
  "correlation_id": "aee-corr-01HZ...",
  "aee_envelope_id": "aee-env-123",
  "aocl_policy_id": "policy.prod.write.requires_hitl",
  "tags": ["prod", "write", "hitl"]
}
```

### 6.3 Payload object (REQUIRED)

`payload` is event-type specific. It **MUST** be safe-by-default:

- include **metadata**, not secrets
- reference attachments via hashes rather than embedding raw output

> **v0.1 scope note:** In v0.1, payloads are metadata-only — e.g., `payload_keys` (field names present in the original message), token counts, tool names, and exit codes. Full-content payloads (message text, raw tool output) are deferred to v0.2 and will require opt-in redaction support (§9) to be conformant.

---

## 7. Event hashing & chaining (normative)

### 7.1 Hash input

The event `hash` is computed as:

`SHA-256( CanonicalJSON( EventWithoutHashField ) )`

Where `EventWithoutHashField` is the event object with the `hash` field removed.

### 7.2 Genesis event

- The first event in a run has:
  - `seq = 1`
  - `prev_hash = "0000...0000"` (64 zeros)

### 7.3 Chain rule

For all events where `seq > 1`:

- `prev_hash` **MUST** equal the `hash` of the event with `seq - 1`.

Verifiers **MUST** fail if the chain rule is violated.

---

## 8. Standard event types (v0.1)

Implementations MAY introduce custom event types, but these are RECOMMENDED standard types for interoperability.

### 8.1 Run lifecycle
- `run.started`
- `run.completed`
- `run.failed`
- `run.cancelled`

### 8.2 AEE messaging
- `aee.envelope.received`
- `aee.envelope.sent`
- `aee.message.parsed`
- `aee.message.rejected`

### 8.3 AOCL policy/control
- `aocl.policy.evaluated`
- `aocl.decision.approved`
- `aocl.decision.denied`
- `aocl.hitl.required`

### 8.4 Tool execution
- `tool.call.requested`
- `tool.call.executed`
- `tool.call.failed`

### 8.5 Human-in-the-loop
- `hitl.requested`
- `hitl.approved`
- `hitl.denied`
- `hitl.timed_out`

### 8.6 Files/network (metadata only)
- `file.read`
- `file.write`
- `net.request`

### 8.7 Model activity (metadata)
- `model.requested`
- `model.responded`

---

## 9. Attachments (content-addressed)

### 9.1 Attachment hashing

Attachment content **MUST** be hashed with SHA-256.

- `attachment_hash`: hex SHA-256
- `content_type`: e.g. `application/json`, `text/plain`, `application/pdf`

### 9.2 Attachment reference in payload

Events that refer to blobs **MUST** reference them by hash:

```json
"payload": {
  "tool_name": "shell",
  "attachment_refs": [
    {
      "hash_alg": "sha256",
      "hash": "e3b0c44298fc1c149afbf4c8996fb924...",
      "content_type": "text/plain",
      "label": "stdout"
    }
  ]
}
```

### 9.3 Storage layout (bundle)

Attachments **SHOULD** be stored under:

- `attachments/<first2>/<hash>`

Example:

- `attachments/e3/e3b0c442...`

---

## 10. Evidence Bundle manifest (v0.1)

A bundle contains a `manifest.json` with REQUIRED fields:

| Field | Type | Notes |
|-------|------|-------|
| `volt_version` | string | `"0.1"` |
| `bundle_id` | string | UUID/ULID |
| `run_id` | string | ties to events |
| `created_ts` | string | ISO-8601 UTC |
| `hash_alg` | string | `"sha256"` |
| `events_file` | string | usually `events.ndjson` |
| `event_count` | integer | count of events |
| `first_event_hash` | string | event seq=1 hash |
| `last_event_hash` | string | event seq=max hash |
| `attachments` | array | optional summary entries |
| `signatures` | array | optional |

Optional but recommended:

- `correlation_id`
- `aee_envelope_id` / `aocl_policy_id` summary
- `redactions_present`: boolean
- `notes`

---

## 11. Verification requirements (normative)

A verifier **MUST**:

1. Parse `manifest.json`
2. Read `events.ndjson` in order
3. For each event:
   - validate required fields exist
   - recompute `hash` from canonical event minus `hash`
   - confirm computed hash equals stored `hash`
   - confirm `prev_hash` equals prior event `hash`
4. Confirm event count matches manifest
5. Confirm manifest `first_event_hash` and `last_event_hash` match
6. If attachments are referenced:
   - confirm file exists
   - recompute SHA-256
   - confirm matches referenced hash
7. If signatures exist:
   - verify signatures according to signature record definition (implementation-defined in v0.1; see roadmap)

A verifier **MUST** output PASS/FAIL and the first failure reason.

---

## 12. Minimal required event examples (v0.1)

Below are examples with abbreviated hashes for readability. Real hashes must be full 64 hex chars.

### 12.1 run.started

```json
{
  "volt_version":"0.1",
  "event_id":"01HZ...A1",
  "run_id":"01HZ...RUN",
  "ts":"2026-02-28T19:12:00.000Z",
  "seq":1,
  "event_type":"run.started",
  "actor":{"actor_type":"system","actor_id":"quox.core"},
  "context":{"correlation_id":"aee-corr-01HZ..."},
  "payload":{"entrypoint":"api.chat","mode":"orchestrated"},
  "prev_hash":"0000000000000000000000000000000000000000000000000000000000000000",
  "hash":"<computed>"
}
```

### 12.2 aee.envelope.received

```json
{
  "volt_version":"0.1",
  "event_id":"01HZ...A2",
  "run_id":"01HZ...RUN",
  "ts":"2026-02-28T19:12:00.050Z",
  "seq":2,
  "event_type":"aee.envelope.received",
  "actor":{"actor_type":"system","actor_id":"quox.aee.gateway"},
  "context":{"correlation_id":"aee-corr-01HZ...","aee_envelope_id":"aee-env-123"},
  "payload":{"channel":"web","size_bytes":1842},
  "prev_hash":"<hash seq1>",
  "hash":"<computed>"
}
```

### 12.3 aocl.policy.evaluated

```json
{
  "volt_version":"0.1",
  "event_id":"01HZ...A3",
  "run_id":"01HZ...RUN",
  "ts":"2026-02-28T19:12:01.000Z",
  "seq":3,
  "event_type":"aocl.policy.evaluated",
  "actor":{"actor_type":"system","actor_id":"quox.aocl"},
  "context":{"correlation_id":"aee-corr-01HZ...","aocl_policy_id":"policy.prod.write.requires_hitl","aocl_decision_id":"dec-789"},
  "payload":{"result":"hitl_required","reason":"prod_write_operation"},
  "prev_hash":"<hash seq2>",
  "hash":"<computed>"
}
```

### 12.4 hitl.requested

```json
{
  "volt_version":"0.1",
  "event_id":"01HZ...A4",
  "run_id":"01HZ...RUN",
  "ts":"2026-02-28T19:12:02.000Z",
  "seq":4,
  "event_type":"hitl.requested",
  "actor":{"actor_type":"system","actor_id":"quox.hitl"},
  "context":{"correlation_id":"aee-corr-01HZ...","aocl_decision_id":"dec-789"},
  "payload":{"request_id":"hitl-555","prompt_summary":"Approve tool write to prod config","expires_ts":"2026-02-28T19:17:02.000Z"},
  "prev_hash":"<hash seq3>",
  "hash":"<computed>"
}
```

### 12.5 hitl.approved

```json
{
  "volt_version":"0.1",
  "event_id":"01HZ...A5",
  "run_id":"01HZ...RUN",
  "ts":"2026-02-28T19:12:30.000Z",
  "seq":5,
  "event_type":"hitl.approved",
  "actor":{"actor_type":"human","actor_id":"user:adam","display_name":"Adam"},
  "context":{"correlation_id":"aee-corr-01HZ...","aocl_decision_id":"dec-789"},
  "payload":{"request_id":"hitl-555","comment":"Proceed"},
  "prev_hash":"<hash seq4>",
  "hash":"<computed>"
}
```

### 12.6 tool.call.requested

```json
{
  "volt_version":"0.1",
  "event_id":"01HZ...A6",
  "run_id":"01HZ...RUN",
  "ts":"2026-02-28T19:12:31.000Z",
  "seq":6,
  "event_type":"tool.call.requested",
  "actor":{"actor_type":"agent","actor_id":"quox.agent.ops"},
  "context":{"correlation_id":"aee-corr-01HZ..."},
  "payload":{"tool_name":"shell","operation":"write_file","target":"prod/nginx.conf","inputs_redacted":true},
  "prev_hash":"<hash seq5>",
  "hash":"<computed>"
}
```

### 12.7 tool.call.executed (with attachment refs)

```json
{
  "volt_version":"0.1",
  "event_id":"01HZ...A7",
  "run_id":"01HZ...RUN",
  "ts":"2026-02-28T19:12:32.000Z",
  "seq":7,
  "event_type":"tool.call.executed",
  "actor":{"actor_type":"runner","actor_id":"runner:vm-prod-01"},
  "context":{"correlation_id":"aee-corr-01HZ..."},
  "payload":{
    "tool_name":"shell",
    "status":"success",
    "duration_ms":812,
    "attachment_refs":[
      {"hash_alg":"sha256","hash":"<stdout_hash>","content_type":"text/plain","label":"stdout"},
      {"hash_alg":"sha256","hash":"<stderr_hash>","content_type":"text/plain","label":"stderr"}
    ]
  },
  "prev_hash":"<hash seq6>",
  "hash":"<computed>"
}
```

### 12.8 run.completed

```json
{
  "volt_version":"0.1",
  "event_id":"01HZ...A8",
  "run_id":"01HZ...RUN",
  "ts":"2026-02-28T19:12:33.000Z",
  "seq":8,
  "event_type":"run.completed",
  "actor":{"actor_type":"system","actor_id":"quox.core"},
  "context":{"correlation_id":"aee-corr-01HZ..."},
  "payload":{"status":"success","summary":"Updated config and reloaded service"},
  "prev_hash":"<hash seq7>",
  "hash":"<computed>"
}
```

---

## 13. Roadmap hooks (non-normative, future)

### 13.1 Replay mode
- Record tool outputs and key environment metadata as attachments.
- Provide a runner that replays tool steps from recorded outputs where possible.

### 13.2 TraceQL
- Query event streams for compliance and incident investigation.

### 13.3 Attestations (signed runners)
- Runners sign the `last_event_hash` or a Merkle root of events for strong non-repudiation.

### 13.4 Optional anchoring
- Periodically anchor `last_event_hash` (or Merkle root) to a public chain for timestamping.

---

## 14. Compliance notes (informative)

- VOLT events are designed to be privacy-safe and exportable.
- Operators should treat bundles as sensitive operational artifacts.
- Redaction rules and key management are defined in `PRIVACY_REDACTION.md` and `SECURITY.md`.

---

## 15. Signatures & attestations (optional in v0.1, schema reserved)

VOLT v0.1 does not REQUIRE signing, but it defines a standard record format so implementations can add signing without breaking interoperability.

### 15.1 Signature record (in manifest)
If a bundle contains signatures, `manifest.json` SHOULD include a `signatures` array where each entry is:

REQUIRED:
- `sig_version`: `"0.1"`
- `sig_type`: `"ed25519"` (recommended)
- `key_id`: stable identifier for the signing key (DID or fingerprint)
- `signed_ts`: ISO-8601 UTC
- `scope`: `"bundle"` (v0.1 only)
- `message`: object defining what was signed
- `signature`: base64 signature bytes

`message` (v0.1 bundle scope) REQUIRED:
- `run_id`
- `bundle_id`
- `hash_alg`
- `first_event_hash`
- `last_event_hash`
- `event_count`

Example:
```json
{
  "sig_version": "0.1",
  "sig_type": "ed25519",
  "key_id": "did:key:z6Mkn...abc",
  "signed_ts": "2026-02-28T19:12:40.000Z",
  "scope": "bundle",
  "message": {
    "run_id": "01HZ...RUN",
    "bundle_id": "01HZ...BUNDLE",
    "hash_alg": "sha256",
    "first_event_hash": "<hash seq1>",
    "last_event_hash": "<hash seqN>",
    "event_count": 128
  },
  "signature": "MEUCIQDx..."
}
```

### 15.2 What signatures mean

- A valid signature indicates the signer attests that the bundle's event chain endpoints and count match the signed message.
- Signatures DO NOT prove that the underlying host was uncompromised; they provide non-repudiation and stronger chain-of-custody.

---

## 16. Manifest schema (normative details)

`manifest.json` MUST be a single JSON object encoded as UTF-8.

### 16.1 Required fields

- `volt_version` (string): MUST equal event `volt_version` for this bundle
- `bundle_id` (string)
- `run_id` (string)
- `created_ts` (string ISO-8601 UTC)
- `hash_alg` (string): MUST be `"sha256"` in v0.1
- `events_file` (string): typically `"events.ndjson"`
- `event_count` (integer)
- `first_event_hash` (string hex64)
- `last_event_hash` (string hex64)

### 16.2 Recommended fields

- `correlation_id` (string)
- `producer` (object):
  - `name` (e.g., `quox-core`)
  - `version`
  - `build` (git sha optional)
- `integrations` (object):
  - `aee` (object) optional summary ids
  - `aocl` (object) optional summary ids
- `redactions_present` (boolean)
- `attachments_present` (boolean)

### 16.3 Attachments summary

If attachments exist, manifest SHOULD include:

```json
"attachments": [
  {
    "hash_alg": "sha256",
    "hash": "<attachment_hash>",
    "content_type": "text/plain",
    "bytes": 48321,
    "path": "attachments/ab/<hash>"
  }
]
```

### 16.4 Integrity note

The manifest itself MAY be signed (see §15).
In v0.1 the manifest is not chained to events; integrity is derived from event hash endpoints and optional signatures.

---

## 17. Streaming & partial bundles

VOLT supports both:

- **final bundles** (run complete)
- **rolling bundles** (periodic snapshots)

### 17.1 Rolling bundles (SHOULD)

A rolling bundle SHOULD include the events up to a cutoff point and may be superseded later.

Rolling bundles SHOULD set:

- `bundle_mode`: `"rolling"`
- `cutoff_ts`: ISO-8601 UTC
- `last_event_hash` corresponding to last included event

### 17.2 Final bundles (RECOMMENDED)

Final bundles SHOULD set:

- `bundle_mode`: `"final"`
- include a terminal event: `run.completed | run.failed | run.cancelled`

### 17.3 Seq gaps

- `seq` MUST be monotonically increasing.
- Gaps SHOULD NOT occur, but if they do, verifiers MAY report a warning if:
  - chain integrity is valid
  - event_count matches content
- In strict mode, verifiers MAY fail on seq gaps.

---

## 18. Verifier output format (normative)

A verifier MUST output a result object (JSON or text). For interoperability, v0.1 RECOMMENDS a JSON report.

### 18.1 JSON report (recommended)

```json
{
  "result": "PASS",
  "run_id": "01HZ...RUN",
  "bundle_id": "01HZ...BUNDLE",
  "volt_version": "0.1",
  "hash_alg": "sha256",
  "event_count": 128,
  "first_event_hash": "<hash1>",
  "last_event_hash": "<hashN>",
  "attachments_verified": true,
  "signatures_verified": false,
  "warnings": []
}
```

On failure:

```json
{
  "result": "FAIL",
  "reason": "EVENT_HASH_MISMATCH",
  "details": {
    "seq": 42,
    "event_id": "01HZ...E42",
    "expected_hash": "<computed>",
    "found_hash": "<stored>"
  }
}
```

### 18.2 Exit codes (CLI conventions)

- `0` = PASS
- `1` = FAIL (verification failed)
- `2` = ERROR (invalid bundle format / IO error)

---

## 19. Conformance levels (normative)

To keep implementations comparable, VOLT defines three conformance targets:

### 19.1 Recorder (VOLT-R)

An implementation is VOLT-R conformant if it:

- emits events matching §6
- computes `hash` and `prev_hash` correctly per §7
- respects privacy constraints in §2

### 19.2 Bundler (VOLT-B)

An implementation is VOLT-B conformant if it:

- produces `manifest.json` per §16
- writes `events.ndjson` per §3
- stores attachments content-addressed per §9

### 19.3 Verifier (VOLT-V)

An implementation is VOLT-V conformant if it:

- verifies events and chain per §11
- verifies attachments referenced by events per §11
- produces a report per §18

---

## 20. Event type naming & registry rules (normative)

### 20.1 Naming

`event_type` MUST be a lowercase dotted string with these conventions:

- domain prefix (e.g., `run`, `aee`, `aocl`, `tool`, `hitl`, `file`, `net`, `model`)
- action (e.g., `started`, `requested`, `executed`, `failed`)

Examples:

- `tool.call.requested`
- `aocl.policy.evaluated`
- `model.responded`

### 20.2 Custom event types

Custom types MAY be added:

- MUST NOT conflict with standard prefixes unless extending them carefully
- SHOULD use a vendor namespace prefix, e.g.:
  - `quox.marketplace.offer.created`
  - `vendorx.routeros.script.deployed`

Verifiers MUST ignore unknown `event_type` values as long as required fields exist.

---

## 21. Security & privacy requirements (short, normative)

- Any field that may contain secrets MUST be omitted or redacted before event creation.
- Tool inputs SHOULD be represented as:
  - parameter names only
  - content hashes (attachments) where appropriate
  - high-level summaries
- Network events SHOULD log:
  - destination host (optional)
  - request method
  - status code
  - timing
  - NEVER raw headers/cookies by default

Detailed guidance belongs in `PRIVACY_REDACTION.md` (Doc later), but these baseline constraints apply at spec level.

---

## 22. Compatibility notes for AEE/AOCL (non-normative)

- `context.correlation_id` SHOULD use the AEE correlation id as the primary run linkage.
- If AOCL produces decision IDs, they SHOULD populate:
  - `context.aocl_policy_id`
  - `context.aocl_decision_id`
- AEE envelope/message identifiers SHOULD populate:
  - `context.aee_envelope_id`
  - `context.aee_message_id`

### 22.1 Bridge activation model

VOLT recording SHOULD be opt-in, controlled by host application settings. The recommended pattern is an **observer bridge** that:

1. Subscribes to AEE envelope storage events and AOCL lifecycle signals.
2. Maps AEE/AOCL events to VOLT trace events (see §8 for type mapping).
3. Records events via the VOLT recorder, creating runs per correlation ID.
4. Handles failures silently — bridge errors **MUST NOT** disrupt the AEE transport pipeline. Implementors SHOULD use fire-and-forget patterns with error logging.

### 22.2 Bridge observability

Implementations SHOULD expose operational counters for monitoring:

- `events_recorded`: Total events successfully persisted.
- `events_failed`: Total events that failed to persist.
- `runs_started` / `runs_completed`: Run lifecycle counters.
- `envelopes_processed`: Total AEE envelopes observed.
- `enabled`: Whether the bridge is currently active.

These counters aid debugging and provide confidence that the audit trail is functioning.

---

## 23. Spec change policy (informative)

- Patch changes (typos, clarifications): no version bump required
- Backwards-compatible schema additions: bump minor (`0.1` → `0.2`)
- Backwards-incompatible changes: bump major (`0.x` → `1.0`)

Changes SHOULD be recorded in `CHANGELOG.md`.
