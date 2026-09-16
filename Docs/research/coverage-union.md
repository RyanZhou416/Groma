# 纯 Rust 密码学功能并集与缺口清单

> **文档类型**：research（调研）
> **所属模块**：Groma
> **创建日期**：2026-09-16
> **实现状态**：已完成本轮全量清点
> **权威范围**：[`../SCOPE.md`](../SCOPE.md)
> **缘起**：完整检索密码学功能并集，排出只有非 Rust 实现的功能，作为自研实现空间
>
> 本文是**功能并集 → 纯 Rust 实现差集**的完整清点。凡"纯 Rust 无"的条目即候选实现空间，
> 但**是否立项须再过 `SCOPE.md` §2.1 的三条准入判据**（有公开规范、有公开向量、有真实消费者）。
> 所有版本与日期为 2026-09-16 快照；**未编译、未执行任何代码**。

---

## 1 判决口径

"纯 Rust"须同时满足：Cargo `links` 为空、无 `*-sys` 依赖、无 C 构建依赖（`cc`／`cmake`／
`bindgen`／`pkg-config`）。

| 结论 | 含义 | 是不是实现空间 |
|------|------|---------------|
| **已缺** | crates.io 上不存在 | ✅ |
| **仅他语言** | 只有 C／C++／Go／Java／Python 实现 | ✅ |
| **有 Rust 壳但默认 C 后端** | 产品路径默认或强制 C | ⚠️ 视目标而定 |
| **壳已有、功能不全** | 缺具体功能 | ✅ 通常工作量最小 |

> **方法学修正（实测）**：一次批量探测曾把 `argon2`、`ed448-goldilocks`、`fido-mds`、`spake2`、
> `opaque-ke`、`xts-mode`、`sm9` 误报为"不存在"，实为传输层 SSL 失败。
> **"探测失败"≠"不存在"**，负结论必须多源复核。

---

## 2 基元层：**基本不缺**

| 域 | 纯 Rust 覆盖 |
|----|-------------|
| 哈希 | SHA-1／SHA-2 全族／SHA-3 全族／SHAKE／BLAKE2／BLAKE3／MD5／RIPEMD／Whirlpool／GOST94／Streebog／**Grøstl／JH／Skein／FSB／Tiger**／Keccak／K12／Ascon-Hash |
| MAC | HMAC／CMAC／PMAC／Poly1305／Polyval／cSHAKE |
| KDF | HKDF／PBKDF2／SP 800-108 KBKDF／concat-KDF／AES-KW＋KWP |
| 口令哈希 | Argon2(d/i/id) 0.6.0／scrypt 0.12.0／bcrypt 0.19.3／yescrypt 0.1.0／Balloon／Lyra2／SHA-crypt／MD5-crypt |
| AEAD | AES-GCM／AES-GCM-SIV／AES-CCM／AES-OCB3／AES-EAX／ChaCha20-Poly1305／XChaCha／Deoxys／Ascon-AEAD |
| 分组密码 | AES／SM4／ARIA／Camellia／DES／3DES／Blowfish／CAST5／CAST6／IDEA／RC2／RC5／Twofish／Serpent／Kuznyechik／Magma |
| 流密码 | ChaCha／Salsa20／HC-256／Rabbit／Grain／Trivium／**ZUC** 0.4.1 |
| 公钥签名 | RSA(PKCS#1v1.5／PSS／OAEP)／ECDSA P-256/384/521／Ed25519／**Ed448**／DSA 0.7.0／**SM2** 0.13.3／**SM9** 0.4.0／k256(BIP340 Schnorr) |
| 密钥交换 | X25519／X448／ECDH P 曲线／**ML-KEM**／DH |
| 后量子 | ML-KEM 0.3.2／ML-DSA 0.1.1／SLH-DSA 0.1.0／FN-DSA 0.4.0／Classic McEliece／FrodoKEM／sntrup761／LMS／XMSS |
| 高级签名 | BLS12-381(zkcrypto 纯 Rust)／BN254／VOPRF 0.5.0／FROST／cggmp21／halo2／bulletproofs／tfhe-rs |
| 格式 | DER／SPKI／PKCS#1／PKCS#8／SEC1／X.509 签发／CSR／COSE／CBOR |

**结论：基元层没有值得立项的空间。** 唯一基元级硬伤是 `rsa` 的
**RUSTSEC-2023-0071（Marvin）`patched = []`**，全部版本受影响。

---

## 3 缺口全集：纯 Rust 无或只有他语言

### 3.1 高优先（有消费者，且 Groma 层内）

| # | 功能 | 纯 Rust 状态 | 现存实现 | Groma 层 |
|---|------|-------------|---------|---------|
| 1 | **通用 provider 抽象** | **生态完全空缺** | 无任何语言的对等物可参照；需自行设计 | **L1** |
| 2 | **PKCS#12 创建＋MAC 校验** | `pkcs12` 0.1.0（**2024-01-04**）有 KDF 与结构，但 `MacData` **不校验 MAC**，TODO 明写缺解密／builder／RC2 | OpenSSL C | L4 |
| 3 | **CMS/PKCS#7 签名验证** | `cms` 0.2.3 纯 Rust 但**无从验签 API**；`pkcs7` 已 DEPRECATED | OpenSSL `CMS_verify` | L4 |
| 4 | **OCSP 吊销检查** | 只有格式层（`x509-ocsp` 0.2.1，2024-01-08，休眠），**无检查器** | OpenSSL C | L5 |
| 5 | **可信路径校验到厂商根** | `rustls-webpki` 有**名称约束**与 **CRL**，**无 OCSP**；无 crate 把根库接上它 | OpenSSL C | L5 |
| 6 | **TPM 2.0 结构与证明验证** | 解析层已有（`tpm2-protocol` 1.2.0，no_std 零依赖）；**验证层残缺**，PS256 无实现 | **Go `go-tpm` 仅用 stdlib**，成熟 | L6 |
| 7 | **PKCS#11 软件令牌的纯 Rust 密码学** | ABI 已纯 Rust（`craton-hsm` 0.10.0 用 Rust 定义 `CK_FUNCTION_LIST`） | **缺口在 ABI 之后的密码学**：其 RSA 私钥操作默认不可用，需 C `aws-lc` | L6 |
| 8 | **SP 800-90B 熵源与健康测试** | **完全空缺**（`entropy` 0.4.3 只是香农熵计算器） | NIST 参考实现（C） | L6 |
| 9 | **可用的 jitter 熵源** | `rand_jitter` 0.6.1 纯 Rust 但**不是** jitterentropy 移植；`jitterentropy-rs` 0.1.1 自述 "**scaffold**" | jitterentropy（C，CMVP E280） | L6 |
| 10 | **密钥序列化补全** | PKCS#1／#8／SPKI 有，但 **PKCS#10 全部版本已 yank**；`rcgen` 默认 `ring`，**纯 Rust 签名需自行实现 `SigningKey`**，RSA 密钥生成走 `aws_lc_rs` | OpenSSL／rcgen+ring | L4 |

### 3.2 中优先（他语言成熟，Rust 侧有壳但不纯）

| # | 功能 | 纯 Rust 状态 | 他语言参照 |
|---|------|-------------|-----------|
| 11 | **WireGuard** | **不纯**：`boringtun` 0.7.1 的 `ring = "0.17"` 是**非可选**依赖 | Go `wireguard-go` |
| 12 | **DTLS** | 每个 Rust DTLS crate 都拉 `ring`／`aws-lc-rs`；`sans-io-dtls`／`rustls-dtls` 不存在 | OpenSSL／mbedTLS |
| 13 | **QUIC** | `quinn` 可换 provider 但**默认** `rustls-ring`；`quiche` **硬绑 BoringSSL**；`s2n-quic` 默认 C `s2n-tls` | quiche(C++)／s2n-quic(C) |
| 14 | **SSH** | `russh` 0.63.3 源码纯 Rust 但 `default=["flate2","aws-lc-rs","rsa"]`——**默认后端是 C** | OpenSSH（C） |
| 15 | **OpenPGP 默认路径** | `sequoia-openpgp` 2.4.1 **默认 `crypto-nettle`**（C）；纯 Rust 替代是 `pgp` 0.20.0（rpgp，**crate 名是 `pgp`**） | GnuPG（C） |
| 16 | **Kerberos 密码学＋GSS-API** | 基本空缺；`gssapi`／`libgssapi` 是 FFI。唯二纯 Rust：`krb5-gss` 0.2.0（**无 RC4-HMAC**）、`krb5-rs` 0.1.0（RFC 3961 仍 "Planned"） | MIT krb5（C）／Go `gokrb5` |
| 17 | **Android Key 证明** | `android-attestation`、`android-key-attestation` 在 crates.io **均不存在**；`octet-attest-verify` 2.2.1 纯 Rust 但**无在线吊销** | Google 自家验证器是 Kotlin |
| 18 | **Apple 匿名证明** | `apple-attestation`、`apple-app-attest` **均不存在**；§8.8 本身只需 SHA-256＋OID＋SPKI 比对，属"没打包"而非"做不到" | — |
| 19 | **FIDO MDS 拉取／刷新／缓存** | `fido-mds` 0.5.5 的 JWS/x5c 走 **OpenSSL**，库内无 HTTP；拉取在独立 `fido-mds-tool`（reqwest），无缓存 | — |
| 20 | **IPsec／IKEv2** | 生产路径是 `strongswan-sys` FFI；纯 Rust 只有 `ryke` 0.3.0（70 下载） | strongSwan（C） |

### 3.3 低优先（生态空白，无消费者——`SCOPE.md` §6 已列为 deferred）

| # | 功能 | 纯 Rust 状态 | 他语言参照 |
|---|------|-------------|-----------|
| 21 | **Signal 协议**（X3DH／Double Ratchet／Sesame） | **crates.io 上零实现**；唯一同名是 2019 年的 FFI 包装 | Signal 官方 libsignal（Rust 但**未发布**） |
| 22 | **J-PAKE** | 不存在 | Go／Python／Java 均有 |
| 23 | SPAKE1／EKE／TLS-PAKE(RFC 8492) | 不存在 | — |
| 24 | Dragonfly／SAE (WPA3) | 只有 C 后端（`shuli`→aws-lc-rs；`supplicant-rs`→ring） | hostapd（C） |
| 25 | Catena／Makwa | 不存在 | PHC 参考实现（C） |
| 26 | 统一 crypt(3) 分派器 | `pwhash` 1.0.0（2021，休眠）覆盖 `$1$,$5$,$6$` 但**缺 `$y$` 与 `$7$`** | glibc／libxcrypt（C） |
| 27 | Yarrow PRNG | 不存在（`yarrow` 0.0.1 是音频 GUI——**命名冲突**） | — |
| 28 | **BIKE** | **完全不存在** | PQClean（C） |
| 29 | Saber | 无已发布纯 Rust（仅 git） | PQClean（C） |
| 30 | HQC | crates.io 上只是**占名**；真实现仅 git；已发布的是 C | PQClean（C） |
| 31 | Classic NTRU | 只有 FFI C | PQClean（C） |
| 32 | SPHINCS+ round-3 参数 | 只有 C；纯 Rust 选项停更于 2023 | PQClean（C） |
| 33 | 群签名 | 只有研究件 `dgsp` 0.1.2，且**经 pqcrypto 变成 C 后端** | — |
| 34 | MPC 框架 | **无已发布主流纯 Rust 框架** | MP-SPDZ、libOTe（C++） |
| 35 | CKKS／BFV 同态（无 C++） | 主流是 SEAL／OpenFHE 绑定；纯 Rust `fhe` 0.1.1 仅早期 | SEAL、OpenFHE（C++） |
| 36 | BBS+／AnonCreds 当前规范 | `bbs` 0.4.1 自 2020 停更；`pairing_crypto` 未发布 | — |
| 37 | **KASUMI／CLEFIA／MISTY1／HIGHT／Noekeon／KATAN／PRESENT／SNOW 3G** | **均不存在**（逐个 404＋搜索总数 0） | 只有 C（OpenSSL 也没有） |
| 38 | **GOST R 34.10-2012 签名** | **签名不存在**，RustCrypto 只覆盖 GOST 分组密码与哈希 | C `gost-engine/engine` |

> **命名冲突警示**（会误导检索）：`present`＝markdown 工具、`simon`＝CLI 参数解析、`seed`＝WASM
> 框架、`falcon`＝二进制分析、`lms`＝文件同步、`sunshine`＝光线投射引擎、`yarrow`＝音频 GUI、
> `aes-kwp` 0.0.0 是占名。**按名字断言"不存在"会出错。**

---

## 4 结果汇总

| 层级 | 判定 | Groma 处置 |
|------|------|-----------|
| **基元层** | 不缺（仅 RSA Marvin 未修补） | L3 引用现成 crate，**不重写** |
| **合规层（FIPS 140-3）** | 纯 Rust 零实现；唯一路径是 FFI 到 C | **显式非目标**（§2.3） |
| **协议层** | 大量"有 Rust 壳、默认 C 后端" | **L7 适配器**：不重写协议，只换后端 |
| **标准／编码层** | PKCS#12 创建、CMS 验签、OCSP 有明确空洞 | **L4／L5** |
| **证明／硬件层** | 缺口最集中 | **L6** |
| **长尾算法** | 不存在，但无消费者 | **deferred**（§6） |
| **熵层** | SP 800-90B 与可用 jitter 空缺 | L6，与 FIPS 非目标联动 |
| **provider 抽象** | 生态空白，无人做过 | **L1，核心** |

### 4.1 三条判断

1. **"纯 Rust 版 aws-lc-rs"的真正含义不是补齐算法**——算法早就齐了；是**补抽象、证据与集成**。
   而 FIPS 这一条对纯 Rust 不是工程问题，是**认证与模块边界问题**。
2. **L1（provider 抽象）＋ L7（适配器）才是"通用"的真正内容，且比移植任何 C 库更划算**：
   不重写协议栈，只让现有纯 Rust 协议接受纯 Rust 后端。
3. **长尾算法虽然"只有他语言"，但不是机会**——Rust 生态没有消费者，做出来没有下游。

---

## 5 已知限制

| 限制 | 说明 |
|------|------|
| 静态快照 | 版本／日期为 2026-09-16；未编译、未执行 |
| 负结论强度 | "不存在"依据 crates.io 404＋搜索总数 0；**本轮已实测出 SSL 失败导致的假 404**，故单次探测不算证据 |
| 第三方 crate 纯度 | `sm9`／`zuc`／`kisaseed`／`simon-speck`／`skipjack`／`xts-mode`／`cipher_magma` 按元数据与描述判为纯 Rust，**未逐一读构建文件** |
| CMVP 检索 | CMVP 索引的是模块**名称**而非语言，故"无纯 Rust 模块"是**强名称证据，非证明** |
| 未核实项 | `ring` 是否内置 jitterentropy；rustls 外部 PSK 现状；`boring-sys` 对应哪张 BoringCrypto 证书；部分 crate 的已发布版本（仅 master 清单可达） |

---

## 修订历史

| 日期 | 变更 |
|------|------|
| 2026-09-16 | 从原项目迁入并改为 Groma 调研文档；按 `SCOPE.md` 分层重排，把无消费者项移入 deferred，补命名冲突与方法学警示。 |
| 2026-09-16 | 建立原始版：完成六层并集清点，排出只有非 Rust 实现的功能。 |
