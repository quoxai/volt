# VOLT Roadmap

This roadmap describes how VOLT evolves beyond v0.1 without losing its core design principles:

- **Privacy by default**
- **Minimal, spec-first**
- **Verifier-first**
- **Separation of concerns** (AEE = envelope, AOCL = control, VOLT = evidence)

VOLT should stay small enough to implement easily, but powerful enough to become an enterprise trust standard.

---

## Guiding rule: keep v0.1 sacred

VOLT v0.1 is the foundation:

✅ Record → Bundle → Verify
✅ Hash-chained events
✅ Portable evidence bundles
✅ Content-addressed attachments
✅ Clear AEE/AOCL integration points
✅ Privacy/redaction guidance

Everything else is additive.

---

## Version plan at a glance

### v0.1 (now) — Evidence substrate
- Event schema + canonicalization + hashing
- Chaining (`prev_hash`)
- Evidence bundles (`manifest.json`, `events.ndjson`, attachments)
- Verifier spec + report format
- Integration hooks for AEE/AOCL/tools/HITL

### v0.2 — Checkpointing & stronger interoperability
- Checkpoint events / periodic run commitments
- Optional Merkle root for event sets
- Event type registry clarifications
- Conformance test vectors (golden bundles)

### v0.3 — Replay fixtures (practical debugging)
- "Replay mode" (best-effort determinism)
- Tool output recording conventions
- Replay runner interface (simulate tool calls from recorded outputs)

### v0.4 — Attestations (real enterprise trust)
- Signed runner attestations
- Cross-signing between orchestrator and runners
- Key identity and revocation guidance

### v0.5 — Query & governance utility
- TraceQL (query language) or a minimal query API
- Compliance queries & policies using VOLT evidence
- Viewer integration patterns

### v1.0 — Stable standard
- Locked canonicalization rules
- Stable schemas and signature formats
- Certification tiers + marketplace integration

---

## v0.2 — Checkpointing & Merkle commitments

### Why
Long runs, streaming traces, and large event sets benefit from checkpointing and Merkle roots:
- faster verification of partial traces
- "proof of inclusion" for specific events
- efficient anchoring to external notaries/chains

### Proposed additions
- New optional fields:
  - `context.checkpoint_id`
  - `payload.merkle_root` (hex)
- New event type:
  - `volt.checkpoint.created`

### Deliverables
- Spec additions in `SPEC.md`
- Example bundles demonstrating checkpoints
- Verifier support (optional validation)

---

## v0.3 — Replay mode (best-effort determinism)

### What replay is (and isn't)
Replay is NOT "re-run the exact LLM output deterministically."
Replay IS:
- recreate the observed tool interactions
- step through the same run timeline
- compare outputs across two runs (regression)

### Proposed deliverables
- Replay conventions for attachments:
  - `tool.call.executed` stores safe stdout/stderr fixtures
  - optionally stores sanitized tool inputs as hashes + categories
- A `volt-replay` runner that can:
  - replay tool calls by returning recorded outputs
  - optionally re-execute tools in "shadow mode" for comparison

### Payoff
- debugging: "why did staging deploy differ from prod?"
- regression testing: "did this workflow change behavior?"
- auditing: "show me the exact steps again"

---

## v0.4 — Attestations & non-repudiation (key milestone)

This is the "enterprise proof" leap.

### Goals
- Prevent "chain rewrite" attacks by adding trusted signatures
- Prove which runner executed which actions
- Enable stronger chain of custody across distributed systems

### Proposed deliverables
1) **Runner attestations**
   - runner signs:
     - `last_event_hash` (or checkpoint root)
     - plus bundle metadata (run_id, event_count)
2) **Orchestrator signature**
   - orchestrator signs the run endpoints too
3) **Cross-signing**
   - orchestrator references runner attestation IDs and vice versa
4) **Key identity & revocation**
   - `key_id` standards (DID recommended)
   - revocation list format
   - rotation guidance

### Optional hardware-backed keys (future)
- TPM/HSM-backed signing keys
- secure enclave attestations (where available)

---

## v0.5 — TraceQL / Query layer

### Purpose
Once you have evidence, people will ask questions like:
- "show all runs that touched production"
- "show runs missing HITL approvals"
- "list tool calls involving SSH to prod servers"
- "show all actions by a specific plugin version"

### Approaches
- A minimal query language (TraceQL) over event streams
- OR a query API with a fixed set of filters

### Deliverables
- `TRACEQL.md` (or `QUERY_API.md`)
- Reference query engine (optional)
- Example compliance queries

---

## v1.0 — Certification & ecosystem

### Goal
Make VOLT the trust layer for an ecosystem:
- workflow/plugin certification tiers
- marketplace trust levels
- audit export standards

### Certification tiers (example)
- **VOLT-Compatible**: emits valid v1.0 events and bundles
- **VOLT-Verified**: includes verification pass + required event coverage
- **VOLT-Attested**: includes trusted runner/orchestrator signatures
- **VOLT-Enterprise**: policy coverage guarantees + export controls

### Deliverables
- `CERTIFICATION.md`
- Conformance suite + official test vectors
- Versioned schemas and validator tooling

---

## Cross-cutting roadmap items (ongoing)

### Privacy improvements
- Field-level commitments (prove integrity while redacting content)
- Redaction proofs (cryptographic transparency for removals)
- Configurable privacy policies per workspace/project

### UX improvements
- Evidence viewer UI component
- Timeline view + filter/search
- "Diff two runs" viewer

### Developer ergonomics
- SDKs in common languages
- Middleware templates for tool wrappers
- "Do not log secrets" lint tooling

---

## What VOLT will not become (explicit non-goals)

To avoid scope creep, VOLT is NOT:
- a SIEM replacement
- a full observability stack
- a policy engine (AOCL does that)
- a database for analytics (you can export into one)
- a blockchain requirement

VOLT remains the evidence substrate.

---

## Recommended next steps (practical)

1) Ship v0.1 in QuoxCORE:
   - recorder middleware + bundler + verifier
2) Add signatures (v0.4 milestone) as soon as feasible:
   - this upgrades evidence from "tamper-evident" to "strongly attributable"
3) Build a simple viewer:
   - makes VOLT immediately useful to humans
4) Then replay mode:
   - makes VOLT useful to engineers every day
