# Groma 目标声明（草稿）

> **文档类型**：goals（草稿，待用户拍板）
> **所属模块**：Groma
> **创建日期**：2026-09-16
> **状态**：**DRAFT** —— 本文件是提案，不是裁决。定稿后并入 [`SCOPE.md`](SCOPE.md) §0/§2，在此之前一切以 `SCOPE.md` 为准
> **权威顺序**：[`SCOPE.md`](SCOPE.md)（权威需求）> [`HANDOFF.md`](HANDOFF.md)（执行细化）> 本文（草稿）> 其他文档
> **来源**：2026-09-16 用户提出"希望 Groma 做到 OpenSSL 的水平"，并明确两点：① 功能覆盖度对标；② 有现成纯 Rust 实现可以引用
> **内容清单以 [`CONTENT.md`](CONTENT.md) 为准**（本文 §3 为其摘要；2026-09-16 加法轮/减法轮后的最终内容见 CONTENT.md）

---

## 0 一句话目标

**Groma 在纯 Rust 生态中提供 OpenSSL 级的密码学供给**：功能覆盖度对标 OpenSSL 中被现代消费者实际使用的算法、密钥与证书格式、信任路径与平台原语，并按三条准入判据追加现代生态超集（无 C 的 PQC、BIP340/国密等）；有现成纯 Rust 实现的直接引用，Groma 自研只做**合同层、验证级编排与差分验证、有界补缺**；以冻结的 provider 合同为架构基石；**取代对象是 aws-lc-rs（rustls 底下的 C），FIPS 段永远不碰**；P1–P6 分期交付，每期机器可判定出口。

---

## 1 语义澄清：什么叫做"对标 OpenSSL"

"对标"分四层，本项目只取第一层：

| 对标层级 | 含义 | Groma 态度 |
|---|---|---|
| **功能覆盖度** | OpenSSL 覆盖的算法/格式/信任/平台面，Groma 也有（供上同样能力） | ✅ **这是目标** |
| 质量与可替换性 | 向量一致、差分对抗、fuzz 干净、后端可换 | ✅ 这是目标 |
| 兼容性 | libcrypto API/ABI、openssl CLI、配置语法、ENGINE 插件 | ❌ 非目标 |
| 合规背书 | FIPS 140-3 验证模块 | ❌ 非目标（R6，**永远不碰**） |

### OpenSSL 为什么庞大（拆解）

OpenSSL 的庞大不是"算法多"，而是四样东西堆在一起：

| 成分 | 占比重（估） | 性质 | Groma 策略 |
|---|---|---|---|
| 算法与密码学原语 | ~20% | 哈希/MAC/AEAD/签名/KEM/口令哈希/随机 | **引用** RustCrypto（P3 单路）/graviola（P4），包装为后端 |
| 密钥与证书格式 | ~25% | PKCS#8/SPKI/PKCS#12/CMS/X.509/CSR/OCSP/CRL/CT | 结构复用现成 crate，Groma 自研**验证级编排＋差分验证** |
| 协议栈与粘合 | ~30% | TLS/QUIC/SSH 本体、libcrypto 上千 API、CLI、配置、ENGINE | 协议本体用 L7 适配器接现有纯 Rust 实现；API/CLI/ENGINE **不做** |
| 历史包袱与合规 | ~25% | RC4/MD4/DES 家族等遗留算法、20 年兼容行为、FIPS 模块 | 遗留算法按三条判据裁决，多数 deferred；FIPS 不做 |

> 关键洞察：OpenSSL 的护城河约有一半是它的负担。Groma 只对标"被现代消费者实际使用"的那 ~45% 供给面，并用"引用生态＋验证级编排"而不是"重写生态"的方式达到覆盖度。

> **2026-09-16 定位扩张（用户拍板）**：在对标面之上按三条判据追加**现代生态超集**——无 C 的 PQC（ML-KEM/ML-DSA）、BIP340/国密等 OpenSSL 没有的算法面，以及 OpenSSL 没有的验证级编排（厂商证明/TPM 证明/CT 客户端层）。定位表述为"**OpenSSL 对标面＋现代生态超集**"。注：BLS/FROST/COSE/HPKE 曾短暂纳入，经减法轮判定为"引用即得/形态冲突/向量 TBA"而删除或指引现成（见 CONTENT.md §7）。

---

## 2 目标拆解

### 2.1 供给层功能覆盖对标

算法、格式、信任、平台四个面的覆盖清单逐项对照 OpenSSL 常用面；纳入与否一律走三条准入判据＋**证据四格**（Rust 现状/死因/跨语言参照/判据落点），裁决留痕。**内容清单的唯一权威是 [`CONTENT.md`](CONTENT.md)**。

### 2.2 复用优先（引用＋验证级编排，不重写）

| 层 | 自研 / 引用 |
|---|---|
| 合同层（`groma-core`/`groma-codec`） | **自研**——这是 Groma 存在的理由 |
| 算法实现 | **全部引用**：RustCrypto（P3 单路，广度）＋graviola（P4，与 rustls 适配器同批）；libcrux 经减法轮砍掉（0.0.x 平行栈） |
| 验证级编排与差分验证（PKCS#12、CMS verify、OCSP HTTP、CT 编排、厂商证明编排、TPM quote 子集、熵健康） | **自研**——"现成不满足才自研"三触发：稳定线铁律／信任缺口（无差分·审计·向量）／形态冲突；自研总量 ~5k 行量级 |
| L7 适配器 | **自研薄层**，让 rustls/quinn/russh/boringtun/OpenPGP 接受 Groma 后端 |

### 2.3 架构基石：冻结的 provider 合同

"良好的架构"具体化为一条：**所有能力挂在冻结的合同上，后端可替换、能力可查询、公开 API 无后端类型**。合同按算法族分组（Digest/Mac/Kdf/Aead/Signer/Verifier/Kem/PasswordHash/Random），对象安全（D2=(d)），规范化类型（封闭枚举＋newtype），分类错误（无效签名≠无效编码≠不支持≠后端故障），纯构造注入（D4=(e)，合同零选择机制）。设计决策 D1–D9 见 DESIGN.md。

### 2.4 质量与信任对标

"OpenSSL 级"同时指质量：共享向量集一致、差分对抗（OpenSSL 是 P2 起差分 oracle）、每个密码学改动带独立来源公开向量＋负向用例、解析入口带 fuzz、交付路径无 C、每 crate 记录审查边界、恒定时间门禁（十二章硬约束见 HANDOFF §7）。

---

## 3 覆盖面对照表（摘要，最终版见 CONTENT.md）

| OpenSSL 常用面 | 代表项 | Groma 层/期 | 来源 |
|---|---|---|---|
| 哈希 | SHA-2、SHA-3、SHAKE、BLAKE2/3 | L3 / P2–P3 | 引用 RustCrypto |
| MAC | HMAC、CMAC、GMAC、KMAC、Poly1305 | L3 / P2–P3 | 引用；GMAC/KMAC 官方拼装（5/50 行） |
| KDF | HKDF、PBKDF2、scrypt | L3 / P2–P3 | 引用（KBKDF 砍——HKDF 已覆盖） |
| 口令哈希 | Argon2、scrypt、bcrypt | L3 / **P2**（R4 承诺） | 引用 |
| AEAD | AES-GCM/CCM、XChaCha、XTS、AES-SIV | L3 / P2–P3 | 引用（OCB3 降级可选 feature） |
| 签名 | ECDSA P-256/384/521、Ed25519、RSA（验签，延后 P4）、Ed448（延后 P4）、secp256k1-BIP340（超集） | L3 / P2–P4 | 引用；Ed25519 **P2**（R4 承诺） |
| 后量子 | ML-KEM、ML-DSA（RustCrypto 单路）；SLH-DSA deferred | L3 / P3 | 引用；无 C 是 OpenSSL 给不了的 |
| 密钥交换 | X25519、ECDH（X448 延后 P4；FFDHE 砍——dh crate 已不存在） | L3 / P3 | 引用 |
| 国密（超集） | SM3/SM4（SM2 延后等 0.14 转正） | L3 / P3 | 引用 |
| RNG/DRBG | OS RNG、HMAC-DRBG、SP 800-90B 运行时健康（小项） | L3/L6 / P6 | 引用＋自研小项 |
| 证书与密钥编码 | PEM、DER、PKCS#8、SPKI、CSR、OID 注册表 | L4 / P1–P3 | 引用＋codec 自研 |
| 密钥库 | PKCS#12 创建＋MAC（**自研 1.5–2k 行＋双向差分**） | L4 / **P2+（获客首发）** | 自研（差分验证） |
| 签名容器 | CMS 验签（**自研 ~500 行编排**） | L4/L5 / P5 | 自研编排 |
| 信任路径 | 路径构建（pkix-path）、OCSP（~300 行编排）、CT（sct＋ct-merkle 编排）、MDS3（fido-mds 策略层） | L5 / P5 | 现成拼装＋自研编排 |
| 厂商证明（超集） | Android/Apple（octet 现成）＋吊销订阅、TPM quote 子集（**自研 1–2k 行**） | L5/L6 / 中期–P6 | 现成＋自研 |
| 协议密码学后端 | rustls/quinn/russh/boringtun/OpenPGP（graviola P4） | L7 / P4 | 适配器自研 |
| 遗留兼容项 | MD5、SHA-1 | — | **均 deferred**（rustls 栈与公共 PKI 2026 已不需要） |
| 长尾 | RC4、MD4、DES/3DES、IDEA、Blowfish、CAST、SEED、Camellia… | — | 三条判据不过，deferred |

---

## 4 显式不对标清单（OpenSSL 有、Groma 不做；含减法轮"改用现成"指引）

| OpenSSL 成分/能力 | Groma | 理由 / 消费者去哪 |
|---|---|---|
| libcrypto API/ABI 兼容 | ❌ | 绑定历史接口，与"冻结自己的合同"矛盾 |
| openssl CLI 与配置语法 | ❌（已决不定期限） | 产品化工具，非库层能力 |
| FIPS 140-3 验证 | ❌ | R6：换语言＝新实现，证书不可继承；**FIPS 段永远不碰** |
| ENGINE/provider 插件机制 | ❌ | Groma 自有 backend 合同替代，且能力可机器查询 |
| 协议栈本体（TLS/QUIC/SSH/IPsec/Kerberos/Signal/WireGuard） | ❌ | 生态已有纯 Rust 实现，Groma 出适配器 |
| 遗留算法（RC4/MD4/DES 家族/IDEA/…） | ❌ | 三条判据过不了，deferred 留痕 |
| 20 年历史兼容行为（quirks） | ❌ | 不复制错误 |
| TSA 一站式 | 不做（减法轮删除自研） | 指引 sigstore-tsa 0.11 |
| COSE_Sign1 独立件 | 不做 | coset 0.4.2 内置验签＋attestation 胶水 |
| HPKE | 不做 | hpke 0.14 开箱即用 |
| FROST 门限签名 | 不做 | frost-* 3.0 成熟 |
| BLS12-381 | 不做 | draft 向量 TBA（判据②不过）；指引 zkcrypto/blsful |
| FFDHE | 不做 | RustCrypto dh crate 已不存在；rustls 内嵌常量 |

---

## 5 "OpenSSL 级"的验收方式（不留空话）

1. **覆盖验收**：§3 清单逐项裁决并留痕（证据四格）；每期结束列出"本期新增覆盖 vs OpenSSL 对应面"
2. **质量验收**：共享向量集一致；P2 起与 OpenSSL 差分对抗（PKCS#12 双向差分即首发件）；不一致一律留档裁决；fuzz 目标随解析入口走
3. **消费端验收**（最硬）：P4 出口判据——rustls/quinn/russh 各自用 Groma 后端跑通自己的测试套件
4. **供给链验收**：锁定版本、许可清单、逐 crate 审查边界、无 C 构建要求，发布绑定 commit＋lock＋支持矩阵＋测试摘要

---

## 6 与既有章程的关系

- 用户权威需求 **R1–R6 全部不动**；本声明是 R1（通用库）与 R2（覆盖尽可能全）的"OpenSSL 对标"具体化
- 本文件定稿后并入 `SCOPE.md`（§0 目标、§2 范围边界），按"范围变更先改权威文档"流程执行
- 不改变任何范围裁决：不重写算法、不做 FIPS、不做协议栈、三条准入判据不变

---

## 7 待拍板项（用户逐条确认后转定稿）

- [x] §0 一句话目标措辞：已吸收双锚点（获客=验证级稀缺件，留存=合同）、取代 aws-lc-rs、超集扩张（2026-09-16 全流程定案）
- [x] §3 覆盖面对照表：已逐项过（加法轮＋复核轮＋补漏轮＋减法轮，最终版见 CONTENT.md）
- [x] §4 CLI：已决不做
- [ ] 定稿后并入 SCOPE.md 的时机（建议：P1 合同冻结出口时一并固化）

---

## 修订历史

| 日期 | 变更 |
|------|------|
| 2026-09-16 | 创建草稿：OpenSSL 对标语义澄清（功能覆盖度＋引用生态）、目标拆解四条、覆盖面对照表、显式不对标清单、验收方式、待拍板项。 |
| 2026-09-16 | 定位扩张：追加现代生态超集（用户拍板）。 |
| 2026-09-16 | 与减法后终版同步：§0 吸收取代 aws-lc-rs/双锚点；§2.2 改为"引用＋验证级编排"（libcrux 砍、graviola P4）；§3 覆盖表对齐 CONTENT.md 最终清单；§4 增"改用现成"指引（TSA/COSE/HPKE/FROST/BLS/FFDHE）；§7 待拍板大部勾销。 |
