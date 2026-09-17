# Groma 开发内容规格（草稿）

> **文档类型**：content-spec（设计，待用户逐节拍板）
> **所属模块**：Groma
> **创建日期**：2026-09-16
> **状态**：**DRAFT** —— 本文件把生态位、合同设计决策、硬约束、调研缺口**合并为一份可执行的内容清单**；经用户确认后与 [`GOALS.md`](GOALS.md)/[`POSITIONING.md`](POSITIONING.md) 一同并入 [`SCOPE.md`](SCOPE.md)
> **权威顺序**：[`SCOPE.md`](SCOPE.md) > [`HANDOFF.md`](HANDOFF.md) > 本文（草稿）> 其他
> **依据**：`SCOPE.md` 分层与分期、`POSITIONING.md` 双锚点生态位（获客=稀缺件，留存=合同）、`DESIGN.md` D1–D9、`HANDOFF.md` §7 十二章硬约束、`CONSTRAINTS-REVIEW.md`、`research/` 缺口清单、四轮调研（含加法轮与减法轮）

---

## 0 内容总则

1. **内容由双锚点推导**：获客件（验证级编排稀缺件）＋留存件（统一合同）＋地基件（合同冻结与 R4 承诺）。凡不能归入这三类的，不做。
2. **准入三判据**（SCOPE §2.1）：有公开规范＋有公开测试向量＋有真实消费者，三条全过才进清单；算法清单受 D3=(a) 封闭枚举约束，加算法=改合同=SCOPE 治理流程。
3. **实现方式受十二章硬约束约束**：引用不重写算法、参考重写留痕、官方零 C、独立向量、恒定时间、依赖白名单、密钥生命周期。
4. 每个条目标注四要素：**状态**（已决/待拍板/deferred）· **层**（L1–L7）· **期**（P1–P6）· **来源**（复用引用/自研补缺/适配器）。
5. **纳入证据四格**：①Rust 现状 ②**死因**（占名未实现／被吸收合并／低需求停摆／停维仍活／架构废弃）③**跨语言参照**（OpenSSL/Go/Java；"他有我无"是稀缺件强定义）④三判据＋双锚点落点。**死因决定动作**；`max_stable_version` 掩盖 pre/rc 线的教训与命名陷阱警告见 §5。
6. **分层归类规则**：①L3 分**算法面**（后端实现的原子算法）与**构造面**（合同之上自研组合）②"结构/格式"归 L4、"验证/信任判断"归 L5 ③L6＝"平台与厂商证明" ④crate 按五簇呈现。
7. **对外叙事收敛**：对外只讲三样——获客件（验证级编排稀缺件）＋统一合同＋无 C 构建；合同覆盖件不得作为卖点。
8. **竞争策略**：取代对象=aws-lc-rs（rustls 是分销渠道）；FIPS 段永远不碰；取代路径三步见 `POSITIONING.md` §5.1。
9. **减法原则（2026-09-16 减法轮定案）**：现成满足（稳定线＋有验证/审计＋形态相合）就引用；自研只在"现成不满足"时成立——三条触发：①稳定线铁律（上游锁 rc/pre）②信任缺口（无差分/无审计/无向量）③形态冲突（协议构造≠算法族）。"官方近零接入"：wrapper ≤~100 行的引用项官方写（成本≈0、保证合同面完整），但对外叙事不卖。

---

## 1 交付物清单（crate 地图，减法后 15 项）

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
| `groma-pkcs12` | L4 | **P2+（获客首发）** | **自研缩范围 ~1.5–2k 行**（创建＋MAC 校验＋OpenSSL/Botan 双向差分；legacy 解析复用 RustCrypto pkcs12） | 已决（形式 (a)） |
| `groma-cms` | L4/L5 | P5 | **自研 ~500 行 verify 编排**（抄 sigstore-tsa 纯 Rust 模式；现成替代带 ring 违反零 C） | 已决 |
| `groma-trust` | L5 | P5 | 编排薄层：路径（pkix-path 封装）＋OCSP（~300 行 HTTP 编排，验签用 pkix-revocation）＋CT（sct＋ct-merkle 拼装）＋MDS3 缓存策略（fido-mds） | 已决 |
| `groma-attestation` | L5/L6 | 中期 | 编排薄层：Android/Apple（octet-attest-verify 现成）＋COSE Sign1 胶水（coset 内置验签）＋Google status-list 吊销订阅 | 已决 |
| `groma-tpm` | L6 | P6 | **自研收窄 1–2k 行**（quote 验签＋PCR 摘要＋AK 链；不重造 eventlog） | 已决 |
| `groma-entropy` | L6 | P6 | **自研小项 300–500 行**（运行时 AP/RC 健康测试；不做实验室级评估） | 已决 |
| `groma-graviola` | L3 | **P4**（与 rustls 适配器同批） | 复用适配（高性能＋rustls 桥） | 已决（延后） |
| `groma-adapter-*`（rustls/quinn/russh；boringtun 缓发、openpgp 接 rpgp） | L7 | P4 | 自研薄适配器 | 已决 |

> **五簇呈现**：**地基簇**＝core/codec/stub/rustcrypto/facade/registry/Tools×2 · **获客簇**＝pkcs12/cms/trust/attestation/tpm/entropy · **适配簇**＝graviola(P4)/adapter-*。
>
> **减法轮删除（7 项，2026-09-16 用户批量确认）**：`groma-hpke`（上游开箱即用纯重复）· `groma-frost`（多轮状态机与单实例 Signer 族形态冲突）· `groma-bls`（draft 向量 TBA，判据②不过）· `groma-tsa`（sigstore-tsa 0.11 一站式）· `groma-cose`（coset 0.4.2 内置验签）· `groma-keywrap`（aes-kw 成熟，cms 直接依赖）· `groma-libcrux`（0.0.x＋平行栈；RustCrypto 单路已稳）。
>
> **契约预留**：X-Wing→L3 混合 KEM 契约（D8）；LMS/XMSS→L3 有状态签名契约（D9）。

---

## 2 算法覆盖清单（L3）

**P2（R4 承诺＋差分基线）**：SHA-256/384/512 · HMAC · HKDF · PBKDF2 · AES-128/256-GCM · ChaCha20-Poly1305 · Ed25519 · Argon2id —— 全部经 `groma-rustcrypto`；SHA-256 另经 `groma-stub` 差分。

**P3 算法面（减法后）**：
- 哈希：SHA-3 全族、SHAKE、BLAKE2/3
- MAC：CMAC、Poly1305；**GMAC（官方做，ghash 之上 ~5 行拼装）**、**KMAC（官方做，cshake 之上 ~50 行拼装＋KAT）**
- KDF/口令哈希：scrypt、bcrypt（KBKDF **砍**——HKDF 已覆盖，rc 转正再收）
- AEAD：AES-CCM、XChaCha20-Poly1305、XTS、AES-SIV；AES-OCB3 **降级**（可选 feature，零需求）
- 签名：ECDSA P-256/384/521（P-521 差异化）；Ed448/X448/RSA 验签/SM2 **延后 P4**（等 pre/rc 转正）；secp256k1 **官方近零接入**
- KEM：X25519、ECDH、**ML-KEM 三参数（RustCrypto 单路）**
- 国密：SM3/SM4（官方近零接入）
- 后量子签名：ML-DSA-44/65/87（RustCrypto 单路）；SLH-DSA **deferred**（触发条件：0.2.0 稳定＋需求信号）
- 构造面：AES-KW/KWP（**官方近零接入**，服务 CMS/PKCS#8；groma-cms 直接依赖 aes-kw）

**砍（减法轮）**：KBKDF（P3；HKDF 覆盖）、FFDHE（RustCrypto dh crate 已不存在，自研模幂不值——TLS 弃用趋势）。

**契约预留（实现暂缓）**：X-Wing 混合 KEM（individual I-D 2026-09 过期；BoringSSL/orion 已实现，码点 0x647A 在用；RFC 定稿转正）· LMS/XMSS 有状态签名（RustCrypto rc/pre 线；OpenSSL 3.6 已有 LMS 验证；CNSA 2.0 固件签名需求真实）。

**裁决项（已决 2026-09-16，经复核轮修订）**：
1. **RSA 与 Marvin**：收，按操作分权——公钥验签纳入（RS256/证书链真实消费者）；私钥操作在 Marvin 未修前禁用；替代现状：graviola 0.2.0（签名，未审计，仅 x86_64/aarch64，无 RSA 加密）、sad-rsa（隐式拒绝，未审计）；私钥解密仍无纯 Rust 安全替代。**复查触发**：RUSTSEC patched 变化、rsa 0.10.0 正式版、或 #680/#702 合并。**期次：延后 P4**（等 0.10 转正）。
2. **MD5/SHA-1**：均 deferred（rustls 生态与公共 PKI 2026 已不需要）。

**Deferred**：RC4、MD4、DES/3DES、IDEA、Blowfish、CAST、SEED、Camellia、KASUMI/CLEFIA/MISTY1/HIGHT/PRESENT/SNOW 3G、GOST 签名等。

---

## 3 格式／信任／平台清单（L4/L5/L6）

**L4 格式**：
- P1：PEM、DER（codec 有界解析子集）
- P3：PKCS#8、SPKI、PKCS#1、SEC1、CSR（装配 x509-cert 的 CertReq；PKCS#10 已由其覆盖，不重造）
- **P2+：PKCS#12 创建＋MAC 校验（获客首发，自研缩范围＋双向差分，见 §1）**
- P5：CMS/PKCS#7 验签编排（自研 ~500 行；底座 cms 0.2.3 解析层；语料 OpenSSL smime-eml/cms-msg 负向＋openssl cms 双向差分）
- OID 注册表：复用 const-oid（L4 基础设施）

**L5 信任**（P5，编排薄层）：
- X.509 路径构建＋名称约束＋CRL：底座 pkix-path 0.3.2（no_std、纯 RustCrypto 依赖；synta 排除——拖 openssl）
- OCSP 验证级客户端：验签用 pkix-revocation（OcspChecker 现成）；Groma 自研 ~300 行 HTTP 编排（nonce/POST/缓存）
- CT 验证子集：sct 0.7.1（SCT 验签）＋ct-merkle 0.3（inclusion/consistency）拼装；自研编排薄层；日志客户端不做
- FIDO MDS3：fido-mds（kanidm 验证已完整）；Groma 只做缓存/刷新/降级策略层
- 厂商根信任对接

**L6 平台与厂商证明**：
- TPM 2.0 attestation 子集（自研收窄 1–2k 行：quote 验签＋PCR 摘要＋AK 链；参照 go-attestation 路径与 trustee 300 行实现；不重造 eventlog；完整客户端不做）
- PKCS#11 密码学缺口（ABI 用 cryptoki，不重造）
- 厂商证明编排（`groma-attestation`）：Android Key/Play Integrity（octet-attest-verify 现成——纯 RustCrypto 栈无 C、离线链验证完整）＋Apple App Attest（同底座）＋COSE Sign1 胶水（coset 内置验签，~50 行）＋Google status-list 吊销订阅（自研小块）
- SP 800-90B 熵健康（自研小项 300–500 行：运行时 AP/RC 健康测试，参考 NIST 公式与内核实现；实验室级 ea_non_iid 不做）
- HMAC-DRBG

---

## 4 L7 适配器清单（P4）

| 适配器 | 目标 | 说明 |
|---|---|---|
| `groma-adapter-rustls` | rustls | 经 groma-graviola / groma-rustcrypto 后端桥接 CryptoProvider（**graviola 与适配器同批 P4**） |
| `groma-adapter-quinn` | quinn | 经 rustls 适配器传导 |
| `groma-adapter-russh` | russh | russh 默认 aws-lc-rs，换 Groma 后端 |
| `groma-adapter-boringtun` | boringtun/WireGuard | ⚠️ 3 处 ring 残留；**缓发＋监控** PR #479（去 ring）；上游活跃（0.7.1） |
| `groma-adapter-openpgp` | rpgp（crate 名 `pgp`） | 已纯 Rust，接 Groma 后端 |

---

## 5 基础设施清单（贯穿全期）

- `Vectors/manifest.toml`＋`Tools/vector-fetch`（D7=(d) 单清单内容寻址；独立来源五要素）
- `Tools/dependency-audit`（无 C 机器门禁）
- Fuzz 目标（P1 三个：cose_key/cbor_bounded/der_signature；随解析入口增长）
- conformance 套件＋双后端差分 harness（P1）
- CI 门禁（fuzz/dudect/无 C/覆盖/禁词/依赖审计）
- 依赖白名单与未修公告裁决账（N2）
- **上游监控清单**：boringtun PR #479、RSA #680/#702（触发复查）、kbkdf 0.1.0、ed448 0.5 观察、x448/rsa/sm2 0.14/0.10 转正、slh-dsa 0.2、sha3-kmac/cshake 线
- **稳定线铁律**（减法轮定案）：只接已在新稳定线上的 crate；rc/pre 一律延后，不例外（当前 rc/pre 清单：kbkdf 0.1.0-rc.1、rsa 0.10.0-rc.18、sm2 0.14.0-rc.15、x448 0.14.0-pre.12、ed448-goldilocks 0.14.0-pre.15、slh-dsa 0.2.0-rc.5）

---

## 6 阶段内容（草案——**顺序未定**）

> ⚠️ 用户指示（2026-09-16）：**实现顺序最后定**。本节记录各期候选内容，顺序仅供依赖分析参考。

| 期 | 内容 | 出口判据（机器可判定） |
|---|---|---|
| **P1 合同冻结** | groma-core（D1–D9 全部落位）＋groma-codec＋groma-stub＋向量/差分/CI 基础设施 | HANDOFF §6 四判据：无泄漏／第三方可增后端／双后端向量一致／解析不崩溃 |
| **P2 兑现与获客首发** | R4：Ed25519/Argon2/AEAD 全向量＋负向；OpenSSL 差分 harness；**PKCS#12（自研＋差分）** | R4 三件套向量全过；PKCS#12 与 OpenSSL 差分一致 |
| **P3 算法面（PQC 前端）** | §2 保留表＋ML-KEM/ML-DSA 单路（RustCrypto）＋GMAC/KMAC 拼装；PKCS#8/SPKI/CSR | 每算法 KAT 过；PQC×无 C 组合可交付 |
| **P4 适配器** | rustls/quinn/russh 适配器＋graviola 后端（boringtun 缓发）＋Ed448/X448/RSA/SM2（转正后） | 各协议栈跑通自带测试套件 |
| **P5 信任与容器** | groma-trust（路径/OCSP/CT/MDS3 编排）＋groma-cms＋attestation 编排 | 与 OpenSSL CLI 差分一致 |
| **P6 平台** | groma-tpm（收窄）＋groma-entropy（小项）＋PKCS#11 缺口 | 公开向量＋真实设备样本 |

---

## 7 明确不做（内容级非目标）

| 不做/改用现成 | 为什么 | 消费者去哪 |
|---|---|---|
| FIPS 140-3 验证 | R6；**FIPS 段永远不碰（用户定）** | aws-lc-rs（有 C）/自接适配器 |
| openssl CLI 等价物 | 产品化工具非库层（已决） | — |
| TSA 一站式 | sigstore-tsa 0.11 已一站式（减法轮删除自研） | sigstore-tsa |
| COSE_Sign1 独立 crate | coset 0.4.2 已内置验签（减法轮删除） | coset＋attestation 胶水 |
| HPKE | hpke 0.14 开箱即用＋套件选择（减法轮删除） | hpke / hpke-dispatch |
| FROST 门限签名 | 形态冲突；frost-* 已成熟（减法轮删除） | frost-* |
| BLS12-381 | draft 向量 TBA（判据②不过）（减法轮删除） | zkcrypto bls12_381 / blsful |
| FFDHE | RustCrypto dh crate 已不存在（减法轮砍） | rustls ffdhe_groups / aws-lc |
| JOSE/JWS | jsonwebtoken 已纯 Rust 化 | jsonwebtoken |
| SSH 密钥格式 | ssh-key 成熟 | ssh-key |
| OpenPGP 协议本体 | rpgp 已纯 Rust | rpgp＋Groma 适配器 |
| PKCS#11 全栈 | cryptoki 成熟 | cryptoki＋Groma 缺口补 |
| WireGuard 协议 | boringtun 主体纯 Rust | 适配器缓发 |
| 协议栈本体（TLS/QUIC/SSH/IPsec…） | 生态已有 | L7 适配器 |
| 长尾算法 | 三判据不过 | 无消费者 |
| libcrypto API/ABI 兼容 | 与"冻结自己合同"矛盾 | — |

---

## 8 待拍板清单（用户）

- [x] 全部内容决策（加法轮/补漏轮/复核轮/分类审查/叙事收敛/竞争策略）
- [x] **减法轮**（四线调研，用户批量确认）：删 7 crate（hpke/frost/bls/tsa/cose/keywrap/libcrux）、延后 P4（Ed448/X448/RSA/SM2/graviola）、砍（KBKDF/FFDHE）、降级（OCB3）、官方近零接入（secp256k1/AES-KW/XTS/AES-SIV/SM3/SM4）、改用现成（Android/Apple→octet、MDS3→fido-mds、CT→sct+ct-merkle、OCSP→pkix-revocation）、自研缩小（PKCS#12 1.5–2k 差分、CMS 500 行、OCSP 300 行、TPM 1–2k、熵 300–500）
- [x] **PKCS#12 首发件形式**：已决 (a) 自研缩范围＋双向差分验证
- [ ] §6 实现顺序：**最后定**（用户指示）
- [ ] 定稿后并入 SCOPE 的时机（建议 P1 出口时）

---

## 修订历史

| 日期 | 变更 |
|------|------|
| 2026-09-16 | 创建草稿：合并生态位/合同决策/硬约束/调研缺口为内容清单（15 项交付物、算法覆盖、格式信任平台清单、适配器清单、不做清单）。 |
| 2026-09-16 | 45 crate 逐个核查＋复核调研轮（四线）＋加法轮（11 收 4 暂缓）＋补漏轮（SM2/3/4、XTS、FFDHE、AES-SIV）＋内部分类审查（分层归类规则、五簇、D8/D9）＋叙事收敛＋竞争策略（取代 aws-lc-rs）。 |
| 2026-09-16 | **减法轮**（四线调研，用户批量确认）：crate 地图 22→15；删 7（hpke/frost/bls/tsa/cose/keywrap/libcrux）；延后 P4（Ed448/X448/RSA 验签/SM2/graviola）；砍 KBKDF/FFDHE；降级 OCB3；官方近零接入 5 项；自研总量收敛至 ~5k 行量级（PKCS#12 1.5–2k＋CMS 500＋OCSP 300＋TPM 1–2k＋熵 300–500）；稳定线铁律入 §5；PKCS#12 形式定案 (a) 自研＋差分。 |
