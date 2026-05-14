# VOLT Codebase Map

<!-- Last verified: 2026-05-14 by codebase scan -->
> Auto-generated structural overview — Last scan: 2026-05-14 UTC

**VOLT — Verifiable Operations Ledger & Trace.** A cryptographically chained event log + portable run bundle + verifier for agentic workflows. The "flight recorder" for agentic workflows with cryptographic integrity.

**IETF Internet-Draft:** [`draft-cowles-volt-00`](https://datatracker.ietf.org/doc/draft-cowles-volt/)

## Protocol Stack Position

```
User Request
    ↓
AEE (messages/envelopes) ↔ Agents
    ↓
AOCL (policies, approvals, control)
    ↓
Tools / Runtimes / Files / Network
    ↓
VOLT (evidence ledger + bundle + verification)
    ↓
WARD (hash-chain witnessing, external anchoring)
```

## Version

- **Protocol:** v0.1.1 (draft)
- **Scope:** Record → Bundle → Verify
- **License:** Apache 2.0
- **Type:** Protocol specification (no runtime code)

## Repository Structure

```
volt/
├── README.md                  # Protocol overview, design principles
├── SPEC.md                    # Normative spec (23 sections)
├── EVIDENCE_BUNDLES.md        # Bundle layout, attachments, retention (13 sections)
├── VERIFICATION.md            # Verifier algorithm (10 steps), report format
├── INTEGRATION.md             # AEE/AOCL/tool integration guide (11 sections)
├── WORKED_EXAMPLES.md         # End-to-end sample runs (4 examples)
├── THREAT_MODEL.md            # Security threat model (10 threats)
├── PRIVACY_REDACTION.md       # Privacy handling, redaction rules (12 sections)
├── SECURITY.md                # Vulnerability reporting policy
├── ROADMAP.md                 # v0.2–v1.0 roadmap (6 versions planned)
├── CONTRIBUTING.md            # Contribution guidelines
├── CHANGELOG.md               # Version history (v0.1.0 → v0.1.1)
├── LICENSE                    # Apache 2.0
├── schemas/                   # JSON Schema definitions (v0.1)
│   ├── event.schema.json          # Trace event schema
│   ├── manifest.schema.json       # Bundle manifest schema
│   ├── signature.schema.json      # Signature record schema
│   └── verification-report.schema.json  # Verifier output schema
└── examples/                  # Placeholder directories
    ├── simple-run/.gitkeep
    └── hitl-run/.gitkeep
```

## JSON Schemas

| Schema | Purpose | Required Fields |
|--------|---------|-----------------|
| `event.schema.json` | Trace event | volt_version, event_id, run_id, ts, seq, event_type, actor, context, payload, prev_hash, hash |
| `manifest.schema.json` | Bundle manifest | volt_version, bundle_id, run_id, created_ts, hash_alg, events_file, event_count, first_event_hash, last_event_hash |
| `signature.schema.json` | Signature record | sig_version, sig_type, key_id, signed_ts, scope, message, signature |
| `verification-report.schema.json` | Verifier output | result (PASS/FAIL/ERROR) |

## Event Schema Structure

```json
{
  "volt_version": "0.1",
  "event_id": "<unique>",
  "run_id": "<unique>",
  "ts": "<ISO 8601 UTC>",
  "seq": "<monotonic int ≥ 1>",
  "event_type": "<dotted path>",
  "actor": { "actor_type": "agent|human|system|tool|runner", "actor_id": "..." },
  "context": { "correlation_id": "...", ... },
  "payload": { "<event-specific, privacy-safe>" },
  "prev_hash": "<64 hex chars, zeros for genesis>",
  "hash": "<SHA-256 of canonical JSON without hash field>"
}
```

## Actor Types

| Type | Description |
|------|-------------|
| `agent` | Autonomous agent (e.g., `quox.agent.routeros`) |
| `human` | Human user (e.g., `user:adam`) |
| `system` | System component (e.g., `quox.core`, `quox.aocl`) |
| `tool` | Tool execution context |
| `runner` | Machine/host executing actions (e.g., `runner:vm-prod-01`) |

## Context Object Fields

| Field | Required | Description |
|-------|----------|-------------|
| `correlation_id` | Yes | AEE correlation ID for run linkage |
| `parent_event_id` | No | For span-like linkage |
| `aee_envelope_id` | No | AEE envelope reference |
| `aee_message_id` | No | AEE message reference |
| `aocl_policy_id` | No | AOCL policy identifier |
| `aocl_decision_id` | No | AOCL decision reference |
| `workspace_id` | No | Workspace scope |
| `project_id` | No | Project scope |
| `tags` | No | Array of strings |

## Standard Event Types

| Domain | Events |
|--------|--------|
| Run lifecycle | `run.started`, `run.completed`, `run.failed`, `run.cancelled` |
| AEE messaging | `aee.envelope.received`, `aee.envelope.sent`, `aee.message.parsed`, `aee.message.rejected` |
| AOCL policy | `aocl.policy.evaluated`, `aocl.decision.approved`, `aocl.decision.denied`, `aocl.hitl.required` |
| Tool execution | `tool.call.requested`, `tool.call.executed`, `tool.call.failed` |
| Human-in-the-loop | `hitl.requested`, `hitl.approved`, `hitl.denied`, `hitl.timed_out` |
| Files/network | `file.read`, `file.write`, `net.request` |
| Model activity | `model.requested`, `model.responded` |

Custom types: Use vendor namespace prefix (e.g., `quox.marketplace.offer.created`)

## Evidence Bundle Layout

```
<bundle_root>/
├── manifest.json              # Required
├── events.ndjson              # Required (NDJSON, ordered by seq)
├── attachments/<first2>/<hash> # Optional, content-addressed
├── signatures/                # Optional (sig-N.json or inline in manifest)
├── redactions/                # Optional (redactions.json)
└── notes/                     # Optional
```

| Mode | Description |
|------|-------------|
| `final` | Complete run with terminal event |
| `rolling` | Periodic snapshot with `cutoff_ts` |

## Conformance Levels

| Level | Requirements |
|-------|--------------|
| VOLT-R (Recorder) | Emits valid events, correct hashing/chaining, privacy constraints |
| VOLT-B (Bundler) | Produces manifest + events.ndjson + content-addressed attachments |
| VOLT-V (Verifier) | Validates chain, attachments, signatures; outputs JSON report |

## Verification Algorithm (10 steps)

1. **Load bundle** — parse manifest.json
2. **Load events** — parse events.ndjson as NDJSON
3. **Validate ordering** — seq monotonic, starts at 1, no gaps/duplicates
4. **Validate fields** — all required event fields present
5. **Validate version** — manifest.volt_version == event.volt_version
6. **Recompute hashes** — SHA-256 of canonical JSON (hash field removed)
7. **Validate chain** — prev_hash matches previous event hash (64 zeros for genesis)
8. **Validate run_id** — all events match manifest.run_id
9. **Validate manifest** — event_count, first_event_hash, last_event_hash
10. **Validate attachments/signatures** — content-addressed blobs + Ed25519 sigs

## Verification Failure Codes

| Code | Meaning |
|------|---------|
| MANIFEST_MISSING | manifest.json not found |
| MANIFEST_UNREADABLE | manifest.json not valid JSON |
| MANIFEST_SCHEMA_INVALID | Missing required fields |
| EVENTS_FILE_MISSING | events.ndjson not found |
| INVALID_EVENT_JSON | Line not valid JSON |
| EVENT_SCHEMA_INVALID | Missing required event fields |
| VERSION_MISMATCH | Event ≠ manifest version |
| RUN_ID_MISMATCH | Event ≠ manifest run_id |
| SEQ_NOT_MONOTONIC | seq not increasing |
| SEQ_DUPLICATE | Duplicate seq |
| SEQ_GAP | Missing seq |
| EVENT_HASH_MISMATCH | Computed ≠ stored hash |
| INVALID_GENESIS_PREV_HASH | First prev_hash ≠ 64 zeros |
| CHAIN_BROKEN | prev_hash ≠ previous event hash |
| MANIFEST_MISMATCH | Count/hash mismatch |
| ATTACHMENT_MISSING | Referenced attachment not found |
| ATTACHMENT_HASH_MISMATCH | Attachment content mismatch |
| SIGNATURE_SCHEMA_INVALID | Bad signature record |
| SIGNATURE_INVALID | Signature verification failed |
| UNSUPPORTED_SIGNATURE_TYPE | Unknown algorithm |

CLI exit codes: 0=PASS, 1=FAIL, 2=ERROR

## Cryptographic Primitives

| Primitive | Algorithm |
|-----------|-----------|
| Event/attachment hashing | SHA-256 |
| Hash encoding | Lowercase hex (64 chars) |
| Signatures | Ed25519 (optional v0.1) |
| Canonical JSON | Sorted keys, no whitespace, NFC normalized |

## Privacy Rules

- Events MUST NOT include secrets (API keys, tokens, passwords, private keys)
- Store metadata/references, not raw content
- v0.1 payloads are metadata-only
- Use `inputs_redacted: true` for sensitive payloads
- Network events: method, status, timing — never raw headers/cookies
- Tool commands: log category/class, not raw command strings with secrets

## Integration Points

### Mandatory Evidence Points (minimum hooks)

| Point | Events |
|-------|--------|
| Run lifecycle | `run.started`, `run.completed`/`failed`/`cancelled` |
| AEE boundaries | `aee.envelope.received`, `aee.envelope.sent` |
| AOCL decisions | `aocl.policy.evaluated`, `aocl.decision.approved`/`denied` |
| HITL lifecycle | `hitl.requested`, `hitl.approved`/`denied`/`timed_out` |
| Tool execution | `tool.call.requested`, `tool.call.executed`/`failed` |

### Bridge Activation Model

- VOLT recording SHOULD be opt-in via host application settings
- Use observer/hook pattern that does not couple transport (AEE) to audit (VOLT)
- Bridge failures MUST NOT disrupt the transport pipeline (fire-and-forget)
- Implementations SHOULD expose counters: `events_recorded`, `events_failed`, `runs_started`, `runs_completed`

### Minimal Interfaces (pseudocode)

```
VoltRecorder:
  start_run(run_meta) -> run_id
  record_event(event_type, actor, context, payload) -> event_id
  attach_bytes(label, content_type, bytes) -> attachment_ref
  finalize_run(status, summary) -> void

VoltBundler:
  finalize_bundle(run_id, mode="final") -> bundle_path

VoltVerifier:
  verify(bundle_path, strict=true) -> VerificationReport
```

## Threat Model Summary

| Threat | Mitigated | Notes |
|--------|-----------|-------|
| T1: Post-hoc tampering | Yes | Hash chain detects edits |
| T2: Event deletion | Yes | Chain breaks + seq gaps |
| T3: Fake event insertion | Yes | Chain breaks unless rebuilt |
| T4: Attachment substitution | Yes | Content-addressed hashes |
| T5: Repudiation | Partial | Full with signatures (v0.4) |
| T6: Silent redaction | Partial | Governance + required events |
| T7: Compromised host | No | Requires remote attestation |
| T8: Key theft | No | HSM/rotation mitigates |
| T9: Missing coverage | No | Integration discipline issue |
| T10: Privacy leakage | Partial | Secret scanning + redaction |

## Roadmap

| Version | Focus |
|---------|-------|
| v0.1 | Record, bundle, verify (current) |
| v0.2 | Checkpointing, Merkle roots, test vectors |
| v0.3 | Replay fixtures (deterministic debugging) |
| v0.4 | Attestations (signed runners, cross-signing) |
| v0.5 | TraceQL query layer |
| v1.0 | Stable standard, certification tiers |

## Explicit Non-Goals

VOLT is NOT:
- A SIEM replacement
- A full observability stack
- A policy engine (AOCL does that)
- A database for analytics
- A blockchain requirement

## Related Protocols

| Protocol | Purpose | Repo |
|----------|---------|------|
| AEE | Message format + correlation IDs | github.com/quoxai/aee |
| AOCL | Orchestration control (policy, HITL) | github.com/quoxai/aocl |
| WARD | Hash-chain witnessing, external anchoring | github.com/quoxai/ward |

## Quick Reference

| Goal | File(s) |
|------|---------|
| Understand VOLT | README.md |
| Build a recorder | SPEC.md §6-7, schemas/event.schema.json |
| Build a bundler | SPEC.md §10,16, EVIDENCE_BUNDLES.md |
| Build a verifier | VERIFICATION.md, schemas/verification-report.schema.json |
| Integrate AEE/AOCL | INTEGRATION.md |
| Add signatures | SPEC.md §15, schemas/signature.schema.json |
| Handle privacy | PRIVACY_REDACTION.md |
| Understand threats | THREAT_MODEL.md |
| See examples | WORKED_EXAMPLES.md |
| Plan implementation | ROADMAP.md |
| Report vulnerabilities | SECURITY.md |
| Contribute changes | CONTRIBUTING.md |
