# VOLT Threat Model (v0.1)

This document describes the threats VOLT is designed to mitigate, the threats it does not (and cannot) fully mitigate, and the practical mitigations recommended for enterprise deployments.

VOLT is an **evidence integrity** protocol: it provides **tamper-evident** traces and portable verification, not perfect truth in adversarial environments.

---

## 1) Security objectives

VOLT v0.1 aims to provide:

1. **Integrity of recorded traces**
   - Detect modification, deletion, or insertion of events after the fact.

2. **Accountability / non-repudiation (partial in v0.1)**
   - Provide strong accountability when optional signatures are used.

3. **Chain of custody**
   - Enable portable Evidence Bundles that can be verified independently.

4. **Auditability of approvals and actions**
   - Prove sequencing such as "AOCL required HITL → human approved → tool executed".

5. **Privacy safety by default**
   - Avoid VOLT becoming a secret/PII leakage system (covered further in `PRIVACY_REDACTION.md`).

---

## 2) Assets to protect

VOLT-related assets include:

- **Trace integrity**
  - `events.ndjson` chain, event hashes, `prev_hash` linkage

- **Evidence Bundle integrity**
  - `manifest.json` correctness, endpoints, counts

- **Attachments integrity**
  - tool outputs, artifacts referenced by content hashes

- **Provenance**
  - linkage to AEE IDs, AOCL policy/decision IDs, HITL approvals

- **Signing keys (if used)**
  - keys used for attestations/signatures are high-value

- **Privacy**
  - preventing secrets/PII from being recorded or exported improperly

---

## 3) Trust boundaries

Typical Quox/VOLT trust boundaries:

- **User/UI boundary**
  - user input enters system via AEE gateway

- **Orchestrator boundary**
  - AOCL decisions + agent routing + tool invocation decisions

- **Tool execution boundary**
  - local tools, remote runners, SSH endpoints, cloud APIs

- **Storage boundary**
  - where bundles and attachments are stored (disk/object store)

- **Export boundary**
  - evidence leaves system (zip export, audit handoff)

Each boundary is a place attackers may attempt tampering, substitution, or data exfiltration.

---

## 4) Threat actors

- **Curious insider**
  - tries to view sensitive data via logs/evidence bundles

- **Malicious insider**
  - tries to alter records to hide actions or fabricate approvals

- **Compromised runner/tool host**
  - attacker controls the system executing tools

- **External attacker**
  - tries to exploit web/API surface, steal keys, or inject false evidence

- **Plugin/workflow author**
  - provides a malicious plugin or tries to evade accountability

---

## 5) Threats VOLT mitigates (v0.1)

### T1 — Post-hoc tampering of events
**Attack:** Modify an event after the run (e.g., change "denied" to "approved").
**Mitigation:** Event hash validation fails (`EVENT_HASH_MISMATCH`).
**Residual risk:** None, unless attacker re-hashes the chain and you have no signatures.

### T2 — Deleting events from the middle
**Attack:** Remove a tool execution event to hide activity.
**Mitigation:** Chain breaks (`CHAIN_BROKEN`) and/or seq gap detection.
**Residual risk:** Same as T1 if attacker rebuilds chain and no signatures exist.

### T3 — Inserting fake events
**Attack:** Insert a fake `hitl.approved` event.
**Mitigation:** Chain breaks unless attacker re-hashes all subsequent events.
**Residual risk:** If attacker can re-hash everything and there are no trusted signatures, they can fabricate a new "consistent" chain.

### T4 — Attachment substitution
**Attack:** Replace tool output attachment with a sanitized/fake version.
**Mitigation:** Attachment hash validation fails (`ATTACHMENT_HASH_MISMATCH`).
**Residual risk:** If attacker can change both attachment and referenced hash by rebuilding events, see T1/T2/T3.

### T5 — Repudiation of approvals and actions (partial in v0.1)
**Attack:** "That approval wasn't mine."
**Mitigation:** If actor identity and/or signatures are used, repudiation is harder.
**Residual risk:** Without signatures/strong identity, v0.1 provides sequencing evidence but weaker non-repudiation.

### T6 — "Silent redaction" / deceptive logging
**Attack:** Hide sensitive tool actions by omitting them silently.
**Mitigation:** VOLT requires explicit redaction flags; missing events remain a governance and detection problem.
**Residual risk:** You can't prove something was recorded if it was never recorded; AOCL/monitoring policies should enforce required event types.

---

## 6) Threats VOLT does NOT fully mitigate (and why)

### T7 — Fully compromised host/runner (truth problem)
**Attack:** Attacker controls runner; it emits a clean but false VOLT trace.
**Why VOLT can't solve alone:** The recorder is running in the compromised environment.
**Mitigations (recommended):**
- Remote runner attestations (roadmap)
- Secure enclaves / TPM-backed keys (future)
- Cross-signing from orchestrator + runner
- Separate logging channel to an append-only store
- AOCL enforcement + network segmentation + least privilege

### T8 — Key theft (signing keys)
**Attack:** Attacker steals signing key; can sign forged bundles.
**Mitigations:**
- store keys in HSM/TPM where possible
- use short-lived keys (rotation)
- maintain key revocation lists
- separate keys per environment (dev/stage/prod)
- require multi-sig for high-risk runs (future)

### T9 — Incomplete evidence (missing coverage)
**Attack:** A tool executes without VOLT hooks; evidence is incomplete.
**Mitigation:** Not a cryptographic issue — it's integration discipline.
**Mitigations:**
- enforce "tool calls must go through middleware" (hard requirement)
- AOCL policy: deny execution if VOLT recorder not active
- CI checks for required event types per run class (prod runs must include HITL)

### T10 — Privacy leakage via evidence bundles
**Attack:** Sensitive data leaks because it got logged or exported.
**Mitigation:** Privacy-first constraints + scanners + redaction policy.
**Mitigations:**
- secret scanning before write
- strict export controls
- role-based access to evidence viewers
- configurable retention and automatic deletion

---

## 7) Security controls & mitigations (recommended)

### 7.1 Controls inside VOLT (protocol level)
- Hash-chained events (tamper-evidence)
- Content-addressed attachments
- Optional signatures/attestations
- Explicit redaction markers

### 7.2 Controls in Quox (system level)
- AOCL policies:
  - require HITL for risky tool actions
  - require VOLT active to execute sensitive tools
  - require "verification PASS" before finalizing prod runs
- Tool middleware enforcement:
  - no direct tool access bypassing recorder
- Access controls:
  - restrict bundle export to privileged roles
  - limit who can view attachments

### 7.3 Operational controls (enterprise hygiene)
- Store bundles encrypted at rest
- Apply retention policies by environment
- Monitor for secret-scanner triggers
- Rotate keys and isolate signing keys
- Segregate production runners

---

## 8) Residual risk summary

VOLT provides strong detection of **post-hoc tampering** of recorded traces.

The biggest remaining risks are:
- **compromised execution environments**
- **missing integration coverage**
- **key theft**
- **privacy leakage through careless recording**

These are mitigated by:
- strong AOCL enforcement
- tool middleware discipline
- optional signing with protected keys
- privacy/redaction guardrails

---

## 9) Roadmap threats addressed later (not v0.1)

Planned improvements that strengthen the threat posture:

- **Remote runner attestations**
  - runner signs run commitments using hardware-backed keys

- **Cross-signing**
  - orchestrator signs chain endpoints; runner signs tool events

- **Merkle roots / checkpointing**
  - allow efficient proofs and periodic "checkpoints"

- **Optional public anchoring**
  - publish chain endpoints to a public timestamping layer (blockchain/notary)

- **Replay harness**
  - supports deeper investigation and regression testing

---

## 10) Practical checklist (deployments)

Before using VOLT for production-grade evidence:

- [ ] Tool calls cannot bypass middleware
- [ ] Secret scanning and redaction are enabled
- [ ] Evidence bundles stored encrypted and access-controlled
- [ ] Export is restricted and logged
- [ ] AOCL requires HITL for high-risk actions
- [ ] Verifier runs in strict mode for production evidence
- [ ] Optional: signatures enabled with protected keys
