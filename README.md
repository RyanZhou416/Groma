# Groma

> **A provider contract and pure-Rust implementations for cryptography.**
>
> 纯 Rust 密码学的定基轴网：一套冻结的后端合同，多个可替换实现。

> **Status: pre-alpha — scope defined, no source code yet.**
> Nothing in this repository is audited, production-ready, or FIPS validated, and no claim to the
> contrary should be inferred from anything here.

---

## Why Groma exists

Rust already has strong pure-Rust implementations of almost every cryptographic primitive. What is
missing is the layer above them:

- **No general, non-TLS provider abstraction.** `rustls::crypto::CryptoProvider` is a `struct`
  shaped around TLS; `rustls_pki_types::SignatureVerificationAlgorithm` covers verification only;
  RustCrypto's `crypto` crate is a version-compatibility re-export; the `aws-lc-rs` equivalent is
  sealed. There is no algorithm-agnostic provider trait in the ecosystem.
- **Bounded gaps remain unfilled.** PKCS#12 creation with MAC verification, CMS signature
  verification, OCSP checking, and attestation trust paths to vendor roots are absent, incomplete,
  or available only through C libraries.
- **The acceptance surface cannot be declared.** Mature options change their accepted algorithms
  with configuration and version, and none can state in machine-checkable form exactly what they
  accept.

Groma fixes the contract first, then fills the gaps, then makes existing pure-Rust protocols accept
a pure-Rust backend.

**Groma does not reimplement algorithms that already exist in pure Rust.** Generality comes from
interface breadth, backend interchangeability, and bounded gap-filling — not from rewriting an
ecosystem.

---

## What Groma is not

| Not this | Why |
|----------|-----|
| An OpenSSL replacement | A different project with different governance; scope grows only by the admission criteria in `../SCOPE.md` |
| A FIPS 140-3 validated module | No pure-Rust module is validated, and validation cannot be inherited across an implementation-language change |
| A protocol stack | TLS, QUIC, SSH, IPsec, Kerberos, Signal and WireGuard are implemented elsewhere; Groma supplies their crypto backend through adapters |
| An HSM or TPM stack | Groma provides the cryptographic core and the adaptation boundary only |
| A product | Ceremonies, account and session models, platform SDKs and wire formats are consumer-side |

---

## Layout

```
Groma/
├─ Crates/         one crate per unit, added as each phase lands
├─ Vectors/        pinned public test vectors with provenance manifests
├─ Fuzz/           fuzz targets and crash regressions
├─ Docs/           charter, design, research
└─ Tools/          vector fetching, dependency audit, differential drivers
```

Crates carry the project prefix: `groma-core`, `groma-codec`, `groma-rustcrypto`, `groma-graviola`,
`groma-libcrux`, `groma-openssl`, `groma-trust`, `groma-pkcs12`, `groma-cms`, `groma-tpm`,
`groma-entropy`, `groma-adapter-*`.

---

## Architecture in one picture

```
        consumers: products, adapters, third-party crates
                          |
        +-----------------+------------------+
        |                 |                  |
   L7 adapters      product code       third-party crates
   rustls/quinn/    (deep
   russh/boringtun/  integrations)
   OpenPGP
        |                 |
        +--------+--------+
                 |
           L1 CONTRACT   <- frozen; no backend type crosses it
                 |
   +---------+---------+---------+
   |         |         |         |
 RustCrypto graviola  libcrux  OpenSSL (differential only)
                 |
    L3 algorithms · L4 formats · L5 trust · L6 platform
```

The contract sits in the middle on purpose: consumers see canonical types only; backends implement
operations only.

---

## Roadmap

| Phase | Content | Exit criterion |
|-------|---------|----------------|
| **P1** | Contract freeze: provider traits by algorithm family, canonical types, capability query, bounded codecs, **two** backends | No backend type in the public API; a third-party backend can be added without touching `core`; both backends agree on shared vectors |
| **P2** | RustCrypto backend + OpenSSL differential; W3C/Wycheproof/ACVP corpora | Applicable vectors pass; every differential mismatch archived and adjudicated |
| **P3** | Algorithm surface and formats (KDF/MAC/KEM/PQC; PKCS#1/#8, SPKI, CSR) | Per-algorithm KATs; fuzz targets clean |
| **P4** | Adapters: rustls, quinn, russh, boringtun, OpenPGP | Each target project passes its own suite with the Groma backend installed |
| **P5** | Trust and containers: X.509 paths, name constraints, CRL, OCSP, CT; PKCS#12, CMS | Conclusions match the OpenSSL CLI differential |
| **P6** | Platform: TPM 2.0 structures and attestation, PKCS#11 core, DRBG and entropy health tests | Public vectors plus real-device samples |

P1 is strictly serial. Algorithm families inside P3 and formats inside P5 are independent.

---

## Documents

| Document | Content |
|----------|---------|
| [`Docs/SCOPE.md`](Docs/SCOPE.md) | **Authoritative charter**: admission criteria, layers, phases, non-goals, quality gates, governance |
| [`Docs/HANDOFF.md`](Docs/HANDOFF.md) | **Temporary handoff brief**: structured owner requirements, scope rulings, P1 deliverables and exit criteria, hard constraints, known traps |
| [`Docs/DESIGN.md`](Docs/DESIGN.md) | Contract design notes and open technical decisions |
| [`Docs/research/`](Docs/research/) | Ecosystem surveys and the evidence behind the scope |

For Chinese-language narrative, `../SCOPE.md` is written in Chinese; this README is the English
entry point.

---

## Contributing

See [`CONTRIBUTING.md`](CONTRIBUTING.md). Every cryptographic change requires test vectors from a
public corpus, and every untrusted parsing entry point requires a fuzz target.

Security reports: see [`SECURITY.md`](SECURITY.md). Please do not open public issues for
vulnerabilities.

---

## Licence

Licensed under either of

- Apache License, Version 2.0 ([`LICENSE-APACHE`](LICENSE-APACHE))
- MIT license ([`LICENSE-MIT`](LICENSE-MIT))

at your option.

Unless you explicitly state otherwise, any contribution intentionally submitted for inclusion in
this project, as defined in the Apache-2.0 licence, shall be dual licensed as above, without any
additional terms or conditions.

Third-party dependencies carry their own licences; see the generated licence inventory once code
lands.
