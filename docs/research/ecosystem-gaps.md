# Rust 纯加密库缺口分析

> **文档类型**：research（调研）
> **所属模块**：Groma
> **创建日期**：2026-09-16
> **实现状态**：已完成本轮调研
> **权威范围**：[`docs/SCOPE.md`](../SCOPE.md)
> **缘起**：核查 Rust 生态是否缺少一个像 `aws-lc-rs` 那样提供加密的纯 Rust 库，并给出缺口结论
>
> 本文只做生态事实核查与缺口判定，不选库、不代表任何候选已达到生产候选。所有版本号与日期是
> 2026-09-16 的资料快照，不是锁定组合。**本文是 `SCOPE.md` 立项的事实依据。**

---

## 1 结论先行

**算法层不缺，缺的是三样别的东西。** 逐项核查后，一个 Relying Party 所需的每一个算法在纯
Rust 下都有现成实现；真正没有现成答案的是：

1. **没有通用的（非 TLS 的）加密提供器抽象**——"可插拔后端"这个概念在 Rust 里只存在于
   **具体消费者内部**，不存在于生态层。这是 Groma 存在的**唯一不可替代的理由**。
2. **纯 Rust 后端与 C 后端之间的可信度落差没有桥**——不是能力落差，是审计证据与 FIPS 的落差。
3. **消费端不可替换是设计性的，不是缺库**——成熟协议库把 C 后端写死且不提供 trait。

---

## 2 参照物：`aws-lc-rs` 到底提供了什么

先把"像 `aws-lc-rs` 一样"拆开，才知道要对标什么。

| 维度 | `aws-lc-rs` 1.18.1 的事实 |
|------|--------------------------|
| 底层语言 | `aws-lc-sys` 是 AWS-LC 的 FFI 绑定；**libcrypto 是 C，libssl 是 C++**，另有 x86／ARM 汇编。谱系 OpenSSL → BoringSSL → AWS-LC |
| 与 ring 的关系 | 在 AWS-LC 之上**重新实现 ring v0.16 API**，不是 ring 的 fork |
| 构建要求 | 官方原文 "Consuming projects will need a C/C++ compiler to build"。非 FIPS：C/C++ 必须，CMake／bindgen／Go 从不需要。FIPS：C/C++＋CMake＋Go＋Perl 全要 |
| **是否有"无 C 工具链"路径** | **没有** |
| 依赖图标记 | `links = "aws_lc_rs_1_18_1_sys"`、`aws-lc-sys` 侧 `links = "aws_lc_0_45_0"`——供零 C 审计机检捕获 |
| 模块面（实测 docs.rs） | 22 个：`aead`／`agreement`／`cipher`／`cmac`／`constant_time`／`digest`／`encoding`／`error`／`hkdf`／`hmac`／`io`／`iv`／`kdf`／`kem`／`key_wrap`／`pbkdf2`／`pkcs8`／`rand`／`rsa`／`signature`／`tls_prf`／`unstable` |
| **缺失面** | **无 X.509、无 TLS、无 CMS/PKCS#7、无 PKCS#12、无 SHA-3/SHAKE/BLAKE2**（`digest` 只做 SHA-2 与 legacy SHA-1）；**要求 `std`**，不支持 `no_std` |
| FIPS | 独立 crate `aws-lc-fips-sys` ＋ `fips` feature。**验证对象是 AWS-LC 的 C 模块，不是 Rust 包装层**。当前绑 **AWS-LC-FIPS 4.x**，官方措辞为"已完成认可实验室测试并提交 NIST 认证"；**已拿证的是 3.0.x**（#5314，需 pin `aws-lc-rs <1.18.0`） |
| 审计 | **未查到针对 `ring` 或 `aws-lc-rs` 的第三方委托审计**。有的是 AWS 自家形式化验证与 2026-03 AISLE 的独立漏洞研究（2 个 CVE），后者是漏洞研究不是委托审计 |

**关键推论**：`aws-lc-rs` 真正卖的不是"算法多"，而是 **FIPS 140-3 验证＋rustls 默认位＋被生态当
成事实标准**；它同时把 **C 工具链**作为不可回避的前置。

---

## 3 算法层：不缺

| 算法 | RustCrypto | graviola 0.4.1 | libcrux 0.0.5 | 备注 |
|------|-----------|----------------|---------------|------|
| ECDSA P-256 验签 | ✅ `p256` 0.14.0 | ✅ | ✅ | 三家都有 |
| **RSA PKCS#1 v1.5 验签** | ✅ `rsa` 0.9.10 | ✅ | **❌ 未实现** | **libcrux 只有 PSS** |
| Ed25519 验签 | ✅ `ed25519-dalek` 3.0.0 | ✅ | ✅ | — |
| SHA-256／384／512 | ✅ `sha2` 0.11.0 | ✅ | ✅ | — |
| AES-GCM／ChaCha20-Poly1305 | ✅ | ✅ | ✅ | — |
| **非 TLS 的 provider trait** | ❌ 只有逐算法 trait 族 | ❌ 只有具体类型 | ❌ 无签名 trait | **三家都没有** |

**libcrux 的 RSA 缺口要单独强调**：`libcrux-rsa` 0.0.8 源码只有 `hacl/rsapss.rs`，
`pkcs1`／`v1.5` 出现次数为 0。形式化验证最好的那家**恰好不覆盖 RS256**。

> 方法学教训：按项目名在 crates.io 搜 `rustls-libcrux` 是空的，因为它实际是 workspace 成员
> `rustls-libcrux-provider`。**登记未发布的 crate 不能只靠注册表名搜索。**

---

## 4 缺口一：没有通用的加密提供器抽象

最结构性的发现。

| 存在的抽象 | 形态 | 为什么不够 |
|-----------|------|-----------|
| `rustls::crypto::CryptoProvider` | **是 struct，不是 trait** | 字段按 TLS 协议形状定义：`cipher_suites`／`kx_groups`／`signature_verification_algorithms`／`secure_random`／`key_provider`。FIPS 开关直接绑 `aws-lc-rs` |
| `rustls_pki_types::SignatureVerificationAlgorithm` | trait，`verify_signature(pubkey, msg, sig)`，dyn-compatible，非 TLS 类型 | **只验签**：无哈希、无随机、无公钥解析。且未发现任何非 TLS crate 真的用它 |
| RustCrypto `crypto` 0.5.1 | 门面 crate | **只是版本兼容的 re-export**，不是可替换后端 |
| `aws-lc-rs` 的 `signature::VerificationAlgorithm` | — | **sealed**，外部无法实现 |
| `ring-compat` 0.8.0 | 逐算法垫片 | 2023-10 后无更新，事实停更 |

`crypto-provider`、`crypto-facade`、`crypto-traits`、`webcrypto`、`universal-crypto` 在 crates.io
**全部不存在**。

**结论**：Rust 里"密码学后端可插拔"**只在具体消费者内部存在**，生态层没有标准。这直接解释了
为什么 Groma 要自己定义 provider trait：**不是重复造轮子，是那个轮子确实不存在。**

---

## 5 缺口二：可信度落差——审计与 FIPS

### 5.1 审计覆盖（已核实的第三方审计）

| crate | 审计方／资助 | 年份 | 局限 |
|-------|-------------|------|------|
| `aes`／`aes-gcm`／`chacha20poly1305`／`chacha20` | NCC Group／MobileCoin | 2020 | 已 6 年 |
| `p256` | zkSecurity／NEAR | 2025-04 | **明确排除**常量时间代码、签名生成、64 位专用代码、ECDH |
| `crypto-bigint` | NCC Group／Entropy | 2023-08 | 自述"实现自上次审计后已显著偏离" |
| `rsa` | Include Security／OTF | 2019 | **早于 Marvin** |
| `subtle`／`curve25519-dalek` | Quarkslab／Tari Labs | 2019-08 | 对 `ed25519-dalek` 只是"quick look" |

**未查到审计的关键项**：`sha2`、`hmac`、`hkdf`、`ed25519-dalek`。

`p256` 自己的 README 至今写着 "never been independently audited … **USE AT YOUR OWN RISK!**"——
**已被审计的那一项，其审计恰好排除了本项目最关心的常量时间。**

### 5.2 一个未修补的公告

**RUSTSEC-2023-0071 / CVE-2023-49092（Marvin Attack）**：`rsa` crate 时序侧信道可致私钥恢复。
`patched = []` 是刻意为之；公告 2026-09-14 更新仍写"截至 2026-09-12，`rsa` 0.9.10 与
0.10.0-rc.18 **仍受影响**"。crypto-bigint 迁移未修复。

**这是算法层唯一的硬伤，且落在 RS256 唯一的纯 Rust 路径上。**

### 5.3 FIPS 没有纯 Rust 路径

CMVP *Modules In Process*（187 个，2026-09-16 快照）与已认证表（1187 张证书）中
Rust／libcrux／Cryspen／Ferrocene／RustCrypto 均为 **0**。所有 Rust FIPS 路径都是 FFI 到已验证的
C 模块（AWS-LC、OpenSSL、BoringCrypto、wolfCrypt、Microsoft SymCrypt——**SymCrypt 是 C**）。

已有人在尝试：`oxicrypt`（"Pure-Rust cryptographic module targeting FIPS 140-3 Level 1"），已实现
AES／SHA／HMAC／KMAC／TupleHash／ParallelHash／DRBG／RSA／ECDSA／ECDH／Ed25519 ＋ ACVP 测试台，
但**仍在 Phase 2、未接触 CST 实验室**。

**这是认证与模块边界问题，不是算法实现问题。**

---

## 6 缺口三：消费端不可替换是设计性的

### 6.1 一个具体样本

| | 某成熟 RP 库 core 0.5.5（稳定版） | 其 0.6.1-dev |
|---|---|---|
| OpenSSL | `openssl`＋`openssl-sys`，**`optional: false`** | **已移除** |
| 可否关闭 | **不能**（0 个 feature；`build.rs` 缺 OpenSSL 直接 panic） | — |
| 替代实现 | — | 纯 Rust 门面 `crypto-glue` |
| **provider trait** | **无** | **仍然无**（整体替换，不是抽象） |

两点必须一起说清：

1. **"必须 OpenSSL"只对旧线成立**，新线已改纯 Rust；**默认分支仍是旧线**，
   **默认分支 ≠ 最新发布版**。
2. **缺口仍在**：源码注释**自称**有 "cryptographic provider abstraction"——这是**过期且未兑现**
   的说法。对两个已发布版本逐一检索，密码学相关的 `pub trait` **一个都没有**。后端类型还
   **泄漏进公开 API**。上游明确表示不打算做可插拔 trait。

### 6.2 顺带发现：一个与加密无关的协议层缺口

该库解析 COSE 用的是 `serde_cbor_2` 而非 `coset`。`serde_cbor_2` 的 `Value::Map` 是 `BTreeMap`，
**重复 COSE 键被静默折叠，后者胜，不报错**；而 `coset` 显式拒绝重复键（`CoseError::DuplicateMapKey`），
源码注释写明 RFC 8152 §14 要求这样做。

**换掉加密后端并不能修好协议层的严格性缺口。** 这正是合同要求 3（有界解析）针对的层。

---

## 7 其他候选后端的定位

| 候选 | 真实定位 | 阻断项 |
|------|---------|--------|
| **graviola 0.4.1** | 唯一"只要 rustc"且**算法面覆盖全部三项**（含 PKCS#1 v1.5 验签）的候选。s2n-bignum 汇编**手工转写成 149 处 Rust `core::arch::asm!`**（0 个 `global_asm!`、0 个 `.s`、无 `build.rs`） | 仅 x86_64＋aarch64，**CPU 特性强制**（缺特性是运行时 panic，非静默降级）。自述 "very new, so exercise due caution"。证明是 s2n-bignum 的 HOL Light——**证的是数学，不含内存安全**，点表选择自述 "non-verified"，**Rust 转写是否被证明覆盖并未声明**。**bus factor ≈ 1** |
| **libcrux 0.0.5** | 形式化验证最强（hacspec → F* → Low* → C via KaRaMel）。**消费端零 C 工具链**。已被 Firefox 用在 MLS 栈 | **RSA 只有 PSS，RS256 不覆盖**。全 0.0.x。无签名 trait。无已发布 rustls provider |
| **`rustls-rustcrypto` 0.0.2-alpha** | — | **事实停更**：最后发布 2024-04-24，依赖仍停在 `p256 ^0.13`／`ecdsa ^0.16`，落后两代。README 原文 **"DO NOT USE THIS IN PRODUCTION"** |
| **`ring`** | — | 最后发布 2025-03-11；无 FIPS；无可核实第三方审计 |

---

## 8 缺口判定汇总

| # | 缺口 | 层级 | 纯 Rust 能否补 | 现状 |
|---|------|------|---------------|------|
| 1 | 算法实现 | 算法 | — | **不缺**（唯一实缺是 libcrux 的 RS256） |
| 2 | 通用（非 TLS）provider trait | 抽象 | 能，但**无人做** | **真缺口（Groma L1）** |
| 3 | 统一后端门面 | 抽象 | 部分有 | `crypto-glue` 存在但绑死单一项目，非通用设施 |
| 4 | 审计覆盖 | 证据 | 需投入，非技术问题 | **真缺口**（Groma 逐 crate 标注，不整体宣称） |
| 5 | 未修补公告 | 证据 | 需上游修 | **真缺口**：RUSTSEC-2023-0071 落在 RS256 路径 |
| 6 | FIPS 140-3 | 合规 | **不能** | **Groma 显式非目标** |
| 7 | 消费端可替换 | 集成 | 需上游改或 Fork | **Groma L7 适配器针对此** |
| 8 | 协议层严格性 | 协议 | 能 | Groma `codec` 的有界解析针对此 |

### 8.1 一句话回答

> Rust 生态**不缺纯 Rust 的加密算法**，缺的是**把"纯 Rust"提升到"可审计、可声明、后端可插拔"
> 这一层**——而这一层的缺失不在算法实现，在**抽象（无通用 provider trait）、证据（审计与 FIPS
> 落差）与集成（消费端写死）**。`aws-lc-rs` 的护城河不是算法，是 **FIPS 140-3 ＋ rustls 默认位**。

---

## 9 已知限制

| 限制 | 说明 |
|------|------|
| 静态快照 | 所有版本／日期为 2026-09-16 资料快照；未运行任何代码、未做性能测试 |
| 审计是"未查到"而非"不存在" | 属**缺乏证据**，不等于证明其从未被审计 |
| 缺 FIPS 证书的穷尽性 | 依据是各库自有文档的沉默与 CMVP 名称检索，**未穷尽查询 CMVP 库**（检索界面为 JS 驱动） |
| 未追踪的依赖边 | 例如某浏览器引入 libcrux 的具体上游边未追踪 |
| 预印本 | IACR 2026/192（libcrux/hpke-rs 漏洞）未经同行评审，只能作线索 |
| `unsafe` 计数口径 | 为源码中 `unsafe` 文本出现次数（含注释），非已审计的 unsafe 块数 |
| 探测失败 ≠ 不存在 | 本轮实测出 SSL 失败导致的假 404，负结论必须多源复核 |

---

## 修订历史

| 日期 | 变更 |
|------|------|
| 2026-09-16 | 从原项目迁入并改为 Groma 调研文档；补 `aws-lc-rs` 实测模块面与 FIPS 4.x 未认证事实、libcrux RS256 缺口、方法学修正。 |
| 2026-09-16 | 建立原始版：核实构建要求、FIPS 状态、审计覆盖与消费端可替换性。 |
