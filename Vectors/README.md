# Vectors/

Pinned public test vectors and their provenance manifests.

## Rules

1. **Provenance is mandatory.** Every vector set has a manifest recording: upstream project, URL,
   exact commit or release tag, retrieval date, licence, and a SHA-256 of the retrieved artefact.
2. **Never edit an upstream vector.** Corrections go in a separate, clearly named local override
   file with a written justification.
3. **Local vectors are ignored by git.** `Vectors/local/` is for scratch work only and is
   `.gitignore`d.
4. **Licence first.** A vector set whose licence does not permit redistribution is fetched by script
   at test time rather than vendored, and the manifest records the fetch parameters.

## Expected sources

| Source | Covers | Used by |
|--------|--------|---------|
| [Wycheproof](https://github.com/C2SP/wycheproof) | known attacks and edge cases across ECDSA, RSA, EdDSA, AEAD | P2 onward |
| [NIST ACVP-Server](https://github.com/usnistgov/ACVP-Server) | FIPS algorithm validation vectors (SHA, HMAC, AES modes, DRBG, ECDSA, RSA, EdDSA, KDFs) | P2 onward |
| [W3C WebAuthn Level 3 test vectors](https://www.w3.org/TR/webauthn-3/#sctn-test-vectors) | registration, authentication, attestation | P2, consumers |
| RFC KATs | algorithm-specific known-answer tests | per algorithm |
| CAVP byte-oriented SHS `.rsp` files | SHA-1 / SHA-2 families | P3 |

## Layout

```
Vectors/
├─ manifest/       one TOML manifest per vector set (provenance, hashes, licence)
├─ wycheproof/     vendored subset, committed
├─ acvp/           vendored subset, committed
├─ w3c-webauthn/   vendored subset, committed
└─ local/          scratch; git-ignored
```

## Why vectors are a first-class artefact

`../SCOPE.md` §2.1 makes public test vectors a condition of admission, and §7 makes them the
mechanism by which correctness is decided. A capability without a vector corpus cannot be verified
without human judgement, and a security gate that needs human judgement is not a gate.
