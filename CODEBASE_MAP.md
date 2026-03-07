<!-- Last verified: 2026-03-06 by /codebase-mirror -->

# VOLT (Verifiable Operations Ledger & Trace) — Codebase Map

## Metrics
| Metric | Count |
|--------|-------|
| Spec Files | 10 markdown |
| JSON Schemas | 4 |
| IETF Status | Submitted (draft-cowles-volt-00) |
| Event Types | 7 |

## Key Specs
- **Model:** Tamper-evident trace events, cryptographically chained, append-only
- **Bundle format:** manifest.json + events.ndjson + optional attachments/
- **Event types:** message, policy_decision, tool_call, human_approval, file_operation, network_request, model_response
- **Verification:** Independent hash chain validation, INTACT/BROKEN/PARTIAL status
- **Lifecycle:** record → finalize → export → verify

## Schemas
| Schema | Purpose |
|--------|---------|
| event.schema.json | Event structure |
| manifest.schema.json | Bundle manifest |
| signature.schema.json | Cryptographic signatures |
| verification-report.schema.json | Verification results |
