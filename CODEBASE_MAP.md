# VOLT Codebase Map

**Repository:** `volt` (VOLT protocol specification)
**Location:** `/home/control/volt`
**Remote:** `github.com/quoxai/volt`
**Type:** Protocol specification (no runtime code)
**Version:** v0.1 (draft)
**IETF Draft:** `draft-cowles-volt-00`

---

## Overview

VOLT (Verifiable Operations Ledger & Trace) is Quox's **evidence layer protocol**. This repository contains:

- The normative protocol specification (SPEC.md)
- JSON Schema definitions for events, manifests, signatures, and verification reports
- Integration guidance for AEE/AOCL/WARD
- Privacy/security/threat modeling documentation
- Worked examples and contribution guidelines

**Not included here:** The reference SDK implementation (`@quox/volt`) lives at `github.com/quoxai/volt-sdk` (private, opens at launch).

---

## Repository Structure

```
volt/
├── README.md                           # Overview, positioning, quickstart
├── SPEC.md                             # Normative protocol spec (v0.1)
├── EVIDENCE_BUNDLES.md                 # Bundle layout, attachments, retention
├── VERIFICATION.md                     # Verifier algorithm + report format
├── INTEGRATION.md                      # AEE/AOCL/tool integration hooks
├── PRIVACY_REDACTION.md                # Privacy-safe logging rules
├── THREAT_MODEL.md                     # Security boundaries, what VOLT mitigates
├── WORKED_EXAMPLES.md                  # End-to-end sample runs
├── ROADMAP.md                          # v0.2+ features, explicit non-goals
├── SECURITY.md                         # Vulnerability reporting policy
├── CONTRIBUTING.md                     # Contribution guidelines
├── CHANGELOG.md                        # Version history
├── LICENSE                             # Apache-2.0
├── CODEBASE_MAP.md                     # This file
├── schemas/                            # JSON Schema definitions (Draft 2020-12)
│   ├── event.schema.json               # VOLT event schema (SPEC §6)
│   ├── manifest.schema.json            # Bundle manifest (SPEC §10, §16)
│   ├── signature.schema.json           # Signature record (SPEC §15)
│   └── verification-report.schema.json # Verifier output (SPEC §18)
└── examples/                           # Bundle example directories
    ├── simple-run/.gitkeep             # Placeholder for basic run
    └── hitl-run/.gitkeep               # Placeholder for HITL run
```

---

## Core Concepts

### 1. What VOLT Is

A **cryptographically chained event log** + **portable run bundle** + **verifier**.

VOLT produces tamper-evident, portable, verifiable traces of what happened during an agent run across agents, tools, approvals, and policy decisions WITHOUT turning your system into a logging swamp or leaking sensitive data.

- **Tamper-evident, not "truth"**: guarantees chain integrity, not correctness if host is fully compromised
- **Privacy by default**: logs metadata + references (hashes), NOT raw content or secrets
- **Portable evidence**: bundles are self-contained (folder or zip format)
- **Verifier-first design**: if it can't be verified independently, it doesn't count as evidence
- **No blockchain required**: optional public anchoring is a roadmap feature

### 2. Three-Phase Flow

1. **Record**: emit VOLT events at key execution points (AEE envelopes, AOCL decisions, tool calls, HITL approvals)
2. **Bundle**: finalize portable Evidence Bundles (`manifest.json`, `events.ndjson`, optional attachments)
3. **Verify**: independent verification checks hash chains, attachments, signatures

### 3. Protocol Relationships

```
User Request
    |
    v
AEE (messages/envelopes) <----> Agents
    |
    v
AOCL (policies, approvals, control)
    |
    v
Tools / Runtimes / Files / Network
    |
    v
VOLT (evidence ledger + bundle + verification)
    |
    v
WARD (content-free witnessing + receipts)
```

#### Separation of Concerns
- **AEE** (github.com/quoxai/aee): message format + correlation IDs for agent-to-agent and human-to-agent envelopes
- **AOCL** (github.com/quoxai/aocl): orchestration control layers (policy decisions, permissions, HITL gates, escalation rules)
- **VOLT** (this repo): evidence recording + integrity guarantees + exportable bundles + verification
- **WARD** (github.com/quoxai/ward): content-free hash-chain witnessing, produces tamper-evident receipts without storing content

#### What VOLT Does NOT Do
- Replace AEE messaging
- Decide policy (that's AOCL)
- Store secrets by default
- Act as a database, SIEM, or analytics warehouse
- Guarantee truth if the host is fully compromised (guarantees tamper-evidence, not miracles)

---

## Key Documents

### Read Order for Implementers
1. **README.md**: protocol introduction, use cases, positioning
2. **SPEC.md**: normative specification (canonical JSON, hashing, chaining, event schema, bundle format)
3. **INTEGRATION.md**: where VOLT hooks into AEE/AOCL/tools/HITL
4. **EVIDENCE_BUNDLES.md**: bundle layout, attachments, retention policies
5. **VERIFICATION.md**: verifier algorithm + report format
6. **PRIVACY_REDACTION.md**: logging safety + redaction rules

### Read Order for Auditors / Security Reviewers
1. **README.md**: protocol overview
2. **THREAT_MODEL.md**: attack surface, what VOLT mitigates and cannot prevent
3. **SECURITY.md**: vulnerability reporting policy
4. **VERIFICATION.md**: how to independently verify bundles
5. **PRIVACY_REDACTION.md**: privacy guarantees and data classification

### Read Order for Contributors
1. **CONTRIBUTING.md**: contribution guidelines
2. **ROADMAP.md**: v0.2+ features (checkpointing, replay, attestations, TraceQL)
3. **CHANGELOG.md**: version history

---

## JSON Schemas (`schemas/`)

All schemas follow **JSON Schema Draft 2020-12**.

### `event.schema.json`
Defines a single VOLT trace event (v0.1). See SPEC.md §6.

**Required fields:**
- `volt_version`: protocol version (e.g., `"0.1"`)
- `event_id`: unique identifier for this event within the run
- `run_id`: unique identifier for the run
- `ts`: ISO 8601 timestamp with UTC Z suffix
- `seq`: monotonically increasing sequence number, starting at 1
- `event_type`: dotted event type path (e.g., `"tool.call.executed"`)
- `actor`: who caused or observed this event
  - `actor_type`: `"agent"`, `"human"`, `"system"`, `"tool"`, or `"runner"`
  - `actor_id`: identifier for the actor
- `context`: correlation and linkage to AEE/AOCL
  - `correlation_id`: AEE correlation ID (required)
  - Optional: `parent_event_id`, `aee_envelope_id`, `aee_message_id`, `aocl_policy_id`, `aocl_decision_id`, `workspace_id`, `project_id`, `tags`
- `payload`: event-type-specific data (privacy-safe)
- `prev_hash`: SHA-256 hash of the previous event (64 hex chars, all zeros for genesis event)
- `hash`: SHA-256 hash of this event (64 hex chars, computed over canonical form with hash field removed)

### `manifest.schema.json`
Defines the bundle manifest (`manifest.json` in Evidence Bundles). See SPEC.md §10, §16.

**Required fields:**
- `volt_version`: protocol version, must match event volt_version
- `bundle_id`: unique identifier for this bundle (UUID or ULID)
- `run_id`: run identifier, must match all events in the bundle
- `created_ts`: ISO 8601 timestamp with UTC Z suffix
- `hash_alg`: hash algorithm used (`"sha256"` in v0.1)
- `events_file`: filename of the NDJSON events file (typically `"events.ndjson"`)
- `event_count`: total number of events in the events file
- `first_event_hash`: hash of the first event (seq=1)
- `last_event_hash`: hash of the last event (seq=max)

**Optional fields:**
- `bundle_mode`: `"rolling"` (periodic snapshot) or `"final"` (run complete)
- `cutoff_ts`: for rolling bundles, the cutoff timestamp
- `correlation_id`: AEE correlation ID for this run
- `producer`: information about the system that produced this bundle (name, version, build)
- `integrations`: summary IDs for AEE/AOCL integration references
- `redactions_present`: boolean flag indicating redacted fields
- `attachments_present`: boolean flag indicating attachments included
- `attachments`: array of attachment metadata (hash_alg, hash, content_type, bytes, path)
- `signatures`: array of signature records (optional in v0.1)
- `notes`: human-readable notes about this bundle

### `signature.schema.json`
Defines signature/attestation records (optional in v0.1). See SPEC.md §15.

**Required fields:**
- `sig_version`: signature record version (e.g., `"0.1"`)
- `sig_type`: signature algorithm (`"ed25519"` only in v0.1)
- `key_id`: stable identifier for the signing key (DID or fingerprint)
- `signed_ts`: ISO 8601 timestamp of when the signature was created
- `scope`: scope of the signature (`"bundle"` only in v0.1)
- `message`: the data that was signed (run_id, bundle_id, hash_alg, first_event_hash, last_event_hash, event_count)
- `signature`: Base64-encoded signature bytes

### `verification-report.schema.json`
Defines verifier report output format (PASS/FAIL/ERROR with structured findings). See SPEC.md §18.

**Result types:**
- `PASS`: bundle has integrity (exit 0)
- `FAIL`: verification failed with specific reason code (exit 1)
- `ERROR`: IO error, parse error, bundle not readable (exit 2)

**Failure reason codes:**
`MANIFEST_MISSING`, `MANIFEST_UNREADABLE`, `MANIFEST_SCHEMA_INVALID`, `EVENTS_FILE_MISSING`, `INVALID_EVENT_JSON`, `EVENT_SCHEMA_INVALID`, `VERSION_MISMATCH`, `RUN_ID_MISMATCH`, `SEQ_NOT_MONOTONIC`, `SEQ_DUPLICATE`, `SEQ_GAP`, `EVENT_HASH_MISMATCH`, `INVALID_GENESIS_PREV_HASH`, `CHAIN_BROKEN`, `MANIFEST_MISMATCH`, `ATTACHMENT_MISSING`, `ATTACHMENT_HASH_MISMATCH`, `SIGNATURE_SCHEMA_INVALID`, `SIGNATURE_INVALID`, `UNSUPPORTED_SIGNATURE_TYPE`

---

## Event Types (from SPEC.md §8)

VOLT v0.1 defines these standard event type namespaces:

### Run lifecycle
- `run.started`: run begins
- `run.completed`: run finished successfully
- `run.failed`: run failed with error
- `run.cancelled`: run cancelled by user/system

### AEE integration (envelopes/messages)
- `aee.envelope.received`: AEE envelope received
- `aee.envelope.sent`: AEE envelope sent
- `aee.message.parsed`: message parsed from envelope
- `aee.message.rejected`: message rejected (validation failed)

### AOCL integration (policy/decision boundaries)
- `aocl.policy.evaluated`: policy evaluation occurred
- `aocl.decision.approved`: policy decision: allow
- `aocl.decision.denied`: policy decision: deny
- `aocl.hitl.required`: HITL approval required

### Human-in-the-loop
- `hitl.requested`: HITL approval requested
- `hitl.approved`: HITL approval granted
- `hitl.denied`: HITL approval denied
- `hitl.timed_out`: HITL request timed out

### Tool execution
- `tool.call.requested`: tool call requested (before execution)
- `tool.call.executed`: tool call executed successfully
- `tool.call.failed`: tool call failed

### Files/network (metadata only)
- `file.read`: file read operation
- `file.write`: file write operation
- `net.request`: network request

### Model activity (metadata)
- `model.requested`: model inference requested
- `model.responded`: model response received

### Custom event types
- MUST use vendor namespace prefix (e.g., `quox.marketplace.offer.created`)
- Verifiers MUST ignore unknown event_type values

---

## Cryptographic Primitives

### Hashing (v0.1)
- **Algorithm:** SHA-256 (MUST)
- **Encoding:** lowercase hex strings of 64 characters (MUST)
- **Storage:** two fields `hash_alg: "sha256"` and `hash: "<64-char-hex>"`

### Canonicalization (for hashing)
Events are hashed using **Canonical JSON** (SPEC.md §3.2):

1. Serialize as JSON with UTF-8 encoding, no insignificant whitespace
2. Object keys MUST be sorted lexicographically (byte-wise) at every nesting level
3. Numbers MUST be represented without exponent notation where possible, without trailing `.0` when integral
4. Strings MUST be Unicode NFC normalized

### Signatures (optional in v0.1)
- **Algorithm:** Ed25519 (SHOULD)
- **Public key identifiers:** DID or key fingerprint (SHOULD be stable)
- **Signatures MUST** be over a clearly defined message (manifest commitments)

---

## Evidence Bundle Format

From `EVIDENCE_BUNDLES.md`:

### Bundle Goals
A VOLT Evidence Bundle MUST be:
- **Portable**: transferable as a folder or a zip file
- **Self-describing**: contains a manifest explaining what's inside
- **Verifiable**: contains an event chain + references that can be checked
- **Privacy-aware**: safe by default, sensitive payloads excluded/redacted
- **Incremental-friendly**: supports rolling bundles and final bundles

### Required Files
1. `manifest.json`
2. `events.ndjson`

### Recommended Layout
```
<bundle_root>/
  manifest.json
  events.ndjson
  attachments/                # optional, content-addressed
    ab/
      ab12...<hash>
  signatures/                 # optional
    sig-1.json
  redactions/                 # optional
    redactions.json
```

---

## Integration Points

From `INTEGRATION.md`:

### Mandatory Evidence Points

| Layer | Event Types |
|-------|-------------|
| Run lifecycle | `run.started`, `run.completed`/`run.failed`/`run.cancelled` |
| AEE boundaries | `aee.envelope.received`, `aee.envelope.sent` |
| AOCL policy | `aocl.policy.evaluated`, `aocl.decision.approved`/`denied`, `aocl.hitl.required` |
| HITL | `hitl.requested`, `hitl.approved`/`denied`/`timed_out` |
| Tool execution | `tool.call.requested`, `tool.call.executed`/`failed` |

### Context Field Mappings

| VOLT field | AEE field | AOCL field |
|------------|-----------|------------|
| `context.correlation_id` | `correlation_id` | `correlation_id` |
| `context.aee_envelope_id` | `envelope_id` | — |
| `context.aee_message_id` | `message_id` | — |
| `context.aocl_policy_id` | — | `policy_id` |
| `context.aocl_decision_id` | — | `decision_id` |

---

## Verification Algorithm

From `VERIFICATION.md`:

A VOLT verifier checks:

1. **Event well-formedness**: valid JSON, required fields present
2. **Hash correctness**: each event hash matches canonical form
3. **Chain integrity**: each `prev_hash` links correctly to previous event (genesis: 64 zeros)
4. **Sequence monotonicity**: `seq` increments by 1
5. **Manifest consistency**: `event_count`, `first_event_hash`, `last_event_hash` match
6. **Attachments**: referenced attachments exist and hashes match
7. **Signatures**: if present, validate for declared scope/message

### Verifier Modes
- **Strict mode** (enterprise/audit): fail on any deviation
- **Permissive mode** (debug/dev): warn on seq gaps if chain integrity valid

---

## Roadmap

From `ROADMAP.md`:

| Version | Focus |
|---------|-------|
| **v0.1** (current) | Evidence substrate: events, chaining, bundles, verifier |
| **v0.2** | Checkpointing, Merkle roots, conformance test vectors |
| **v0.3** | Replay mode (tool output fixtures, replay runner) |
| **v0.4** | Attestations (signed runners, cross-signing, key identity) |
| **v0.5** | TraceQL / query layer |
| **v1.0** | Stable standard, certification tiers, marketplace integration |

### Explicit Non-Goals
- NOT a SIEM replacement
- NOT a full observability stack
- NOT a policy engine (AOCL does that)
- NOT a database for analytics
- NOT a blockchain requirement

---

## Reference SDK

A conformant reference implementation of VOLT v0.1:

- **Package:** `@quox/volt` (TypeScript / Node.js, zero runtime dependencies)
- **Location:** `github.com/quoxai/volt-sdk` (private, opening at launch)
- **Implements:** Full Record, Bundle, Verify loop
  - Canonical JSON (SPEC §3.2, strictly enforced)
  - Event hashing and chaining (SPEC §7)
  - Manifest build (SPEC §16)
  - Ed25519 signing with manifest-bound signatures (SPEC §15)
  - Verifier algorithm (VERIFICATION.md)
  - `volt-verify` CLI
- **Conformance:** Anchored to frozen external hash vectors

---

## Related Quox Protocols

| Protocol | Purpose | Repo |
|----------|---------|------|
| **AEE** | Agent Envelope Exchange (messaging, correlation) | github.com/quoxai/aee |
| **AOCL** | Agent Orchestration Control Layers (policy, HITL) | github.com/quoxai/aocl |
| **VOLT** | Verifiable Operations Ledger & Trace (evidence) | github.com/quoxai/volt |
| **WARD** | Witnessing And Receipt Distribution (content-free witnessing) | github.com/quoxai/ward |

---

## Contributing

See `CONTRIBUTING.md` for contribution guidelines.

## Security

See `SECURITY.md` for vulnerability reporting policy.

## License

Apache-2.0

---

**Last updated:** 2026-08-11 (auto-generated codebase map)
