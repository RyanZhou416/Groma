# Contribution Guidelines

> **Status**: pre-alpha. The contract is not frozen, so large changes to public API shape are
> expected. Please open an issue before starting substantial work.

---

## 1 Before you write code

1. Read [`Docs/SCOPE.md`](Docs/SCOPE.md). If your change is not permitted by the layers and admission
   criteria there, the correct first step is a scope discussion — not a pull request.
2. Read the relevant public specification. Behaviour is defined by RFC / NIST / W3C / ISO documents,
   not by another language's implementation.
3. Check the licence of every dependency and reference you consult.

## 2 Non-negotiable rules

| Rule | Reason |
|------|--------|
| Never implement a curve, bignum arithmetic, hash function, or RNG from scratch | Use vetted crates; record their review boundaries instead |
| Never port code from another language into this repository | Establish the specification, vectors, and licence first |
| Every untrusted parsing entry point is bounded before parsing | Size, depth, and element count limits are mandatory |
| Verification must not panic, abort, or leak secrets | Backend panics are caught and reported as backend failure, never as an invalid signature |
| No `unsafe` without a `// SAFETY:` justification | and a reviewer who agrees |
| No algorithm is added silently | Adding an algorithm must never enlarge the accepted surface automatically |

## 3 Required evidence for a cryptographic change

A pull request touching cryptography is accepted only with:

- **Test vectors** from a public corpus (ACVP/CAVP, Wycheproof, W3C, RFC KATs) and a note of which
  corpus and version.
- **A negative test**: at least one invalid input that must be rejected.
- **A fuzz target** if the change adds or modifies a parsing entry point, plus a regression entry if
  it fixes a crash.
- **A differential note** if the change affects behaviour that the OpenSSL differential harness
  covers.
- **Licence and provenance** for any new dependency or vendored vector set.

Machine-decidable evidence only. Screenshots, manual logins, and single-browser demonstrations are
not security gates.

## 4 Documentation language

- Narrative documents under `Docs/` are written in **Chinese**.
- `README.md` is the English entry point and may also be bilingual.
- Code, identifiers, paths, commands, and file/folder names are **English**.
- Commit messages: English, imperative mood.

## 5 Commit and branch conventions

- Keep history linear on `main`; use short-lived topic branches.
- One logical change per commit; documentation-only changes in their own commit.
- Do not commit generated artefacts, local vector overrides, or scratch files.

## 6 Review

- Every cryptographic change needs review by someone other than the author.
- A reviewer's job includes attempting to falsify the change, not confirming it.
- Reviewers must state residual uncertainty explicitly rather than approving silently.

## 7 What will be rejected

- Claims of "production safe", "audited", "constant-time", or "FIPS compliant" without evidence that
  supports the exact claim.
- Formal-verification claims stated beyond their actual coverage.
- Scope expansion argued from "another library has it".
- Dependencies that introduce a C/C++ build requirement into `core` or a shipping backend path.
