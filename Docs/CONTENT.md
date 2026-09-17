# Groma 开发内容规格（草稿）

> **文档类型**：content-spec（设计，待用户逐节拍板）
> **所属模块**：Groma
> **创建日期**：2026-09-16
> **状态**：**DRAFT** —— 本文件把生态位、合同设计决策、硬约束、调研缺口**合并为一份可执行的内容清单**；经用户确认后与 [`GOALS.md`](GOALS.md)/[`POSITIONING.md`](POSITIONING.md) 一同并入 [`SCOPE.md`](SCOPE.md)
> **权威顺序**：[`SCOPE.md`](SCOPE.md) > [`HANDOFF.md`](HANDOFF.md) > 本文（草稿）> 其他
> **依据**：`SCOPE.md` 分层与分期、`POSITIONING.md` 双锚点生态位（获客=稀缺件，留存=合同）、`DESIGN.md` D1–D7（已全部拍板）、`HANDOFF.md` §7 十二章硬约束、`CONSTRAINTS-REVIEW.md` 四线调研、`research/` 缺口清单

---

## 0 内容总则

1. **内容由双锚点推导**：获客件（无 C 稀缺件）＋留存件（统一合同）＋地基件（合同冻结与 R4 承诺）。凡不能归入这三类的，不做。
2. **准入三判据**（SCOPE §2.1）：有公开规范＋有公开测试向量＋有真实消费者，三条全过才进清单；算法清单受 D3=(a) 封闭枚举约束，加算法=改合同=SCOPE 治理流程。
3. **实现方式受十二章硬约束约束**：引用不重写算法、参考重写留痕、官方零 C、独立向量、恒定时间、依赖白名单、密钥生命周期。
4. 每个条目标注四要素：**状态**（已决/待拍板/deferred）· **层**（L1–L7）· **期**（P1–P6）· **来源**（复用引用/自研补缺/适配器）。
5. **纳入证据四格**（2026-09-16 用户确立）：每个候选条目须过——①Rust 现状（有/无、活/死、版本/维护/采用）②**死因**（若死：占名未实现／被吸收合并／低需求停摆／停维仍活／架构废弃）③**跨语言参照**（OpenSSL/Go/Java 有无成熟实现；"他有我无"是稀缺件的强定义）④三判据＋双锚点落点。**死因决定动作**：占名未实现且 C 有 → **自构补缺**（真稀缺件）；被吸收合并 → 指向后继不动作；低需求停摆 → deferred（判据③不过）；停维但需求仍活 → 依赖＋N2 标注（收养维护列预案）；架构废弃 → 指向后继不动作。
   **方法论警告（复核轮教训）**：`max_stable_version` 字段会掩盖活跃的 pre/rc 发布线——判定"死/停维"必须同时查：稳定版＋pre/rc 线＋上游仓库真实活动；crates.io 命名陷阱（占位 crate、同名不同义、未发布 workspace 成员）须逐一甄别。
6. **分层归类规则**（2026-09-16 内部分类审查定案）：①L3 分两个子面——**算法面**（后端实现的原子算法）与**构造面**（HPKE/FROST/AES-KW 等合同之上自研组合，消费合同而非实现合同）②"**结构/格式**"归 L4、"**验证/信任判断**"归 L5（TSA 结构→L4、验证→L5，crate 内分模块）③L6 广义化为"**平台与厂商证明**"（TPM/Android Key/Apple App Attest/PKCS#11/熵）；FIDO MDS3 归 L5（与 OCSP/CT 同质）④crate 粒度维持 22 项，按五簇组织呈现（地基/算法/获客/平台/适配）。
7. **对外叙事收敛规则**（2026-09-16 用户拍板）：对外沟通只以三样为主体——**获客件（验证级编排稀缺件）＋统一合同＋无 C 构建**；合同覆盖件（secp256k1/BLS/HPKE/FROST/AES-KW 等适配）**不得作为卖点**（开发者直接引用上游同样成立），官方适配器标"低优先级/社区可做"（合同开放，第三方写适配器同样成立）；官方精力集中在获客件与 rustls 适配器。
8. **竞争策略**（2026-09-16 用户拍板）：取代对象是 **aws-lc-rs**（rustls 底下的 C），不是 rustls（rustls 是分销渠道）；**aws-lc-rs 的 FIPS 段永远不碰**（R6），其余市场段错位竞争；取代路径三步见 `POSITIONING.md`。

---

## 1 交付物清单（crate 地图，共 15 个）

| crate/工具 | 层 | 期 | 性质 | 状态 |
|---|---|---|---|---|
| `groma-core` | L1 | P1 | 自研（合同层，项目存在的理由） | 已决 |
| `groma-codec` | L1 | P1 | 自研（有界 CBOR/COSE/DER） | 已决 |
| `groma-stub` | L3 | P1 | 自研（SHA-256 差分后端，非生产） | 已决 |
| `groma-rustcrypto` | L3 | P2 | 复用适配（RustCrypto 0.11 世代） | 已决 |
| `groma` | 伞 | P1–P2 | 自研（facade，可叠加装载 feature） | 已决 |
| `groma-registry` | 伞 | P1–P2 | 自研（普通对象注册表，非全局） | 已决 |
| `Tools/vector-fetch` | 工具 | P1 | 自研（D7=(d) 单清单内容寻址） | 已决 |
| `Tools/dependency-audit` | 工具 | P1 | 自研（links/cc/cmake 机器门禁） | 已决 |
| `groma-pkcs12` | L4 | **P2+（获客首发）** | 自研补缺（创建＋MAC 校验，纯 Rust 创建为零） | 已决 |
| `groma-libcrux` | L3 | P3 前端 | 复用适配（ML-KEM/ML-DSA/SLH-DSA，PQC 提前） | 已决 |
| `groma-graviola` | L3 | P3/P4 | 复用适配（高性能＋rustls 桥） | 已决 |
| `groma-cms` | L4 | P5 | 自研补缺（SignedData/EnvelopedData 验签） | 已决 |
| `groma-trust`（OCSP/路径/CRL/CT） | L5 | P5 | 自研补缺 | 已决 |
| `groma-tpm` | L6 | P6 | 自研补缺（TPM 2.0 结构与证明验证） | 已决＋路线图提前公告 |
| `groma-entropy` | L6 | P6 | 自研补缺（SP 800-90B 熵健康＋DRBG） | 已决 |
| `groma-adapter-*`（rustls/quinn/russh/boringtun/openpgp） | L7 | P4 | 自研薄适配器 | 已决 |
| `groma-hpke` | L3 | P2 | 自研构造（RFC 9180＝KEM/KDF/AEAD 组合；可适配 hpke 0.14 或自组于合同） | 已决（低优先级/社区可做） |
| `groma-frost` | L3 | P2/P3 | 复用适配（frost-* 3.0，RFC 9591 定稿） | 已决（低优先级/社区可做） |
| `groma-bls` | L3 | P2 | 复用适配（zkcrypto bls12_381；签名层按 CFRG draft 语义） | 已决（低优先级/社区可做） |
| `groma-keywrap` | L4 | P1/P2 | 复用适配（aes-kw 0.3.1；服务 PKCS#8/CMS/JWE 格式层） | 已决（低优先级/社区可做） |
| `groma-cose` | L4 | P2 | 自研编排（coset 结构＋Sig_structure 组装＋backend 验签） | 已决（加法轮） |
| `groma-attestation` | L6 | 中期 | 自研补缺（Android Key/Play Integrity＋Apple App Attest＋FIDO MDS3 验证与缓存） | 已决（加法轮） |
| `groma-tsa` | L4/L5 | 中期 | 自研编排（RFC 3161；x509-tsp/cms 积木＋验证） | 已决（加法轮） |

> **五簇呈现（分类审查定案）**：**地基簇**＝groma-core/codec/stub/rustcrypto/groma/groma-registry/Tools×2 · **算法簇**＝groma-libcrux/graviola/bls/frost/hpke/keywrap · **获客簇**＝groma-pkcs12/cms/trust/cose/tsa · **平台簇**＝groma-tpm/entropy/attestation · **适配簇**＝groma-adapter-*。
>
> **层位映射提案（待内部分类审查确认，2026-09-16）**：secp256k1→groma-rustcrypto 扩展（L3）；OID 注册表→复用 const-oid（L4 基础设施，无新 crate）；X-Wing→L3 契约预留（Kem 混合模式，实验后端）；LMS/XMSS→L3 契约预留（有状态签名家族，实现暂缓）。

---

## 2 算法覆盖清单（L3）

**P2（R4 承诺＋差分基线）**：SHA-256/384/512 · HMAC · HKDF · PBKDF2 · AES-128/256-GCM · ChaCha20-Poly1305 · Ed25519 · Argon2id —— 全部经 `groma-rustcrypto`；SHA-256 另经 `groma-stub` 差分。

**P3（算法面，PQC 前端）**：
- 哈希：SHA-3 全族、SHAKE、BLAKE2/3
- MAC：CMAC、Poly1305（引用）；**GMAC、KMAC（拼装件，P3 后段）**——基座成熟：GMAC=ghash 0.6.0（1.57 亿下载）＋AES 派生 H/J0；KMAC=cshake 0.2.1（sha3 0.12 已拆出）或 sha3-kmac 0.3.0 之上装配；向量充足（CAVP GCMVS 含 GMAC 组、Wycheproof aes_gmac 414 条含 324 负向、kmac 435 条＋SP 800-185 样例），oracle=openssl mac/dgst 差分。难度易-中，立即可做
- KDF：KBKDF（SP 800-108；依赖须用 0.1.0-rc.1——0.0.1 已与 digest rc.11 编译断裂，预发布生态锁定；或经 hmac 自构三模式）、scrypt
- AEAD：AES-CCM、AES-OCB3、XChaCha20-Poly1305
- 签名：ECDSA P-256/384/521、Ed448（**复核轮修正：非停维**——RustCrypto 收养，0.14.0-pre 线活跃至 2026-06；依赖＋跟踪 0.14 正式）、**RSA PKCS#1 v1.5/PSS（见裁决）**
- KEM/密钥交换：X25519、X448（**同 Ed448 修正**：RustCrypto 收养、0.14.0-pre 活跃）、ECDH、**ML-KEM-768（优先）/512/1024**
- 后量子签名：**ML-DSA-44/65/87**（经 `groma-libcrux`＋RustCrypto 双路）；**SLH-DSA 有条件纳入**（复核轮修正：非停滞——0.1.0 即 FIPS 205 定稿版、0.2.0-rc.5 活跃；跟踪 0.2.0 正式版，P3 后段、ML-DSA 之后；libcrux 无 SLH-DSA 计划）
- 口令哈希：scrypt、bcrypt

**加法轮新增（2026-09-16 用户全收；适配类标"低优先级/社区可做"——对外叙事不作为卖点）**：
- **secp256k1**（k256 0.14 适配：ECDSA＋BIP340 Schnorr）——Bitcoin/Cosmos 生态规模消费者，P2；OpenSSL 有曲线无 BIP340（半超对标面）；**低优先级/社区可做**
- **HPKE**（RFC 9180 **构造面**：适配 hpke 0.14 或于合同自组 KEM/KDF/AEAD 组合）——MLS/ECH/OHTTP 消费者，P2；OpenSSL 3.2+ 有（对标内）；**低优先级/社区可做**
- **AES-KW/KWP**（**构造面**：aes-kw 0.3.1；SP 800-38F＋RFC 3394/5649＋Wycheproof aes_kwp）——服务 PKCS#8/CMS/JWE（pkcs8 0.11 尚未接 aes-kw，Groma 补），P2；**低优先级/社区可做**
- **BLS12-381 签名**（**算法面**：zkcrypto bls12_381 0.9 适配；CFRG draft-07 语义，向量用 ETH2 consensus-spec-tests 补附录 TBA）——**超出 OpenSSL 对标面**（OpenSSL 至今无 BLS），Ethereum/Filecoin 消费者，P2；**低优先级/社区可做**
- **FROST 门限签名**（**构造面**：frost-* 3.0 适配；RFC 9591 定稿含向量）——**超出对标面**，MPC 钱包消费者，P2/P3；**低优先级/社区可做**

**补漏轮（2026-09-16 OpenSSL 对照发现，用户全收；均为引用适配）**：
- **SM2/SM3/SM4 国密**（RustCrypto sm2 0.13.3/sm3/sm4；GM/T 规范＋官方向量）——中国市场合规消费者；OpenSSL 对标面的正面缺口，P3
- **XTS**（RustCrypto xts-mode；IEEE 1619 向量）——全磁盘加密（dm-crypt/BitLocker/VeraCrypt），P3
- **FFDHE（RFC 7919）**（RustCrypto DH；RFC 7919 向量）——TLS DHE 套件消费者，P3
- **AES-SIV（RFC 5297）**（RustCrypto aes-siv；RFC 5297 向量）——密钥包装类确定性加密，P3

**契约预留（实现暂缓）**：
- **X-Wing 混合 KEM**：individual I-D 已于 2026-09 过期未转 RFC（三判据①不齐），但 BoringSSL/CIRCL/orion 已实现、码点 0x647A 在用——L3 只抽象"混合 KEM"契约，X-Wing 作实验后端，RFC 定稿转正
- **LMS/XMSS 有状态签名**：RustCrypto 官方仍在 rc/pre 线；OpenSSL 3.6 已有 LMS 验证（对标面在动）——L3 预留"有状态签名"契约（含状态管理 API），实现暂不承诺；CNSA 2.0 固件签名需求真实（oxicrypt 即为此而生）

**P6**：HMAC-DRBG、熵源健康（SP 800-90B 对接）。

**裁决项（已决 2026-09-16，经复核轮修订）**：
1. **RSA 与 Marvin**：**收，按操作分权**——公钥验签纳入（RS256/证书链真实消费者，L5 绕不开）；私钥操作（签名/解密）在 Marvin 未修前禁用。**替代现状（复核轮核实）**：RSA 私钥**签名**可用 graviola 0.2.0（PKCS#1/PSS，CRT＋固定窗口＋验后复验；"very new"、未审计、仅 x86_64/aarch64、无 RSA 加密）；sad-rsa（隐式拒绝 fork，未审计）为备选；私钥**解密**仍无纯 Rust 安全替代。**复查触发**：RUSTSEC patched 字段变化、rsa 0.10.0 正式版发布、或修复 PR #680/#702 合并时复核。
2. **MD5/SHA-1**：**均 deferred**（复核轮修正：此前"SHA-1 收为 legacy"的判断不成立——rustls 生态 2026 完全不接受 SHA-1，公共 PKI 已清除 SHA-1 交叉证书，MD5 零现代消费者；若未来出现真实消费者再按判据重开）。

**Deferred（三判据不过，SCOPE §6 已有）**：RC4、MD4、DES/3DES、IDEA、Blowfish、CAST、SEED、Camellia、KASUMI/CLEFIA/MISTY1/HIGHT/PRESENT/SNOW 3G、GOST 签名等。

---

## 3 格式／信任／平台清单（L4/L5/L6）

**L4 格式**：
- P1：PEM、DER（codec 有界解析子集）
- P3：PKCS#8、SPKI、PKCS#1、SEC1、CSR（**复核轮修正：PKCS#10 已由 x509-cert＋rcgen 覆盖，Groma 不重造**，装配 x509-cert 的 CertReq）
- **加法轮：COSE_Sign1 编排（`groma-cose`）**——coset 0.4.2 结构底座（23.2M 下载、Android vendor）；Groma 自研 Sig_structure 组装＋backend 验签映射（ES256/RS256/EdDSA）；RFC 9052/9053＋cose-wg Examples 官方向量；**OpenSSL 无 COSE**（超对标）；消费者 FIDO/passkey/SCITT，P2
- **加法轮：TSA 时间戳（`groma-tsa`）**——RFC 3161；x509-tsp/cms/sigstore-tsa 积木全纯 Rust；Groma 自研一站式客户端＋TimeStampToken 验证（**结构→L4、验证→L5，crate 内分模块**）；oracle=openssl ts；代码签名/PDF/eIDAS 消费者，中期
- **加法轮：OID 注册表**——复用 const-oid 0.10（4.6 亿下载），L4 基础设施无新 crate
- **P2+：PKCS#12 创建＋MAC 校验（获客首发）**——**形式已决（2026-09-16）：自研（缩范围 ~1.5–2k 行）**：照 RFC 7292 自写创建＋MAC 校验（现代算法路径：AES-256-CBC PBE＋HMAC-SHA256；legacy 解析复用 RustCrypto pkcs12）；**自带 OpenSSL/Botan 双向差分验证（获客件的信任核心）**。减法轮发现 p12-keystore 0.3.2 已覆盖创建，但踩 pre 线（pkcs12 0.2.0-pre＋cms 0.3.0-pre）、无差分、文档 42%——"现成不满足才自研"成立（稳定线铁律＋无差分＋无审计三点均不满足）；PKCS#12 同时是合同 Digest/Mac/Kdf 族的第一个真实消费者（合同练兵）。向量：OpenSSL ~40＋Botan ~50 个 .p12 样本（含 RFC 9579 PBMAC1 全套负向）＋openssl pkcs12 全参数双向差分——立即可做
- P5：CMS/PKCS#7——底座 cms 0.2.3 解析层完整；Groma 自建 verify()（300–500 行：eContent 摘要→signedAttrs→验签→x509-cert 链校验）；语料 OpenSSL smime-eml/cms-msg 负向＋openssl cms 双向差分——立即可做

**L5 信任**（P5）：
- **X.509 路径构建＋名称约束＋CRL**：底座 pkix-path 0.3.2（2026-06，no_std、纯 RustCrypto 依赖；排除 synta-x509-verification——默认拖 openssl crate 非纯 Rust）；Groma 做验证级编排封装
- **OCSP 验证级客户端**：底座 x509-ocsp 0.2.1 格式层完整；Groma 自建验签/nonce/CertID 匹配/时间窗口/签名者授权编排；OpenSSL ocsp-tests 10+ 类负向＋离线 round-trip 差分——立即可做
- **CT 验证子集**：SCT 验证（sct 0.7.1 底座）＋Merkle inclusion 证明自研（RFC 6962 §2.1.3 示例＋ct-go testdata）；日志客户端/监控/STH 全栈不做
- **加法轮：FIDO MDS3 验证与缓存**（fido-mds 0.6-dev 已纯 Rust 未发布；Groma 接 L1 provider；**归 L5——与 OCSP/CT 同质的元数据信任判断**）——消费者 WebAuthn RP，中期
- 厂商根信任对接

**L6 平台与厂商证明**（P6）：
- **TPM 2.0**：底座 tpm2-protocol 1.2.0（wire 编解码，作者为内核 TPM 维护者 Jarkko）；Groma 做 attestation 验证子集（quote 验签＋PCR 摘要，难度中）；完整客户端（会话 HMAC/命令级 API）不做——P 期再议；oracle=TCG 模拟器＋tpm2-tools
- PKCS#11 密码学缺口（ABI 用 cryptoki，不重造）
- **加法轮：厂商证明验证（`groma-attestation`）**——Android Key/Play Integrity 证明（octet-attest-verify 底座：嵌入 Google 根含 2026-02 P-384 新根；无在线吊销列已知限制）、Apple App Attest（CBOR＋链到 Apple 根＋nonce/counter）；消费者 WebAuthn RP（webauthn-rs 链验证至今走 OpenSSL）；中期（MDS3 已移 L5）
- **SP 800-90B 熵健康＋jitter 熵源**（复核轮确认：唯一完整真缺口——纯 Rust 只有 GPL 未验证脚手架；NIST 评估工具与 libjitterentropy 为 C；自研评估套件＋可验证熵源，P6）
- HMAC-DRBG

---

## 4 L7 适配器清单（P4）

| 适配器 | 目标 | 说明 |
|---|---|---|
| `groma-adapter-rustls` | rustls | 经 groma-graviola / groma-rustcrypto 后端桥接 CryptoProvider |
| `groma-adapter-quinn` | quinn | 经 rustls 适配器传导 |
| `groma-adapter-russh` | russh | russh 默认 aws-lc-rs，换 Groma 后端 |
| `groma-adapter-boringtun` | boringtun/WireGuard | ⚠️ boringtun 带 ring 残留（**复核轮修正：3 处**——handshake 静态公钥比对＋rate_limiter MAC1/MAC2 常量时间比较）；**缓发＋监控**：去 ring PR #479（改 subtle）已存在未合并，合并后即可收；上游新维护者活跃（0.7.1，2026-05） |
| `groma-adapter-openpgp` | rpgp（crate 名 `pgp`） | 已纯 Rust，接 Groma 后端 |

---

## 5 基础设施清单（贯穿全期）

- `Vectors/manifest.toml`＋`Tools/vector-fetch`（D7=(d) 单清单内容寻址；独立来源五要素：出处/版本/sha256/许可/用途）
- `Tools/dependency-audit`（无 C 机器门禁：links/cc/cmake/bindgen）
- Fuzz 目标（P1 三个：cose_key/cbor_bounded/der_signature；随解析入口增长）
- conformance 套件＋双后端差分 harness（P1）
- CI 门禁（fuzz/dudect/无 C/覆盖/禁词/依赖审计，P1 接入）
- 依赖白名单与未修公告裁决账（N2 章程）
- **上游监控清单**（复核轮确立）：boringtun PR #479（去 ring）、RSA #680/#702（Marvin 修复，触发=patched 变化或 0.10 正式版）、kbkdf 0.1.0、ed448-goldilocks/x448 0.14 正式、pkcs1 0.8、slh-dsa 0.2、sha3-kmac/cshake 线

---

## 6 阶段内容（草案——**顺序未定**）

> ⚠️ 用户指示（2026-09-16）：**实现顺序最后定**。本节记录各期候选内容，顺序仅供依赖分析参考。

| 期 | 内容 | 出口判据（机器可判定） |
|---|---|---|
| **P1 合同冻结** | groma-core（D1–D7 全部落位）＋groma-codec＋groma-stub＋向量/差分/CI 基础设施 | HANDOFF §6 四判据：无泄漏／第三方可增后端／双后端向量一致／解析不崩溃 |
| **P2 兑现与获客首发** | R4：Ed25519/Argon2/AEAD 全向量＋负向（玩家服务排期承诺）；OpenSSL 差分 harness；**PKCS#12（获客首发件）** | R4 三件套向量全过；PKCS#12 与 OpenSSL 差分一致（创建/解析/MAC） |
| **P3 算法面（PQC 前端）** | 哈希/MAC/KDF/AEAD/签名/KEM 全表＋**ML-KEM/ML-DSA 双路后端（libcrux＋RustCrypto）**；PKCS#8/SPKI/CSR | 每算法 KAT 过；PQC×无 C 组合可交付 |
| **P4 适配器** | rustls/quinn/russh 适配器（boringtun 缓发） | 各目标协议栈跑通自带测试套件（消费端验收） |
| **P5 信任与容器** | 顺序：路径构建→OCSP→CT→CMS→PKCS#12 完善 | 结论与 OpenSSL CLI 差分一致 |
| **P6 平台** | TPM 2.0 结构与证明、PKCS#11 缺口、熵健康/DRBG | 公开向量＋真实设备样本 |

**顺序调整说明（本次定稿的关键变更）**：
1. **PKCS#12 从 P5 前置到 P2+**——双锚点推论"第一件稀缺件落地=第一次获客"；PKCS#12 依赖链短（合同＋DER＋digest/mac/kdf 适配，P2 即具备），无需等 P3 算法面。这也是此前"第一件稀缺件"决策（PKCS#12 优于 PQC 首发）的落位。
2. **PQC 提前到 P3 前端**——窗口最尖锐（ml-kem 增速 70%、rustls 默认全绑 aws-lc-rs），向量最充足（FIPS 203/204/205 官方）。
3. **TPM 留在 P6 但提前公告路线图**——最痛缺口，但最重；纯 Rust 竞争者窗口尚在（tss-esapi 8.0 仍 alpha）。
4. P4 与 P5 的先后：P4（适配器）在 P3 算法面之后、P5 之前——适配器需要算法面就绪。

---

## 7 明确不做（内容级非目标）

| 不做 | 为什么 | 消费者去哪 |
|---|---|---|
| FIPS 140-3 验证 | R6；换语言不可继承；**FIPS 段永远不碰（2026-09-16 用户定）** | aws-lc-rs（有 C）/自接适配器 |
| openssl CLI 等价物 | **已决（2026-09-16）：不做**——产品化工具非库层 | — |
| JOSE/JWS | jsonwebtoken 已纯 Rust 化（trait 化后端） | jsonwebtoken |
| SSH 密钥格式 | ssh-key 成熟（1140 万下载） | ssh-key |
| OpenPGP 协议本体 | rpgp 已纯 Rust | rpgp＋Groma 适配器 |
| PKCS#11 全栈 | cryptoki 成熟（267 万下载） | cryptoki＋Groma 缺口补 |
| WireGuard 协议 | boringtun 主体纯 Rust | 适配器缓发 |
| 协议栈本体（TLS/QUIC/SSH/IPsec…） | 生态已有 | L7 适配器 |
| 长尾算法（§2 deferred 清单） | 三判据不过 | 无消费者，不做 |
| libcrypto API/ABI 兼容 | 与"冻结自己合同"矛盾 | — |

---

## 8 待拍板清单（用户）

- [x] §1 crate 地图 15 项：已确认（含 7 红灯复审）
- [x] §2 算法覆盖清单逐项：已确认（GMAC/KMAC 自构补缺、SLH-DSA deferred、Ed448/X448/pkcs1 N2 标注、KBKDF 备注）
- [x] §2 裁决项：已决（RSA 按操作分权；SHA-1 legacy opt-in、MD5 deferred）
- [x] §7 不做清单：已确认（含 CLI 不做）
- [x] **复核调研轮**：已完成（四线：跨语言差集／自构可行性／裁决项复核／依赖上游）——真缺口收窄（SP 800-90B、CT 客户端层）、SLH-DSA 有条件纳入、SHA-1/MD5 均 deferred、Ed448/X448/pkcs1 撤销停维、KMAC/GMAC 改拼装件、PKCS#10 改装配 x509-cert、X.509 底座 pkix-path、TPM 改 attestation 子集、RSA 增复查触发、boringtun 3 处 ring＋PR #479 监控
- [x] **加法轮**（用户指示"继续做加法"，两线调研 15 候选）：11 项全收（COSE_Sign1/Android/Apple/MDS3/TSA/secp256k1/HPKE/AES-KW/OID/BLS/FROST），4 项暂缓（X-Wing、LMS/XMSS 契约预留；CMP/EST/SCEP 中 SCEP 排除；Kerberos 远期）；定位扩张为"OpenSSL 对标面＋现代生态超集"（GOALS §1 已记）
- [x] **内部分类审查**：已定案——L3 分算法面/构造面、结构→L4 验证→L5、MDS3 归 L5、L6 广义化"平台与厂商证明"、五簇呈现、契约预留补记 DESIGN.md（D8 混合 KEM/D9 有状态签名）
- [x] **对外叙事收敛**（2026-09-16 用户拍板）：获客件＋合同＋无 C 为主体；BLS/FROST/secp256k1/HPKE/AES-KW 官方适配器标"低优先级/社区可做"；官方精力集中在获客件与 rustls 适配器
- [x] **竞争策略**：取代对象=aws-lc-rs（非 rustls）；FIPS 段永远不碰；其余错位竞争（对照表与取代路径三步见 POSITIONING）
- [x] **补漏轮**（OpenSSL 对照发现 4 项）：SM2/SM3/SM4、XTS、FFDHE、AES-SIV 全收（引用适配，P3）
- [ ] §6 实现顺序：**最后定**（用户指示）
- [ ] 定稿后并入 SCOPE 的时机（建议 P1 出口时）

---

## 修订历史

| 日期 | 变更 |
|------|------|
| 2026-09-16 | 创建草稿：合并生态位/合同决策/硬约束/调研缺口为内容清单——15 项交付物、算法覆盖分 P2/P3/P6、格式信任平台清单、适配器清单、阶段顺序定稿提案（PKCS#12 前置、PQC 提前）、不做清单、待拍板项。 |
| 2026-09-16 | 45 crate 逐个核查（crates.io API）：7 项红灯复审；按"纳入证据四格"（Rust 现状/死因/跨语言参照/三判据落点）重裁：GMAC/KMAC 自构补缺、SLH-DSA deferred、Ed448/X448/pkcs1 N2 标注；裁决项定案（RSA 按操作分权、SHA-1 legacy/MD5 deferred、CLI 不做）；§6 顺序改为"最后定"（用户指示）。 |
| 2026-09-16 | 复核调研轮（四线）修正：真缺口收窄（SP 800-90B 评估、CT 客户端层）；"算法缺失"稀缺性改为"验证级编排缺失"；SLH-DSA 有条件纳入（0.2.0-rc 活跃、0.1.0 即 FIPS 205 定稿）；SHA-1/MD5 均 deferred（rustls 栈与公共 PKI 已不需要 SHA-1）；Ed448/X448/pkcs1 撤销停维（RustCrypto 收养/活跃 RC）；KMAC/GMAC 改拼装件（cshake/sha3-kmac/ghash 基座）；PKCS#10 改装配 x509-cert；X.509 底座定 pkix-path（排除 synta 的 openssl 依赖）；TPM 改 attestation 验证子集；RSA 增复查触发（#680/#702）；boringtun 修正为 3 处 ring＋PR #479 监控；kbkdf 用 0.1.0-rc.1；上游监控清单建立。 |
| 2026-09-16 | 加法轮（两线 15 候选四格核查）：11 项全收（COSE_Sign1 编排、Android/Play Integrity、Apple App Attest、FIDO MDS3、TSA、secp256k1、HPKE、AES-KW、OID、BLS12-381、FROST），4 项暂缓/契约预留（X-Wing 规范过期、LMS/XMSS rc 线、CMP/EST/SCEP、Kerberos）；定位扩张"OpenSSL 对标面＋现代生态超集"；crate 地图增至 22 项。 |
| 2026-09-16 | 内部分类审查定案：分层归类规则入 §0（L3 算法面/构造面、结构→L4 验证→L5、L6=平台与厂商证明、五簇呈现）；MDS3 移 L5；TSA 结构/验证分层；契约预留补记 DESIGN.md D8/D9。 |
