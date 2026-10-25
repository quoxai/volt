<!-- Last verified: 2026-10-25 by /codebase-mirror -->

# VOLT Codebase Map

**Repository:** `volt` (VOLT protocol specification)
**Location:** `/home/control/repos/volt`
**Remote:** `github.com/quoxai/volt`
**Type:** Protocol specification. No runtime code, no build, no tests, no routes, no registries.
**Version:** spec 0.1.1 (CHANGELOG: 0.1.0 and 0.1.1 both dated 2026-02-28)
**IETF Draft:** [`draft-cowles-volt`](https://datatracker.ietf.org/doc/draft-cowles-volt/) (linked from README; the draft XML is not stored in this repo)
**License:** Apache-2.0

VOLT (Verifiable Operations Ledger & Trace) is Quox's evidence layer: a SHA-256 chained event log, a portable run bundle, and an independent verifier. AEE is how agents talk, AOCL is how they are controlled, VOLT is how you prove what happened.

**Not in this repo:** the reference SDK `@quox/volt` (TypeScript, zero runtime deps, `volt-verify` CLI) lives at `github.com/quoxai/volt-sdk` (private, opening at launch).

## Metrics

| Metric | Count |
|--------|-------|
| JSON Schemas (Draft 2020-12) | 4 |
| Spec and supporting docs (markdown) | 12 |
| Normative SPEC sections | 23 |
| Standard event types | 24 |
| Verifier failure reason codes | 20 |
| Conformance levels | 3 (VOLT-R, VOLT-B, VOLT-V) |
| Example bundles | 0 (two `.gitkeep` placeholders) |
| Entry points / routes / services / tests | 0 (spec-only repo) |

## Repository Structure

```
volt/
├── README.md                # Overview, "The short version" on-ramp, hero image, quickstart
├── SPEC.md                  # Normative protocol spec, sections 1-23 (821 lines)
├── EVIDENCE_BUNDLES.md      # Bundle layout, attachments, rolling vs final, retention
├── VERIFICATION.md          # Verifier algorithm, strict/permissive modes, report format
├── INTEGRATION.md           # AEE / AOCL / tool middleware / HITL hook points
├── PRIVACY_REDACTION.md     # Privacy-safe logging and redaction rules
├── THREAT_MODEL.md          # What VOLT mitigates and what it cannot
├── WORKED_EXAMPLES.md       # End-to-end sample runs
├── ROADMAP.md               # v0.2 to v1.0, explicit non-goals
├── SECURITY.md              # Vulnerability reporting policy
├── CONTRIBUTING.md          # Contribution guidelines
├── CHANGELOG.md             # 0.1.0, 0.1.1
├── LICENSE                  # Apache-2.0
├── CODEBASE_MAP.md          # This file (generated)
├── docs/assets/readme-hero.png   # Project architecture hero artwork (added 2026-10-22)
├── schemas/                 # 4 JSON Schemas, $id under github.com/quoxai/volt/schemas/
│   ├── event.schema.json
│   ├── manifest.schema.json
│   ├── signature.schema.json
│   └── verification-report.schema.json
└── examples/                # Placeholders only
    ├── simple-run/.gitkeep
    └── hitl-run/.gitkeep
```

## Authoritative Files

| File | Purpose | Count |
|------|---------|-------|
| `SPEC.md` | Normative protocol definition | 23 sections |
| `schemas/event.schema.json` | Event shape (SPEC 6) | 11 required fields, no optional top-level props |
| `schemas/manifest.schema.json` | Bundle manifest (SPEC 10, 16) | 9 required, 10 optional fields |
| `schemas/signature.schema.json` | Signature record (SPEC 15) | 7 required fields |
| `schemas/verification-report.schema.json` | Verifier output (SPEC 18) | 3 result shapes, 20 reason codes |
| `CHANGELOG.md` | Version history | 2 releases |
| `ROADMAP.md` | Future versions and non-goals | 6 milestones |

## Registration Chains

- **New standard event type:** add to SPEC section 8 under the right namespace, follow the naming rules in section 20.1, add a minimal example in section 12 if it is a mandatory evidence point, update INTEGRATION.md hook table, bump CHANGELOG (minor).
- **New verifier reason code:** add to the `reason` enum in `schemas/verification-report.schema.json`, document in SPEC 18.1 and VERIFICATION.md, bump CHANGELOG (minor).
- **New manifest field:** add to `schemas/manifest.schema.json`, document in SPEC 16.2 (recommended) or 16.1 (required, breaking), bump CHANGELOG.
- **Custom vendor event types** need no spec change: use a vendor prefix (for example `quox.marketplace.offer.created`); verifiers MUST ignore unknown types.

## Invariants

| Check | Status | Details |
|-------|--------|---------|
| All 4 schemas parse as JSON and declare Draft 2020-12 | ✓ pass | Verified with jq |
| Schema `$id` URLs point at this repo | ✓ pass | `https://github.com/quoxai/volt/schemas/<name>` |
| Event schema required set matches SPEC 6 | ✓ pass | 11 fields |
| Reason codes in schema match VERIFICATION.md | ✓ pass | 20 codes |
| Example bundles present | ⚠ warn | `examples/simple-run` and `examples/hitl-run` are `.gitkeep` placeholders only |
| CHANGELOG version matches README/SPEC | ✓ pass | SPEC header says v0.1, CHANGELOG top is 0.1.1 (patch) |

## Core Model

Three phases: **Record** (emit events at AEE envelope, AOCL decision, tool call and HITL boundaries), **Bundle** (`manifest.json` + `events.ndjson` + optional content-addressed attachments), **Verify** (independent PASS/FAIL over hashes, chain, manifest, attachments, signatures).

| Protocol | Role | Repo |
|----------|------|------|
| AEE | Message envelopes and correlation IDs | github.com/quoxai/aee |
| AOCL | Policy, permissions, HITL gates | github.com/quoxai/aocl |
| VOLT | Evidence recording, bundles, verification | this repo |
| WARD | Content-free hash-chain witnessing and receipts | github.com/quoxai/ward |

VOLT does NOT replace AEE, decide policy, store secrets by default, act as a SIEM or database, or guarantee truth on a fully compromised host (tamper evidence, not miracles).

## SPEC.md Section Guide

| Sections | Contents |
|----------|----------|
| 1-3 | Terminology, design constraints (privacy by default, minimal schema, tamper evidence not truth), NDJSON transport, canonical JSON (3.2) |
| 4-5 | SHA-256 lowercase 64-hex, optional Ed25519; IDs and ISO 8601 UTC time |
| 6-7 | Event schema with actor/context/payload objects; hashing and `prev_hash` chaining (genesis is 64 zeros) |
| 8 | Standard event types (table below) |
| 9-10 | Content-addressed attachments; bundle manifest |
| 11-12 | Verification requirements; eight minimal event examples |
| 13-14 | Roadmap hooks (replay, TraceQL, attestations, anchoring); compliance notes |
| 15-16 | Signature records (Ed25519, bundle scope); manifest required and recommended fields |
| 17-18 | Rolling and final bundles, seq gaps; verifier JSON report and exit codes |
| 19-20 | Conformance levels VOLT-R / VOLT-B / VOLT-V; event type naming and registry rules |
| 21-23 | Security and privacy requirements; AEE/AOCL bridge model (opt-in observer, MUST NOT disrupt transport, SHOULD expose counters); change policy |

## JSON Schemas

**`event.schema.json`**: required `volt_version`, `event_id`, `run_id`, `ts`, `seq` (monotonic from 1), `event_type` (dotted path), `actor` (`actor_type` enum agent/human/system/tool/runner plus `actor_id`), `context` (`correlation_id` required; optional `parent_event_id`, `aee_envelope_id`, `aee_message_id`, `aocl_policy_id`, `aocl_decision_id`, `workspace_id`, `project_id`, `tags`), `payload` (metadata-only in v0.1), `prev_hash`, `hash`.

**`manifest.schema.json`**: required `volt_version`, `bundle_id`, `run_id`, `created_ts`, `hash_alg` (`sha256`), `events_file`, `event_count`, `first_event_hash`, `last_event_hash`. Optional `bundle_mode` (rolling/final), `cutoff_ts`, `correlation_id`, `producer`, `integrations`, `redactions_present`, `attachments_present`, `attachments[]`, `signatures[]`, `notes`.

**`signature.schema.json`**: required `sig_version`, `sig_type` (`ed25519` only), `key_id`, `signed_ts`, `scope` (`bundle` only), `message` (run_id, bundle_id, hash_alg, first/last event hash, event_count), `signature` (Base64).

**`verification-report.schema.json`**: `oneOf` three shapes. `PASS` (exit 0, ids, hashes, counts, `warnings[]`), `FAIL` (exit 1, `reason` code plus optional `details`), `ERROR` (exit 2, IO or parse). Reason codes: `MANIFEST_MISSING`, `MANIFEST_UNREADABLE`, `MANIFEST_SCHEMA_INVALID`, `EVENTS_FILE_MISSING`, `INVALID_EVENT_JSON`, `EVENT_SCHEMA_INVALID`, `VERSION_MISMATCH`, `RUN_ID_MISMATCH`, `SEQ_NOT_MONOTONIC`, `SEQ_DUPLICATE`, `SEQ_GAP`, `EVENT_HASH_MISMATCH`, `INVALID_GENESIS_PREV_HASH`, `CHAIN_BROKEN`, `MANIFEST_MISMATCH`, `ATTACHMENT_MISSING`, `ATTACHMENT_HASH_MISMATCH`, `SIGNATURE_SCHEMA_INVALID`, `SIGNATURE_INVALID`, `UNSUPPORTED_SIGNATURE_TYPE`.

## Standard Event Types (SPEC section 8)

| Namespace | Events |
|-----------|--------|
| Run lifecycle | `run.started`, `run.completed`, `run.failed`, `run.cancelled` |
| AEE messaging | `aee.envelope.received`, `aee.envelope.sent`, `aee.message.parsed`, `aee.message.rejected` |
| AOCL policy | `aocl.policy.evaluated`, `aocl.decision.approved`, `aocl.decision.denied`, `aocl.hitl.required` |
| HITL | `hitl.requested`, `hitl.approved`, `hitl.denied`, `hitl.timed_out` |
| Tool execution | `tool.call.requested`, `tool.call.executed`, `tool.call.failed` |
| Files/network (metadata only) | `file.read`, `file.write`, `net.request` |
| Model activity (metadata) | `model.requested`, `model.responded` |

## Hashing and Canonicalization

SHA-256 only, lowercase 64-hex. Each event is hashed over its canonical form with the `hash` field removed; `prev_hash` links to the previous event, genesis is 64 zeros. Canonical JSON (SPEC 3.2): UTF-8, no insignificant whitespace, byte-wise sorted keys at every level, no exponent notation or trailing `.0`, NFC-normalised strings. Signatures are optional in v0.1: Ed25519 over manifest commitments.

## Bundles and Verification

Bundle: required `manifest.json` and `events.ndjson`; optional `attachments/` (content-addressed, sharded by hash prefix), `signatures/`, `redactions/`. Portable as folder or zip, rolling or final mode.

Verifier order: event well-formedness, hash correctness, chain integrity, sequence monotonicity, manifest consistency, attachment existence and hashes, signature validity. Strict mode (audit) fails on any deviation; permissive mode (dev) warns on seq gaps when the chain is intact.

## Integration Points (INTEGRATION.md)

Mandatory evidence points: run lifecycle, AEE envelope boundaries, AOCL policy decisions, HITL events, tool execution. Context mappings: `context.correlation_id` from AEE/AOCL `correlation_id`; `context.aee_envelope_id` / `aee_message_id` from AEE; `context.aocl_policy_id` / `aocl_decision_id` from AOCL. The bridge is an opt-in observer and MUST NOT disrupt the AEE transport on failure (SPEC 22).

## Roadmap (ROADMAP.md)

| Version | Focus |
|---------|-------|
| v0.1 (current) | Evidence substrate: events, chaining, bundles, verifier |
| v0.2 | Checkpointing, Merkle roots, conformance test vectors |
| v0.3 | Replay mode (tool output fixtures, replay runner) |
| v0.4 | Attestations (signed runners, cross-signing, key identity) |
| v0.5 | TraceQL / query layer |
| v1.0 | Stable standard, certification tiers, marketplace integration |

Non-goals: not a SIEM, not an observability stack, not a policy engine, not an analytics database, no blockchain requirement.

## Read Order

- **Implementers:** README, SPEC, INTEGRATION, EVIDENCE_BUNDLES, VERIFICATION, PRIVACY_REDACTION
- **Auditors:** README, THREAT_MODEL, SECURITY, VERIFICATION, PRIVACY_REDACTION
- **Contributors:** CONTRIBUTING, ROADMAP, CHANGELOG
