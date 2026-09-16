# 接手需求书（临时）

> **文档类型**：handoff（临时交接待办）
> **所属模块**：Groma
> **创建日期**：2026-09-16
> **实现状态**：待接手 Agent 执行；一经接手即可删除或归档
> **权威顺序**：[`SCOPE.md`](SCOPE.md)（用户已认可的权威需求）> 本文（执行性细化）> 其他一切文档
> **移交来源**：2026-09-16 用户决定独立立项并建设通用纯 Rust 密码学库；本文由立项会话整理，交给承接工作区的 Agent 执行

---

## 0 一句话

用户要建一个**通用**纯 Rust 密码学库，覆盖尽量全、分期交付；并要求**用户自己项目（PlayerServices 玩家服务）所需的"轮子"部分必须被这个库覆盖**，深度适配不归本库。立项已完成，现在从 **P1（合同冻结）** 开始实施。

---

## 1 背景与决策链（为什么会有这个项目）

按时间顺序，理解这条链才能理解范围为什么长这样：

1. 玩家服务（PlayerServices）为身份侧选型时受**零 C／C++ 依赖**约束（[网络层决策 42](../../Network/Docs/决策记录/42_Rust服务端依赖政策.md) 档 0 ＋ 本项目 B3），禁名单含 `ring`／`aws-lc-rs`／`openssl-sys`／`native-tls`。TLS 已选 `rustls`＋`rustls-graviola`（B4）。
2. 核查发现成熟 WebAuthn 协议层把 OpenSSL 写死，因此原先立了一个**窄范围**项目（VeriCeremony，只做 WebAuthn 加密 provider）。
3. 进一步核查生态后确认：**算法层不缺**（RustCrypto／graviola／libcrux 覆盖极广），缺的是**通用 provider 抽象、审计与 FIPS 证据、消费端可替换性**三样。
4. 用户裁决：**不做窄项目，做通用库**。覆盖全是好事，但不能一次全做完，要分期。
5. 用户进一步明确：**必须保证玩家服务的需求被这个库覆盖**——是"轮子"部分（哈希／签名／AEAD／口令哈希／密钥与证书格式等），不是"深度适配"部分（WebAuthn ceremony 政策、SNSM 格式、账号会话模型、平台 SDK）。
6. 用户定名 **Groma**（罗马测量师的定基仪器：先定轴网，再往上盖），并要求目录全英文、文档可为中文、建立公开仓库。
7. 于是 `SCOPE.md` 被**重写**（不是改窄项目文档）：旧文档"完整 OpenSSL 对标库不是本项目目标"这条**已删除**，改为三层准入判据＋L1–L7 分层＋P1–P6 分期＋明确的不做清单。

**核查证据**在 [`research/ecosystem-gaps.md`](research/ecosystem-gaps.md) 与 [`research/coverage-union.md`](research/coverage-union.md)，接手前应先读这两份，避免重复调研或踩已知坑。

---

## 2 权威需求（用户原话意图的结构化）

| # | 需求 | 含义 |
|---|------|------|
| R1 | **通用库** | 不是 WebAuthn 专用；面向"纯 Rust 密码学供给"这一整层 |
| R2 | **覆盖尽可能全** | 缺口清单里凡符合条件的都可纳入；覆盖全是好事 |
| R3 | **分期交付** | 不要求一次做完；每期独立可用、有可判定的出口 |
| R4 | **必须服务用户自己的项目** | 玩家服务所需的**轮子**必须在这个库里被满足，不能"库很通用但用户自己没得到好处" |
| R5 | 深度适配不进库 | 玩家服务的业务与协议政策逻辑留在消费者侧 |
| R6 | 不做 FIPS 140-3 | 纯 Rust 无已验证模块；CMVP porting 只管换运行环境，换语言是新实现，证书不可继承 |

**R4 的兑现方式已经写进 `SCOPE.md` §5**：玩家服务的日常密码学需求（Ed25519 签名、Argon2、AEAD）在 **P2** 就满足，不必等通用性做完。**接手 Agent 必须保住这条排期承诺。**

---

## 3 范围裁决（重要，勿走偏）

| 事项 | 裁定 | 出处 |
|------|------|------|
| 是否做 OpenSSL 全对标 | **否**，是独立的另一个项目 | `SCOPE.md` §2.3 |
| 是否做 FIPS 140-3 | **否**，显式非目标 | §2.3 |
| 是否重写已有纯 Rust 算法 | **否**。通用性靠**接口广度＋后端可替换＋有界补缺**达成 | §0.2 |
| 是否做协议栈（TLS／QUIC／SSH／IPsec／Kerberos／Signal／WireGuard） | **否**，用 L7 适配器让现有纯 Rust 协议接受纯 Rust 后端 | §2.3 |
| 是否做长尾无消费者算法（KASUMI／CLEFIA／MISTY1／HIGHT／PRESENT／SNOW 3G／GOST 签名…） | **否，列为 deferred**；满足三条判据才可移出 | §2.2／§6 |
| 是否做 WebAuthn 证明语句验证逻辑 | **大部分不进库**（属深度适配）。只有其下的 X.509 路径／OCSP／TPM 结构／COSE·DER 编解码进库 | §2.4 |
| 加入新能力的判据 | **三条全过**：①有公开规范 ②有公开测试向量 ③有真实消费者 | §2.1 |

---

## 4 目录与命名约定（已完成，勿再重命名）

| 约定 | 值 |
|------|-----|
| 顶层目录 | **PascalCase**：`Crates/` `Docs/` `Fuzz/` `Tools/` `Vectors/` |
| 仓库根文件 | 保持约定俗成的形式：`.gitignore` `.gitattributes` `Cargo.toml` `README.md` `LICENSE-*` `SECURITY.md` `CONTRIBUTING.md` `rust-toolchain.toml` |
| **crate 目录名** | **必须小写**（`Crates/groma-core` 等）——Cargo 的 package 名不允许大写，这是工具链硬约束，不是风格选择 |
| crate 命名 | `groma-*` 家族，见 `Crates/README.md` |
| 文档语言 | `Docs/` 下叙述用中文；`README.md` 为英文入口 |
| 代码／标识符／路径／命令 | 英文 |
| 行尾 / 编码 | **LF ＋ UTF-8 无 BOM**；已用 `.gitattributes` 强制（`* text=auto eol=lf`），Windows 下 `core.autocrlf=true` 已不再影响 |

> **注意**：`git core.ignorecase=true`。将来若再改目录大小写，必须**两步走**（先 `git mv a _tmp`，再 `git mv _tmp A`），否则 git 视为无变化。

---

## 5 仓库与发布状态

| 项 | 值 |
|---|---|
| 本地 | `C:\Project\Groma`（**Git 仓库**，与 P4 工程 `TheSunNeverSets` 无关，不要混用版本控制） |
| 远端 | <https://github.com/RyanZhou416/Groma>（**PUBLIC**），默认分支 `main` |
| 许可 | `MIT OR Apache-2.0`（自研代码） |
| 当前阶段 | 仅文档，**无源码、无 crate、无 CI、无发布** |
| `groma-*` crate 名 | 2026-09-16 核查**全部在 crates.io 空闲**，但**未占名**（未发布任何 crate） |

---

## 6 立即要做的：P1（合同冻结）

**目标**：冻结 provider 合同，并用两个后端证明抽象正确。

**交付物**

1. `Crates/groma-core`
   - provider trait **按算法族**分组（`Digest`／`Mac`／`Kdf`／`Aead`／`Signer`／`Verifier`／`Kem`／`PasswordHash`／`Random`），**不要一个算法一个 trait**
   - 规范化类型：**封闭算法枚举**、key／signature／nonce／tag 等 newtype
   - 分类错误枚举：**无效签名 ≠ 无效编码 ≠ 不支持算法 ≠ 后端故障**
   - 能力查询（backend 可声明自己接受哪些算法）
   - `no_std` ＋ `alloc`
2. `Crates/groma-codec`
   - **有界** CBOR／COSE／DER 解析（大小、深度、元素数上限先于解析）
   - `no_std` ＋ `alloc`
3. **两个后端**（第二个即为抽象正确性的证明）
   - `Crates/groma-rustcrypto`：第一生产候选
   - 第二个后端：可先只实现 SHA-256（甚至一个最小 in-tree stub），**关键是证明"换后端不改合同"**
4. 验证与工具
   - 编译期测试：公开 API **不出现任何后端私有类型**
   - 两后端在同一向量集上结论一致
   - `Tools/dependency-audit`：断言交付路径**不含 C 构建要求**（检测 `links=`、`*-sys`、`cc`／`cmake`／`bindgen` 构建依赖）
   - `Fuzz/` 三目标：`cose_key`、`cbor_bounded`、`der_signature`

**出口判据（机器可判定）**

- [ ] 公开 API 无后端类型泄漏（编译期测试）
- [ ] 第三方可在**不改 `groma-core`** 的前提下新增后端
- [ ] 两个后端在共享向量集上结论一致
- [ ] COSE/CBOR/DER 解析对超大、超深、重复键、非规范输入不崩溃、不越界、不无界分配

**P1 必须串行。合同错了全盘返工。**

---

## 7 必须遵守的硬约束

> **状态（2026-09-16）**：本清单已经四线调研（成熟项目实践／标准框架／事故复盘／工具现状）系统验证并按用户逐条拍板修订，详见 [`CONSTRAINTS-REVIEW.md`](CONSTRAINTS-REVIEW.md)。共十二章：原九条修订＋三条新增（恒定时间／依赖治理／密钥生命周期）。

1. **不重写已有纯 Rust 算法**：引用生态实现（RustCrypto／graviola／libcrux），不自研算法集（白名单见第 11 章）。
2. **参考加速、重写实现**：实现可参考任何语言的开源实现加速开发（AI 跨语言重写为常态），但必须：①保持 Groma 自有架构 ②照规范＋**外部**公开向量＋负向用例独立验证——参考来源不构成正确性证据 ③逐文件记录参考来源＋许可审查，**copyleft（GPL/MPL/AGPL）来源禁止翻译采用**（双许可防污染）④输出是真正的 Rust 重写而非机械转译。
3. **不自研椭圆曲线／RSA 大数／哈希／随机数**：并入第 11 章白名单治理。
4. **不可信解析入口先限长再解析**：失败关闭；错误可诊断但**与秘密脱钩**；不记录密钥材料或秘密。
5. **宣称纪律**：不得未经独立审计就宣称"生产安全""已审计""可替代 OpenSSL"；形式化验证只按其真实覆盖表述；行业措辞模板（BoringSSL／aws-lc 的 FIPS.md）作附录。
6. **FIPS 140-3 显式非目标**：不得暗示合规；CI 禁词扫描对外文案（"FIPS validated／认证／compliant"）。
7. **正确性只能由独立证据证明**：①公开向量须来自**独立来源**（NIST/CAVP 文件、Wycheproof、CCTV 跨实现交叉、第三方实现）；自产向量不得作为正确性证据（自测/往返健全性检查不禁）②负向用例须含协议／状态机负向与规范超限拒绝（P1＝CBOR/COSE/DER 重复键/深度/尺寸/非规范编码，协议状态机至 P4 全面铺开）③每个公开密码学 API ≥1 正向＋≥1 负向（按家族语义），CI 覆盖率兜底 ④不可信解析入口带 cargo-fuzz 目标，自建 CI（ubuntu＋nightly）⑤截图／手工登录／单浏览器演示不是安全门。
8. **官方全交付零 C**：官方所有 crate 依赖闭包无 C 构建（`links`／cc／cmake 机器门禁）；不发 ring／aws-lc-rs 适配器；FIPS 消费者经开放合同自接适配器。
9. **范围变更必须先改 `SCOPE.md`**，不能靠"顺手加个算法"扩范围。
10. **恒定时间（N1）**：Groma 自写代码中所有秘密数据路径恒定时间（tag/MAC 比较用 subtle 原语、禁秘密相关除法）；dudect 门禁（逐步引入 crabgrind）；后端内部 CT 由后端自身审计覆盖并在第 11 章留痕。
11. **密码学依赖治理（N2）**：白名单制＋版本锁定＋升级 SLA；未修公告（如 `rsa` Marvin `patched=[]`）按**操作**裁决（私钥操作禁用或替代，公钥验签允许并留痕）；逐依赖记录审查边界与公告裁决。
12. **密钥生命周期（N6）**：秘密类型 zeroize-on-drop；禁 Debug/Display 输出秘密；密钥生成包装带自检向量（ROCA 类检测）；零化路径写测试。

---

## 8 待用户拍板的事项（接手前或 P1 期间需确认）

| # | 事项 | 说明 | 建议 |
|---|------|------|------|
| A1 | `LICENSE-APACHE` 目前**不是 Apache-2.0 全文**（为免大段法律文本重复，写了规范头＋指向 ASF 官方文本＋双许可理由） | 导致 GitHub 识别 licence 为 `Other` 而非 `Apache-2.0` | ✅ **已决**：保持 MIT OR Apache-2.0；`LICENSE-APACHE` 已替换为逐字全文，版权署名维持 "2026 Groma contributors" |
| A2 | `groma-*` 命名**未在 crates.io 占名** | 仓库已公开，存在被抢注风险 | ✅ **已决**：首次发布前再做（发布真实 crate 时自然占名，不发 pointer crate） |
| A3 | 是否引入 CI | 当前无任何 CI | ✅ **已决**：已加发布级 CI（`.github/workflows/ci.yml`：fmt／clippy／test 矩阵［ubuntu＋windows × 1.98.1＋1.89.0 MSRV］／no_std 交叉检查／cargo-deny 禁 C 依赖＋cargo-audit／rustdoc／最小版本／覆盖率／tag 触发 semver-checks＋publish dry-run）＋dependabot |
| A4 | 第二个后端选谁 | `SCOPE.md` §D6 列为待决 | ✅ **已决（2026-09-16）**：P1 用最小 in-tree stub（`groma-stub`，自写 SHA-256，差分专用）；graviola/libcrux 留到 P3/P4 |
| A5 | `no_std` 口径（D1）／trait 动态性（D2）／算法枚举扩展方式（D3）／后端选择机制（D4） | 见 [`DESIGN.md`](DESIGN.md) §2 | P1 必须定，晚了改不动；**D1 已决：方案 (a) 合同层 `no_std`+`alloc`、后端不限**；**D2 已决：方案 (d) 单层对象安全合同（工厂化构造）**；**D3 已决：方案 (a) 封闭 `#[non_exhaustive]` 枚举**；**D4 已决：方案 (e) 纯构造注入（合同零选择机制）** |
| A6 | "做到 OpenSSL 的水平"的目标声明与生态位 | 用户 2026-09-16 提出：功能覆盖度对标＋有现成纯 Rust 实现则引用 | 🔶 部分已决：生态位＝方案 A（集成供给层），表述已按开发者视角调整为**双锚点**（获客=无 C 稀缺件，留存=统一合同），见 [`POSITIONING.md`](POSITIONING.md) §0/§1.1；目标声明（[`GOALS.md`](GOALS.md)）与阶段顺序、开发内容、并入 SCOPE 时机待全貌后定 |

---

## 9 已知坑（接手前必读，省几轮返工）

1. **"探测失败"≠"不存在"**：本轮实测出传输层 SSL 失败导致 7 个 crate（含 `argon2`／`spake2`／`sm9`／`xts-mode`）被**误判为不存在**。负结论必须多源复核。
2. **命名冲突会骗人**：`present`＝markdown 工具、`simon`＝参数解析、`seed`＝WASM 框架、`falcon`＝二进制分析、`lms`＝文件同步、`sunshine`＝光线投射引擎、`yarrow`＝音频 GUI。**按名字断言"不存在"会出错。**
3. **未发布的 crate 搜不到**：`rustls-libcrux` 实际是 workspace 成员 `rustls-libcrux-provider`，按项目名搜 crates.io 是空的。
4. **`rsa` 有未修补公告**：RUSTSEC-2023-0071（Marvin）`patched = []`，**全部版本受影响**，crypto-bigint 迁移未修复。落在 RS256 唯一纯 Rust 路径上，必须显式记录与裁决，不能藏。
5. **`aws-lc-fips-sys` 当前绑 FIPS 4.x＝"实验室测完、NIST 待批"**；**已拿证的是 3.0.x**（#5314，需 pin `aws-lc-rs <1.18.0`）。FIPS 文档里不要写错。
6. **`craton-hsm` 的 PKCS#11 ABI 是纯 Rust，但 RSA 私钥操作默认不可用**（需 C `aws-lc`）。"有纯 Rust PKCS#11 令牌"这句话要加限定。
7. **`libcrux` 的 RSA 只有 PSS，没有 PKCS#1 v1.5**，即**不覆盖 RS256**。
8. **`sequoia-openpgp` 默认 `crypto-nettle`（C）**；rpgp 的 crate 名是 **`pgp`**，不是 `rpgp`。
9. **PowerShell 大小写陷阱**（本仓库在 Windows 上维护）：`-eq`／`-ne`／`Select-String` **默认不区分大小写**，做大小写敏感的替换／比较必须用 `-ceq`／`-cne`／`-CaseSensitive`；`Get-ChildItem -Recurse` **默认跳过隐藏文件**（`.gitignore` 等），需 `-Force`，或直接用 `git ls-files` 枚举。

---

## 10 与玩家服务的边界（R4／R5 的具体化）

| 玩家服务需要 | 层 | 何时可用 | 归谁 |
|---|---|---|---|
| Ed25519 签发材料签名 | L1＋L3 | **P2** | Groma |
| 口令哈希（Argon2） | L3 | **P2** | Groma |
| 会话／字段加密 AEAD | L3 | **P2** | Groma |
| 密钥序列化（PKCS#8／SPKI） | L4 | P3 | Groma |
| TLS 密码学后端 | L7 | P4 | Groma（适配器） |
| PKCS#12 密钥库文件 | L4 | P5 | Groma |
| OCSP 与信任路径 | L5 | P5 | Groma |
| 证明原语（TPM 结构等） | L6 | P6 | Groma |
| **SNSM 签发材料格式** | — | — | **玩家服务** |
| **`NetworkIdentityAssertion` 线格式** | — | — | **玩家服务** |
| **账号／会话／刷新家族模型** | — | — | **玩家服务** |
| **WebAuthn ceremony 与政策（challenge／origin／RP ID）** | — | — | **玩家服务** |
| **Steam／微信平台 SDK 接入** | — | — | **玩家服务** |

---

## 11 接手确认清单

- [ ] 已读 [`SCOPE.md`](SCOPE.md)（权威）＋ [`DESIGN.md`](DESIGN.md)（设计取向与待决 D1–D7）
- [ ] 已读 [`research/ecosystem-gaps.md`](research/ecosystem-gaps.md) 与 [`research/coverage-union.md`](research/coverage-union.md)
- [ ] 已确认 §3 的范围裁决与 §7 的硬约束
- [ ] 已确认 §8 的待拍板事项，并向用户确认后再动 P1 的接口设计
- [ ] 已确认：本仓库用 **Git**（非 P4），远端为公开仓库
- [ ] 已确认：crate 目录名小写是工具链硬约束，不会"顺手改成大写"
- [ ] 已确认：P1 出口判据可机器判定，且保住 R4 的 P2 排期承诺

---

## 修订历史

| 日期 | 变更 |
|------|------|
| 2026-09-16 | 创建临时接手需求书：结构化用户权威需求 R1–R6、范围裁决、命名与目录约定、P1 交付与出口判据、硬约束、待拍板事项、已知坑、玩家服务边界、接手确认清单。 |
| 2026-09-16 | 项目正式化：A1 已决（双许可保持，LICENSE-APACHE 换逐字全文）、A2 已决（首发前占名）、A3 已决（发布级 CI 落地）；工具链政策定案（MSRV 1.89，开发工具链 1.98.1，CI 双版本矩阵），写入 `DESIGN.md` §2.1。 |
| 2026-09-16 | 新增待拍板 A6：OpenSSL 对标目标声明（草稿 `GOALS.md`，DRAFT）；推进方式定案为"边做边定"。 |
| 2026-09-16 | §7 硬约束经四线调研系统验证并按用户拍板改写为十二章终版（#2 参考加速四护栏、#7 独立证据四强化、#8 官方零 C、新增 N1 恒定时间／N2 依赖治理／N6 密钥生命周期）；验证过程与证据见 `CONSTRAINTS-REVIEW.md`。 |
