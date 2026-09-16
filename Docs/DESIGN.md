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
| D1 | `no_std` posture | (a) `core`+`codec` `no_std`+`alloc`, backends unconstrained (b) `std` everywhere | ✅ **已决 2026-09-16：方案 (a)**——合同层 `no_std`+`alloc`，后端适配器不受限（ring/aws-lc-rs 需要 std，不许其后端受限会丢 FIPS 路径） |
| D2 | Trait dynamism | (a) static generics only (b) `dyn`-compatible traits (c) dual-layer (d) single-layer object-safe contract | ✅ **已决 2026-09-16：方案 (d)**——族 trait 从第一天起对象安全（构造走 provider 工厂，`finalize` 类用 `mut self: Box<Self>` 接收者，MSRV 1.89 ≥ 1.88 满足）；一套 trait 同时服务静态（零开销）与动态（运行时选后端）两种用法，第三方后端实现一次两种用法自动获得 |
| D3 | Algorithm enum extensibility | (a) closed non-exhaustive enum (b) open registry | ✅ **已决 2026-09-16：方案 (a)**——封闭 `#[non_exhaustive]` 枚举；加算法=改合同=SCOPE 治理流程（硬约束 9 的机器执行），第三方扩展的正当代言在 D4（后端）而非 D3；`non_exhaustive` 使加变体不打断下游 |
| D4 | Backend selection | (a) compile-time features (b) runtime registration (c) both | ✅ **已决 2026-09-16：方案 (e) 纯构造注入**（研究后选型，替代原 (c)）——合同层零选择机制（无 feature 选择、无全局注册表、无默认后端、无 `install()`）；装载＝普通依赖（伞 crate feature 只作可叠加的 re-export 糖）；选择＝组合根显式传值；注册表＝外围 crate 普通对象。根除 feature 互斥冲突／全局安装／"库替应用做决定"三类生态顽疾 |
| D5 | Error taxonomy depth | (a) coarse (b) fine-grained per family | ✅ **已决 2026-09-16：方案 (d) 统一分类枚举＋源链**（研究后选型，替代原 (a)/(b)）——单一 `#[non_exhaustive]` 枚举，变体＝类别（`InvalidSignature`／`InvalidEncoding`／`UnsupportedAlgorithm`／`BackendFailure`／`ParameterError`）而非原因；原因走 `source()` 链（signature crate 的 hashed-error 模式）；Display 不含密钥/明文。可匹配防 fail-open、细节不入 match 防错误预言机、枚举不爆炸防 semver 抖动；统一类型也是 D2=(d) 对象安全合同的必然推论 |
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
| 2026-09-16 | Decided D1: no_std posture = (a) contract layer `no_std`+`alloc`, backends unconstrained. |
| 2026-09-16 | Decided D2: trait dynamism = (d) single-layer object-safe contract (factory-based construction, `Box<Self>` receivers); one trait serves both static and dynamic dispatch. |
| 2026-09-16 | Decided D3: algorithm naming = (a) closed `#[non_exhaustive]` enum; adding an algorithm is a governed contract change. |
| 2026-09-16 | Decided D4: backend selection = (e) pure constructor injection — zero selection mechanism in the contract, loading via ordinary (additive) dependencies, choice made at the composition root, registry as an out-of-contract crate. |
| 2026-09-16 | Decided D5: error model = (d) unified classified enum + source chain — variants are classes not causes, hashed-error source chain, no secrets in Display. |
