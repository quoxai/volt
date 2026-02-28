# VOLT Evidence Bundles (v0.1)

This document defines the **portable Evidence Bundle** format for VOLT v0.1.

An Evidence Bundle is a self-contained package that allows a third party (or future-you) to:
- inspect what happened in a run
- verify the trace has not been tampered with
- validate referenced attachments
- optionally validate signatures/attestations

VOLT bundles are designed for:
- audit and compliance evidence
- incident reconstruction
- workflow/plugin accountability
- portability across systems

---

## 1) Bundle goals

A VOLT Evidence Bundle MUST be:
- **Portable**: transferable as a folder or a zip file.
- **Self-describing**: contains a manifest explaining what's inside.
- **Verifiable**: contains an event chain + references that can be checked.
- **Privacy-aware**: safe by default; sensitive payloads should be excluded/redacted.
- **Incremental-friendly**: supports rolling bundles (periodic snapshots) and final bundles.

---

## 2) Bundle formats

VOLT supports two equivalent representations:

### 2.1 Folder format (recommended for internal systems)
A directory structure on disk.

### 2.2 Zip format (recommended for export/sharing)
A zip archive containing the same directory layout.

**Rule:** The bundle contents MUST be identical regardless of container (folder vs zip).

---

## 3) Required files

Every bundle MUST contain:

1) `manifest.json`
2) `events.ndjson`

These file names MAY be customized via `manifest.events_file`, but the defaults are recommended.

---

## 4) Recommended bundle layout

```text
<bundle_root>/
  manifest.json
  events.ndjson

  attachments/                # optional
    ab/
      ab12...<hash>           # content-addressed blobs

  signatures/                 # optional (if you store signatures separately)
    sig-1.json

  redactions/                 # optional
    redactions.json

  notes/                      # optional
    notes.md
```

---

## 5) manifest.json (bundle index)

The manifest is the index and summary record for the bundle.

### 5.1 Minimum required fields (from SPEC)

- `volt_version`
- `bundle_id`
- `run_id`
- `created_ts`
- `hash_alg` (`sha256` in v0.1)
- `events_file`
- `event_count`
- `first_event_hash`
- `last_event_hash`

### 5.2 Recommended fields (strongly suggested)

- `bundle_mode`: `"rolling"` or `"final"`
- `cutoff_ts` (for rolling)
- `correlation_id` (AEE correlation id)
- `producer` object: `{ name, version, build }`
- `redactions_present`: boolean
- `attachments_present`: boolean
- `integrations`: summary IDs for AEE/AOCL references

Example `producer`:

```json
"producer": {
  "name": "quox-core",
  "version": "0.9.3",
  "build": "git:8f3a3b1"
}
```

---

## 6) events.ndjson (the trace)

`events.ndjson` contains one JSON object per line.

Rules:

- UTF-8
- must be ordered by `seq` ascending
- each event MUST include `hash` and `prev_hash`
- events MUST obey the chain rule defined in `SPEC.md`

---

## 7) Attachments (optional but common)

Attachments are blobs referenced by events (e.g., tool outputs, generated artifacts, snapshots, reports).

### 7.1 Content-addressing rule

Attachment files MUST be stored by hash:

- Use SHA-256 over raw bytes
- Store file at:
  - `attachments/<first2>/<hash>`

Example:

- hash: `ab12...ff`
- path: `attachments/ab/ab12...ff`

### 7.2 Attachment metadata

Events that reference attachments SHOULD include, per attachment:

- `hash_alg` (sha256)
- `hash` (hex)
- `content_type` (MIME type)
- `label` (human-friendly)

Example attachment reference:

```json
"attachment_refs": [
  {
    "hash_alg": "sha256",
    "hash": "ab12...ff",
    "content_type": "text/plain",
    "label": "stdout"
  }
]
```

### 7.3 What belongs as an attachment

**Good candidates:**

- tool stdout/stderr (sanitized)
- JSON tool results (sanitized)
- generated artifacts (reports, configs, exports)
- screenshots or UI captures (if allowed)
- policy evaluation explanations (if non-sensitive)
- replay fixtures (future)

**Avoid by default:**

- raw prompts that include secrets
- raw HTTP headers/cookies
- raw database dumps
- private keys or tokens
- full file contents of sensitive locations (e.g., `/etc/shadow`)
- personal data unless required and approved

### 7.4 Size management

Attachments can be large. Implementations SHOULD support:

- max attachment size (configurable)
- truncation with a "truncated" marker in payload
- external storage references (future; not v0.1)

---

## 8) Redactions

Redaction is how VOLT stays privacy-first while still remaining useful.

### 8.1 Redaction strategy (v0.1)

In v0.1, redaction is simple and explicit:

- omit sensitive fields
- replace with placeholders like:
  - `"redacted": true`
  - `"inputs_redacted": true`

Example:

```json
"payload": {
  "tool_name": "shell",
  "operation": "write_file",
  "inputs_redacted": true
}
```

### 8.2 redactions.json (optional)

If your system redacts data, you MAY include `redactions/redactions.json` to describe:

- which event IDs had redactions
- which fields were removed
- the redaction reason category (e.g., `secret`, `pii`, `policy`)

Example:

```json
{
  "volt_version": "0.1",
  "run_id": "01HZ...RUN",
  "items": [
    {
      "event_id": "01HZ...A6",
      "fields_removed": ["payload.inputs"],
      "reason": "secret"
    }
  ]
}
```

**Note:** v0.1 does not require cryptographic "proof of redaction correctness". That's a roadmap feature.

---

## 9) Signatures & attestations

Bundles MAY include signatures either:

- inline in `manifest.json` under `signatures`, or
- as files under `signatures/`

### 9.1 Recommended approach (v0.1)

Keep it simple:

- store signatures in `manifest.json`
- or store each as `signatures/sig-<n>.json` with the same schema

Verifiers SHOULD verify signatures if present.

---

## 10) Rolling vs final bundles

### 10.1 Rolling bundle

Use rolling bundles when you want:

- periodic evidence checkpoints
- long-running workflows
- reduced loss if a run crashes mid-way

Rolling bundle SHOULD include:

- `bundle_mode: "rolling"`
- `cutoff_ts`
- valid `last_event_hash` at cutoff

### 10.2 Final bundle

Final bundle SHOULD include:

- `bundle_mode: "final"`
- a terminal event (`run.completed`, `run.failed`, `run.cancelled`)
- complete event chain from seq=1 to terminal

---

## 11) Retention & storage recommendations (practical)

VOLT bundles are operational evidence. Treat them like logs — but often more sensitive.

Recommended practice:

- store bundles in encrypted storage (at rest)
- restrict access by role (audit vs operator vs developer)
- define retention by environment:
  - dev: short retention
  - staging: medium
  - production: policy-driven
- support "export for audit" as a controlled action

---

## 12) Export/import expectations

### 12.1 Export

Exporting a bundle SHOULD:

- produce a zip (if leaving the system)
- include a verification report if requested (see `VERIFICATION.md`)
- include only allowed attachments (respect privacy policies)

### 12.2 Import

Importing a bundle SHOULD:

- verify it immediately
- quarantine failed bundles
- record provenance metadata (who imported, when)

---

## 13) Minimal implementation checklist

To be "VOLT-B (Bundler) conformant":

- [ ] Create `manifest.json` with required fields
- [ ] Write `events.ndjson` ordered by `seq`
- [ ] Store attachments content-addressed
- [ ] Ensure verifier can re-hash and confirm attachment hashes
- [ ] Keep secrets out of payloads
