# Groma 生态位方案书（草案）

> **文档类型**：positioning（生态位提案，待用户拍板）
> **所属模块**：Groma
> **创建日期**：2026-09-16
> **状态**：**DRAFT** —— 与 [`GOALS.md`](GOALS.md) 同为提案；两者定稿后并入 [`SCOPE.md`](SCOPE.md)
> **权威顺序**：[`SCOPE.md`](SCOPE.md) > [`HANDOFF.md`](HANDOFF.md) > [`GOALS.md`](GOALS.md)（草稿）> 本文（草稿）> `research/`
> **证据来源**：仓库内调研 [`research/ecosystem-gaps.md`](research/ecosystem-gaps.md)、[`research/coverage-union.md`](research/coverage-union.md)，加 2026-09-16 四路外部调研（[`research/ecosystem-maturity.md`](research/ecosystem-maturity.md) 全景与成熟度、[`research/gaps-verification.md`](research/gaps-verification.md) 缺口实证、provider 抽象竞争格局、消费端需求信号——后者两路为会话内报告）

---

## 0 一句话生态位

> **Groma 是纯 Rust 生态的"密码学供给层集成者"：冻结的算法族级 provider 合同，把 RustCrypto／graviola／libcrux 等零散实现装配成"无 C 构建、后端可替换、能力可查询"的统一供给层，并以有界补缺（TPM／PKCS#12／OCSP／CMS／路径验证）填平被 C 锁死的互操作空洞；no_std＋alloc 首日成立，审计与 PQC 沿路线图推进。**

否定式定位同样重要——**Groma 不是**：
又一个算法实现集合（RustCrypto 已垄断基元层）／ TLS 协议栈／ FIPS 140-3 模块／ HSM／ aws-lc-rs 克隆（那意味着带 C）／ orion 式"自家实现全家桶"（我们引用生态，不自研算法）。

---

## 1 交叉验证后的生态事实（四条线 × 两份仓库内调研，2026-09-16 快照）

1. **纯 Rust 已是主流，不是替代品**：sha2 单 crate 9.5 亿下载；RustCrypto 0.11 世代 2026 年迁移完成（digest 0.11/sha2 0.11/aes-gcm 0.11/ed25519-dalek 3.0），全线 MSRV 1.85 + edition 2024。例外：`rsa` 停在 0.9.10（0.10 已 18 个 rc），Marvin 公告 `patched=[]` 依旧。
2. **算法层不缺**：哈希/MAC/KDF/AEAD/签名/KEM/口令哈希/PQC 全覆盖（三线一致）。基元级没有立项空间。
3. **provider 抽象是真实空白**：`crypto-provider`/`crypto-facade` 等名字 crates.io 不存在；rustls CryptoProvider 是 TLS 形状的 struct（cipher_suites/kx_groups 字段 + 进程级 `install_default`）；RustCrypto `crypto` facade 只是版本兼容 re-export 且 3 年未更新（issue #1633 挂着）；aws-lc-rs 的 trait sealed。**"算法族级（Digest/Mac/Kdf/Aead/Signer/Verifier/Kem/PasswordHash/Random）+ 能力查询 + 值式可替换 + no_std+alloc" 无人在做。**
4. **最近竞品 rtc-crypto（webrtc-rs，2026-09-01）验证了时机**：开放 trait + `supports()` 能力查询 + 值式 provider + 公开一致性套件 `conformance::assert_provider`——但它是 WebRTC/DTLS/SRTP 形状，明确放弃 no_std、无 KDF/口令哈希族、KEM 被 key-exchange 形状替代、错误是 `CryptoError(String)`。
5. **"无 C"与"合规"冲突是真实采购障碍**：Google Cloud Rust SDK 官方 PR #4348（2026-01 合并）原文——客户要么禁 aws-lc-rs（含 C 代码），要么禁 ring（非 FIPS）。同时满足两者 = 两头通吃。
6. **PQC × 纯 Rust 互斥是当前最尖锐空白**：rustls 的 X25519MLKEM768 已提为最高优先级默认、0.23.44 起 ML-DSA 默认启用，但全部绑定 aws-lc-rs；ring 无 PQC；纯 Rust 侧只有 graviola（缺 ML-DSA/SHA-1、仅新 x86_64/aarch64）。`ml-kem` crate 近 90 天下载占总量 **70%**（全场增速第一）；Mozilla NSS 3.118 vendor 了 libcrux 的 ML-KEM——需求爆发、供给错位。
7. **C 依赖存续的头号原因是 FIPS 140-3**（aws-lc-rs 背后 6 张证书），其次是构建税与惯性；纯 Rust 无任何 FIPS 路径。Groma 的对策只能是：FIPS 非目标（自证）+ 合同层支持声明"本后端对接了 FIPS 模块"的边界。
8. **ring 停摆是战略风险信号**：0.17.14 后 18 个月零发布，<0.17 被 RUSTSEC-2025-0010 标 unmaintained；rustls 官方改推 aws-lc-rs（增速 37%/90 天 ≈ ring 的 1.8 倍）。生态整体滑向 C 后端，"无 C"用户被反向排挤。
9. **补缺的真实缺口按痛感排序**：TPM 2.0 纯 Rust 栈 > PKCS#12 创建/导出 > OCSP 验证级客户端 > CMS 健壮解析+EnvelopedData > X.509 全量路径验证。已解决、不值得做：PKCS#11（cryptoki）、JOSE/JWS（jsonwebtoken）、SSH 密钥格式（ssh-key）、OpenPGP（rpgp）。
10. **整合型竞争者全都有短板**：orion（活跃、含 PQC，但无审计、无 RSA/ECDSA/Ed25519）、rscrypto/Crown/purecrypto（2025-2026 新出，无审计无生态）、q-periapt（研究级）、graviola（高性能纯 Rust 上限样本，但 CPU 硬门槛、无第三方审计、bus factor≈1）、libcrux（形式化验证 ≠ 产品成熟：2026 上半年一批公告，全 crate <0.1）。

**买方排序**（按付费意愿/痛感）：① 受监管行业 Rust 服务端（无 C + FIPS 夹击）→ ② 多目标交叉编译 SDK/网关厂商 → ③ 网络/身份基础设施（quinn/russh/webauthn-rs 去 C 运动）→ ④ IoT/嵌入式（NoxTLS 2026-05 商业进场证明付费意愿）→ ⑤ 要 PQC 但不要 C 的团队 → ⑥ 通行密钥/WebAuthn 服务 → ⑦ 游戏服务器（传播层，付费意愿低）。

---

## 2 候选生态位方案

| 方案 | 内容 | 优点 | 缺点 |
|---|---|---|---|
| **A 集成供给层（推荐）** | 合同（L1）+ 多后端适配器 + 有界补缺（L4–L6）+ no_std + 审计路线图 + PQ 就绪，即现有 SCOPE 全谱系 | 差异化最全、同时命中买方 ①–⑤、与已定章程一致 | 盘子大、周期长、执行压力真实 |
| B 补缺专精 | 不做合同，直接用 RustCrypto traits，只做 TPM/PKCS#12/OCSP/CMS/路径 | 痛点最实、单点易成 | 放弃生态位皇冠（合同层），沦为"又一个小众 crate 集"；X.509/OCSP 窗口仅 6–12 个月（pkix-path/synta 竞争）；TPM 极重 |
| C 合同优先 | 只做 trait 合同 + 一致性套件 + 后端适配器，补缺全留给生态 | 最快兑现差异化，直接回应 rtc-crypto 空白 | 鸡生蛋问题：没有消费与补缺支撑，合同无人用；不解决 TPM/PKCS#12 等痛点，买方 ①④ 不买单 |
| D PQC 先锋 | 先做 ML-KEM/ML-DSA 的无 C 后端组合（RustCrypto + libcrux 适配） | ml-kem 增速 70%、PQC×无 C 空白最尖锐、时间窗明确 | 单点窄；标准面仍在变；审计要求高 |

**推荐：A 为骨架，D 提前，B 按痛点排序融入**——即"全谱系定位 + 锋利切入顺序"（见 §4）。理由：四条调研线独立收敛到同一结论（空白在"抽象+集成+补缺"三层，不在算法），而买方 ①④⑤ 的订单只在 A 的交集上；纯 B/C/D 都会退化成又一个有短板的竞争者（§1 第 10 条已经列满了）。

---

## 3 差异化清单（对每个竞品一条）

1. **vs rustls CryptoProvider**：算法族合同而非 TLS 形状（cipher_suites/kx_groups 对非 TLS 消费者无意义）；provider 是普通值 + 可选注册表，不做 `install_default` 式进程全局；同时提供 rustls CryptoProvider 桥，让 Groma 后端直接喂给 rustls。
2. **vs RustCrypto `crypto` facade**：不是"版本兼容打包"，而是 dyn 兼容的开放 trait + 统一错误分类 + 能力查询，补齐 facade 漏掉的 Kdf/Kem/Random 三族；RustCrypto 实现是一等后端适配器。若官方 facade 复活，适配而非对抗。
3. **vs rtc-crypto**：no_std+alloc 首日成立（它明确放弃、q-periapt 证明可行）；补 KDF 族（HKDF/PBKDF2/Argon2 形态）与 KEM 族（ML-KEM 优先）；结构化错误而非 `CryptoError(String)`；学它的两件已验证设计——密钥化操作工厂化（评测：每次重算 HMAC key schedule 占 RTCP 40%）与公开一致性套件 `conformance::assert_provider`。
4. **vs graviola**：不拼性能，拼覆盖与可替换——graviola 只是五个示范后端之一（另：RustCrypto、libcrux、ring、aws-lc-rs 的适配器，证明"第三方可插"）；CPU 硬门槛由多后端能力查询兜底（协商前失败，而非运行时 panic）。
5. **vs orion/rscrypto 等整合型**：它们绑定自家实现；Groma 的护城河是后端可替换 + 审计路线图 + 逐 crate 证据标注，不宣称整体已审计。
6. **vs aws-lc-rs**：无 C 构建要求 + trait 永久开放（它 sealed）+ 能力查询；FIPS 不做（非目标），但合同层给"FIPS 后端"留声明边界。

---

## 4 切入顺序建议（对 SCOPE 阶段表的微调建议，需拍板后生效）

| 阶段 | 原内容 | 建议 |
|---|---|---|
| P1 | 合同冻结 + 两后端 | **不变**（合同是全部差异化的地基） |
| P2 | R4 兑现（Ed25519/Argon2/AEAD）+ RustCrypto 后端 | **不变**（玩家服务排期承诺优先） |
| P3 | 算法面 + 格式 | **PQC 提前**：ml-kem/ml-dsa/slh-dsa 后端适配（RustCrypto + libcrux 两路）进 P3 前端，抢"PQC × 无 C"窗口 |
| P4 | 适配器（rustls/quinn/russh/boringtun） | 不变；graviola 适配器放这里 |
| P5 | 信任与容器 | 顺序按痛点：**OCSP → CMS → PKCS#12 → 路径验证**；注意 pkix-path/synta 的 6–12 个月窗口，设计与调研提前启动 |
| P6 | 平台（TPM/PKCS#11/熵） | 不变，但**提前公告 TPM 路线图承诺**（最痛缺口，买家 ①④ 加分项）；PKCS#11 引用 cryptoki，熵测试自研 |

---

## 5 风险与反证（诚实清单）

| 风险 | 评估 | 应对 |
|---|---|---|
| RustCrypto 官方 facade 复活（issue #1633） | 挂 3 年未动，概率中等 | 合同层自研先发；真复活则提供"官方 facade 后端适配器"，把竞争变互操作 |
| rtc-crypto 扩张成通用层 | 它在 webrtc-rs 内、协议形状惯性大；概率低 | no_std + 族齐全 + 先发时间差 |
| 大厂（AWS/Google）自研纯 Rust 供给层 | Google PR #4348 显示他们要的是"可剪除 provider"，不是新库；概率中 | 合同层正好是解药；若发生，我们的多后端合同仍是中立集成点 |
| 审计资源不足 | 真实 | P2 起逐 crate 记录审查边界与证据，严禁整体宣称"已审计"；审计路线图进文档 |
| graviola bus factor≈1 | 真实 | 只作后端之一，任何后端都可被替换正是合同的意义 |
| X.509/OCSP 窗口 6–12 个月 | 真实 | P5 设计提前启动；最坏情况适配 pkix-path/synta 为后端而非重写 |
| A 方案盘子大、执行压力 | 真实 | "A 骨架 + 锋利切入"：每期出口判据机器可判定，范围变更走 SCOPE；不并行堆功能 |
| 买方 ① 的 FIPS 期望 | 我们给不了证书 | 明示非目标（R6），提供"对接 FIPS 模块的边界 + 证据标注"；不暗示合规 |
| TPM 被 tss-esapi 锁死 | tss-esapi 8.0 仍 alpha | 纯 Rust 窗口尚在，P6 前持续监控，不轻易承诺日期 |

---

## 6 待拍板（用户）

- [x] 生态位方案选型：**已决（2026-09-16）＝ 方案 A（集成供给层）**；执行节奏按 E（A+D+B）精神：PQC 提前、补缺按痛点排序，具体阶段顺序待全貌考虑后定
- [ ] 阶段顺序微调：**暂缓**——待 P1 合同设计决策（D1–D7/A5）全部拍板后，连同全貌一并定
- [ ] 本文 + `GOALS.md` 定稿后并入 `SCOPE.md` 的时机：**暂定默认＝P1 出口时一并固化**（待最终确认）

---

## 修订历史

| 日期 | 变更 |
|------|------|
| 2026-09-16 | 创建草案：基于四路外部调研 + 两份仓库内调研的交叉结论，提出生态位一句话定位、四个候选方案（推荐 A）、六条差异化清单、阶段顺序微调建议、风险清单。 |
| 2026-09-16 | 用户拍板：生态位＝方案 A（集成供给层）；阶段顺序调整暂缓（待 D1–D7 全貌）；并入 SCOPE 暂定 P1 出口时。 |
