# Groma — Scope and Charter

> **Document type**: charter (project scope, authoritative)
> **Module**: Groma
> **Created**: 2026-09-16
> **Status**: scope defined; implementation not started
> **Supersedes**: the former `VeriCeremony` WebAuthn-provider charter (renamed and widened here)
> **Authority**: owner decision of 2026-09-16 — build a *general-purpose* pure-Rust cryptography
> library, phased; the owner's own products consume the reusable "wheel" parts, not deep integrations.
>
> This document is the authoritative scope. Where any other document, issue, or pull request
> conflicts with it, this document wins. Narrowing or widening the scope is an explicit edit to this
> file, never a side effect of adding an algorithm.

---

## 0 Name, one-line goal, status

**Project name: Groma.** A *groma* was the Roman surveyor's instrument — a staff carrying a cross
that fixed the cardinal axes and the right angle *before* anything was built. Every street and plot
of a legionary camp was measured from that grid. The metaphor is deliberate: this library fixes the
backend contract first, and implementations and consumers build on those axes.

> **Groma — a provider contract and pure-Rust implementations for cryptography.**
>
> 中文：**Groma — 纯 Rust 密码学的定基轴网：一套冻结的后端合同，多个可替换实现。**

**Status:** scope defined. No source code, no published crate, no remote release. The GitHub
repository exists but contains documentation only.

### 0.1 The problem

Rust already has excellent pure-Rust implementations of essentially every cryptographic primitive:
the RustCrypto organisation, `graviola`, and `libcrux` between them cover hashing, MACs, KDFs,
AEADs, block and stream ciphers, digital signatures, key exchange, post-quantum schemes, and the
common key/certificate encodings.

What the ecosystem does **not** have is the layer that makes those implementations *interchangeable
and declarable*:

1. **No general, non-TLS provider abstraction.** `rustls::crypto::CryptoProvider` is a `struct`, not
   a trait, and is shaped around TLS (cipher suites, key-exchange groups, webpki algorithms).
   `rustls_pki_types::SignatureVerificationAlgorithm` is a trait but covers verification only.
   RustCrypto's `crypto` crate is a version-compatibility re-export facade, not a swappable backend.
   The `aws-lc-rs` equivalent is sealed. Names such as `crypto-provider`, `crypto-facade`,
   `crypto-traits`, `webcrypto` and `universal-crypto` do not exist on crates.io.
2. **No bounded coverage for several standard seams.** PKCS#12 creation with MAC verification, CMS
   signature verification, OCSP checking, and attestation trust-path building to vendor roots are
   each either absent or incomplete in pure Rust.
3. **No way to declare a fixed acceptance surface.** Every mature option either changes its accepted
   algorithm set with configuration and version, or cannot state in machine-checkable form exactly
   which algorithms it accepts.

### 0.2 What Groma is — and is not

**Groma is** the provider contract plus pure-Rust implementations and adapters that fill the gaps
above, phased so that each phase is independently useful.

**Groma is not** an OpenSSL equivalent, a protocol stack, a compliance certification, or a rewrite
of algorithms that already exist in pure Rust.

**The single most important design rule:** *Groma does not reimplement existing pure-Rust
algorithms.* Generality is achieved through **interface breadth plus backend interchangeability
plus bounded gap-filling** — not through rewriting 200+ existing crates. Reimplementing them would
take years and produce worse code.

---

## 1 Terminology

| Term | Meaning in this document |
|------|--------------------------|
| Provider contract | The stable trait/type surface through which a backend performs cryptographic operations |
| Backend | A concrete implementation of the provider contract (e.g. the RustCrypto backend) |
| Reference backend | A mature implementation used only for differential testing; never a product dependency |
| Acceptance surface | The exact set of algorithms and encodings a build accepts, queryable at runtime |
| Wheel | A reusable, general-purpose component with no product-specific policy |
| Deep integration | Product-specific ceremony, policy, account/session, platform-SDK or wire-format logic |
| Consumer | A project that depends on Groma wheels (for example the owner's player-services backend) |
| Phase | A delivery stage with machine-checkable exit criteria (P1…P6) |
| Layer | A functional stratum of the library (L1…L7) |

---

## 2 Scope

### 2.1 Admission criteria

A capability enters Groma **only if all three hold**:

1. **Public specification** — an RFC, NIST/ISO/IEC/3GPP publication, W3C Recommendation, or an
   equivalent normative document defines the behaviour.
2. **Public test vectors** — ACVP/CAVP, Wycheproof, W3C, RFC KATs, or an equivalent machine-readable
   corpus exists, so correctness is decidable without human judgement.
3. **A real consumer** — either one of the owner's products or a credible ecosystem consumer needs
   it. "Someone might want it" is not a consumer.

If a capability fails any criterion, it is recorded as *deferred* in §6 rather than implemented.

### 2.2 Layers

| Layer | Contents | Rationale |
|-------|----------|-----------|
| **L1 Contract** | Provider traits by algorithm family; canonical types; closed algorithm enums; capability query; classified errors; backend registration | The ecosystem gap; must be frozen before anything else |
| **L2 Backends** | RustCrypto, graviola, libcrux, OpenSSL (differential only) | Interchangeability; the proof that L1 is correct |
| **L3 Algorithms** | Hashing, MAC, KDF, AEAD, block/stream ciphers, signatures, KEM/PQC, password hashing, RNG/DRBG | Generality over the common surface |
| **L4 Formats** | DER, SPKI, PKCS#1/#8/#12, CMS/PKCS#7, CSR, COSE, CBOR | Bounded, spec-defined, largely missing |
| **L5 Trust** | X.509 path validation, name constraints, CRL, OCSP, CT | Missing; blocks real verification work |
| **L6 Platform** | TPM 2.0 structures and attestation verification, PKCS#11 cryptographic core, entropy sources and health tests | Vendor-independent, spec-defined, missing |
| **L7 Adapters** | rustls, quinn, russh, boringtun, OpenPGP backend swaps | Highest leverage: makes existing pure-Rust protocols accept a pure-Rust backend without rewriting protocol code |

### 2.3 Explicit non-goals

| Non-goal | Why |
|----------|-----|
| **FIPS 140-3 validated module** | No pure-Rust module is validated; CMVP porting rules cover a change of operational environment, not a change of implementation language, so validation cannot be inherited. A submission is a multi-year certification project, not a library task. Groma states clearly that it is **not** FIPS validated. |
| **OpenSSL parity** | An OpenSSL-equivalent library is a different project with different governance. Scope grows by the §2.1 criteria, never by "OpenSSL has it too". |
| **Protocol stacks** (TLS, DTLS, QUIC, SSH, IPsec, Kerberos, Signal, WireGuard) | Implemented or best implemented elsewhere; Groma supplies the crypto backend via L7 adapters. |
| **HSM firmware, TPM stack, PKCS#11 modules as products** | Separate domains. Groma provides only the cryptographic core and the adaptation boundary. |
| **Long-tail ciphers without consumers** | KASUMI, CLEFIA, MISTY1, HIGHT, Noekeon, KATAN/KTANTAN, PRESENT, SNOW 3G, and GOST R 34.10-2012 signatures have specifications but effectively no Rust consumers. Deferred per §2.1 criterion 3. |
| **Product policy** | Ceremonies, challenges, origin/RP-ID policy, account and session models, platform SDKs, and product wire formats are deep integrations and live in the consumer. |
| **Self-invented cryptography** | No new algorithms, curves, signature formats, or RNGs. |

### 2.4 The wheel / deep-integration boundary

This boundary is the answer to "will the owner's own products actually benefit?".

| Category | Examples | In Groma? |
|----------|----------|-----------|
| Wheel | hashing, MAC, KDF, AEAD, signatures, KEM, RNG, key/certificate encodings, X.509 paths, OCSP, TPM structures, COSE/DER codecs, provider contract | ✅ yes |
| Deep integration | WebAuthn ceremony state machine; challenge/origin/RP-ID policy; attestation-statement *policy*; account/session/refresh models; platform SDKs; product wire formats | ❌ no — consumer side |

**Consequence:** the WebAuthn *attestation statement* logic is largely a deep integration. Only the
general parts beneath it (X.509 paths, OCSP, TPM structure parsing, COSE/DER codecs) belong in Groma.

---

## 3 Architecture

### 3.1 Layering

```
                     consumers (products, other projects)
                                  |
        +-------------------------+-------------------------+
        |                         |                         |
   L7 adapters              product code              third-party crates
   (rustls, quinn,          (deep integrations)       (via L1 contract)
    russh, boringtun,
    OpenPGP)
        |                         |
        +-----------+-------------+
                    |
              L1 CONTRACT  <- frozen; no backend types leak across it
                    |
    +-------+-------+-------+-------+
    |       |       |       |       |
 L2 RustCrypto  graviola  libcrux  OpenSSL (differential only)
    |
    +--- L3 algorithms --- L4 formats --- L5 trust --- L6 platform
```

The contract sits in the middle by design: consumers see only canonical types, and backends see only
the operations they must implement.

### 3.2 Contract requirements

These are inherited from the original charter and remain binding.

1. Algorithms are a **closed enum**; unknown algorithms fail; the algorithm is never inferred from
   input length.
2. Key type and algorithm are checked against each other; algorithm confusion is rejected.
3. All external bytes are untrusted input; parsing is bounded in size, depth, and element count.
4. Invalid signature and invalid encoding are reported distinctly to the consumer, while the
   product-facing response may be uniform.
5. Verification functions have **no implicit global state, network, file, or clock dependencies.**
   *Clarification (2026-09-16):* this forbids provider-owned process-wide mutable state and any
   hidden I/O or clock reads. It does **not** forbid lazily-initialised immutable tables (for
   example a CPU-feature cache) or a per-backend error queue, provided both are documented and
   bounded. Without this clarification the requirement would exclude every resource-pooling
   implementation, including all current Rust candidates.
6. Backend errors must not panic, abort, or leak secrets; unsupported platforms fail explicitly at
   initialisation.
7. The public API never exposes backend-private types.
8. Provider version and algorithm set are queryable; **adding an algorithm must never enlarge the
   accepted set automatically** — acceptance is explicit configuration.
9. Replacing a backend must not change the canonical input; the same valid or invalid vector must be
   differentially comparable across backends.
10. Production callers use the high-level safe interface; there is no raw-primitive combination
    surface that can bypass policy checks.

### 3.3 Workspace layout

```
Groma/
├─ Cargo.toml              workspace root (members added per phase)
├─ rust-toolchain.toml     pinned toolchain
├─ crates/                 one directory per crate, added per phase
├─ vectors/                pinned public test vectors and provenance manifests
├─ fuzz/                   fuzz targets and crash regressions
├─ docs/                   charter, design, research (Chinese narrative)
└─ tools/                  repository tooling (vector fetch, audit, differential drivers)
```

Crate naming follows the project prefix: `groma-core`, `groma-codec`, `groma-rustcrypto`,
`groma-graviola`, `groma-libcrux`, `groma-openssl`, `groma-trust`, `groma-pkcs12`, `groma-cms`,
`groma-tpm`, `groma-entropy`, and `groma-adapter-*`.

---

## 4 Phases

Each phase has an exit criterion that is machine-decidable. No phase is "done" on a demo or a
screenshot.

### P1 — Contract freeze (L1)

- Provider traits grouped **by algorithm family** (`Digest`, `Mac`, `Kdf`, `Aead`, `Signer`,
  `Kem`, `PasswordHash`, `Random`), not one trait per algorithm.
- Canonical types: algorithm enum, key/signature/ciphertext newtypes, classified error enum.
- Capability query and backend registration.
- `no_std` + `alloc` for `core` and `codec`.
- Bounded CBOR/COSE/DER parsing in `codec`.
- **Two backends**, even if the second implements only one algorithm, to prove the abstraction.

**Exit:** the public API exposes no backend type (compile-time test); a third-party backend can be
added without touching `core`; the two backends agree on a shared vector set.

### P2 — RustCrypto backend and OpenSSL differential (L2)

- Full verify/hash/AEAD/password-hash/RNG path on the RustCrypto backend.
- OpenSSL backend used **only** in test targets for differential comparison.
- W3C WebAuthn Level 3, Wycheproof, and ACVP corpora wired in.

**Exit:** applicable vectors pass; every differential mismatch is archived and adjudicated — never
resolved by majority vote.

### P3 — Algorithm surface and formats (L3, L4)

- KDF/MAC/KEM/PQC surface; PKCS#1/#8, SPKI, CSR.
- Per-algorithm KATs; fuzzing on every untrusted parsing entry point.

**Exit:** each algorithm has a KAT from a public corpus; fuzz targets run clean over the corpus.

### P4 — Adapters (L7)

- rustls, quinn, russh, boringtun, OpenPGP backend swaps.

**Exit:** each target project passes its own test suite with the Groma backend installed.

### P5 — Trust and containers (L5, L4)

- X.509 path validation with name constraints, CRL, OCSP, CT.
- PKCS#12 creation and MAC verification; CMS signature verification.

**Exit:** conclusions match the OpenSSL CLI differential across the corpus.

### P6 — Platform (L6)

- TPM 2.0 structures and attestation verification (including RSASSA-PSS AIK signatures).
- PKCS#11 cryptographic core; DRBG and SP 800-90B entropy sources and health tests.

**Exit:** public vectors pass plus real-device samples for the declared platforms.

### Parallelism

P1 is strictly serial (a wrong contract forces total rework). Within P3, algorithm families are
independent. Within P5, formats are independent. P4 depends on P1–P3.

---

## 5 Consumers and the benefit guarantee

The owner's products are the **first consumers**, and the phasing is arranged so they receive value
early rather than after the general library matures.

| Consumer need | Layer | Available from |
|---------------|-------|----------------|
| Ed25519 signing of issued material | L1 + L3 | **P2** |
| Password hashing (Argon2) | L3 | **P2** |
| AEAD for session and field encryption | L3 | **P2** |
| Key serialisation (PKCS#8, SPKI) | L4 | P3 |
| TLS crypto backend | L7 | P4 |
| PKCS#12 keystore files | L4 | P5 |
| OCSP and trust paths | L5 | P5 |
| Attestation primitives | L6 | P6 |

**The guarantee:** a product's ordinary cryptographic needs are satisfied at **P2**, not at the end
of the roadmap. Generality accumulates at P3–P6; it is never a precondition for usefulness.

---

## 6 Deferred capabilities

Recorded so that "later" is explicit rather than forgotten. Moving an item out of this table
requires satisfying all three admission criteria in §2.1.

| Capability | Reason deferred |
|------------|-----------------|
| FIPS 140-3 module | Certification project, not a library task (§2.3) |
| KASUMI, CLEFIA, MISTY1, HIGHT, Noekeon, KATAN/KTANTAN, PRESENT, SNOW 3G | No consumer (criterion 3) |
| GOST R 34.10-2012 signature | No consumer; no Rust implementation exists today |
| J-PAKE, SPAKE1, EKE, TLS-PAKE | No consumer identified |
| Catena, Makwa | No consumer identified |
| BIKE, Saber, HQC (published pure Rust) | Standardisation still settling; no consumer |
| Classic NTRU/SPHINCS+ round-3 parameters | Superseded parameter sets; no consumer |
| Group signatures, MPC frameworks | No consumer; research-grade |
| Signal protocol, Kerberos, IPsec crypto | No consumer in the owner's products; large surface |
| Yarrow PRNG, SP 800-90B entropy certification | Follows from the FIPS non-goal; revisit with P6 |
| Unified crypt(3) dispatcher | No consumer |

---

## 7 Validation and quality gates

| Category | Required machine evidence |
|----------|--------------------------|
| Specification vectors | Applicable W3C / NIST / RFC vectors pass item by item; inapplicable items carry a reason and a tracking entry |
| Known attacks | Wycheproof conclusions match for every applicable family; no invalid signature is accepted |
| Differential | Identical canonical input yields identical conclusions on reference and candidate backends; mismatches are archived and adjudicated |
| Parsing | Truncated, duplicated-key, non-canonical, over-deep, oversized, and random inputs neither crash, over-read, nor allocate unboundedly |
| Properties | encode/decode round-trips only on canonical representations; key type, algorithm and signature never cross-mix |
| Fuzzing | Every untrusted parsing and verification entry point has a durable fuzz target; crashes become fixed regressions |
| Cross-platform | `core` and applicable backends build across the support matrix; runtime capability is reported separately from successful compilation |
| Resources | Malicious large inputs, bulk invalid verifications, and concurrent calls are bounded; measured figures are recorded, never asserted |
| Supply chain | Locked versions, licence scan, provenance, and release artefacts traceable to a source commit and a test summary |
| Review | Each crate carries its own audit status. No aggregate claim of being audited or production-safe may be made before independent review exists. |

Test vectors find many known errors but do not replace audit. Formal verification, memory safety,
and absence of `unsafe` do not by themselves prove protocol, boundary, or build-artifact correctness.

---

## 8 Licensing and governance

- Original code: **MIT OR Apache-2.0** (Rust-ecosystem convention; the owner may change this before
  first release, but not after).
- Every dependency records version, licence, and purpose. Attributions and `NOTICE` obligations are
  preserved.
- `webauthn-rs` is MPL-2.0 and is **not** vendored into this repository; it is a consumer-side
  concern if used at all.
- `ed25519-dalek` is BSD-3-Clause; the licence inventory must reflect it.
- Reference implementations in other languages may be compared for behaviour and vectors. Porting
  code requires per-file licence and attribution review, and is avoided in favour of implementing
  from specifications.
- Security reports use a private channel; high-risk issues are coordinated before disclosure. See
  `SECURITY.md`.
- Each release binds a source commit, a lock file, the support matrix, a test summary, and known
  limitations.
- If maintenance capacity fails, the project must state that clearly rather than carrying an
  outdated release as an authority.

---

## 9 Relationship to consumers

Groma is an independent library. Consumers depend on released interfaces and artefacts; they do not
own its code or schedule.

| Situation | Consumer behaviour |
|-----------|--------------------|
| Groma incomplete for a need | Consumer ships with its own temporary arrangement; no product feature is deleted to wait for Groma |
| Experimental Groma build | Offline and shadow-differential use only; it does not decide a product outcome |
| Groma reaches a declared phase | Consumer adopts only after passing its own interface, data-compatibility, device, load, and failure acceptance |
| Differential mismatch | The existing authority is retained; a minimised, redacted reproduction is archived. "The new library is purer" is never a reason to accept a difference |
| Groma stalls | Consumers are not locked in; they keep using a validated backend |

The boundary must keep backends replaceable: no consumer account, session, revocation, or
authorisation logic may depend on Groma-private types. Groma never reads a product database or takes
over account authority.

---

## 10 Working contract for implementers

1. Read this document, the relevant specification, and the target dependency's security and licence
   notes before writing code.
2. Start at P1. Do not accumulate algorithms before the contract, threat model, and differential
   harness exist.
3. Prefer contributing an upstream provider boundary over forking. A fork must keep a minimal patch
   queue and evidence of upstream synchronisation.
4. Never implement elliptic curves, RSA bignum arithmetic, hash functions, or randomness from
   scratch. Use vetted crates and record their review boundaries.
5. Do not translate other languages' reference code into the repository. Establish the specification,
   the test vectors, and the licence first.
6. The first runnable output claims P1 only. README and API docs state the experimental status and
   the uncovered surface.
7. All tests are machine-decidable. Screenshots, manual logins, and single-browser demos are not
   security gates.
8. Bound every untrusted input before parsing. Fail closed; make errors diagnosable; never log
   credential material or secrets.
9. Each phase reports scope, evidence, residual risk, and the entry point of the next phase.
   Widening the scope requires editing this document first.
10. Do not write "production safe", "audited", or "OpenSSL replacement" before independent review
    exists. Describe formal verification only to its actual coverage.

---

## 11 Acceptance checklist

- [ ] Repository location, licence, and maintainer responsibility confirmed by the owner.
- [ ] Name and crate coordinates re-checked; conflicts and reservations recorded.
- [ ] P1 threat model, provider API, algorithm table, error table, and non-goals complete.
- [ ] `core`/`codec` depend on no specific backend and contain no network or account logic.
- [ ] Two backends agree on shared specification, negative, and differential vectors.
- [ ] CBOR/COSE/DER fuzz targets and crash regressions established.
- [ ] Support matrix distinguishes compilation, unit verification, real-device coverage, and
      production candidacy.
- [ ] README states experimental status, covered and uncovered surface.
- [ ] Dependency licences and supply-chain summary machine-generatable.
- [ ] Conclusions, fixes, and versions traceable across external security review.

---

## 12 Known limitations and next step

The scope above is defined; no source code, crate, CI, release, or review engagement exists yet.
Candidate backend versions, MSRV, the concrete API, and all performance and compatibility figures
must be validated by the implementation project. This document records **no** performance or
security measurement.

**Next step: P1.** Freeze the provider contract and ship two backends that agree on a shared vector
set. Only after that is it possible to judge how much of the remaining surface is adaptation and how
much is genuinely new implementation.

---

## Revision history

| Date | Change |
|------|--------|
| 2026-09-16 | Renamed the project from VeriCeremony to **Groma** and widened the scope from a WebAuthn-only crypto provider to a general pure-Rust cryptography library. Added the three admission criteria, L1–L7 layering, P1–P6 phasing, the explicit non-goal on FIPS 140-3, the wheel/deep-integration boundary, and the consumer benefit guarantee. Clarified contract requirement 5 (lazy immutable tables and per-backend error queues are permitted). |
| 2026-09-16 | (as VeriCeremony) Established scope for a standalone WebAuthn crypto-provider project. |
