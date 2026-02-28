# VOLT Verification (v0.1)

This document defines how a VOLT verifier validates an Evidence Bundle.

Verification is the heart of VOLT: if a bundle can't be verified independently, it isn't evidence.

---

## 1) What verification guarantees

A successful verification means:

- Events are well-formed for VOLT v0.1
- Each event hash matches the event content
- Each event links correctly to the previous event (`prev_hash`)
- The manifest matches the bundle contents (count + endpoints)
- Any referenced attachments exist and match their hashes
- Any included signatures (if present) validate for the declared scope/message

In short: **the trace has integrity** and **the bundle is complete**.

---

## 2) What verification does NOT guarantee

Verification does not prove:

- the host/runner was uncompromised during execution
- the tool results are "true" beyond what was recorded
- the AI model's reasoning was correct
- the run complied with external laws/policies not recorded as events

VOLT is a **tamper-evidence and chain-of-custody** protocol, not an oracle.

---

## 3) Verifier modes

A verifier SHOULD support two modes:

### 3.1 Strict mode (enterprise/audit)
- Fail on any schema deviation
- Fail on missing required fields
- Fail on `seq` gaps or non-monotonic `seq`
- Fail if manifest fields don't match exactly
- Fail if attachments referenced are missing

### 3.2 Permissive mode (debug/dev)
- Warn on `seq` gaps if chain integrity is valid
- Warn on unknown optional fields
- Allow extra files in bundle
- Still FAIL on:
  - hash mismatches
  - chain breaks
  - invalid JSON
  - missing required fields

---

## 4) Verification inputs & outputs

### 4.1 Inputs
- Bundle root directory OR zip file
- Optional flags:
  - `--strict` (default true recommended for production)
  - `--no-attachments` (verify chain only)
  - `--no-signatures` (skip signature checks)
  - `--report json|text` (default json recommended)

### 4.2 Outputs
- PASS/FAIL/ERROR result
- Machine-readable report (recommended JSON)
- Exit code:
  - `0` PASS
  - `1` FAIL (verification failed)
  - `2` ERROR (IO, parse error, bundle not readable)

---

## 5) Verification algorithm (normative)

### Step 0 — Load bundle
1. Locate `manifest.json`
2. Read and parse JSON (UTF-8)
3. Validate required manifest fields exist (see SPEC §16)

If missing/invalid → **ERROR**

---

### Step 1 — Load events
1. Read `events_file` from manifest (default `events.ndjson`)
2. Parse as NDJSON: one JSON object per line
3. Collect events in file order

If any line is not valid JSON → **FAIL** (`INVALID_EVENT_JSON`)

---

### Step 2 — Validate event ordering
1. Ensure events are ordered by `seq` ascending
2. Ensure `seq` starts at 1 (recommended)
3. Ensure `seq` is monotonically increasing

Strict mode:
- FAIL if `seq` does not start at 1
- FAIL if any gap exists (`SEQ_GAP`)
- FAIL if duplicates (`SEQ_DUPLICATE`)
- FAIL if decreasing (`SEQ_NOT_MONOTONIC`)

Permissive mode:
- WARN on gaps
- FAIL on duplicates or decreasing

---

### Step 3 — Validate required event fields
For each event, verify required fields exist and types match:
- `volt_version` (string)
- `event_id` (string)
- `run_id` (string)
- `ts` (string)
- `seq` (int)
- `event_type` (string)
- `actor` (object)
- `context` (object)
- `payload` (object)
- `prev_hash` (string)
- `hash` (string)

Missing/invalid → **FAIL** (`EVENT_SCHEMA_INVALID`)

---

### Step 4 — Validate version compatibility
- `manifest.volt_version` MUST match `event.volt_version` for all events

Mismatch → **FAIL** (`VERSION_MISMATCH`)

---

### Step 5 — Recompute and validate event hashes (core)
For each event:
1. Create a copy of the event object with `hash` removed
2. Serialize using **Canonical JSON** rules (SPEC §3.2)
3. Compute `SHA-256` digest of canonical bytes
4. Compare computed hex digest to stored `event.hash`

Mismatch → **FAIL** (`EVENT_HASH_MISMATCH`)

---

### Step 6 — Validate the chain
For the first event (`seq=1`):
- `prev_hash` MUST be 64 hex zeros (`"0000000000000000000000000000000000000000000000000000000000000000"`)

Else → **FAIL** (`INVALID_GENESIS_PREV_HASH`)

For each subsequent event:
- `event.prev_hash` MUST equal previous event's `hash`

Mismatch → **FAIL** (`CHAIN_BROKEN`)

---

### Step 7 — Validate run_id consistency
- All events MUST have the same `run_id` as `manifest.run_id`

Mismatch → **FAIL** (`RUN_ID_MISMATCH`)

---

### Step 8 — Validate manifest counts and endpoints
1. Count events in file: must equal `manifest.event_count`
2. Confirm:
   - `manifest.first_event_hash` == hash of first event
   - `manifest.last_event_hash` == hash of last event

Mismatch → **FAIL** (`MANIFEST_MISMATCH`)

---

### Step 9 — Validate attachments (if enabled)
If attachment verification is enabled:

For each event, locate any `payload.attachment_refs` entries.
For each attachment ref:
1. Locate file under `attachments/<first2>/<hash>`
2. Read raw bytes
3. Compute SHA-256
4. Compare to referenced `hash`

Missing file → **FAIL** (`ATTACHMENT_MISSING`)
Hash mismatch → **FAIL** (`ATTACHMENT_HASH_MISMATCH`)

If `--no-attachments` is used:
- verifier MAY output `attachments_verified=false` and warn if refs exist

---

### Step 10 — Validate signatures (if present and enabled)
If signatures exist (manifest `signatures` or files under `signatures/`):

For each signature record:
1. Validate signature record schema
2. Reconstruct the signature `message` object exactly as defined (SPEC §15.1)
3. Verify signature bytes using declared algorithm (`ed25519` recommended)
4. Ensure scope is supported (v0.1 supports `"bundle"`)

Invalid signature → **FAIL** (`SIGNATURE_INVALID`)

If `--no-signatures`:
- signatures are ignored and `signatures_verified=false`

---

## 6) Verification report format (recommended JSON)

### 6.1 PASS report

```json
{
  "result": "PASS",
  "run_id": "01HZ...RUN",
  "bundle_id": "01HZ...BUNDLE",
  "volt_version": "0.1",
  "hash_alg": "sha256",
  "event_count": 128,
  "first_event_hash": "<hash1>",
  "last_event_hash": "<hashN>",
  "attachments_verified": true,
  "signatures_verified": false,
  "warnings": []
}
```

### 6.2 FAIL report

```json
{
  "result": "FAIL",
  "reason": "CHAIN_BROKEN",
  "details": {
    "seq": 42,
    "event_id": "01HZ...E42",
    "prev_hash_found": "<prev_hash>",
    "prev_hash_expected": "<previous_event_hash>"
  }
}
```

### 6.3 ERROR report (IO/parse)

```json
{
  "result": "ERROR",
  "reason": "MANIFEST_UNREADABLE",
  "details": {
    "message": "manifest.json missing or invalid JSON"
  }
}
```

---

## 7) Standard failure reasons (recommended list)

Verifiers SHOULD use consistent reason codes:

- `MANIFEST_MISSING`
- `MANIFEST_UNREADABLE`
- `MANIFEST_SCHEMA_INVALID`
- `EVENTS_FILE_MISSING`
- `INVALID_EVENT_JSON`
- `EVENT_SCHEMA_INVALID`
- `VERSION_MISMATCH`
- `RUN_ID_MISMATCH`
- `SEQ_NOT_MONOTONIC`
- `SEQ_DUPLICATE`
- `SEQ_GAP`
- `EVENT_HASH_MISMATCH`
- `INVALID_GENESIS_PREV_HASH`
- `CHAIN_BROKEN`
- `MANIFEST_MISMATCH`
- `ATTACHMENT_MISSING`
- `ATTACHMENT_HASH_MISMATCH`
- `SIGNATURE_SCHEMA_INVALID`
- `SIGNATURE_INVALID`
- `UNSUPPORTED_SIGNATURE_TYPE`

---

## 8) Common scenarios & what they mean

### 8.1 Someone edited an event

- You'll see `EVENT_HASH_MISMATCH` at the modified event.

### 8.2 Someone deleted an event

- You'll see `SEQ_GAP` (strict) and/or `CHAIN_BROKEN`.

### 8.3 Someone inserted an event in the middle

- You'll see `CHAIN_BROKEN` unless they re-hashed all subsequent events.

### 8.4 Someone re-hashed everything

- Without signatures, this can be harder to detect.
- With signatures/attestations from a trusted key, re-hashing will fail signature verification.

This is why signing is a key roadmap milestone for strong non-repudiation.

---

## 9) CI integration recommendations

A VOLT verifier is designed to be automation-friendly.

Example CI policy:

- require PASS in strict mode for production runs
- store verification reports alongside build artifacts
- fail the pipeline if:
  - any run touching `prod` lacks `hitl.approved` event
  - any run includes `tool.call.executed` without a preceding `aocl.policy.evaluated` event

(Note: Those policies are AOCL-level governance checks; VOLT is the evidence substrate.)

---

## 10) Minimal CLI interface (suggested)

Examples:

```bash
volt-verify ./bundle.zip --strict --report json
volt-verify ./bundle/ --no-attachments --report text
```

Exit codes:

- `0` PASS
- `1` FAIL
- `2` ERROR
