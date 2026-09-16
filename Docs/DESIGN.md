# Design Notes

> **Document type**: design (working notes)
> **Module**: Groma
> **Created**: 2026-09-16
> **Status**: draft — decisions below are proposed, not frozen
> **Related**: [`SCOPE.md`](SCOPE.md) is authoritative; this document elaborates *how*.

本文记录合同层的设计取向与尚未决定的技术选择。与 `SCOPE.md` 冲突时以 `SCOPE.md` 为准。

---

## 1 Contract shape

### 1.1 Traits by family, not by algorithm

One trait per algorithm produces dozens of traits and cannot be extended. Groma groups by family:

| Trait | Covers | Notes |
|-------|--------|-------|
| `Digest` | SHA-2, SHA-3, BLAKE2/3, … | streaming update/finalize |
| `Mac` | HMAC, CMAC, GMAC, KMAC, Poly1305, … | keyed; verify uses constant-time compare |
| `Kdf` | HKDF, PBKDF2, KBKDF, scrypt, Argon2, … | parameters are explicit, never defaulted implicitly |
| `Aead` | AES-GCM, ChaCha20-Poly1305, AES-CCM, AES-OCB3, … | nonce and AAD handling is explicit |
| `Signer` / `Verifier` | ECDSA, RSA (PKCS#1 v1.5, PSS), Ed25519/Ed448, ML-DSA, SLH-DSA | signer and verifier are separate traits |
| `Kem` | ML-KEM, ECDH-based KEM, … | encapsulate/decapsulate |
| `PasswordHash` | Argon2, scrypt, bcrypt, … | distinct from `Kdf` because parameters are policy |
| `Random` | OS RNG access, DRBG | failure is explicit; never silently weakened |

### 1.2 Canonical types

- `Algorithm` — a **closed enum**, not a string. Unknown values fail to parse.
- Key, signature, nonce, tag and ciphertext newtypes, each carrying its algorithm where it matters.
- `Error` — a classified enum distinguishing *invalid signature* from *invalid encoding* from
  *unsupported algorithm* from *backend failure*. The distinction is preserved even when a product
  presents a uniform response.

### 1.3 Capability query

A backend reports what it accepts. Acceptance is explicit configuration; upgrading a dependency must
never enlarge the accepted set on its own. This is the machine-checkable form of the safety property
that generic libraries cannot express.

---

## 2 Open decisions (must be settled before or during P1)

| # | Decision | Options | Notes |
|---|----------|---------|-------|
| D1 | `no_std` posture | (a) `core`+`codec` `no_std`+`alloc`, backends unconstrained (b) `std` everywhere | (a) is proposed; changing later is expensive for embedded consumers |
| D2 | Trait dynamism | (a) static generics only (b) `dyn`-compatible traits | Affects object safety and runtime backend selection |
| D3 | Algorithm enum extensibility | (a) closed non-exhaustive enum (b) open registry | A closed enum is required by contract rule 1; `#[non_exhaustive]` allows growth without breaking |
| D4 | Backend selection | (a) compile-time features (b) runtime registration (c) both | Both is proposed: features for products, registration for tests and differential runs |
| D5 | Error taxonomy depth | (a) coarse (b) fine-grained per family | Rule 4 requires distinguishing signature vs encoding at minimum |
| D6 | Which second backend proves P1 | (a) graviola (b) a minimal in-tree stub (c) libcrux | (b) is cheapest and keeps P1 honest; (a)/(c) depend on availability |
| D7 | Vector storage | (a) vendored in-tree (b) fetched by hash | Licensing and size trade-off; provenance manifest either way |

### 2.1 Environment policy (decided 2026-09-16)

Frozen with the owner while the repository was formalised:

- **MSRV = 1.89** (`rust-version` in the workspace manifest). The floor is set by the ecosystem,
  not by taste: edition 2024 requires 1.85, RustCrypto's current generation (digest/sha2 0.11)
  requires 1.85, and graviola 0.3+ requires 1.89. Anything lower would mean pinning outdated
  dependency versions, which a security library must not do.
- **Development toolchain pinned to 1.98.1** (`rust-toolchain.toml`). Contributors and CI build
  with one exact compiler; the pin is bumped deliberately every few months, never floating.
- **CI keeps the promise.** The matrix builds and tests on 1.89 (MSRV) and 1.98.1 (stable), on
  Ubuntu and Windows. Day-to-day code may use newer language features only if the MSRV leg still
  passes; `resolver = "3"` plus `rust-version` keeps transitive dependencies inside the same floor
  (MSRV-aware resolution).
- **MSRV changes** happen only when (a) an upgraded dependency requires it, or (b) an essential
  language feature requires it; only in a minor release; always with a changelog entry.

---

## 3 Backend notes

### 3.1 RustCrypto (first production candidate)

- Broadest coverage; `no_std`-capable; per-crate audit status varies and must be recorded **per
  crate**, never by organisation.
- Known hard spot: the `rsa` crate carries `RUSTSEC-2023-0071` (Marvin) with `patched = []`, affecting
  private-key operations. Verification uses public keys, but the advisory must be documented and the
  risk explicitly adjudicated rather than hidden.

### 3.2 graviola (second candidate)

- Builds with rustc alone; assembly is hand-transcribed from s2n-bignum into Rust `core::arch::asm!`.
- x86_64 and aarch64 only, with **mandatory CPU features**; missing features panic at runtime rather
  than degrading silently. CPU capability must therefore be an explicit, documented precondition.
- Documented as very new. Proofs cover the mathematics, not memory safety; the point-selection path
  is stated to be unverified.

### 3.3 libcrux (formal-verification lineage)

- Verified extraction from HACL* rather than hand-written Rust.
- Coverage caveat: RSA is **PSS only** — PKCS#1 v1.5 is absent, so RS256 is not covered.
- Pre-1.0 throughout.

### 3.4 OpenSSL (differential only)

- Never a product dependency and never a Groma backend in shipping builds.
- Its value is breadth: it is the only single implementation covering primitives, certificate paths,
  CMS, PKCS#12 and OCSP, which makes it a strong differential oracle.
- Its FIPS provider certificate is FIPS 140-2 (#4282), not 140-3.

### 3.5 Why none of these is "the" backend

The contract exists because consumers must be able to switch. No backend is privileged; the
differential harness is the arbiter.

---

## 4 Security engineering notes

- Every untrusted parsing entry point is bounded **before** parsing: size, depth, element count.
- Verification must not panic; backend panics are caught at the boundary and reported as backend
  failure, never as "invalid signature".
- Constant-time comparison for all MAC and tag checks; use audited primitives rather than writing
  comparisons by hand.
- Error messages never contain key material, credential bytes, or plaintext.
- Differential mismatches are archived as minimised, redacted reproductions and adjudicated by
  reading the specification — never by majority vote and never by assuming the newer library is
  right.

---

## 5 Revision history

| Date | Change |
|------|--------|
| 2026-09-16 | Initial design notes: contract shape, open decisions D1–D7, backend characteristics, security engineering rules. |
| 2026-09-16 | Added §2.1 environment policy (decided): MSRV 1.89, dev toolchain 1.98.1, CI matrix, MSRV change rules. |
