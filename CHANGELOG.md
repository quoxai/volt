# Changelog — VOLT (Verifiable Operations Ledger & Trace)

All notable changes to the **VOLT protocol specification** will be documented in this file.

This project follows a pragmatic versioning approach:
- **Patch**: clarifications/typos only (no behavior change)
- **Minor**: backwards-compatible additions
- **Major**: breaking changes

## [0.1.0] — 2026-02-28

### Added
- Initial VOLT v0.1 protocol specification:
  - NDJSON event format (`events.ndjson`)
  - Canonical JSON rules for hashing
  - SHA-256 event hashing and `prev_hash` chaining
  - Required event fields and core schema
  - Standard event type conventions
- Evidence Bundle format:
  - `manifest.json` required fields
  - Content-addressed attachments layout and referencing
  - Rolling vs final bundle concepts
- Verification process:
  - Verifier algorithm (strict/permissive modes)
  - PASS/FAIL report format and reason codes
  - CLI exit code conventions
- Integration guidance:
  - Hook points for AEE envelope in/out
  - Hook points for AOCL policy decisions and HITL gating
  - Tool middleware lifecycle events (requested/executed/failed)
  - Correlation mapping rules (AEE/AOCL → VOLT context)
- Privacy and security baseline:
  - Privacy-first logging guidance and redaction rules
  - Threat model for tamper-evidence and operational risks
- Worked examples and roadmap.

### Notes
- Optional signatures/attestations are defined as a reserved schema pattern in v0.1, but are not required for conformance.
- Deterministic replay, TraceQL, remote runner attestations, and external anchoring are explicitly out of scope for v0.1 and listed on the roadmap.
