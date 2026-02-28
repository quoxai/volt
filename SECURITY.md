# Security Policy

## Supported Versions

VOLT is a protocol/spec repo. Security issues here usually relate to:
- reference implementations (if/when included),
- verifier/bundler tooling,
- documentation errors that could cause insecure deployments.

**Supported:** the latest released protocol version and the `main` branch.

---

## Reporting a Vulnerability

If you believe you've found a security vulnerability in VOLT (spec, examples, tooling, or guidance):

1) **Do not open a public GitHub issue** if it includes exploit details, sensitive data, or guidance that could be abused.

2) Instead, report privately:
- **Email:** security@quox.ai
- **Subject:** `VOLT Security Report`

Include:
- A clear description of the issue and impact
- Steps to reproduce (as safely as possible)
- Affected files/sections (links if possible)
- Any suggested fix or mitigation
- Your preferred attribution name (optional)

If email is not possible, you may open a GitHub issue titled:
- `SECURITY: Private disclosure requested`
…and include **no details**. We'll respond with a private channel.

---

## What to Expect

We aim to:
- Acknowledge receipt within **72 hours**
- Provide an initial assessment and next steps
- Coordinate disclosure timing if needed
- Credit reporters who want attribution (optional)

---

## Security Philosophy (VOLT-specific)

VOLT is an evidence protocol. Many "security issues" are actually **privacy/safety** issues such as:
- accidental logging of secrets
- unsafe examples that encourage recording sensitive payloads
- verifier weaknesses that allow tampering to go undetected

If you spot anything that could cause:
- secret leakage,
- weak evidence integrity,
- misleading verification outcomes,
please report it.

---

## Safe Harbor

We support good-faith security research and responsible disclosure.
Please avoid:
- accessing data you do not own or have permission to access,
- disrupting systems,
- publishing exploit details before coordination.
