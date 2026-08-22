<!-- Last verified: 2026-08-22 by /codebase-mirror -->

# VOLT: Codebase Map

**VOLT (Verifiable Operations Ledger & Trace).** A cryptographically chained event log + portable run bundle + verifier for agentic workflows. The "flight recorder" with cryptographic integrity. Part of the Quox protocol family (AEE, AOCL, VOLT, WARD). Specification only, no runtime code: there are no entry points, registries, API routes, build configs, or tests in this repo, only normative docs and JSON Schemas.

**IETF Internet-Draft:** [`draft-cowles-volt-00`](https://datatracker.ietf.org/doc/draft-cowles-volt/)

**Reference SDK:** `@quox/volt` (TypeScript / Node, zero runtime dependencies) at github.com/quoxai/volt-sdk. Implements the full Record, Bundle, Verify loop plus a `volt-verify` CLI, with conformance anchored to frozen external hash vectors. The SDK repo is private today, opening at launch with the npm publish.

## Metrics

| Metric | Value |
|--------|-------|
| Protocol Version | `0.1` |
| Doc-set Version | `0.1.1` (2026-02-28) |
| Schema files | 4 (JSON Schema Draft 2020-12, all parse clean) |
| Spec docs (Markdown) | 12 |
| Example bundles | 0 (2 stub dirs: `simple-run`, `hitl-run`) |
| Total lines | ~4,076 (excluding this map) |
| License | Apache 2.0 |

## Directory Structure

```
volt/
├── README.md                    # Overview, positioning, design principles, SDK pointer (180 lines)
├── SPEC.md                      # Normative protocol spec, 23 sections (821 lines)
├── EVIDENCE_BUNDLES.md          # Bundle layout, attachments, retention (337 lines)
├── VERIFICATION.md              # Verifier algorithm (10 steps) + report format (345 lines)
├── INTEGRATION.md               # AEE/AOCL/tool hook points, middleware patterns (391 lines)
├── PRIVACY_REDACTION.md         # Logging safety, redaction strategies (332 lines)
├── THREAT_MODEL.md              # Security objectives, mitigations, residual risks (247 lines)
├── WORKED_EXAMPLES.md           # End-to-end sample runs with event sequences (281 lines)
├── ROADMAP.md                   # v0.2 to v1.0 evolution plan (223 lines)
├── SECURITY.md                  # Vulnerability reporting policy (68 lines)
├── CONTRIBUTING.md              # Contribution guidelines (124 lines)
├── CHANGELOG.md                 # Version history (0.1.0, 0.1.1) (48 lines)
├── CODEBASE_MAP.md              # This file
├── LICENSE                      # Apache 2.0 (190 lines)
├── schemas/
│   ├── event.schema.json        # Trace event schema (135 lines)
│   ├── manifest.schema.json     # Bundle manifest schema (159 lines)
│   ├── signature.schema.json    # Signature/attestation record schema (88 lines)
│   └── verification-report.schema.json  # Verifier output (PASS/FAIL/ERROR) (107 lines)
└── examples/
    ├── simple-run/              # Stub (.gitkeep only)
    └── hitl-run/                # Stub (.gitkeep only)
```

## Authoritative Files

| File | Purpose |
|------|---------|
| `SPEC.md` | Normative spec: canonicalization, crypto, event schema, chaining, manifest, signatures, conformance levels |
| `schemas/event.schema.json` | Event schema (11 required fields) |
| `schemas/manifest.schema.json` | Bundle manifest (9 required fields, 10 optional) |
| `schemas/signature.schema.json` | Signature record (7 required fields, versioned by `sig_version`) |
| `schemas/verification-report.schema.json` | Verifier output (`result` required) with 20 standard failure codes |

## Core Concepts

### Event Schema (Required Fields)
- `volt_version`: Protocol version (e.g., `"0.1"`)
- `event_id`, `run_id`: Unique identifiers (UUIDv4 or ULID)
- `ts`: ISO 8601 UTC timestamp with `Z` suffix
- `seq`: Monotonically increasing from 1
- `event_type`: Dotted path (e.g., `tool.call.executed`)
- `actor`: `{actor_type, actor_id}`, types: `agent`, `human`, `system`, `tool`, `runner`
- `context`: `{correlation_id}` + optional AEE/AOCL linkage
- `payload`: Event-specific data (privacy-safe, metadata-only in v0.1)
- `prev_hash`: SHA-256 of previous event (64 zeros for genesis)
- `hash`: SHA-256 of canonical event (without `hash` field)

### Evidence Bundle Layout
```
<bundle>/
  manifest.json      # Index + integrity anchors
  events.ndjson      # Ordered event chain (newline-delimited JSON)
  attachments/       # Content-addressed blobs (optional)
    <first2>/<hash>  # e.g., attachments/ab/ab12...ff
  signatures/        # Optional attestations
  redactions/        # Optional redaction audit
```

### Verification Algorithm (10 Steps)
1. Load `manifest.json`, validate required fields
2. Load `events.ndjson`, one JSON object per line
3. Validate ordering: `seq` monotonically increasing from 1
4. Validate required event fields per schema
5. Validate version consistency: all events match manifest `volt_version`
6. Recompute SHA-256 event hashes via Canonical JSON
7. Validate chain: `prev_hash` equals previous event's `hash` (genesis = 64 zeros)
8. Validate `run_id` consistency across all events
9. Validate manifest endpoints: `first_event_hash`, `last_event_hash`, `event_count`
10. Validate attachments (if referenced) and signatures (if present)

**Exit codes:** 0=PASS, 1=FAIL, 2=ERROR

## Standard Event Types (v0.1)

| Domain | Event Types |
|--------|-------------|
| Run lifecycle | `run.started`, `run.completed`, `run.failed`, `run.cancelled` |
| AEE messaging | `aee.envelope.received`, `aee.envelope.sent`, `aee.message.parsed`, `aee.message.rejected` |
| AOCL policy | `aocl.policy.evaluated`, `aocl.decision.approved`, `aocl.decision.denied`, `aocl.hitl.required` |
| Tool execution | `tool.call.requested`, `tool.call.executed`, `tool.call.failed` |
| HITL | `hitl.requested`, `hitl.approved`, `hitl.denied`, `hitl.timed_out` |
| Files/network | `file.read`, `file.write`, `net.request` |
| Model activity | `model.requested`, `model.responded` |

Custom types use vendor namespace prefix (e.g., `quox.marketplace.offer.created`).

## Crypto Primitives

| Primitive | Algorithm | Format |
|-----------|-----------|--------|
| Event/attachment hashing | SHA-256 | Lowercase 64-char hex |
| Signatures (optional) | Ed25519 | Base64-encoded |
| Canonical JSON | Sorted keys (byte-wise), no whitespace, NFC strings, no exponent notation | n/a |

## Conformance Levels

| Level | Abbrev | Requirements |
|-------|--------|--------------|
| Recorder | VOLT-R | Emit valid events per §6, compute hashes per §7, respect privacy per §2 |
| Bundler | VOLT-B | Produce `manifest.json` per §16, write `events.ndjson` per §3, store attachments content-addressed per §9 |
| Verifier | VOLT-V | Verify events/chain per §11, verify attachments per §11, produce report per §18 |

## Verification Failure Codes

| Code | Meaning |
|------|---------|
| `MANIFEST_MISSING` / `MANIFEST_UNREADABLE` / `MANIFEST_SCHEMA_INVALID` | Manifest problems |
| `EVENTS_FILE_MISSING` / `INVALID_EVENT_JSON` / `EVENT_SCHEMA_INVALID` | Event file problems |
| `VERSION_MISMATCH` / `RUN_ID_MISMATCH` | Consistency failures |
| `SEQ_NOT_MONOTONIC` / `SEQ_DUPLICATE` / `SEQ_GAP` | Ordering failures |
| `EVENT_HASH_MISMATCH` / `INVALID_GENESIS_PREV_HASH` / `CHAIN_BROKEN` | Integrity failures |
| `MANIFEST_MISMATCH` | Endpoints/count don't match |
| `ATTACHMENT_MISSING` / `ATTACHMENT_HASH_MISMATCH` | Attachment failures |
| `SIGNATURE_SCHEMA_INVALID` / `SIGNATURE_INVALID` / `UNSUPPORTED_SIGNATURE_TYPE` | Signature failures |

## Integration Points

### Mandatory Evidence Points
1. **Run lifecycle**: `run.started` (early), `run.completed`/`failed`/`cancelled` (terminal)
2. **AEE boundaries**: `aee.envelope.received`, `aee.envelope.sent`
3. **AOCL decisions**: `aocl.policy.evaluated`, `aocl.decision.approved`/`denied`
4. **HITL gating**: `hitl.requested`, `hitl.approved`/`denied`/`timed_out`
5. **Tool execution**: `tool.call.requested` then `tool.call.executed`/`failed`

### Context Mapping
- `context.correlation_id` from AEE correlation ID (primary linkage)
- `context.aee_envelope_id`, `context.aee_message_id` from AEE identifiers
- `context.aocl_policy_id`, `context.aocl_decision_id` from AOCL identifiers
- `context.parent_event_id` for nested/causal flows

## Related Protocols

| Protocol | Role | Integration |
|----------|------|-------------|
| **AEE** | Message format + correlation IDs | VOLT records `aee.envelope.*` events, uses AEE `correlation_id` |
| **AOCL** | Policy decisions, HITL gates | VOLT records `aocl.*` events, links `aocl_policy_id` / `aocl_decision_id` |
| **WARD** | Content-free hash-chain witnessing | Produces tamper-evident receipts of VOLT bundles without storing content |

## Roadmap

| Version | Focus |
|---------|-------|
| v0.1 (current) | Record, Bundle, Verify |
| v0.2 | Checkpointing, Merkle roots, conformance test vectors |
| v0.3 | Replay fixtures, tool output recording |
| v0.4 | Signed runner attestations, cross-signing, key management |
| v0.5 | TraceQL / query layer |
| v1.0 | Stable standard, certification tiers, marketplace integration |

## Privacy & Security Summary

**Non-negotiables:**
- Events MUST NOT contain secrets (API keys, passwords, tokens, private keys)
- Default to metadata + references (hashes), not raw content
- Explicit redaction flags (`payload.redacted: true`, `inputs_redacted: true`)

**Threat mitigations:**
- Post-hoc tampering: `EVENT_HASH_MISMATCH` detection
- Event deletion: `CHAIN_BROKEN` / `SEQ_GAP` detection
- Attachment substitution: `ATTACHMENT_HASH_MISMATCH` detection
- Non-repudiation (v0.4+): Ed25519 signatures from runners/orchestrators

## Invariants

| Check | Status |
|-------|--------|
| `volt_version` required in event + manifest schemas (signature uses `sig_version`; optional in verification-report) | PASS |
| All 4 JSON schemas valid (Draft 2020-12, parse clean) | PASS |
| Example bundles populated | WARN (stubs only) |
