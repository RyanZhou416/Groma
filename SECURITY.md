# Security Policy

> **Status**: pre-alpha. There is no released code and no security guarantee of any kind.

---

## No guarantees at this stage

Groma is not audited. Nothing in this repository has been independently reviewed, and no crate here
should be used to protect real secrets. The project makes no claim of being production-safe, and no
claim of FIPS 140-3 compliance — that is an explicit non-goal (see `../SCOPE.md` §2.3).

## Reporting a vulnerability

**Do not open a public issue for a security problem.**

Use GitHub's private vulnerability reporting on this repository
(*Security* → *Report a vulnerability*), or contact the maintainer directly through the GitHub
account that owns this repository.

Please include:

- The affected crate, version, and commit.
- A description of the impact, including whether key material, credentials, or plaintext can be
  recovered, and whether the issue is remotely triggerable.
- A minimal reproduction, if one exists.
- Any suggested fix or mitigation.

## What to expect

| Stage | Commitment |
|-------|-----------|
| Acknowledgement | Best effort, within a few days |
| Assessment | Severity, affected versions, and whether the issue is in Groma code or in a dependency |
| Fix and disclosure | Coordinated; details published after a fix is available |
| Dependency issues | Reported upstream where applicable; Groma records the advisory and the affected range |

This is a small project without a dedicated security team. Timelines are honest intentions, not
service levels.

## Scope of reports

In scope:

- Groma's own crates: `groma-core`, `groma-codec`, and later backend, format, trust, and platform
  crates.
- The provider contract itself: algorithm-confusion, acceptance-surface, or canonicalisation flaws.
- Any case where a Groma API accepts input it documents as invalid, or rejects input the relevant
  specification requires it to accept.

Out of scope:

- Vulnerabilities in third-party dependencies. Report those upstream; tell us as well so the
  advisory can be tracked.
- The absence of FIPS validation, which is a documented non-goal.
- Missing algorithms listed as deferred in `../SCOPE.md` §6.
- Denial of service through deliberately unbounded use of a documented low-level API by a caller who
  bypassed the bounded entry points.

## Hardening expectations for contributors

- Bound every untrusted input before parsing.
- Fail closed; never treat a backend failure as "valid".
- Keep error output free of key material, credential bytes, and plaintext.
- Use constant-time comparison primitives for all tag and MAC checks.
