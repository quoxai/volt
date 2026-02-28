# Contributing to VOLT (Verifiable Operations Ledger & Trace)

Thanks for helping improve VOLT.

VOLT is a **protocol/specification** repo. The goal is interoperability, clarity, and security — not endless features.

If you're proposing a change, treat it like changing a public standard: small, careful, testable.

---

## What this repo contains

- The VOLT protocol specification (`SPEC.md`)
- Supporting documentation (bundles, verification, integration, privacy, threat model)
- Worked examples and roadmap
- (Optional, if present) reference tooling such as verifier/bundler utilities

---

## Ground rules

### 1) Privacy-first is non-negotiable
Any change that increases the chance of recording secrets/PII will be rejected unless it includes strong guardrails.

See: `PRIVACY_REDACTION.md`

### 2) Verifier-first
If a change can't be verified independently, it's not evidence.

Any change that affects hashing/canonicalization MUST include:
- updated spec language
- updated examples
- clear verifier behavior

### 3) Separation of concerns
- AEE = envelopes/messages
- AOCL = policy/control/HITL
- VOLT = evidence integrity & portability

VOLT should not absorb policy logic, orchestration logic, or observability tooling.

### 4) Keep it small
If you propose a feature that belongs on the roadmap, we'll ask you to move it to `ROADMAP.md` first.

---

## How to propose changes

### Option A — Small doc fixes (typos/clarity)
1. Open a PR
2. Clearly state what changed and why
3. No spec version bump required for pure clarifications

### Option B — Spec changes (schema/behavior)
1. Open a GitHub issue first describing:
   - problem statement
   - proposed solution
   - impact on existing implementations
2. Then open a PR that includes:
   - changes to `SPEC.md`
   - changes to `WORKED_EXAMPLES.md` (if relevant)
   - changes to verifier rules in `VERIFICATION.md` (if relevant)
   - changelog entry (see below)

---

## Versioning & changelog rules

VOLT uses simple protocol versioning:

- **Patch**: wording/clarifications only (no behavioral change)
- **Minor**: backwards-compatible additions (new optional fields, new event types, new docs)
- **Major**: breaking changes (field requirements change, canonicalization/hash rules change)

If your PR changes behavior, update:
- `CHANGELOG.md` (add an Unreleased section if needed)
- and any impacted docs/examples.

---

## "Definition of done" for spec-impacting PRs

A spec PR is considered complete when it includes:

- [ ] Clear rationale and scope
- [ ] Normative language in `SPEC.md` (MUST/SHOULD/MAY)
- [ ] Compatibility notes (what breaks, what doesn't)
- [ ] Updated examples (where applicable)
- [ ] Updated verification rules (where applicable)
- [ ] Updated roadmap (if it's a future milestone)
- [ ] Updated changelog entry

---

## Event types & extensions

- Standard event types should remain small and stable.
- New standard event types should only be added when multiple implementations benefit.
- Vendor-specific event types should use a namespace prefix, e.g.:
  - `quox.marketplace.offer.created`
  - `vendorx.routeros.script.deployed`

---

## Security issues

If your contribution relates to a security or privacy vulnerability, please follow `SECURITY.md` and report privately before opening a public issue/PR.

---

## Code contributions (if tooling exists)

If this repo contains reference tooling (e.g., `volt-verify`):

- Keep dependencies minimal
- Prefer clear, readable code over clever code
- Include tests or golden test vectors where feasible
- Do not introduce behavior that logs secrets

---

## License

By contributing, you agree that your contributions will be licensed under the repo's `LICENSE`.
