# Groma 路径与分布规格（草稿）

> **文档类型**：paths-spec（设计，待用户逐节拍板）
> **所属模块**：Groma
> **创建日期**：2026-09-16
> **状态**：**DRAFT** —— 规划内容的逻辑路径（层→crate→模块→API）与代码路径（文件系统→依赖图→发布路径）；定稿后并入 [`SCOPE.md`](SCOPE.md) §3.3
> **依据**：`CONTENT.md`（15 crate 终版）、`DESIGN.md` D1–D9、十二章硬约束、`SCOPE.md` §3.3 既有布局约定

---

## 1 文件系统布局（代码路径总树）

```
Groma/
├─ Cargo.toml                 workspace root（resolver=3、MSRV 1.89、workspace.lints）
├─ rust-toolchain.toml        1.98.1（dev）/ MSRV 1.89（CI 腿）
├─ deny.toml                  C 依赖禁令＋许可白名单
├─ Cargo.lock                 提交（仓库政策）
├─ Crates/
│  ├─ groma-core/              L1 合同（唯一零第三方依赖的库 crate）
│  ├─ groma-codec/             L1 有界解析（CBOR/COSE/DER/PEM）
│  ├─ groma-stub/              L2 差分后端（自写 SHA-256）
│  ├─ groma-rustcrypto/        L2 后端（RustCrypto 0.11 世代适配）
│  ├─ groma-graviola/          L2 后端（P4，与适配器同批）
│  ├─ groma/                   facade（装载 feature，无代码只有 re-export）
│  ├─ groma-registry/          普通对象注册表（可选）
│  ├─ groma-pkcs12/            L4 获客首发（P2+）
│  ├─ groma-cms/               L4/L5（P5）
│  ├─ groma-trust/             L5 编排（路径/OCSP/CT/MDS3）（P5）
│  ├─ groma-attestation/       L5/L6 编排（Android/Apple/COSE Sign1 胶水）（中期）
│  ├─ groma-tpm/               L6（P6）
│  ├─ groma-entropy/           L6（P6）
│  ├─ groma-adapter-rustls/    L7（P4）
│  ├─ groma-adapter-quinn/     L7（P4）
│  └─ groma-adapter-russh/     L7（P4）
├─ Tools/
│  ├─ vector-fetch/            独立 manifest（非 workspace 成员）
│  └─ dependency-audit/        独立 manifest（非 workspace 成员）
├─ Fuzz/
│  ├─ cose_key/                cargo-fuzz 目标（独立 manifest）
│  ├─ cbor_bounded/
│  └─ der_signature/
├─ Vectors/
│  ├─ manifest.toml            单一清单（D7=(d)）
│  ├─ *.json / *.rsp           vendored 小向量（<100KB）
│  └─ local/                   （gitignore）内容寻址缓存
├─ Docs/                       章程与设计文档
└─ .github/                    CI + dependabot
```

**规则**：crate 目录名全小写（工具链硬约束）；Tools/Fuzz 为独立 manifest（不进 workspace members——避免污染发布面/MSRV 面/审计面）。

---

## 2 逻辑路径（层→crate 映射，五簇）

```
消费者（应用 / 协议栈 / 第三方 crate）
      │
  L7  groma-adapter-{rustls,quinn,russh}      ← 适配簇
      │
  L5/L6  groma-trust · groma-cms · groma-attestation · groma-tpm · groma-entropy   ← 获客簇
      │
  L4  groma-pkcs12（获客首发）───────────────────────────┘
      │
  L3/L2  groma-rustcrypto · groma-stub · groma-graviola(P4)   ← 后端（只实现合同）
      │
  L1  groma-core（合同）· groma-codec（有界解析）              ← 地基
      │
  groma（facade）· groma-registry   ← 装载/选择（可选外围）
```

---

## 3 crate 依赖图（决定顺序的关键）

**方向铁律**：一切指向 L1；**后端只实现合同、从不被获客件硬依赖**；第三方依赖只走稳定线。

```
groma-core ──(零 groma 依赖；第三方仅 subtle/ctutils 等 N1 原语)
groma-codec ──→ core（统一 Error）
groma-stub ───→ core
groma-rustcrypto ──→ core
groma-registry ──→ core
groma（facade）──→ core＋codec＋rustcrypto＋stub（可叠加 feature 装载）
groma-pkcs12 ──→ core＋codec          ★算法执行经 provider 注入，零后端硬依赖
groma-cms ────→ core＋codec＋(x509-cert 直接；链校验经注入 provider)
groma-trust ──→ core＋codec＋pkix-path＋x509-ocsp＋sct＋ct-merkle＋fido-mds
groma-attestation ──→ core＋codec＋coset＋octet-attest-verify
groma-tpm ────→ core＋tpm2-protocol
groma-entropy ──→ core
groma-graviola ──→ core＋graviola（P4）
groma-adapter-rustls ──→ rustls＋rustls-pki-types＋core＋(graviola|rustcrypto 桥)
groma-adapter-quinn ──→ quinn＋adapter-rustls
groma-adapter-russh ──→ russh＋core＋rustcrypto
Tools/* · Fuzz/* ──→ 独立（各自 manifest）
```

**★关键设计决定（D4=(e) 的推论）**：获客件与编排件（pkcs12/cms/trust/attestation/tpm/entropy）**零后端硬依赖**——它们的算法执行（PBKDF2/HMAC/验签）全部经 `&dyn Provider` 注入。收益三连：①获客件成为合同的**真实消费者**（第二个练兵场，第一个是 stub）②消费者可换后端（获客件本身可插拔）③获客件依赖树纯净，零 C 审计面最小。测试经 dev-deps 装配 rustcrypto。

---

## 4 模块路径（每个 crate 的 API 草图）

```
groma-core/src/
├─ lib.rs            #![no_std] + alloc；pub use 各模块
├─ algorithm.rs      封闭 #[non_exhaustive] 枚举（D3）
├─ error.rs          统一分类错误＋源链（D5）
├─ digest.rs / mac.rs / kdf.rs / aead.rs /
│  signer.rs / verifier.rs / kem.rs /
│  password_hash.rs / random.rs     九族对象安全 trait＋工厂（D2）
├─ capability.rs     能力查询
├─ key.rs            key/signature/nonce/tag/ciphertext newtype
└─ no_leak.rs        编译期"公开 API 无后端类型"断言

groma-codec/src/
├─ lib.rs            #![no_std] + alloc
├─ limits.rs         大小/深度/元素数上限（先于解析）
├─ cbor.rs / cose.rs / der.rs / pem.rs
└─ error 映射 → core::Error::InvalidEncoding

groma-pkcs12/src/
├─ lib.rs            pub use create/parse
├─ create.rs         端到端创建（现代算法路径）
├─ parse.rs          解析＋MAC 校验（legacy 复用 RustCrypto pkcs12）
├─ kdf.rs            PKCS#12 KDF 变体（经合同 Kdf）
├─ mac.rs            PBMAC1 封装（经合同 Mac）
└─ tests/differential.rs  OpenSSL/Botan 双向差分（Tools 驱动）

groma-rustcrypto/src/  digest/ mac/ kdf/ aead/ signer/ kem/ password_hash/
                      各一模块，适配 RustCrypto 0.11 世代；构造器 pub fn provider()
groma（facade）/src/lib.rs   仅 feature 门控的 re-export（零逻辑）
groma-registry/src/lib.rs    Registry::new().register(name, provider)（普通对象）
```

其余 crate 同模式：`lib.rs → 按规范分模块 → tests/ 含负向与差分`。

---

## 5 引用路径规范（第三方依赖怎么走）

1. **稳定线铁律**：官方 crate 只依赖新稳定线（rc/pre 清单见 CONTENT.md §5，一律延后）
2. **N2 白名单**：第三方依赖进 `deny.toml` 白名单＋逐依赖记录审查边界；未修公告按操作裁决（rsa 验签延后 P4）
3. **零 C 门禁**：`Tools/dependency-audit` 扫全官方 crate 的 links/cc/cmake——发布前必过
4. **dev-deps 例外**：差分 oracle（openssl CLI 子进程、Botan 样本）只出现在 dev-deps/Tools，永不进正常依赖
5. **feature 只在 facade**：core/codec/获客件零 feature；facade 的 feature = 装载（可叠加）
6. **MSRV 传导**：resolver=3 自动把传递依赖锁在 1.89 内

---

## 6 数据与工具路径

- **向量路径**：`Vectors/manifest.toml` 条目 → `vector-fetch sync` → `Vectors/local/<sha256>` → 测试按 ID 加载（D7=(d)）
- **差分路径**：获客件 tests/differential.rs → Tools 驱动 openssl/botan 子进程 → 结论归档（不一致留档裁决，DESIGN §4）
- **fuzz 路径**：Fuzz/<target> 独立 cargo-fuzz 工程 → CI ubuntu+nightly 短跑＋定期长跑
- **CI 路径**：无 C 检查（dependency-audit）→ 向量 check（vector-fetch check）→ 测试矩阵（1.98.1/1.89 × ubuntu/windows）→ no_std 交叉 → fuzz → dudect → 覆盖 → 禁词

---

## 7 发布路径

- crate 名 = 目录名（groma-*，crates.io 首发时占名，A2 已决）
- 版本：workspace.package 统一版本＋各 crate `version.workspace = true`（合同冻结后 0.1.0 起跳）
- 发布顺序 = 依赖拓扑序：core → codec/stub/rustcrypto → 获客件 → 适配器
- 发布前门禁：semver-checks＋publish dry-run（CI tag 腿已备）

---

## 8 待拍板

- [ ] §3 关键决定：获客件**零后端硬依赖**（算法经 provider 注入）——是/否
- [ ] §1：Tools/Fuzz 独立 manifest 不进 workspace members——是/否
- [ ] §7：统一 workspace 版本——是/否（vs 各 crate 独立版本）
- [ ] §4 模块草图逐 crate 粒度是否够用（实现会话可按此开工）
- [ ] 定稿后并入 SCOPE §3.3 的时机（与其余草案同批）

---

## 修订历史

| 日期 | 变更 |
|------|------|
| 2026-09-16 | 创建草稿：文件系统布局、逻辑路径五簇映射、crate 依赖图（含"获客件零后端硬依赖"关键决定）、模块 API 草图、引用路径规范、数据工具路径、发布路径、待拍板。 |
