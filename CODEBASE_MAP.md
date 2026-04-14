<!-- Last verified: 2026-04-14T18:45:00Z by codebase-mirror scan -->

# VOLT (Verifiable Operations Ledger & Trace) — Codebase Map

## Overview

Evidence layer for the Quox ecosystem. Produces tamper-evident, portable, verifiable traces of agent runs. Cryptographically chained event log + portable run bundle + independent verifier.

| Field | Value |
|-------|-------|
| Version | 0.1 (draft) |
| IETF Draft | draft-cowles-volt-00 |
| Stack | Protocol specification (JSON schemas, Markdown docs) |
| Remote | github.com/quoxai/volt |

## Core Concept

```
User Request → AEE (messages) → AOCL (policy) → Tools → VOLT (evidence)
```

VOLT records what happened at each boundary, chains events cryptographically, and packages them into portable Evidence Bundles that can be verified independently.

## Metrics

| Metric | Count |
|--------|-------|
| Schema Files | 4 |
| Documentation Files | 12 |
| Example Directories | 2 |
| Total Lines (docs + schemas) | 4,285 |

## Directory Structure

```
volt/
├── schemas/                    # JSON Schema definitions (v0.1)
│   ├── event.schema.json                 # Single VOLT trace event (11 required fields, 135 lines)
│   ├── manifest.schema.json              # Evidence Bundle manifest (9 required fields, 159 lines)
│   ├── signature.schema.json             # Signature/attestation record (7 required fields, 88 lines)
│   └── verification-report.schema.json   # Verifier output (PASS/FAIL/ERROR, 107 lines)
├── examples/
│   ├── simple-run/             # Placeholder for basic tool call example
│   └── hitl-run/               # Placeholder for HITL-gated write example
├── SPEC.md                     # Normative protocol specification (822 lines, 23 sections)
├── INTEGRATION.md              # AEE/AOCL/tool hook points (391 lines)
├── VERIFICATION.md             # Verifier algorithm (10-step) and report format (345 lines)
├── EVIDENCE_BUNDLES.md         # Bundle format, layout, retention (337 lines)
├── PRIVACY_REDACTION.md        # Logging safety, redaction rules (332 lines)
├── WORKED_EXAMPLES.md          # End-to-end trace examples (281 lines)
├── THREAT_MODEL.md             # Security objectives and mitigations (247 lines)
├── ROADMAP.md                  # v0.2+ features (checkpointing, replay, attestations, TraceQL) (223 lines)
├── README.md                   # Overview and entry point (170 lines)
├── CONTRIBUTING.md             # Contribution guidelines (124 lines)
├── SECURITY.md                 # Vulnerability reporting policy (68 lines)
├── CHANGELOG.md                # Version history (48 lines)
└── LICENSE                     # Apache 2.0 (190 lines)
```

## Schema Files

| Schema | Purpose | Required Fields |
|--------|---------|-----------------|
| `event.schema.json` | Single trace event | volt_version, event_id, run_id, ts, seq, event_type, actor, context, payload, prev_hash, hash |
| `manifest.schema.json` | Bundle index | volt_version, bundle_id, run_id, created_ts, hash_alg, events_file, event_count, first_event_hash, last_event_hash |
| `signature.schema.json` | Attestation record | sig_version, sig_type, key_id, signed_ts, scope, message, signature |
| `verification-report.schema.json` | Verifier output | result (PASS/FAIL/ERROR), reason codes, details |

## Standard Event Types (v0.1)

| Category | Event Types |
|----------|-------------|
| Run lifecycle | `run.started`, `run.completed`, `run.failed`, `run.cancelled` |
| AEE messaging | `aee.envelope.received`, `aee.envelope.sent`, `aee.message.parsed`, `aee.message.rejected` |
| AOCL policy | `aocl.policy.evaluated`, `aocl.decision.approved`, `aocl.decision.denied`, `aocl.hitl.required` |
| Tool execution | `tool.call.requested`, `tool.call.executed`, `tool.call.failed` |
| HITL gates | `hitl.requested`, `hitl.approved`, `hitl.denied`, `hitl.timed_out` |
| File/network | `file.read`, `file.write`, `net.request` |
| Model activity | `model.requested`, `model.responded` |

## Evidence Bundle Layout

```
<bundle_root>/
  manifest.json           # Required: bundle index
  events.ndjson           # Required: NDJSON event chain
  attachments/            # Optional: content-addressed blobs
    <first2>/<hash>
  signatures/             # Optional: signature records
  redactions/             # Optional: redaction log
```

## Event Schema (Key Fields)

```json
{
  "volt_version": "0.1",
  "event_id": "unique-per-run",
  "run_id": "unique-run-id",
  "ts": "2026-04-14T12:00:00.000Z",
  "seq": 1,
  "event_type": "run.started",
  "actor": { "actor_type": "agent|human|system|tool|runner", "actor_id": "..." },
  "context": { "correlation_id": "...", "aee_envelope_id": "...", "aocl_policy_id": "..." },
  "payload": { /* event-specific, privacy-safe */ },
  "prev_hash": "0000...0000",  // 64 zeros for genesis
  "hash": "sha256-hex"
}
```

## Verification Algorithm (10 Steps)

1. Load `manifest.json` and validate required fields
2. Load `events.ndjson` as NDJSON
3. Validate event ordering (seq monotonic, starts at 1)
4. Validate required event fields per schema
5. Validate version compatibility (manifest ↔ events)
6. Recompute and validate event hashes (SHA-256 of canonical JSON)
7. Validate chain integrity (prev_hash links)
8. Validate run_id consistency
9. Validate manifest counts and endpoints match
10. Validate attachments and signatures (if present)

## Verification Failure Codes

`MANIFEST_MISSING`, `MANIFEST_UNREADABLE`, `MANIFEST_SCHEMA_INVALID`, `EVENTS_FILE_MISSING`, `INVALID_EVENT_JSON`, `EVENT_SCHEMA_INVALID`, `VERSION_MISMATCH`, `RUN_ID_MISMATCH`, `SEQ_NOT_MONOTONIC`, `SEQ_DUPLICATE`, `SEQ_GAP`, `EVENT_HASH_MISMATCH`, `INVALID_GENESIS_PREV_HASH`, `CHAIN_BROKEN`, `MANIFEST_MISMATCH`, `ATTACHMENT_MISSING`, `ATTACHMENT_HASH_MISMATCH`, `SIGNATURE_SCHEMA_INVALID`, `SIGNATURE_INVALID`, `UNSUPPORTED_SIGNATURE_TYPE`

## Conformance Levels

| Level | Name | Requirements |
|-------|------|--------------|
| VOLT-R | Recorder | Emits valid events, correct hashing/chaining, privacy constraints |
| VOLT-B | Bundler | Produces manifest.json, events.ndjson, content-addressed attachments |
| VOLT-V | Verifier | Verifies events/chain, attachments, produces report per spec |

## Cryptographic Primitives

| Primitive | Algorithm | Encoding |
|-----------|-----------|----------|
| Event hash | SHA-256 | 64-char lowercase hex |
| Attachment hash | SHA-256 | 64-char lowercase hex |
| Signatures | Ed25519 (recommended) | Base64 |
| Canonical JSON | UTF-8, sorted keys, no whitespace, NFC normalized |

## Integration Points (AEE/AOCL)

| Boundary | VOLT Hook | Event Types |
|----------|-----------|-------------|
| AEE Gateway | Envelope in/out | `aee.envelope.received`, `aee.envelope.sent` |
| AOCL Engine | Policy evaluation | `aocl.policy.evaluated`, `aocl.decision.*` |
| HITL Service | Approval gates | `hitl.requested`, `hitl.approved`, `hitl.denied` |
| Tool Middleware | Execution lifecycle | `tool.call.requested`, `tool.call.executed`, `tool.call.failed` |
| Run Lifecycle | Start/end | `run.started`, `run.completed`, `run.failed` |

## Minimal Integration Interfaces

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

## Privacy Rules (Non-negotiable)

- MUST NOT log secrets (API keys, passwords, tokens, private keys)
- MUST default to metadata + references (not raw content)
- MUST mark redactions explicitly (`payload.redacted: true`)
- SHOULD scan for secrets before writing events/attachments
- SHOULD NOT log raw shell commands, HTTP headers, or file contents

## Threat Model Summary

| Threat | Mitigated By |
|--------|--------------|
| Post-hoc event tampering | Event hash validation (`EVENT_HASH_MISMATCH`) |
| Event deletion | Chain breaks (`CHAIN_BROKEN`), seq gap detection |
| Event insertion | Chain integrity validation |
| Attachment substitution | Content-addressed hash validation |
| Repudiation (partial v0.1) | Actor identity, optional signatures |
| Silent redaction | Explicit redaction flags (governance concern) |

**Not fully mitigated (v0.1):** Compromised host/runner, key theft, incomplete coverage, privacy leakage. These require operational controls, AOCL enforcement, and roadmap features (attestations).

## Roadmap (Post v0.1)

| Version | Features |
|---------|----------|
| v0.2 | Checkpointing, Merkle roots, conformance test vectors |
| v0.3 | Replay fixtures, tool output recording |
| v0.4 | Runner attestations, cross-signing, key management |
| v0.5 | TraceQL queries, compliance policies |
| v1.0 | Stable standard, certification tiers |

## Related Protocols

| Protocol | Relationship |
|----------|--------------|
| [AEE](https://github.com/quoxai/aee) | Message format + correlation IDs (VOLT records AEE envelopes) |
| [AOCL](https://github.com/quoxai/aocl) | Policy/permissions/HITL gates (VOLT records AOCL decisions) |
| [WARD](https://github.com/quoxai/ward) | Content-free witnessing layer (signs VOLT bundle hashes) |

## Invariants

| Check | Status | Details |
|-------|--------|---------|
| schema-files | pass | 4 JSON schemas (event, manifest, signature, verification-report) |
| spec-complete | pass | Full v0.1 spec documented in SPEC.md (23 sections) |
| examples-exist | pass | simple-run, hitl-run directories (placeholders) |
| privacy-guidance | pass | PRIVACY_REDACTION.md defines safe defaults |
| threat-model | pass | THREAT_MODEL.md covers mitigations |
| verification-spec | pass | VERIFICATION.md defines 10-step algorithm |
| integration-guide | pass | INTEGRATION.md maps AEE/AOCL hook points |
