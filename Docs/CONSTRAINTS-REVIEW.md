# Groma 硬约束修订建议书（草稿）

> **文档类型**：constraints-review（调研结论 + 修订提案，待用户拍板）
> **所属模块**：Groma
> **创建日期**：2026-09-16
> **状态**：**DRAFT** —— 经用户逐条拍板后改写 [`HANDOFF.md`](HANDOFF.md) §7 与 [`SCOPE.md`](SCOPE.md) 相关条款
> **证据来源**：2026-09-16 四条调研线（会话内报告）——①成熟项目实践（BoringSSL/Go/rustls/RustCrypto/ring/aws-lc 的一手 CONTRIBUTING/SECURITY/README 对照）②标准与合规框架（CAVP/ACVP、ISO 19790、CMVP IG、ASVS、Wycheproof、SLSA/SSDF/ANSSI）③事故复盘（18 起真实事故逐条查证）④工具与方法现状（OSS-Fuzz/cargo-fuzz/dudect/crabgrind/zeroize/cargo-deny 等 2026-09 现状）

---

## 1 总评（三轴交叉）

九条硬约束是 2026-09-16 立项会话写成、未做系统溯源的经验共识。四线验证结论：**方向基本正确、无一条应删除**；但存在：①两条"绝对化"条款与行业证据冲突（#2 禁止移植、#8 禁 C 的一刀切边界）②一条的致命细节缺失（#7 未写"向量须来自独立来源"，会重蹈自证循环）③**事故轴反推出六条缺口约束**（现有九条拦不住 13/18 起真实事故，时序侧信道 6 起居首）④工具轴证明：可机器执行的必须机器化（#8 最强、#7 主体、#3 白名单），只能流程的如实标注（#5/#6/#9）。

---

## 2 九条逐条裁决

| # | 裁决 | 依据要点 | 修订方向 |
|---|---|---|---|
| 1 不重写已有纯 Rust 算法 | **保留＋精确化** | 共识度高（rustls provider 委托、Go fiat-crypto）；"记录审查边界"无行业成文对应但合理 | 增加依赖白名单机制（与 #3 合并为"密码学依赖治理"新条）；明确与 #8 的边界（白名单内不得含 C 构建项） |
| 2 不移植源码、照规范＋向量实现 | ✅ **已决（2026-09-16）：用户实际政策**——参考加速为常态（AI 跨语言理解＋Rust 重写是常规工作方式），但四条护栏：①保持 Groma 自有架构 ②照规范＋外部公开向量＋负向用例独立验证（参考来源不构成正确性证据，KyberSlash 教训）③逐文件记录参考来源＋许可审查，copyleft（GPL/MPL/AGPL）来源禁止翻译采用（双许可防污染）④参考是理解手段非复制，输出必须是真正的 Rust 重写而非机械转译 | 四条款版见左，替换原"绝对禁令" |
| 3 不自研 EC/RSA 大数/哈希/随机数 | **保留＋精确化** | 共识度高（Go math/big 出安全边界；ASVS 6.2.2 禁止自写密码）；但"crate 审计≠算法验证证书"（比 FIPS 松，如实标注） | 与 #1 合并进"密码学依赖治理"：白名单（elliptic-curves/rsa/sha2/rand_core…）＋禁自制大数（num-bigint 等）＋逐依赖记录审查边界与公告裁决 |
| 4 限长解析/失败关闭/可诊断/不记录秘密 | **保留＋精确化** | 标准支持度最强（ASVS L1 条目直接对应）；rustls SECURITY.md 几乎逐条对应 | "错误可诊断"加限定"诊断信息与秘密脱钩"（呼应 ASVS 7.4.1 通用消息与 D5=(d) 的 Display 纪律）；"不记录秘密"标注为流程条款（无工具） |
| 5 未经审计不得宣称；形式化验证按覆盖表述 | **保留** | 行业是姿态（BoringSSL"not intended for general use"、ring"experiment"）而非条款；Groma 章程化是合理强化 | 附录 BoringSSL/aws-lc 的 FIPS.md 措辞模板；"形式化验证按真实覆盖表述"保留为自律条款并标注"无标准原文" |
| 6 FIPS 非目标、不得暗示合规 | **保留＋工具化** | 行业模板：BoringSSL"整体非 FIPS validated"（仅 BCM 获证）、aws-lc 精确措辞；oxicrypt"holds no NIST certificate"是正面样板 | 附行业措辞模板；加 CI 禁词扫描（"FIPS validated/认证/compliant"出现在对外文案即红） |
| 7 公开向量＋负向用例＋fuzz；演示非安全门 | **保留＋强化（本条最重要）** | 成分均有依据（CAVP/Wycheproof/SSDF/ISO 19790 PCT）；两个失败模式：向量粒度不足（SHAKE 分块、x25519 双向）与缺协议负向（rustls 被审计仍漏两年）；**致命细节：向量必须来自独立来源**（CAVP 刻意由 NIST 生成向量打破自证循环） | 强化为：①向量来源须独立于实现（NIST 文件/Wycheproof/第三方实现交叉）②负向用例须含协议/状态机负向与规范超限拒绝③每个公开密码学 API ≥1 正＋≥1 负（覆盖门禁）④fuzz 自建 cargo-fuzz CI（OSS-Fuzz 对 RustCrypto 覆盖空白，不能指望外部） |
| 8 交付路径无 C/C++ 构建 | **保留＋重新定位** | 无任何标准依据（SLSA/SSDF 不管语言）；行业路线选择（共识度低）；但机器可执行度最高（links 字段＋cc/cmake build-dep 检测＋deny bans）；事故视角：严格执行反而规避 aws-lc-rs CCM 类公告 | 如实标注性质："语言安全策略自选，非行业共识"，但它是**用户的核心诉求**（编译不方便），保留为硬性；边界精确化：约束**合同层与默认交付路径**（groma-core/codec/rustcrypto 后端）；FIPS 需求经"可选后端适配器"满足并显式标注 C 构建要求（与 D1=(a)、D4=(e) 一致） |
| 9 范围变更先改章程 | **保留** | 行业通用做法是"先讨论/提案后实现"（Go proposal、rustls"先开 issue"、BoringSSL API-CONVENTIONS）；Groma 的"先改 SCOPE"是机制化表达 | 保留原样 |

---

## 3 新增约束提案（事故轴反推，工具轴确认可执行）

| 新条 | 依据事故（频率） | 工具支撑（2026-09 现状） |
|---|---|---|
| **N1 恒定时间硬约束** | 时序侧信道 ×6 居首（Marvin/KyberSlash/Minerva/ml-dsa/aws-lc CCM/dalek）——dalek 案例证明**源码 CT ≠ 二进制 CT**（LLVM 优化破坏） | 完整：dudect-bencher 0.7.0（crypto-bigint 已用）、crabgrind（ctgrind 继任，graviola 有 CI）、subtle/ctutils/aarch64-dit 原语、Kani 小规模 CT 证明 |
| **N2 密码学依赖治理** | Marvin（rsa `patched=[]` 落在 RS256 路径）、libcrux 2026 公告潮、"用库≠安全" | cargo-deny bans/duplicates、cargo-audit（RUSTSEC）、cargo-vet；白名单＋版本锁定＋升级 SLA＋未修公告禁用清单（rsa 私钥解密路径等） |
| **N3 状态机与协议负向/差分测试** | rustls 加密层边界 bug 存活两年（同款 Go CVE）、SHAKE 增量输出 | 增量 API 分块组合向量；协议跨状态负向；graviola `crosschecks.rs`＋proptest 差分模板 |
| **N4 规范边界与解析语义测试** | AES-GCM AAD 超限未拒（SP 800-38D）、serde_cbor 重复键静默折叠（签名/规范化绕过） | Wycheproof 风格边界向量；重复键拒绝/规范形式显式定义 |
| **N5 公开 API 测试覆盖门禁** | libcrux x25519 夹紧写反（合法密钥全拒）、poly1305 密钥长度 panic（CVSS 8.7） | 每个公开密码学 API ≥1 正＋≥1 负；CI 覆盖门禁（可并入 #7 强化） |
| **N6 密钥生命周期** | ROCA（密钥生成）、Heartbleed（内存消毒） | zeroize 1.9.0（6.9 亿下载，RustCrypto 事实标准）；秘密类型禁 Debug；密钥生成自检向量（ROCA 检测）；graviola `tests/zeroing.rs` 模板 |

---

## 4 机器执行蓝图（修订后章程的 CI 门禁总表）

| 门禁 | 工具 | 对应条款 |
|---|---|---|
| 无 C 构建检测 | `cargo metadata` 过滤 `links` ＋ build-dep 扫描 ＋ cargo-deny bans(cc/cmake/bindgen/*-sys) | #8 |
| 依赖白名单/公告 | cargo-deny bans＋cargo-audit＋cargo-vet | N2、#1/#3 |
| 向量/负向/覆盖 | wycheproof crate＋KAT＋API 覆盖门禁 | #7、N4、N5 |
| fuzz | 自建 cargo-fuzz CI（arbitrary） | #7 |
| 差分 | 两后端同向量集成测试＋proptest（graviola crosschecks 模板） | P1 出口判据 |
| 恒定时间 | dudect-bencher 门禁＋（逐步）crabgrind CI | N1 |
| 零化 | zeroize 依赖强制＋zeroing 路径测试 | N6 |
| Miri | 纯 Rust 部分 UB 门禁（nightly job） | #4 辅助 |
| 文案禁词 | CI grep（FIPS validated 等） | #6 |

---

## 5 待拍板清单

- [x] #2 降级：**已决**——四条款版（参考加速为常态＋重写实现＋留痕/许可护栏＋独立验证）
- [ ] #8 边界："交付路径"→"合同层与默认交付路径"（可选 FIPS 后端显式标注 C）
- [ ] #7 强化四项（独立来源/协议与边界负向/API 覆盖门禁/自建 fuzz）
- [ ] 新增 N1–N6 是否全收（N5 可并入 #7）
- [ ] 其余（#1/#3 合并入 N2、#4/#5/#6 精确化、#9 保留）按建议直接改
- [ ] 修订后的 HANDOFF §7 与 SCOPE 相关条款改写时机（建议与 GOALS/POSITIONING 并入 SCOPE 同期）

---

## 修订历史

| 日期 | 变更 |
|------|------|
| 2026-09-16 | 创建草稿：四线调研交叉结论、九条逐条裁决、六条新增提案、机器执行蓝图、待拍板清单。 |
