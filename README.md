# VOLT — Verifiable Operations Ledger & Trace

VOLT is Quox's **evidence layer**.

It produces **tamper-evident, portable, verifiable traces** of what happened during an agent run — across agents, tools, approvals, and policy decisions — without turning your system into a logging swamp or leaking sensitive data.

If **AEE** is how agents communicate, and **AOCL** is how they're controlled, then **VOLT** is how you **prove** what actually happened.

**IETF Internet-Draft:** [`draft-cowles-volt-00`](https://datatracker.ietf.org/doc/draft-cowles-volt/)

---

## Why VOLT exists

Modern agent systems are powerful, but they're often a **black box**:

- "Who approved that change?"
- "What tool was executed, with what inputs?"
- "Did the log get edited after the incident?"
- "Can we reproduce what happened and explain it to an auditor?"

VOLT answers those questions by generating **Evidence Bundles** that can be verified independently.

---

## What VOLT is (in one line)

A **cryptographically chained event log** + **portable run bundle** + **verifier**.

VOLT does **not** require a blockchain.
(Optionally, you can later anchor bundle hashes to a chain as a "public notary stamp".)

---

## Where VOLT fits

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
```

### Responsibilities (clear separation)
- **[AEE](https://github.com/quoxai/aee)**: message format + correlation IDs for agent-to-agent and human-to-agent envelopes.
- **[AOCL](https://github.com/quoxai/aocl)**: orchestration control layers (policy decisions, permissions, HITL gates, escalation rules).
- **VOLT**: evidence recording + integrity guarantees + exportable bundles + verification.

### VOLT explicitly does NOT
- Replace AEE messaging
- Decide policy (that's AOCL)
- Store secrets by default
- Act as a database, SIEM, or analytics warehouse
- Guarantee truth if the host is fully compromised (it guarantees **tamper-evidence**, not miracles)

---

## Core features (v0.1)

### 1) Tamper-evident trace events
Every meaningful action becomes an event:
- message in/out
- policy allow/deny
- tool call requested/executed
- human approval granted/denied
- file read/write (metadata only)
- network request (metadata only)
- model response produced (metadata + references)

Each event is hashed and **linked to the previous event** (an append-only chain).

### 2) Evidence Bundles (portable)
A run can be exported as a bundle:
- `manifest.json`
- `events.ndjson`
- optional `attachments/` (tool outputs, artifacts, snapshots)
- optional redaction maps / signatures

### 3) Independent verification
A verifier can check:
- event hashes are correct
- the chain hasn't been modified
- referenced attachments exist and match hashes
- optional signatures/attestations are valid

---

## Quickstart (conceptual)

### Record
Your orchestrator/runner emits VOLT events at key points:
- when an AEE envelope is received/sent
- when AOCL makes a policy decision
- before/after tool execution
- at HITL checkpoints

### Bundle
At the end of the run (or periodically), you finalize a bundle:
- events are written as NDJSON
- attachments are stored by content hash
- manifest links everything together

### Verify
Anyone can run verification on the bundle:
- PASS/FAIL with a clear report of what broke (if anything)

---

## Use cases

- **Enterprise audit & compliance** (prove approvals, actions, and policy gates)
- **Incident reconstruction** (what happened, in order, with trace integrity)
- **Marketplace trust** (plugins/workflows can be "VOLT-compatible")
- **Human-in-the-loop proof** (approvals become verifiable events)
- **Debugging** (replay/compare runs is a roadmap feature)

---

## Design principles

- **Privacy by default**: do not log secrets; log metadata + references.
- **Minimal but extensible**: small schema, versioned, forward-compatible.
- **Separation of concerns**: AEE = envelope, AOCL = control, VOLT = evidence.
- **Portable evidence**: bundles are self-contained and exportable.
- **Verifier-first**: if it can't be verified independently, it doesn't count as evidence.

---

## Status

- **Protocol Version:** VOLT v0.1 (draft)
- **Scope:** record → bundle → verify
- **Roadmap:** deterministic replay harness, TraceQL queries, remote runner attestations, optional public anchoring

See: [SPEC.md](SPEC.md), [EVIDENCE_BUNDLES.md](EVIDENCE_BUNDLES.md), [VERIFICATION.md](VERIFICATION.md), [INTEGRATION.md](INTEGRATION.md)

---

## Documents

- **[SPEC.md](SPEC.md)** — normative protocol spec (schema, hashing, chaining, versioning)
- **[EVIDENCE_BUNDLES.md](EVIDENCE_BUNDLES.md)** — bundle layout, attachments, retention
- **[VERIFICATION.md](VERIFICATION.md)** — verifier algorithm + report format
- **[INTEGRATION.md](INTEGRATION.md)** — how VOLT hooks into AEE + AOCL + tool execution
- **[PRIVACY_REDACTION.md](PRIVACY_REDACTION.md)** — logging safety + redaction rules
- **[THREAT_MODEL.md](THREAT_MODEL.md)** — what VOLT mitigates and what it cannot
- **[WORKED_EXAMPLES.md](WORKED_EXAMPLES.md)** — end-to-end sample runs
- **[ROADMAP.md](ROADMAP.md)** — v0.2+ features and explicit non-goals
- **[SECURITY.md](SECURITY.md)** — vulnerability reporting policy
- **[CONTRIBUTING.md](CONTRIBUTING.md)** — contribution guidelines
- **[CHANGELOG.md](CHANGELOG.md)** — version history

---

## Naming

**VOLT** stands for **Verifiable Operations Ledger & Trace**.

It's a "flight recorder" for agentic workflows — with cryptographic integrity.
