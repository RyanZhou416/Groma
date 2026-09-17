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

---

## 2 算法覆盖清单（L3）

**P2（R4 承诺＋差分基线）**：SHA-256/384/512 · HMAC · HKDF · PBKDF2 · AES-128/256-GCM · ChaCha20-Poly1305 · Ed25519 · Argon2id —— 全部经 `groma-rustcrypto`；SHA-256 另经 `groma-stub` 差分。

**P3（算法面，PQC 前端）**：
- 哈希：SHA-3 全族、SHAKE、BLAKE2/3
- MAC：CMAC、Poly1305（引用）；**GMAC、KMAC（自构补缺，P3 后段）**——两者 crates.io 均为占名未实现（0.0.0），OpenSSL 有对应实现；GMAC 经成熟的 `ghash` crate 装配，KMAC 为 cSHAKE 之上的薄层
- KDF：KBKDF（SP 800-108；上游仅 0.0.1，必要时经 hmac 自构）、scrypt
- AEAD：AES-CCM、AES-OCB3、XChaCha20-Poly1305
- 签名：ECDSA P-256/384/521、Ed448（**上游停维 3.5 年，依赖＋N2 标注，收养预案**）、**RSA PKCS#1 v1.5/PSS（见裁决）**
- KEM/密钥交换：X25519、X448（**上游停维 6 年，同 Ed448 处理**）、ECDH、**ML-KEM-768（优先）/512/1024**
- 后量子签名：**ML-DSA-44/65/87**（经 `groma-libcrux`＋RustCrypto 双路）；**SLH-DSA deferred**——Rust 侧停滞两年、C 侧仅经 oqs-provider 冷供给、三判据③消费者不足，跟随生态成熟度
- 口令哈希：scrypt、bcrypt

**P6**：HMAC-DRBG、熵源健康（SP 800-90B 对接）。

**裁决项（已决 2026-09-16，复核调研轮可修正）**：
1. **RSA 与 Marvin**：**收，按操作分权**——公钥验签纳入（RS256/证书链真实消费者，L5 绕不开）；私钥操作（签名/解密）在 Marvin 未修前禁用，P3/P4 由 graviola 的 RSA 签名替代。
2. **MD5/SHA-1**：**SHA-1 收为 legacy**（仅验签/兼容上下文，opt-in，永不默认）；**MD5 deferred**（无现代消费者）。

**Deferred（三判据不过，SCOPE §6 已有）**：RC4、MD4、DES/3DES、IDEA、Blowfish、CAST、SEED、Camellia、KASUMI/CLEFIA/MISTY1/HIGHT/PRESENT/SNOW 3G、GOST 签名等。

---

## 3 格式／信任／平台清单（L4/L5/L6）

**L4 格式**：
- P1：PEM、DER（codec 有界解析子集）
- P3：PKCS#8、SPKI、PKCS#1、SEC1、CSR、PKCS#10
- **P2+：PKCS#12 创建＋MAC 校验（获客首发，理由见 §6）**
- P5：CMS/PKCS#7（SignedData 验签＋EnvelopedData）

**L5 信任**（P5，顺序见 §6）：X.509 路径构建＋名称约束＋CRL → OCSP 验证级客户端 → CT 日志校验 → 厂商根信任对接。

**L6 平台**（P6）：TPM 2.0 结构编解码＋证明验证（PS256）· PKCS#11 密码学缺口（ABI 用 cryptoki，不重造）· SP 800-90B 熵健康＋可用 jitter 熵源 · HMAC-DRBG。

---

## 4 L7 适配器清单（P4）

| 适配器 | 目标 | 说明 |
|---|---|---|
| `groma-adapter-rustls` | rustls | 经 groma-graviola / groma-rustcrypto 后端桥接 CryptoProvider |
| `groma-adapter-quinn` | quinn | 经 rustls 适配器传导 |
| `groma-adapter-russh` | russh | russh 默认 aws-lc-rs，换 Groma 后端 |
| `groma-adapter-boringtun` | boringtun/WireGuard | ⚠️ boringtun 带 ring 残留（一处常量时间比较）；**缓发**：等上游去 ring 或 fork 去 C（官方零 C 约束 8） |
| `groma-adapter-openpgp` | rpgp（crate 名 `pgp`） | 已纯 Rust，接 Groma 后端 |

---

## 5 基础设施清单（贯穿全期）

- `Vectors/manifest.toml`＋`Tools/vector-fetch`（D7=(d) 单清单内容寻址；独立来源五要素：出处/版本/sha256/许可/用途）
- `Tools/dependency-audit`（无 C 机器门禁：links/cc/cmake/bindgen）
- Fuzz 目标（P1 三个：cose_key/cbor_bounded/der_signature；随解析入口增长）
- conformance 套件＋双后端差分 harness（P1）
- CI 门禁（fuzz/dudect/无 C/覆盖/禁词/依赖审计，P1 接入）
- 依赖白名单与未修公告裁决账（N2 章程）

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
| FIPS 140-3 验证 | R6；换语言不可继承 | aws-lc-rs（有 C）/自接适配器 |
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
- [ ] **复核调研轮**（用户指示，进行中）：①跨语言差集全量复核 ②自构补缺可行性（KMAC/GMAC 基础组件与向量现状）③裁决项复核（Marvin 最新状态、boringtun 去 ring 进度、Ed448/X448 后继/替代）④依赖风险项上游活动
- [ ] §6 实现顺序：**最后定**（用户指示）
- [ ] 定稿后并入 SCOPE 的时机（建议 P1 出口时）

---

## 修订历史

| 日期 | 变更 |
|------|------|
| 2026-09-16 | 创建草稿：合并生态位/合同决策/硬约束/调研缺口为内容清单——15 项交付物、算法覆盖分 P2/P3/P6、格式信任平台清单、适配器清单、阶段顺序定稿提案（PKCS#12 前置、PQC 提前）、不做清单、待拍板项。 |
| 2026-09-16 | 45 crate 逐个核查（crates.io API）：7 项红灯复审；按"纳入证据四格"（Rust 现状/死因/跨语言参照/三判据落点）重裁：GMAC/KMAC 自构补缺、SLH-DSA deferred、Ed448/X448/pkcs1 N2 标注；裁决项定案（RSA 按操作分权、SHA-1 legacy/MD5 deferred、CLI 不做）；§6 顺序改为"最后定"（用户指示）。 |
