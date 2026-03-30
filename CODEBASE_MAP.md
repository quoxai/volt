<!-- Last regenerated: 2026-03-30T18:13Z by codebase-mirror -->

# VOLT (Verifiable Operations Ledger & Trace) — Codebase Map

## Overview

VOLT is Quox's **evidence layer** — a protocol for producing tamper-evident, portable, verifiable traces of agent runs. It records what happened (messages, approvals, tool calls, policy decisions) without becoming a logging swamp or leaking sensitive data.

**Type:** Protocol specification (no runtime code — schemas + documentation only)

**Position in stack:** AEE (messaging) → AOCL (control) → Tools → **VOLT (evidence)** → WARD (witnessing)

## Spec Status

| Field | Value |
|-------|-------|
| Version | 0.1 (draft) |
| IETF Draft | [`draft-cowles-volt-00`](https://datatracker.ietf.org/doc/draft-cowles-volt/) |
| License | MIT |
| Scope | Record → Bundle → Verify |

## Metrics

| Metric | Count |
|--------|-------|
| JSON Schemas | 4 |
| Documentation Files | 11 |
| Example Directories | 2 |

## Core Concepts

| Concept | Description |
|---------|-------------|
| **Event** | Atomic record of an action (message, decision, tool call, approval) |
| **Trace** | Ordered sequence of events for a run, hash-chained via `prev_hash` |
| **Evidence Bundle** | Portable package: `manifest.json` + `events.ndjson` + attachments |
| **Attachment** | Content-addressed blob (tool output, artifact) referenced by SHA-256 |
| **Verification** | Independent check that chain is intact and hashes match |

## Repository Structure

```
volt/
├── SPEC.md                    # Normative protocol specification (23 sections)
├── README.md                  # Overview and design principles
├── EVIDENCE_BUNDLES.md        # Bundle format, attachments, redactions
├── VERIFICATION.md            # Verifier algorithm and report format
├── INTEGRATION.md             # AEE/AOCL/tool integration guide
├── PRIVACY_REDACTION.md       # Privacy-first logging rules
├── THREAT_MODEL.md            # Security analysis and mitigations
├── WORKED_EXAMPLES.md         # End-to-end sample runs
├── ROADMAP.md                 # v0.2+ features and non-goals
├── SECURITY.md                # Vulnerability reporting
├── CONTRIBUTING.md            # Contribution guidelines
├── CHANGELOG.md               # Version history
├── LICENSE                    # MIT
├── schemas/
│   ├── event.schema.json      # Event structure (JSON Schema 2020-12)
│   ├── manifest.schema.json   # Bundle manifest schema
│   ├── signature.schema.json  # Signature/attestation record
│   └── verification-report.schema.json  # Verifier output format
└── examples/
    ├── simple-run/            # Basic run example (placeholder)
    └── hitl-run/              # HITL approval flow example (placeholder)
```

## Schema Summary

### Event Schema (`schemas/event.schema.json`)

Required fields for every VOLT event:

| Field | Type | Description |
|-------|------|-------------|
| `volt_version` | string | Protocol version (e.g., `"0.1"`) |
| `event_id` | string | Unique per run |
| `run_id` | string | Unique per run |
| `ts` | string | ISO 8601 UTC timestamp |
| `seq` | integer | Monotonic sequence starting at 1 |
| `event_type` | string | Dotted path (e.g., `tool.call.executed`) |
| `actor` | object | Who caused the event (`actor_type`, `actor_id`) |
| `context` | object | Correlation IDs, AEE/AOCL linkage |
| `payload` | object | Event-specific data (privacy-safe) |
| `prev_hash` | string | SHA-256 hex of previous event (64 zeros for genesis) |
| `hash` | string | SHA-256 hex of canonical event (without hash field) |

### Actor Types

`agent` | `human` | `system` | `tool` | `runner`

### Standard Event Types (v0.1)

| Category | Event Types |
|----------|-------------|
| Run lifecycle | `run.started`, `run.completed`, `run.failed`, `run.cancelled` |
| AEE messaging | `aee.envelope.received`, `aee.envelope.sent`, `aee.message.parsed`, `aee.message.rejected` |
| AOCL control | `aocl.policy.evaluated`, `aocl.decision.approved`, `aocl.decision.denied`, `aocl.hitl.required` |
| Tool execution | `tool.call.requested`, `tool.call.executed`, `tool.call.failed` |
| HITL | `hitl.requested`, `hitl.approved`, `hitl.denied`, `hitl.timed_out` |
| Files/network | `file.read`, `file.write`, `net.request` |
| Model | `model.requested`, `model.responded` |

### Manifest Schema (`schemas/manifest.schema.json`)

Required fields for bundle manifest:

- `volt_version`, `bundle_id`, `run_id`, `created_ts`
- `hash_alg` (must be `"sha256"`)
- `events_file`, `event_count`
- `first_event_hash`, `last_event_hash`

Optional: `bundle_mode` (`rolling` | `final`), `cutoff_ts`, `correlation_id`, `producer`, `integrations`, `redactions_present`, `attachments_present`, `attachments`, `signatures`, `notes`

### Signature Schema (`schemas/signature.schema.json`)

Optional Ed25519 attestations over bundle endpoints:

- `sig_version`, `sig_type` (`ed25519`), `key_id`, `signed_ts`
- `scope` (`bundle` in v0.1)
- `message`: `{ run_id, bundle_id, hash_alg, first_event_hash, last_event_hash, event_count }`
- `signature`: Base64-encoded bytes

### Verification Report (`schemas/verification-report.schema.json`)

Exit codes: `0` = PASS, `1` = FAIL, `2` = ERROR

Standard failure reasons:
- `EVENT_HASH_MISMATCH`, `CHAIN_BROKEN`, `MANIFEST_MISMATCH`
- `ATTACHMENT_MISSING`, `ATTACHMENT_HASH_MISMATCH`
- `SEQ_GAP`, `SEQ_DUPLICATE`, `SEQ_NOT_MONOTONIC`
- `SIGNATURE_INVALID`, `RUN_ID_MISMATCH`, `VERSION_MISMATCH`
- `MANIFEST_MISSING`, `MANIFEST_UNREADABLE`, `MANIFEST_SCHEMA_INVALID`
- `EVENTS_FILE_MISSING`, `INVALID_EVENT_JSON`, `EVENT_SCHEMA_INVALID`
- `INVALID_GENESIS_PREV_HASH`, `SIGNATURE_SCHEMA_INVALID`, `UNSUPPORTED_SIGNATURE_TYPE`

## Verification Algorithm (10 Steps)

1. **Load bundle** — Parse `manifest.json`, validate required fields
2. **Load events** — Read `events.ndjson`, parse each line
3. **Validate ordering** — `seq` must start at 1, monotonically increase
4. **Validate event schema** — All required fields present with correct types
5. **Validate version** — Manifest and event `volt_version` must match
6. **Recompute hashes** — Canonical JSON → SHA-256 → compare to stored `hash`
7. **Validate chain** — Genesis `prev_hash` = 64 zeros; subsequent = prior `hash`
8. **Validate run_id** — All events match `manifest.run_id`
9. **Validate manifest** — `event_count`, `first_event_hash`, `last_event_hash` match
10. **Validate attachments/signatures** — If present and enabled

## Conformance Levels

| Level | Code | Requirements |
|-------|------|--------------|
| Recorder | VOLT-R | Emits valid events, correct hashing/chaining, privacy constraints |
| Bundler | VOLT-B | Produces valid manifest + events.ndjson + content-addressed attachments |
| Verifier | VOLT-V | Verifies events, chain, attachments; produces standard report |

## Integration Points

VOLT hooks into Quox at these boundaries:

1. **AEE Gateway** → `aee.envelope.received/sent`
2. **AOCL Engine** → `aocl.policy.evaluated`, `aocl.decision.*`
3. **HITL Service** → `hitl.requested/approved/denied`
4. **Tool Middleware** → `tool.call.requested/executed/failed`
5. **Run Lifecycle** → `run.started/completed/failed`

## Cryptographic Primitives

| Primitive | Algorithm | Format |
|-----------|-----------|--------|
| Hash | SHA-256 | 64-char lowercase hex |
| Signatures | Ed25519 (optional in v0.1) | Base64 |
| Canonicalization | Sorted keys, no whitespace, NFC-normalized strings | UTF-8 JSON |

## Privacy Rules (Non-negotiables)

- **Never log**: API keys, tokens, passwords, private keys, auth headers
- **Default to metadata**: Tool names, timing, status — not raw content
- **Explicit redaction**: Use `redacted: true` flags when omitting data
- **Attachments**: Sanitize before storing; content-addressed by hash

## Bundle Formats

| Format | Use Case |
|--------|----------|
| Folder | Internal systems, development |
| Zip | Export, sharing, archival |

### Bundle Layout

```
<bundle_root>/
  manifest.json           # Required: index and summary
  events.ndjson           # Required: trace events
  attachments/            # Optional: content-addressed blobs
    <first2>/<hash>
  signatures/             # Optional: separate signature files
  redactions/             # Optional: redaction metadata
  notes/                  # Optional: human notes
```

## Roadmap

| Version | Focus |
|---------|-------|
| v0.1 | Evidence substrate (current) |
| v0.2 | Checkpointing, Merkle roots, conformance tests |
| v0.3 | Replay fixtures for debugging |
| v0.4 | Runner attestations, cross-signing |
| v0.5 | TraceQL query layer |
| v1.0 | Stable standard, certification tiers |

## Related Protocols

| Protocol | Relationship |
|----------|--------------|
| [AEE](https://github.com/quoxai/aee) | Message format + correlation IDs |
| [AOCL](https://github.com/quoxai/aocl) | Policy decisions, HITL gates |
| [WARD](https://github.com/quoxai/ward) | Content-free hash-chain witnessing |

## Invariants

| Check | Status | Details |
|-------|--------|---------|
| schema-files | ✓ pass | 4 JSON schemas present and valid |
| spec-complete | ✓ pass | Full spec (23 sections) + threat model + examples |
| privacy-guidance | ✓ pass | PRIVACY_REDACTION.md defines non-negotiables |
| verification-algorithm | ✓ pass | 10-step normative algorithm in VERIFICATION.md |
