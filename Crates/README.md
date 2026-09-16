# Crates/

One directory per crate, added as each phase of [`../Docs/SCOPE.md`](../Docs/SCOPE.md) lands.

Planned units (the whole `groma-*` namespace was verified free on crates.io on 2026-09-16):

| Crate | Layer | Phase | Purpose |
|-------|-------|-------|---------|
| `groma-core` | L1 | P1 | Provider traits, canonical types, capability query, errors |
| `groma-codec` | L1 | P1 | Bounded CBOR / COSE / DER parsing and canonicalisation |
| `groma-rustcrypto` | L2 | P2 | RustCrypto backend (first production candidate) |
| `groma-graviola` | L2 | P3 | graviola backend, with explicit CPU-feature admission |
| `groma-libcrux` | L2 | P3 | libcrux backend (formally verified lineage) |
| `groma-openssl` | L2 | P2 | OpenSSL **differential only**; never a product dependency |
| `groma-trust` | L5 | P5 | X.509 paths, name constraints, CRL, OCSP, CT |
| `groma-pkcs12` | L4 | P5 | PKCS#12 creation and MAC verification |
| `groma-cms` | L4 | P5 | CMS / PKCS#7 signature verification |
| `groma-tpm` | L6 | P6 | TPM 2.0 structures and attestation verification |
| `groma-entropy` | L6 | P6 | DRBG, SP 800-90B entropy sources and health tests |
| `groma-adapter-*` | L7 | P4 | rustls / quinn / russh / boringtun / OpenPGP backend swaps |

**The directory stays empty until P1.** The workspace root deliberately declares no members, so the
repository never advertises a crate that does not exist.

Crate-level rules:

- A crate declares its own audit status. There is no aggregate "this project is audited" claim.
- No crate in the shipping path may introduce a C/C++ build requirement.
- `groma-core` and `groma-codec` are `no_std` + `alloc`; they must not depend on any concrete backend.
