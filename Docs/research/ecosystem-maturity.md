# 纯 Rust 密码学 crate 生态调研报告（2026-09-16）

> 调研对象：Groma 项目"纯 Rust 通用密码学供给层/后端合同库"定位决策
> 数据源：crates.io API（当日实时）、rustsec.org 公告库（2025–2026 全部 457 条公告全量扫描）、各项目 GitHub README/Cargo.toml/CHANGELOG 原文。
> 提示：crates.io 探测失败 ≠ crate 不存在；present/simon/seed/falcon/lms 等名称均为无关项目，本报告未采用任何此类误认。

---

## 一、RustCrypto 组织

**成熟度摘要**：纯 Rust 生态的事实标准组织（40 个仓库）。2026 年是"0.11 世代"完成年：`digest 0.11.0` 于 **2026-02-13** 稳定（0.11.3 现为最新），`sha2 0.11.0`（2026-03-25）、`aes-gcm 0.11.1`、`sha3 0.12.0`、`hmac/hkdf/pbkdf2 0.13.0`、`ecdsa 0.17.0`、`p256/p384/k256/p521 0.14.0`、`argon2 0.6.0`（2026-08-27）均已跟进；`ed25519-dalek 3.0.0`/`curve25519-dalek 5.0.0`（dalek 家族，2026-07-06）同步完成迁移。**MSRV 全线 1.85**（Rust 2024 edition）：digest Cargo.toml 明确 `edition="2024", rust-version="1.85"`（https://github.com/RustCrypto/traits/blob/master/digest/Cargo.toml），traits 仓库 README 亦以 1.85 徽章标注全部 trait crate（https://github.com/RustCrypto/traits）。唯一例外：**`rsa` 仍停在 0.9.10，0.10.0 已发 18 个 rc**（rc.18，2026-04-27）——RSA 是迁移最慢的主线。PQC 命名定型：`ml-kem 0.3.2`（RustCrypto/KEMs）、`ml-dsa 0.1.1`、`slh-dsa 0.1.0`（RustCrypto/signatures，取代 fips203/fips204/fips205 命名）。
**审计**：历史上有 NCC Group 2020 年对 AES-GCM/ChaCha20+Poly1305 的公开审计（https://www.nccgroup.com/research/public-report-rustcrypto-aesgcm-and-chacha20pluspoly1305-implementation-review/），Bitwarden 2025 年委托第三方出具《RustCrypto Library Security Report》（https://bitwarden.com/assets/5YGFfe5sU4Op1aWvFnSi6b/1f059c111445194bce1c185c8a3fdbaf/2025_Bitwarden_RustCrypto_Library_Security_Report.pdf）。
**安全公告**：2025-12-12 的 **RUSTSEC-2025-0144**（ml-dsa/fips204，CVE-2026-22705）：ML-DSA 分解步骤中对秘密数据做变时除法的时间侧信道，修复于 0.1.0-rc.3（https://rustsec.org/advisories/RUSTSEC-2025-0144.html）。未见 RustCrypto 侧 patched=[] 的未修复公告。
**采用度**：sha2 总下载 9.46 亿、近 90 天 2.52 亿；digest 10.7 亿/2.78 亿（https://crates.io/api/v1/crates/sha2）。
**风险**：低（单 crate 质量高）；生态级风险是世代迁移期版本分裂与 rsa 0.10 未定稿。

## 二、graviola（ctz/rustls 创始人 Joe Birr-Pixton）

**版本**：graviola 0.4.1（2026-06-24）、rustls-graviola 0.4.0（2026-06-17）；仓库 2026-08 仍在活跃提交。
**覆盖**：SHA2/SHA3/SHAKE/HMAC、AES-GCM、ChaCha20-Poly1305、X25519、P256/P384 KEX、**ML-KEM-768**、Ed25519、ECDSA(P256/P384)、RSA-PSS/PKCS#1 签名与验签（**明确拒绝实现 RSA 加密**："These are not made. They should never be made."）、RSA 密钥生成（2048–8192 五档，**自述目前非侧信道安全**）。
**实现**：底层采用 AWS s2n-bignum **形式化证明的汇编**（P256/P384/P521、X25519、ML-KEM、Keccak-f 等）；Wycheproof 测试向量；声称性能对标 ring/aws-lc-rs/RustCrypto（https://jbp.io/graviola/）。
**rustls 集成**：rustls-graviola 是 rustls 官方 README 列出的第三方 provider（https://github.com/rustls/rustls），0.4.0 已稳定发布；graviola README 称"本 crate 的首要目标是经 rustls-graviola 供 rustls 使用"。
**硬性架构限制**：**仅 x86_64 / aarch64**。x86_64 强制要求 `aes, ssse3, avx, avx2, adx, bmi2, pclmulqdq`（约 2014 年后 CPU，无运行时降级路径——AES/GHASH 只有 intrinsic 实现）；aarch64 要求 `aes, sha2, pmull, neon`（树莓派 4 及更早不支持，Pi 5 支持）（https://github.com/ctz/graviola）。
**成熟度信号**：README 明示 "This project is very new, so exercise due caution"；**无第三方安全审计记录**；采用度低（总下载 25.6 万，rustls-graviola 13.9 万）。
**风险**：新、审计缺失、CPU 特性硬门槛；但构建零 C 依赖（纯 rustc，<1s 编译）。

## 三、libcrux（CELA/celabshq，原 Cryspen）

**版本**：全仓库 pre-1.0：libcrux 0.0.5（2026-07-15），算法按 crate 分发：libcrux-ml-kem 0.0.10、libcrux-ml-dsa 0.0.10、libcrux-sha2/sha3/hmac/ecdh/poly1305/chacha20poly1305 0.0.8、**libcrux-rsa 0.0.8（仅 RSA-PSS，总下载 4345——极新）**。README 明确"所有 crate 版本 <0.1；生产使用前请联系维护者"（https://github.com/celabshq/libcrux）。仓库已从 cryspen/libcrux 迁移至 celabshq/libcrux（GitHub 重定向确认），维护极活跃（2026-09-16 当日仍在提交）。
**覆盖**：ML-KEM、ML-DSA、Ed25519、P256 ECDSA/DHKEM、X25519、XWing 与 X25519MlKem768Draft00 混合 KEM、AES-GCM/CCM（已更名 libcrux-aes）、ChaCha20Poly1305、SHA2/SHA3、BLAKE2、HKDF/HMAC/Poly1305、PSQ 协议。**RSA 缺口确认：仅 PSS**（无 PKCS#1 v1.5、无 RSA 加密）。
**形式化验证表述**（README 原话）：HACL* 提取代码验证内存安全、对高层规格的功能正确性与 secret-independence；部分 Rust 代码用 hax 工具链验证"无 panic + 相对数学规格功能正确"（hax 论文 https://eprint.iacr.org/2025/142）；**但"编译产物不保证侧信道安全"**；部分 crate 标 pre-verification 徽章（大部分默认 feature 未验证）。MSRV 1.89。
**安全公告（重要）**：2026 年密集公告——X25519 校验缺陷/夹紧检查错误（RUSTSEC-2026-0023/0024，CVE-2026-76234）、libcrux-psq 解密 panic（0025）、Ed25519 seed 多余夹紧损失熵（0026）、poly1305 独立 MAC panic（0073）、增量 SHAKE 输出错误（0074）、ML-DSA 签名提示解码 panic（0076）、AVX2 未完全约减（0126 notice）、**aarch64 常量时间 swap/select 可能输出错误（0212）**、AES-GCM 未强制 AAD 长度限制（0209）、libcrux-intrinsics aarch64 计算错误（RUSTSEC-2025-0133）。均为 patched 公告，但密度说明胶水层（非验证核心）是风险区。
**风险**：高保证叙事 ≠ 产品成熟；pre-1.0 语义不承诺稳定。

## 四、传统强者：ring / aws-lc-rs / openssl-sys·native-tls

### ring
0.17.14 之后 **18 个月无发布**（最后发布 2025-03-11），仓库 0.17 分支有零星提交（最近 2026-07-22）但不出新版本。RustSec 曾公告"ring 已无人维护"（RUSTSEC-2025-0007，2 天后撤回），正式版 **RUSTSEC-2025-0010 将全部 <0.17 版本标记 unmaintained、patched=[]**（https://rustsec.org/advisories/RUSTSEC-2025-0010.html）。rustls 官方 README 将 aws-lc-rs 列为推荐第一方 provider，并明说 rustls-ring "功能集更有限（无后量子算法）"。构建要求：crates.io 发布包**仅需 C 工具链**（Perl 产物已打包、Windows 汇编预编译）；从 Git 构建需 Perl+NASM（https://github.com/briansmith/ring/blob/main/BUILDING.md）。仍被大量使用：737M 总下载、近 90 天 157M——惯性 + 构建简单 + 无 FIPS 需求场景。

### aws-lc-rs
1.18.1（2026-09-01），AWS 背书、极活跃。背后是 **6 张 FIPS 140-3 证书**的 AWS-LC：v1.0 #4631、NetOS 动态 #5146、v2.0 动态 #5429/静态 #4816、**v3.1 动态 #5298/静态 #5314**；v4.0 已送审在途；熵源证书 #E77(2023)/#E280(2025-08)（https://github.com/aws/aws-lc/blob/main/crypto/fipsmodule/FIPS.md）。aws-lc-rs 1.18.x 绑定 **AWS-LC-FIPS 4.x（尚未获证）**，3.0.x 模块需锁 <1.18.0。构建要求：非 FIPS 构建需 C/C++ 编译器（CMake/bindgen/Go 均不需要），**FIPS 构建需 CMake+Go+可能 bindgen**，Windows x86/x86-64 需 NASM（https://github.com/aws/aws-lc-rs/blob/main/aws-lc-rs/README.md）。含后量子算法（ML-KEM/ML-DSA）。**它也有真漏洞**：2026-03-02 一批 aws-lc-sys/aws-lc-fips-sys 公告（如 RUSTSEC-2026-0045：AES-CCM 计时侧信道，CVE-2026-3337，修复于 0.38.0）。采用度：224M 总下载、近 90 天 83M（增速已超 ring）。

### openssl-sys / native-tls
openssl 0.10.81 / openssl-sys 0.9.117（2026-06-12）仍活跃维护；native-tls 0.2.18（2026-02-18）走平台原生后端（Windows SChannel、macOS Security.framework、Linux→OpenSSL，https://docs.rs/native-tls）。openssl crate 本身出过内存漏洞：**CVE-2025-24898**（`ssl::select_next_proto` use-after-free，RUSTSEC-2025-0004，修复于 0.10.70）。构建要求：需系统 OpenSSL 或 `vendored` 特性从源码编译（C+Perl+make）。合计 8 亿+下载，靠生态惯性（reqwest 默认后端历史、系统信任锚）维持。
**三者小结**：仍被使用的原因排序 = **FIPS/合规认证 > 长期审计记录与供应商背书 > 生态惯性 > 性能**；纯 Rust 侧目前没有任何 FIPS 140-3 验证路径，这是 C 依赖最硬的护城河。

## 五、其他值得注意的项目

| 项目 | 状态 | 要点 |
|---|---|---|
| **orion** | 0.18.0（2026-08-30），活跃 | 纯 Rust"一站式"：ML-KEM/ML-DSA/X-Wing/HPKE/Argon2/scrypt/SHA2-3/BLAKE2-3/X25519/(X)ChaCha20-Poly1305；unsafe-forbidden；MSRV 1.87；X25519/Poly1305 用 Fiat Crypto 验证算法。**README 自述"未经过任何第三方安全审计，使用风险自负"**（https://github.com/orion-rs/orion）。12.8M 下载。 |
| **dalek 家族** | curve25519-dalek 5.0.0 / ed25519-dalek 3.0.0 / x25519-dalek 3.0.0（2026-07-06） | MSRV 1.85、edition 2024、signature 3.0、sha2 0.11（https://github.com/dalek-cryptography/curve25519-dalek/blob/main/ed25519-dalek/CHANGELOG.md）；Quarkslab 2019 年审计（https://blog.quarkslab.com/security-audit-of-dalek-libraries.html）。Ed25519/X25519 事实标准。 |
| **hacspec / hacl-rs / EverCrypt** | 基本休眠 | hacspec-lib 最后发布 2023（总下载仅 3023）；evercrypt-sys 0.0.9 止于 2022-02；cryspen/hacl-rs 仓库 404。HACL* 的 Rust 成果实际通过 libcrux 与 RustCrypto（fiat-crypto）间接进入生态。 |
| **libsodium 绑定** | sodiumoxide 0.2.7 死于 2021-06 | 8.4M 下载但 5 年无维护；替代品 **dryoc 1.0.0（2026-07-22）**——纯 Rust 复刻 libsodium API（https://github.com/SimenB/dryoc），采用度尚低（24 万下载）。 |
| **pqcrypto 家族** | **2026-06-04 集体死亡** | RustSec 同一天 9 条 unmaintained 公告（pqcrypto/pqcrypto-internals/traits/mlem/mldsa/sphincsplus/falcon/hqc/classicmceliece），**全部 patched=[]**，根因是上游 PQClean 项目归档（https://rustsec.org/advisories/RUSTSEC-2026-0160.html）。pqcrypto-kyber 0.8.1 也已 2.5 年无更新。基于 pqcrypto 的旧后端是死路。 |
| **RustCrypto PQC（续）** | ml-kem 0.3.2 / ml-dsa 0.1.1 / slh-dsa 0.1.0 | 2026 年事实标准；ml-kem 近 90 天 500 万下载。fips203/204/205 旧命名停止演进（2024-12 后无新版本）。 |

## 六、crates.io 下载量（2026-09-16 实时抓取）

**总下载 / 近 90 天下载**（https://crates.io/api/v1/crates/&lt;name&gt; 的 `downloads`/`recent_downloads` 字段）：

| crate | 版本 | 总下载 | 近90天 |
|---|---|---|---|
| digest | 0.11.3 | 1,070M | 278M |
| sha2 | 0.11.0 | 946M | 252M |
| rustls | 0.23.45 | 916M | 200M |
| ring | 0.17.14 | 737M | 157M |
| hmac | 0.13.0 | 582M | 148M |
| openssl-sys | 0.9.117 | 448M | 76M |
| aws-lc-rs | 1.18.1 | 225M | 83M |
| rsa | 0.9.10 | 221M | 51M |
| ed25519-dalek | 3.0.0 | 212M | 55M |
| aes-gcm | 0.11.1 | 156M | 43M |
| ml-kem | 0.3.2 | 7.1M | 5.0M |
| libcrux-ml-kem | 0.0.10 | 3.0M | 1.3M |
| orion | 0.18.0 | 12.8M | 1.6M |
| graviola | 0.4.1 | 0.26M | 0.16M |
| rustls-graviola | 0.4.0 | 0.14M | 0.09M |
| dryoc | 1.0.0 | 0.24M | 0.06M |
| pqcrypto-kyber | 0.8.1 | 1.7M | 0.19M |

趋势：纯 Rust（RustCrypto/dalek）总盘与增量均统治；aws-lc-rs 增量（83M/90d）已超 ring 增量的一半并高速追赶；ML-KEM 采用爆发（近 90 天 500 万）。

## 七、全景总表

| 项目 | 版本(2026-09) | MSRV | 纯Rust | 第三方审计 | 活跃度 | 采用度 | 主要风险 |
|---|---|---|---|---|---|---|---|
| RustCrypto | 0.11 世代（digest 0.11.3/sha2 0.11.0/aes-gcm 0.11.1；rsa 0.9.10→0.10rc） | 1.85 | 是 | 部分（NCC 2020、Bitwarden 2025；ml-dsa 有 CVE） | 极活跃 | 统治级（sha2 9.5 亿） | rsa 0.10 未定稿；无 FIPS |
| dalek | ed25519-dalek 3.0.0 / curve25519-dalek 5.0.0 | 1.85 | 是 | Quarkslab 2019 | 活跃 | 极高 | 算法面窄 |
| graviola | 0.4.1 | 1.85?（未明示，新库） | 是 | 无 | 活跃（ctz 单人主导） | 低 | 新+无审计；x86_64/aarch64+强 CPU 特性硬要求 |
| libcrux | <0.1（0.0.x 系列） | 1.89 | 是 | 无传统审计（形式化验证替代叙事） | 极活跃 | 低（libcrux-ml-kem 300 万） | pre-1.0；2026 公告密集；RSA 仅 PSS |
| ring | 0.17.14（2025-03 后停更） | 1.66 | 否(C+asm) | 有（历史；供应商背书弱化） | 低（零星提交无发布） | 高（惯性） | <0.17 官方 unmaintained；无 PQ |
| aws-lc-rs | 1.18.1 | 1.66+ | 否(C/asm) | AWS 生态+FIPS 认证 | 极活跃 | 高且增速最快 | C 构建链；FIPS 需 CMake+Go；2026-03 有 CVE 批次 |
| openssl/native-tls | 0.10.81 / 0.2.18 | 低 | 否 | OpenSSL 本体审计 | 活跃（维护者少） | 高（惯性） | UAF 类 CVE 史；构建链重 |
| orion | 0.18.0 | 1.87 | 是 | **无（自述）** | 活跃 | 低-中 | 无审计 |
| pqcrypto | 0.8.x/0.18.x | — | 否(PQClean C) | 无 | **2026-06 官方 unmaintained** | 低且流失 | 死项目 |
| libsodium 绑定 | sodiumoxide 0.2.7 死；dryoc 1.0.0 | — | dryoc 是 | 无 | dryoc 活跃 | 低 | 生态断裂 |
| hacspec/EverCrypt/hacl-rs | 休眠 | — | 是 | — | 停 | 无 | 停摆，成果被 libcrux/RustCrypto 吸收 |

## 八、生态十大事实（对 Groma 定位影响排序）

1. **0.11 世代迁移 2026 年才完成**：digest 0.11.0 发布于 2026-02-13、sha2 0.11.0 于 2026-03-25、ed25519-dalek 3.0.0 于 2026-07-06，全线 MSRV 1.85（Rust 2024 edition）——Groma 的 trait 合同必须对齐 0.11/0.13 世代而非 0.10。依据：https://crates.io/api/v1/crates/digest/versions 、https://github.com/RustCrypto/traits/blob/master/digest/Cargo.toml
2. **纯 Rust 已是主流而非边缘**：sha2 总下载 9.46 亿且近 90 天仍有 2.52 亿——"Rust 项目都用 C 库做加密"的时代已结束。依据：https://crates.io/api/v1/crates/sha2
3. **RustSec 公告生态对纯 Rust 是净加分**：ml-dsa 的 CVE-2026-22705（RustCrypto，2025-12）在 rc 阶段即修复，而 rustls 本身 2026-09-14 还有 TLS 1.3 加密层边界公告——公告密度是"被盯着"的信号，不是衰败信号。依据：https://rustsec.org/advisories/RUSTSEC-2025-0144.html 、https://rustsec.org/advisories/RUSTSEC-2026-0285.html
4. **ring 实际停摆**：0.17.14（2025-03-11）后 18 个月零发布，<0.17 被 RustSec 正式标记 unmaintained（patched=[]），rustls 官方已改推 aws-lc-rs——依赖 ring 的后端合同有战略风险。依据：https://rustsec.org/advisories/RUSTSEC-2025-0010.html 、https://github.com/rustls/rustls
5. **C 依赖存续的头号原因是 FIPS 140-3 认证**：aws-lc-rs 背后的 AWS-LC 握有 6 张证书（v3.1 #5298/#5314，v4.0 送审中）；纯 Rust 没有任何 FIPS 验证路径——这是 Groma 无法替代、只能"对接"的边界。依据：https://github.com/aws/aws-lc/blob/main/crypto/fipsmodule/FIPS.md
6. **C 依赖的构建税仍然沉重**：aws-lc-rs 需要 C/C++ 编译器（FIPS 需 CMake+Go、Windows 需 NASM），ring 从 Git 构建需 Perl+NASM——"纯 Rust、零工具链"是 graviola/gorion 路线可量化的卖点。依据：https://github.com/aws/aws-lc-rs/blob/main/aws-lc-rs/README.md 、https://github.com/briansmith/ring/blob/main/BUILDING.md
7. **graviola 是"高性能纯 Rust"上限样本但带硬约束**：s2n-bignum 形式化验证汇编 + rustls provider 0.4.0 已发布，但仅 x86_64/aarch64、x86_64 强制 AVX2/ADX/BMI2/PCLMULQDQ 且无软件回退，README 自认 "very new"——适合作为 Groma 的可插拔后端而非兜底。依据：https://github.com/ctz/graviola
8. **libcrux 表明"形式化验证"标签不能当产品成熟度用**：全部 crate <0.1、README 要求生产使用前联系维护者，且 2026 上半年连续爆出 CVE-2026-76234 等一串胶水层公告，RSA 仅 PSS——高保证核心 ≠ 无 bug 产品。依据：https://github.com/celabshq/libcrux 、https://rustsec.org/advisories/RUSTSEC-2026-0023.html
9. **pqcrypto 家族 2026-06-04 集体死亡**（9 条 unmaintained、patched=[]，上游 PQClean 归档）——2026 年 PQC 标准实现的赢家是 RustCrypto 的 ml-kem/ml-dsa/slh-dsa 与 libcrux，选后端时要按"RustSec 存活名单"过滤。依据：https://rustsec.org/advisories/RUSTSEC-2026-0160.html
10. **一站式纯 Rust 竞品 orion 缺审计、RustCrypto 单 crate 缺统一供给层**：orion 0.18.0 活跃且算法面全（ML-KEM/ML-DSA/X-Wing/HPKE/Argon2）但自述"未经过任何第三方审计"，而 RustCrypto 是几十个 crate 的松散联盟——"以 RustCrypto traits 为合同、多后端可插拔（RustCrypto/graviola/libcrux/aws-lc-rs）的统一供给层"恰是生态空白，这正是 Groma 的定位机会。依据：https://github.com/orion-rs/orion 、https://github.com/RustCrypto/traits

---

### 附：2025-2026 关键安全公告速查（本报告引用）

| 公告 | 日期 | 对象 | 结论 |
|---|---|---|---|
| RUSTSEC-2025-0010 | 2025-03 | ring | <0.17 unmaintained，patched=[] |
| RUSTSEC-2025-0004 | 2025-02 | openssl crate | CVE-2025-24898 UAF，修复 0.10.70 |
| RUSTSEC-2025-0133 | 2025-12 | libcrux-intrinsics | aarch64 计算错误 |
| RUSTSEC-2025-0144 | 2025-12 | ml-dsa(RustCrypto) | CVE-2026-22705 时间侧信道，修复 0.1.0-rc.3 |
| RUSTSEC-2026-0023/24/25/26 | 2026-01~02 | libcrux-ecdh/psq/ed25519 | CVE-2026-76234 系列 |
| RUSTSEC-2026-0042~48 | 2026-03 | aws-lc-sys/-fips-sys | 含 CVE-2026-3337（AES-CCM 侧信道） |
| RUSTSEC-2026-0098/0104 | 2026-04 | rustls-webpki | 名称约束/CRL panic |
| RUSTSEC-2026-0160~68 | 2026-06 | pqcrypto×9 | unmaintained，patched=[] |
| RUSTSEC-2026-0209/0210 | 2026-06~07 | libcrux-aesgcm | AAD 限制缺失；更名 libcrux-aes |
| RUSTSEC-2026-0285 | 2026-09-14 | rustls | TLS 1.3 加密层边界缺陷，修复 0.23.45 |
