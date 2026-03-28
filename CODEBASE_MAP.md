<!-- Last verified: 2026-03-27 by /codebase-mirror -->

# VOLT (Verifiable Operations Ledger & Trace) — Codebase Map

## Spec Status
| Field | Value |
|-------|-------|
| Version | 0.1 |
| IETF Draft | draft-cowles-volt-00 |
| License | MIT |

## Architecture
Cryptographically chained event log + portable evidence bundles. Records message in/out, policy allow/deny, tool calls, human approvals, file access, network requests, model responses.

## Schemas (4)
| Schema | Purpose |
|--------|---------|
| `schemas/event.schema.json` | Individual event schema |
| `schemas/manifest.schema.json` | Bundle manifest (SHA-256 chaining) |
| `schemas/signature.schema.json` | Signature records |
| `schemas/verification-report.schema.json` | Verification output |

## Key Files
| File | Purpose |
|------|---------|
| `README.md` | Concept overview |
| `SPEC.md` | Normative protocol spec |
| `EVIDENCE_BUNDLES.md` | Bundle layout, attachments, retention |
| `VERIFICATION.md` | Verifier algorithm + report format |
| `INTEGRATION.md` | AEE + AOCL hooks |
| `PRIVACY_REDACTION.md` | Logging safety rules |
| `THREAT_MODEL.md` | Threat analysis |
| `WORKED_EXAMPLES.md` | Sample runs |
