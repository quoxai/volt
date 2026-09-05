# VOLT Codebase Map

**Repository:** `volt` (VOLT protocol specification)
**Location:** `/home/control/repos/volt`
**Remote:** `github.com/quoxai/volt`
**Type:** Protocol specification (no runtime code, no build, no tests)
**Version:** v0.1 (spec at 0.1.1 per CHANGELOG, both releases 2026-02-28)
**IETF Draft:** [`draft-cowles-volt`](https://datatracker.ietf.org/doc/draft-cowles-volt/)

---

## Overview

VOLT (Verifiable Operations Ledger & Trace) is Quox's **evidence layer protocol**: a cryptographically chained event log plus a portable run bundle plus a verifier. A flight recorder for agent runs. If AEE is how agents communicate and AOCL is how they're controlled, VOLT is how you prove what actually happened.

This repo contains the normative spec, four JSON Schemas, and supporting docs. There are no entry points, registries, routes, or config files: it is documentation plus schemas only.

**Not included here:** the reference SDK `@quox/volt` (TypeScript/Node, zero runtime deps, `volt-verify` CLI) lives at `github.com/quoxai/volt-sdk` (private, opening at launch).

---

## Repository Structure

```
volt/
├── README.md                # Overview, "short version" on-ramp, positioning, quickstart
├── SPEC.md                  # Normative protocol spec, sections 1-23
├── EVIDENCE_BUNDLES.md      # Bundle layout, attachments, retention
├── VERIFICATION.md          # Verifier algorithm + report format
├── INTEGRATION.md           # AEE/AOCL/tool/HITL hook points
├── PRIVACY_REDACTION.md     # Privacy-safe logging + redaction rules
├── THREAT_MODEL.md          # What VOLT mitigates and cannot
├── WORKED_EXAMPLES.md       # End-to-end sample runs
├── ROADMAP.md               # v0.2+ features, explicit non-goals
├── SECURITY.md              # Vulnerability reporting policy
├── CONTRIBUTING.md          # Contribution guidelines
├── CHANGELOG.md             # 0.1.0 and 0.1.1 (both 2026-02-28)
├── LICENSE                  # Apache-2.0
├── CODEBASE_MAP.md          # This file (generated)
├── schemas/                 # JSON Schema Draft 2020-12, 4 schemas
│   ├── event.schema.json
│   ├── manifest.schema.json
│   ├── signature.schema.json
│   └── verification-report.schema.json
└── examples/                # Placeholders only (.gitkeep)
    ├── simple-run/
    └── hitl-run/
```

---

## Core Model

Three-phase flow: **Record** (emit events at AEE envelope, AOCL decision, tool call, and HITL boundaries) then **Bundle** (`manifest.json` + `events.ndjson` + optional content-addressed attachments) then **Verify** (independent PASS/FAIL check of hashes, chain, manifest, attachments, signatures).

Protocol stack and separation of concerns:

| Protocol | Role | Repo |
|----------|------|------|
| AEE | Message envelopes + correlation IDs | github.com/quoxai/aee |
| AOCL | Policy, permissions, HITL gates | github.com/quoxai/aocl |
| VOLT | Evidence recording, bundles, verification | this repo |
| WARD | Content-free hash-chain witnessing + receipts | github.com/quoxai/ward |

VOLT explicitly does NOT: replace AEE, decide policy, store secrets by default, act as a SIEM/database, or guarantee truth on a fully compromised host (tamper-evidence, not miracles).

---

## SPEC.md Section Guide

| Sections | Contents |
|----------|----------|
| 1-3 | Terminology, design constraints (privacy by default, minimal schema), NDJSON transport, canonical JSON rules (3.2) |
| 4-5 | Crypto primitives (SHA-256, lowercase 64-hex; Ed25519 optional), IDs and time |
| 6-7 | Event schema; actor/context/payload objects; hashing and `prev_hash` chaining (genesis = 64 zeros) |
| 8 | Standard event types (see below) |
| 9-10 | Content-addressed attachments; bundle manifest |
| 11-12 | Verification requirements; minimal event examples |
| 13-14 | Roadmap hooks (replay, TraceQL, attestations, anchoring); compliance notes |
| 15-16 | Signature records (Ed25519, bundle scope); manifest schema details |
| 17-18 | Streaming/rolling bundles; verifier output format |
| 19-20 | Conformance levels (VOLT-R recorder, VOLT-B bundler, VOLT-V verifier); event type naming and registry rules |
| 21-23 | Security/privacy requirements; AEE/AOCL compatibility notes (bridge is opt-in observer, failures MUST NOT disrupt transport); spec change policy |

---

## JSON Schemas (`schemas/`, Draft 2020-12)

### `event.schema.json` (SPEC section 6)
Required (all 11 fields, no optional top-level props): `volt_version`, `event_id`, `run_id`, `ts` (ISO 8601 UTC Z), `seq` (monotonic from 1), `event_type` (dotted path), `actor` (`actor_type`: agent/human/system/tool/runner + `actor_id`), `context` (`correlation_id` required; optional `parent_event_id`, `aee_envelope_id`, `aee_message_id`, `aocl_policy_id`, `aocl_decision_id`, `workspace_id`, `project_id`, `tags`), `payload` (metadata-only in v0.1), `prev_hash`, `hash`.

### `manifest.schema.json` (SPEC sections 10, 16)
Required: `volt_version`, `bundle_id`, `run_id`, `created_ts`, `hash_alg` (`sha256`), `events_file`, `event_count`, `first_event_hash`, `last_event_hash`.
Optional: `bundle_mode` (rolling/final), `cutoff_ts`, `correlation_id`, `producer`, `integrations`, `redactions_present`, `attachments_present`, `attachments[]`, `signatures[]`, `notes`.

### `signature.schema.json` (SPEC section 15)
Required: `sig_version`, `sig_type` (`ed25519` only in v0.1), `key_id` (DID or fingerprint), `signed_ts`, `scope` (`bundle` only in v0.1), `message` (run_id, bundle_id, hash_alg, first/last event hash, event_count), `signature` (Base64).

### `verification-report.schema.json` (SPEC section 18)
A `oneOf` over three result shapes: `PASS` (exit 0; run/bundle ids, hashes, counts, `warnings[]`), `FAIL` (exit 1; requires `reason` code + optional `details`), `ERROR` (exit 2, IO/parse). Failure reason codes: `MANIFEST_MISSING`, `MANIFEST_UNREADABLE`, `MANIFEST_SCHEMA_INVALID`, `EVENTS_FILE_MISSING`, `INVALID_EVENT_JSON`, `EVENT_SCHEMA_INVALID`, `VERSION_MISMATCH`, `RUN_ID_MISMATCH`, `SEQ_NOT_MONOTONIC`, `SEQ_DUPLICATE`, `SEQ_GAP`, `EVENT_HASH_MISMATCH`, `INVALID_GENESIS_PREV_HASH`, `CHAIN_BROKEN`, `MANIFEST_MISMATCH`, `ATTACHMENT_MISSING`, `ATTACHMENT_HASH_MISMATCH`, `SIGNATURE_SCHEMA_INVALID`, `SIGNATURE_INVALID`, `UNSUPPORTED_SIGNATURE_TYPE`.

---

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

Custom types MUST use a vendor namespace prefix (e.g. `quox.marketplace.offer.created`); verifiers MUST ignore unknown types.

---

## Hashing and Canonicalization

- SHA-256 (MUST), lowercase 64-char hex, stored as `hash_alg` + `hash`.
- Events hashed over canonical form with the `hash` field removed; `prev_hash` links each event to the previous (genesis: 64 zeros).
- Canonical JSON (SPEC 3.2): UTF-8, no insignificant whitespace, byte-wise sorted keys at every level, no exponent notation or trailing `.0`, NFC-normalized strings.
- Signatures optional in v0.1: Ed25519 over manifest commitments.

---

## Evidence Bundle Layout (EVIDENCE_BUNDLES.md)

Required: `manifest.json`, `events.ndjson`. Optional: `attachments/` (content-addressed, sharded by hash prefix), `signatures/`, `redactions/`. Bundles are portable (folder or zip), self-describing, privacy-safe by default, and support rolling and final modes.

---

## Verification (VERIFICATION.md)

Checks in order: event well-formedness, hash correctness, chain integrity, sequence monotonicity, manifest consistency, attachment existence and hashes, signature validity. Strict mode (audit) fails on any deviation; permissive mode (dev) warns on seq gaps when the chain is intact.

---

## Integration Points (INTEGRATION.md)

Mandatory evidence points: run lifecycle, AEE envelope boundaries, AOCL policy decisions, HITL events, tool execution. Context field mappings: `context.correlation_id` maps to AEE/AOCL `correlation_id`; `context.aee_envelope_id`/`aee_message_id` to AEE `envelope_id`/`message_id`; `context.aocl_policy_id`/`aocl_decision_id` to AOCL `policy_id`/`decision_id`. Per SPEC section 22, the VOLT bridge SHOULD be opt-in, observer-pattern, and MUST NOT disrupt the AEE transport pipeline on failure.

---

## Roadmap (ROADMAP.md)

| Version | Focus |
|---------|-------|
| v0.1 (current) | Evidence substrate: events, chaining, bundles, verifier |
| v0.2 | Checkpointing, Merkle roots, conformance test vectors |
| v0.3 | Replay mode (tool output fixtures, replay runner) |
| v0.4 | Attestations (signed runners, cross-signing, key identity) |
| v0.5 | TraceQL / query layer |
| v1.0 | Stable standard, certification tiers, marketplace integration |

Explicit non-goals: not a SIEM, not an observability stack, not a policy engine, not an analytics database, no blockchain requirement.

---

## Read Order

- **Implementers:** README, SPEC, INTEGRATION, EVIDENCE_BUNDLES, VERIFICATION, PRIVACY_REDACTION
- **Auditors:** README, THREAT_MODEL, SECURITY, VERIFICATION, PRIVACY_REDACTION
- **Contributors:** CONTRIBUTING, ROADMAP, CHANGELOG

License: Apache-2.0

---

**Last updated:** 2026-09-05 (auto-generated codebase map)
