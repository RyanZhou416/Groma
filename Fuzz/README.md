# Fuzz/

Fuzz targets and crash regressions.

## Rules

1. **Every untrusted parsing entry point has a durable target.** This is required by
   `../SCOPE.md` §3.2 rule 3 and §7.
2. **Every crash becomes a fixed regression.** A crashing input is minimised, committed under
   `Fuzz/regression/`, and referenced from a test that fails if it crashes again.
3. **Targets must be bounded.** A fuzz target feeds data to the *bounded* entry points, so that a
   finding is a real defect rather than a caller misusing a documented low-level API.
4. **Artifacts and corpus are not committed.** `Fuzz/artifacts/`, `Fuzz/corpus/` and `Fuzz/target/`
   are git-ignored. Only minimised regression inputs are committed.

## Planned targets

| Target | Entry point | Phase |
|--------|-------------|-------|
| `cose_key` | `groma-codec` COSE_Key parsing | P1 |
| `cbor_bounded` | `groma-codec` bounded CBOR (duplicate keys, depth, size) | P1 |
| `der_signature` | DER ECDSA `Ecdsa-Sig-Value` decoding | P1 |
| `pem_der_chain` | certificate and key encoding parsing | P3 |
| `attestation_object` | attestation statement parsing | P5/P6 |

## What fuzzing does and does not establish

Fuzzing finds crashes, over-reads, unbounded allocation, and non-termination. It does **not**
establish that accepted inputs are *semantically* correct — that is what the vector corpora in
[`../Vectors/`](../Vectors/) are for. Neither replaces independent audit.
