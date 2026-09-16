# Rust 密码学"纯 Rust 缺口"实证调研报告（2026-09-16）

调研对象：Groma（计划中的"无 C 构建要求的纯 Rust 密码学供给层"）生态位决策依据。
方法：web_search / web_fetch 多源交叉（crates.io API、docs.rs、GitHub Cargo.toml 源码、users.rust-lang.org、GitHub issues），版本与更新时间均以 2026-09-16 抓取为准。

---

## 一、知名"纯 Rust 项目"仍依赖 C 密码学的现状（2026）

| 项目 | 现状 | C 依赖 | 证据 |
|---|---|---|---|
| **webauthn-rs（Kanidm）** | 0.5.5（2026-04）核心库 **硬依赖 openssl+openssl-sys**（非可选）；0.4.9（2022）起即如此 | OpenSSL | [webauthn-rs-core Cargo.toml](https://github.com/kanidm/webauthn-rs/blob/master/webauthn-rs-core/Cargo.toml)、[0.5.5 deps](https://crates.io/api/v1/crates/webauthn-rs-core/0.5.5/dependencies) |
| **quinn（QUIC）** | 0.11.12（2026-09）默认 `rustls-ring`（2024-09 起）；可选 aws-lc-rs | ring（C+asm） | [quinn/Cargo.toml](https://github.com/quinn-rs/quinn/blob/main/quinn/Cargo.toml)、[crates.io 0.11.12 features](https://crates.io/crates/quinn/0.11.12/features) |
| **russh（SSH）** | 0.63.3 默认 `aws-lc-rs`（备选 ring）做签名后端，其余全 RustCrypto | aws-lc-rs | [russh/Cargo.toml](https://github.com/warp-tech/russh/blob/main/russh/Cargo.toml)、[rpg#603 抱怨 aws-lc-sys 拖慢构建](https://github.com/NikolayS/rpg/issues/603) |
| **boringtun（WireGuard）** | 0.7.1 仍带 ring 0.17，但**仅用于 `ring::constant_time::verify_slices_are_equal` 一处**（防 DoS MAC 比较）；X25519/ChaCha20Poly1305/Blake2s 全是 RustCrypto/x25519-dalek | ring（残留，替换成本极低） | [rate_limiter.rs 源码](https://github.com/cloudflare/boringtun/blob/master/boringtun/src/noise/rate_limiter.rs) |
| **sequoia-openpgp** | 2.4.1 默认后端 **crypto-nettle**（C 库）；存在 crypto-rust（RustCrypto+ML-KEM/ML-DSA/SLH-DSA）但非默认 | Nettle | [features 页](https://docs.rs/crate/sequoia-openpgp/latest/features) |
| **rpgp（crate 名 `pgp`）** | 0.20.0（2026-06）**全纯 Rust**（RustCrypto + zlib-rs），可选 PQC draft，620 万下载，活跃 | 无（bzip2 可选） | [rpgp Cargo.toml](https://github.com/rpgp/rpgp/blob/master/Cargo.toml) |
| **rustls** | 0.23.45（2026-09）默认 `aws_lc_rs`（+新默认 `prefer-post-quantum`）；**0.24.0-dev 已把 provider 全部移出核心**（default=log+webpki） | aws-lc-rs（默认） | [crates.io rustls features](https://crates.io/crates/rustls/0.23.45/features)、[0.24.0-dev.1](https://crates.io/crates/rustls/0.24.0-dev.1) |
| **X.509 解析链** | x509-parser 0.18.1（1.6 亿下载，纯 Rust 解析；但 `verify` 特性绑 ring / `verify-aws` 绑 aws-lc-rs）；rustls-pki-types 1.15.1（5 亿下载，纯类型无加密） | 仅 verify 特性 | [crates.io x509-parser](https://crates.io/crates/x509-parser/0.18.1/features)、[rustls-pki-types](https://crates.io/crates/rustls-pki-types) |

**已出现的纯 Rust TLS 供给**（Groma 的对标/竞品，注意区分）：
- **graviola 0.4.1**（2026-06，rustls 作者 ctz）：仅需 rustc 编译（无 C/无汇编器），内部用 s2n-bignum 形式化验证汇编；限 x86_64（2014 后 CPU）/aarch64。README 自述"very new"。→ [github.com/ctz/graviola](https://github.com/ctz/graviola)
- **rustls-graviola 0.3.1**、**rustls-rustcrypto**（n0-computer）、**oxitls**（2026，"rustls+RustCrypto zero FFI" facade）——2026 年已出现多个"纯 Rust rustls 拼装层"，说明该层正在被抢占。

---

## 二、8 项能力逐项核查（版本=2026-09-16 crates.io）

| 能力 | 纯 Rust 现状 | 成熟度 | 真实消费者 | 三判据结论 |
|---|---|---|---|---|
| **PKCS#12 解析**（含 MAC） | `pkcs12` 0.1.0（2024-01，仅 356 行）；0.2.0-pre.0（2026-01）补 kdf/PbeParams/MacData | 解析可用、**创建（导出 .pfx）完全缺失**；稳定版 MAC 能力不完整 | 无纯 Rust 消费者；需求走 openssl crate | 规范✓(RFC7292)；向量△（无官方向量集）；消费者✓但未被满足 → **真实缺口** |
| **CMS/PKCS#7 签名验证** | `cms` 0.2.3（2024-01，稳定，1.2 亿下载）SignedData 签名/验证；EnvelopedData 仅在 0.3.0-pre.2（2026-01） | 中等偏下：真实世界消息解析报错（见论坛案例）；encryption 未发稳定版 | 少（rcgen 等用 x509-cert 而非 cms）；AdES 类需求在用 openssl | 规范✓(RFC5652)；向量✓（RFC 附录样例）；消费者△ → **半缺口** |
| **OCSP 客户端与响应解析** | `x509-ocsp` 0.2.1（**2024-01 后停更**，仅类型+请求构造）；`rasn-ocsp` 0.28.14（活跃，ASN.1 类型）；客户端仅 `ocsp-stapler` 0.4.9（blind-oracle，11.7 万下载，支持 rustls） | 碎片化：无"验证级"维护良好的客户端，无与路径验证的集成 | ocsp-stapler / tsp-ltv（2026，15 万下载） | 规范✓(RFC6960 附录有向量✓)；消费者✓ → **真实缺口** |
| **X.509 路径构建/名称约束/CRL** | `pkix-path` 0.2.1（2026-05，RFC5280 §6，纯 Rust no_std，含 NameConstraints；CRL/OCSP 走 pkix-revocation）；`synta-x509-verification` 0.3.4（2026-09，abbra）；rustls-webpki 只做 TLS 链 | **2026 年才出现，无大规模实战记录**；rustls 栈无内置 CRL/名称约束 | 尚无大消费者（x509-cert 3.9 亿下载仅做结构与签发） | 规范✓；向量✓（NIST PKITS 公开）；消费者△ → **真实缺口（时间窗正在收窄）** |
| **TPM 2.0 结构与证明** | 纯 Rust：`tpm2`(tpm-rs) 0.1.0（2026-04，早期）、purecrypto-tpm 0.1.0、hyde-tpm 0.3.0、crumpet 0.1.2——全部 0.x/零下载；主流仍是 **tss-esapi 7.7.0（C tpm2-tss 绑定，350 万下载）**，微软 ms-tpm-20-ref-rs 也是 C 绑定 | 纯 Rust 栈**空白/胚胎期** | Azure 机密计算 az-snp/vtpm（33 万下载，依赖 tss-esapi）等全部绑 C | 规范✓(TCG 公开)；向量✓（TCG 测试套件+官方模拟器）；消费者✓被 C 锁死 → **最大缺口** |
| **PKCS#11 客户端** | `cryptoki` 0.12.1（**2026-09-16 当天发版**，纯 Rust dlopen 封装，267 万下载） | 成熟、活跃 | Parsec、parallaxsecond 系 key-manager | 规范✓(OASIS)；向量✓(SoftHSM)；消费者✓ → **已解决，无需做** |
| **JOSE/JWS（RS256、EdDSA）** | `jose-jws` 0.1.2（2023-08 后停更，RFC7515，支持 RS256/ES256 等）；**jsonwebtoken 11.1.0（189 亿次下载，10.0 起 trait 化后端：rust_crypto 纯 Rust / aws_lc_rs 可选；≤9 用 ring）** | 主流路径已纯 Rust 化 | 全生态 | 规范✓；向量✓(RFC7515 附录)；消费者✓ → **已解决** |
| **SSH 密钥格式（RFC 4716）** | `ssh-key` 0.6.7 / 0.7.0-rc.11（2026-06，1140 万下载，RFC4716+OpenSSH+sshsig+证书） | 成熟、活跃 | russh 及大量 SSH 工具链 | 规范✓；向量✓(RFC4716 示例)；消费者✓ → **已解决** |

---

## 三、社区需求信号（2024–2026）

**被问得最多、最缺的主题**（按证据强度排序）：

1. **"免 C 构建"是高频痛点**：reqwest#2937「Windows 上 aws-lc-sys 强制 CMake/NASM/clang，`--no-default-features -F rustls` 也不行」[链接](https://github.com/seanmonstar/reqwest/issues/2937)；sqlx#4033 要求「rustls 无默认 provider/根证书」[链接](https://github.com/transact-rs/sqlx/issues/4033)；rpg#603「russh 拉入 aws-lc-sys 构建极慢」[链接](https://github.com/NikolayS/rpg/issues/603)；论坛 [Rustls vs openssl 2024](https://users.rust-lang.org/t/rustls-vs-openssl-2024/111754)。
2. **PKCS#12**：论坛求助「Legacy PKCS12 TLS Support」[链接](https://users.rust-lang.org/t/legacy-pkcs12-tls-support/95213)；lib.rs/crates/pkcs12 长期存在且使用者众，但只能解析。
3. **CMS/PKCS#7**：2024-07 论坛帖——用 RustCrypto `cms` 解析真实（微软时间戳）CMS 直接 panic `TagUnexpected`，"OpenSSL 解析正常"，90 天无人解答被自动关闭 [链接](https://users.rust-lang.org/t/rustcrypto-cms-tagunexpected-error-but-works-fine-in-openssl/114912)；另有 [Extract certificate from pkcs7](https://users.rust-lang.org/t/extract-certificate-from-pkcs7/75979)（2280 次浏览）。
4. **OCSP**：crates.io 搜索 "ocsp" 有 123 个 crate 但无公认主力；x509-ocsp 停更、ocsp-x509 已弃用（"DEPRECATED. Use RustCrypto's x509-ocsp"）。
5. **TPM**：纯 Rust 尝试层出不穷（tpm-rs/purecrypto-tpm/crumpet/hyde-tpm 全部 2026 年发布、0 下载），反证需求未获满足；真实消费者（Azure 机密计算、Parsec、fidorium 等）全绑 tss-esapi(C)。
6. **"纯 Rust 化"的主动工程信号**：codemonger-io 分支专门移除 webauthn-rs 的 OpenSSL 依赖 [commit](https://github.com/codemonger-io/webauthn-rs/commit/1c4e6201f4d6d5e897ee47b066055be4b0c4723d)；kanidm PR#534「6.0: Remove caBLE OpenSSL dependency」[链接](https://github.com/kanidm/webauthn-rs/pull/534)；boringtun 多个 fork（Pothulapati/hackclub/defguard）都在去 ring。

---

## 四、"最痛的五个缺口"排名

| # | 缺口 | 痛因 | 关键证据 URL |
|---|---|---|---|
| 1 | **TPM 2.0 纯 Rust 栈（结构编解码+证明）** | 唯一现实选择是 C 绑定（tss-esapi 350 万下载）；微软、Azure 机密计算全绑 C；纯 Rust 项目全部 0.x 且 2026 年才起步 | [tpm-rs](https://github.com/tpm-rs/tpm-rs)、[ms-tpm-20-ref-rs](https://github.com/microsoft/ms-tpm-20-ref-rs)、[az-cvm-vtpm deps(tss-esapi)](https://crates.io/api/v1/crates/az-cvm-vtpm/0.8.2/dependencies)、[tss-esapi](https://crates.io/crates/tss-esapi) |
| 2 | **PKCS#12 完整支持（创建/导出 + MAC 全流程）** | RustCrypto pkcs12 仅 356 行解析代码、创建能力为零；稳定版 0.1.0 已两年未动 | [pkcs12 0.2.0-pre docs](https://docs.rs/pkcs12/0.2.0-pre.0/pkcs12/)、[论坛求助](https://users.rust-lang.org/t/legacy-pkcs12-tls-support/95213) |
| 3 | **OCSP 验证级客户端** | 解析类型有三个 crate 却停更/碎片化；唯一客户端 ocsp-stapler 单人维护；与 X.509 路径验证零集成 | [x509-ocsp(2024 停更)](https://crates.io/crates/x509-ocsp)、[ocsp-stapler](https://crates.io/crates/ocsp-stapler) |
| 4 | **CMS/PKCS#7 健壮解析 + EnvelopedData** | 稳定版只能 SignedData 且真实消息解析崩溃；envelope 两年停留在 pre 版；AdES/电子签场景被迫回 OpenSSL | [cms 0.2.3](https://docs.rs/crate/cms/0.2.3)、[解析崩溃案例](https://users.rust-lang.org/t/rustcrypto-cms-tagunexpected-error-but-works-fine-in-openssl/114912) |
| 5 | **X.509 全量路径验证（名称约束/策略/CRL+OCSP 集成）** | 直到 2026-05 才出现 pkix-path（无实战背书）；rustls 家族只做 TLS 链；synta 同月出现——窗口期 1 年以内 | [pkix-path docs](https://docs.rs/pkix-path/0.2.1/pkix_path/)、[synta-x509-verification](https://crates.io/crates/synta-x509-verification) |

---

## 五、对 Groma 的启示

**值得自研补缺（判据三合一 + 无成熟供给）：**
1. **TPM 2.0 纯 Rust 供给层**——最高优先。做"结构编解码（TCG TPM2B/TSS 数据结构）+ 命令封装 + 证明（quote/attestation）验证"纯 Rust 栈，直接对标 tss-esapi 的 C 绑定生态位（Azure/微软/机密计算都是现成消费者）。测试向量有官方模拟器，判据最硬。
2. **PKCS#12 创建+解析一体化**——解析依赖 RustCrypto pkcs12 即可，核心补"创建/导出"（含 MAC 计算、PBE 加密），这是 openssl crate 用户最常被迫引入 C 的场景。
3. **OCSP 客户端（含响应签名验证、与 X.509 链集成）**——在 x509-ocsp/rasn-ocsp 类型之上做验证逻辑与 HTTP 客户端，并可对接 pkix-path 做吊销检查。
4. **CMS 解析加固 + EnvelopedData**——先做"容错解析 + SignedData 验证"，再做加密；与 PKCS#12 共享 ASN.1 底座。
5. **X.509 路径验证**——pkix-path/synta 尚无消费者，仍有 6-12 个月窗口；可自研或与 pkix-path 合作（提供 CRL/OCSP 集成与审计测试）。

**不值得做（生态已解决或被抢占）：**
- PKCS#11（cryptoki 成熟 + Parsec 背书）、JOSE/JWS（jsonwebtoken 10+ 已纯 Rust 化）、SSH/RFC4716（ssh-key 成熟）、OpenPGP（rpgp 纯 Rust 且活跃）、TLS 原语（graviola/rustls-rustcrypto/oxitls 已占位，rustls 0.24 将去默认 provider）、WireGuard 原语（boringtun 的 ring 仅剩一行常数时间比较，fork 即可去 C）。
- **注意**：quinn/russh/webauthn-rs 等仍默认 C 后端，Groma 的价值不是重写它们，而是提供"provider 可插拔的验证/格式层"，让这类项目能一键切到纯 Rust。

**战略建议**：Groma 的差异化定位应是"**无 C 构建的互操作/验证层**"（PKI 格式 + 协议 + 验证策略），而不是再做一个 RustCrypto 算法集或 rustls provider——后者已被 graviola（rustls 作者）与 rustls-rustcrypto 卡位。切入顺序建议：TPM → PKCS#12 → OCSP → CMS → 路径验证。
