# Groma 目标声明（草稿）

> **文档类型**：goals（草稿，待用户拍板）
> **所属模块**：Groma
> **创建日期**：2026-09-16
> **状态**：**DRAFT** —— 本文件是提案，不是裁决。定稿后并入 [`SCOPE.md`](SCOPE.md) §0/§2，在此之前一切以 `SCOPE.md` 为准
> **权威顺序**：[`SCOPE.md`](SCOPE.md)（权威需求）> [`HANDOFF.md`](HANDOFF.md)（执行细化）> 本文（草稿）> 其他文档
> **来源**：2026-09-16 用户提出"希望 Groma 做到 OpenSSL 的水平"，并明确两点：① 功能覆盖度对标；② 有现成纯 Rust 实现可以引用

---

## 0 一句话目标

**Groma 在纯 Rust 生态中提供 OpenSSL 级的密码学供给**：功能覆盖度对标 OpenSSL 中被现代消费者实际使用的算法、密钥与证书格式、信任路径与平台原语；有现成纯 Rust 实现的直接引用，Groma 自研只做合同层、有界补缺与适配器；以冻结的 provider 合同为架构基石，P1–P6 分期交付，每期机器可判定出口。

---

## 1 语义澄清：什么叫做"对标 OpenSSL"

"对标"分四层，本项目只取第一层：

| 对标层级 | 含义 | Groma 态度 |
|---|---|---|
| **功能覆盖度** | OpenSSL 覆盖的算法/格式/信任/平台面，Groma 也有（供上同样能力） | ✅ **这是目标** |
| 质量与可替换性 | 向量一致、差分对抗、fuzz 干净、后端可换 | ✅ 这是目标 |
| 兼容性 | libcrypto API/ABI、openssl CLI、配置语法、ENGINE 插件 | ❌ 非目标 |
| 合规背书 | FIPS 140-3 验证模块 | ❌ 非目标（R6） |

### OpenSSL 为什么庞大（拆解）

OpenSSL 的庞大不是"算法多"，而是四样东西堆在一起：

| 成分 | 占比重（估） | 性质 | Groma 策略 |
|---|---|---|---|
| 算法与密码学原语 | ~20% | 哈希/MAC/AEAD/签名/KEM/口令哈希/随机 | **引用** RustCrypto/graviola/libcrux，包装为后端 |
| 密钥与证书格式 | ~25% | PKCS#8/SPKI/PKCS#12/CMS/X.509/CSR/OCSP/CRL/CT | 编解码归 codec，容器与路径归 L4/L5，**有界补缺自研** |
| 协议栈与粘合 | ~30% | TLS/QUIC/SSH 本体、libcrypto 上千 API、CLI、配置、ENGINE | 协议本体用 L7 适配器接现有纯 Rust 实现；API/CLI/ENGINE **不做** |
| 历史包袱与合规 | ~25% | RC4/MD4/DES 家族等遗留算法、20 年兼容行为、FIPS 模块 | 遗留算法按三条判据裁决，多数 deferred；FIPS 不做 |

> 关键洞察：OpenSSL 的护城河约有一半是它的负担。Groma 只对标"被现代消费者实际使用"的那 ~45% 供给面，并用"引用生态"而不是"重写生态"的方式达到覆盖度——这正是用户"有现成纯 Rust 实现可以引用"的意图。

---

## 2 目标拆解

### 2.1 供给层功能覆盖对标

算法、格式、信任、平台四个面的覆盖清单逐项对照 OpenSSL 常用面；纳入与否一律走三条准入判据（公开规范＋公开向量＋真实消费者），裁决留痕。覆盖清单的初始版本见 §3，P1 期间逐项确认。

### 2.2 复用优先（引用，不重写）

| 层 | 自研 / 引用 |
|---|---|
| 合同层（`groma-core`/`groma-codec`） | **自研**——这是 Groma 存在的理由 |
| 算法实现 | **全部引用**：RustCrypto（广度）、graviola（s2n-bignum 验证汇编）、libcrux（HACL* 提取）；只写薄包装后端 |
| 有界补缺（PKCS#12 创建、CMS 验证、OCSP、证明路径等） | **自研**，照规范＋公开向量实现，不移植其他语言源码 |
| L7 适配器 | **自研薄层**，让 rustls/quinn/russh/boringtun/OpenPGP 接受 Groma 后端 |

### 2.3 架构基石：冻结的 provider 合同

"良好的架构"具体化为一条：**所有能力挂在冻结的合同上，后端可替换、能力可查询、公开 API 无后端类型**。合同按算法族分组（Digest/Mac/Kdf/Aead/Signer/Verifier/Kem/PasswordHash/Random），规范化类型（封闭枚举＋newtype），分类错误（无效签名≠无效编码≠不支持≠后端故障）。这一条是 P1 的出口判据，不是口号。

### 2.4 质量与信任对标

"OpenSSL 级"同时指质量：共享向量集一致、差分对抗（OpenSSL 本身就是 P2 的差分 oracle）、每个密码学改动带公开向量＋至少一个负向用例、解析入口必有 fuzz target、交付路径无 C 构建要求、每 crate 记录审查边界。

---

## 3 覆盖面对照表（初始草案，P1 期间逐项确认）

| OpenSSL 常用面 | 代表项 | Groma 层/期 | 来源 |
|---|---|---|---|
| 哈希 | SHA-2、SHA-3、BLAKE2/3 | L3 / P2 | 引用 RustCrypto |
| MAC | HMAC、CMAC、GMAC、KMAC、Poly1305 | L3 / P2–P3 | 引用 |
| KDF | HKDF、PBKDF2、KBKDF | L3 / P3 | 引用 |
| 口令哈希 | Argon2、scrypt、bcrypt | L3 / **P2**（R4 承诺） | 引用 |
| AEAD | AES-GCM/CCM、ChaCha20-Poly1305 | L3 / **P2**（R4 承诺） | 引用 |
| 分组/流密码 | AES、ChaCha20 | L3 / P2 | 引用 |
| 签名 | ECDSA P-256/384/521、Ed25519/Ed448、RSA PKCS#1 v1.5/PSS | L3 / P2–P3 | 引用；Ed25519 **P2**（R4 承诺） |
| 后量子 | ML-KEM、ML-DSA、SLH-DSA | L3 / P3 | 引用 libcrux/graviola |
| 密钥交换 | X25519、ECDH、ML-KEM KEM | L3 / P3 | 引用 |
| RNG/DRBG | OS RNG、HMAC-DRBG | L3 / P6 | 引用 |
| 证书与密钥编码 | PEM、DER、PKCS#8、SPKI、CSR | L4 / P3 | codec 自研＋格式自研 |
| 密钥库 | PKCS#12 | L4 / P5 | 自研补缺 |
| 签名容器 | CMS | L4 / P5 | 自研补缺 |
| 信任路径 | X.509 路径构建、名称约束、CRL、OCSP、CT | L5 / P5 | 自研补缺 |
| 平台 | TPM 2.0 结构、PKCS#11 core、DRBG 熵健康 | L6 / P6 | 自研 |
| 协议密码学后端 | rustls/quinn/russh/boringtun/OpenPGP | L7 / P4 | 适配器自研 |
| 遗留兼容项 | MD5、SHA-1（证书链兼容场景） | L3 / 按判据 | 仅当真实消费者存在；标注 legacy |
| 长尾 | RC4、MD4、DES/3DES、IDEA、Blowfish、CAST、SEED、Camellia… | — | 按三条判据裁决，默认 deferred |

---

## 4 显式不对标清单（OpenSSL 有、Groma 不做）

| OpenSSL 成分 | Groma | 理由 |
|---|---|---|
| libcrypto API/ABI 兼容 | ❌ | 绑定历史接口，与"冻结自己的合同"矛盾 |
| openssl CLI 与配置语法 | ❌（暂） | 产品化工具，非库层能力；可后续单独立项 |
| FIPS 140-3 验证 | ❌ | R6：换语言＝新实现，证书不可继承 |
| ENGINE/provider 插件机制 | ❌ | Groma 自有 backend 合同替代，且能力可机器查询 |
| 协议栈本体（TLS/QUIC/SSH/IPsec/Kerberos/Signal/WireGuard） | ❌ | 生态已有纯 Rust 实现，Groma 出适配器 |
| 遗留算法（RC4/MD4/DES 家族/IDEA/…） | ❌ | 三条判据过不了，deferred 留痕 |
| 20 年历史兼容行为（quirks） | ❌ | 不复制错误 |
| S/MIME、TSA、cms 之外的边缘 CLI 工具 | ❌ | 消费者侧或另立项目 |

---

## 5 "OpenSSL 级"的验收方式（不留空话）

1. **覆盖验收**：§3 清单逐项裁决并留痕；每期结束列出"本期新增覆盖 vs OpenSSL 对应面"
2. **质量验收**：共享向量集一致；P2 起与 OpenSSL 差分对抗，不一致一律留档裁决；fuzz 目标随解析入口走
3. **消费端验收**（最硬）：P4 出口判据——rustls/quinn/russh/boringtun/OpenPGP 各自用 Groma 后端跑通自己的测试套件
4. **供给链验收**：锁定版本、许可清单、逐 crate 审查边界、无 C 构建要求，发布绑定 commit＋lock＋支持矩阵＋测试摘要

---

## 6 与既有章程的关系

- 用户权威需求 **R1–R6 全部不动**；本声明是 R1（通用库）与 R2（覆盖尽可能全）的"OpenSSL 对标"具体化
- 本文件定稿后并入 `SCOPE.md`（§0 目标、§2 范围边界），按"范围变更先改权威文档"流程执行
- 不改变任何范围裁决：不重写算法、不做 FIPS、不做协议栈、三条准入判据不变

---

## 7 待拍板项（用户逐条确认后转定稿）

- [ ] §0 一句话目标措辞（定稿时吸收 [`POSITIONING.md`](POSITIONING.md) §1.1 的双锚点：获客=无 C 稀缺件，留存=统一合同；"组合现有库"为非卖点）
- [ ] §3 覆盖面对照表的粒度与缺项（P1 期间逐项过）
- [ ] §4 是否确认"暂不做 CLI"
- [ ] 定稿后并入 SCOPE.md 的时机（建议：P1 合同冻结出口时一并固化）

---

## 修订历史

| 日期 | 变更 |
|------|------|
| 2026-09-16 | 创建草稿：OpenSSL 对标语义澄清（功能覆盖度＋引用生态）、目标拆解四条、覆盖面对照表、显式不对标清单、验收方式、待拍板项。 |
