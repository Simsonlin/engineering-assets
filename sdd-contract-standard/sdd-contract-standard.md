# SDD + Contract 工程规范

**Spec-Driven Development with Contract-First Verification**

> 版本：1.0（2026-08-19 首稿）
> 状态：活文档。本规范由一次真实 cutover（SPEC-027 v2→v3）的问题提炼而成，
> 背景与过程问题见 [background-and-rationale.md](./background-and-rationale.md)。
> 各项目应用时可按 §9–§10 取舍调整。

---

## 0. 一句话定义

功能演进由**分阶段授权 + 决策记录**驱动（SDD）；"必须满足什么"由**机器可读、单一真相的合同层**承载
（Contract-First）；验收套件与发布门禁全部由合同层**派生**，而不是由按历史 spec 身份组织的手写检查维护。

---

## 1. 适用场景与边界

**适合**：

- 长期演进、多版本更迭的项目（合同会换代）；
- 需要分阶段人类授权、强可追溯性的项目（内部工具、领域规范库、受监管产品）；
- 验收诉求明确、需要"发布前证明满足"的项目。

**不适合（信号）**：

- 一次性原型、纯探索、无验收诉求的脚本；
- 合同层投入大于收益的小型项目（可只取 §9 的轻量变体）。

**允许调整**：本规范是基线，不是教条。取舍点见 §9（项目间变体）与 §10（兼容历史）。

---

## 2. 核心原则

| # | 原则 | 一句话 |
|---|---|---|
| P1 | **真相单点** | 每类"必须满足的事实"只允许一个权威载体（schema / 类型 / 行为测试），禁止复制到文档或多个测试里 |
| P2 | **合同即代码** | 可机器检查的 WHAT 进合同层（结构合同 + 行为合同）；spec 文档只记录 WHY、范围与授权 |
| P3 | **版本化合同** | 合同有 semver 与 `CURRENT` 指针；换代 = 升版本 + 归档旧版，不修改旧版 |
| P4 | **结构性退休** | 归档合同在结构与工具上默认不参与验收与门禁（不是靠"约定不运行"的纪律） |
| P5 | **验收按合同版本组织** | 一份验收套件对应一个合同版本，不按历史 spec 身份组织；换代 = 指向新版本，不是逐个改断言 |
| P6 | **门禁数据驱动** | release gate 读 `gate-manifest.json`（current / required / archived），不硬编码检查列表 |
| P7 | **人类授权门** | 阶段化推进；promote / release 等关键动作必须独立授权；"验收 PASS ≠ 发布授权" |
| P8 | **文档不进 gate** | spec 文档不参与发布判定（最多做链接/拼写卫生检查），避免"文档卫生"与"发布资格"耦合 |

---

## 3. 五层架构

```
specs/          决策记录层（文档，只讲 WHY）      ← 人类授权、决策可追溯
gate/           门禁层（读 gate-manifest）        ← 发布判定
verification/   验收层（按合同版本组织套件）      ← 证明"runtime 符合当前合同"
ux-proto/       运行层（runtime 消费合同）        ← 实现 + 校验
contracts/      合同层（机器可读，唯一真相）      ← 结构合同 + 行为合同
```

- **依赖方向**：上层依赖下层，禁止反向（runtime 不得依赖 verification；spec 不得复制合同的 WHAT）。
- **合同层是唯一真相**：runtime 与 verification 都 import 同一份合同（schema / 类型 / 行为测试），
  谁都不许另写一份。
- **spec 层是唯一授权记录**：gate 与验收不读 spec 文档。

---

## 4. 目录模板（最小骨架）

```
<project>/
  contracts/
    v1/                            # 历史合同版本（结构上退休后默认不运行）
      manifest.schema.json
      envelope.schema.json
      relations.ts
    v2/                            # 当前合同版本
      manifest.schema.json         # 结构合同：JSON Schema（可换成 TS 类型）
      envelope.schema.json         # 例：envelope 的 status/data/error 形状只在这里定义一次
      relations.ts                 # 例：关系类型（assetLink / public-import / entrypoint）
      behavior/
        bootstrap.test.mjs         # 行为合同：语义测试（顺序、失败原子性、安全边界）
        expand.test.mjs
        build.test.mjs
    CURRENT → v2                   # 版本指针（符号链接或 manifest 字段）
  ux-proto/assets/core/            # 运行层：实现 + 校验（import contracts/ 的 schema/类型）
  verification/
    suites/
      current/                     # 当前验收套件：验证 runtime 符合 contracts/CURRENT
        manifest-suite.mjs
        behavior-suite.mjs
      archived/                    # 归档套件：显式登记，默认不运行
  gate/
    gate-manifest.json             # {"current": "v2", "requiredChecks": [...], "archived": [...]}
    check-release-version.mjs      # 读 gate-manifest 驱动，不硬编码列表
  specs/
    001-<name>/                    # 决策记录层
      brief.md                     # 目标、范围、授权阶段（WHY）
      decisions.md                 # 决策与理由（含 supersession 记录）
      tasks.md                     # 阶段与授权门
    # 注意：没有 requirements/acceptance 的副本——它们活在 contracts/ 里
```

**教学示例**（envelope 合同，取自真实教训）：

```jsonc
// contracts/v2/envelope.schema.json —— 唯一真相
{
  "schemaVersion": 2,
  "command": "bootstrap",
  "status": "success | partial | failed",
  "data": { "fullAssets": [...], "linkedAbstracts": [...] },
  "error": { "category": "...", "message": "...", "context": {} }
}
```

runtime 的 CLI 按此 schema 输出；验收套件按此 schema 断言。谁都不许在别处再写
"envelope 长这样"。

---

## 5. 合同层细则

### 5.1 结构合同 vs 行为合同的划分标准

| 类型 | 载体 | 用于 | 例子 |
|---|---|---|---|
| 结构合同 | JSON Schema / TS 类型 | 形状、字段集合、枚举、必填 | manifest 形状、envelope、descriptor、关系类型、版本绑定 |
| 行为合同 | 测试代码（语义测试） | 顺序、失败原子性、安全边界、退出码、副作用 | "partial 必须保留可消费 data"、"失败不得留下 partial authority"、"degraded 是 partial 而非 failed" |

划分经验：**能 schema 化的尽量 schema 化**；行为合同只留 schema 表达不了的部分。
禁止把结构事实写进行为测试（那是重复真相的开始）。

### 5.2 版本化与 `CURRENT` 指针

- 合同跟随 semver：不兼容变更升 major（`v2` → `v3`）。
- `CURRENT` 指针（符号链接或 manifest 字段）是 runtime / 验收 / 门禁读取合同的唯一入口；
  任何代码不得硬编码合同版本号。
- 旧版本目录**不修改**（冻结）；换代只新增新版本 + 移动指针。

### 5.3 禁止写进合同的内容

- 实现细节（内部路径、私有函数、引擎版本）；
- 决策理由（进 decisions.md）；
- 目标与范围（进 brief.md）；
- 任何"当前版本"的快照事实（版本号本身、gitBase、digest——这些进 gate-manifest 或独立身份文件）。

---

## 6. 验收层细则

### 6.1 套件 = 合同版本 × 能力域

- 套件目录按合同版本组织（`suites/current/`），能力域为文件粒度。
- 一份套件的职责：**证明 runtime 符合某个合同版本**。不按"哪个历史 spec 写的"组织。
- 共享断言模块：跨套件重复的断言（envelope 形状、关系类型、版本绑定）抽到
  `verification/shared/`，套件 import，禁止各写一份。

### 6.2 archived 的结构性退休

- 归档 = 把套件移入 `suites/archived/` 并在 `gate-manifest.json` 的 `archived` 登记，
  同时记录归档理由（指向 supersession 决策）。
- 默认不运行：gate 只跑 `requiredChecks`；归档套件保留为历史证据，可手动运行。
- 归档不是删除：文件保留，历史可查。

---

## 7. 门禁层细则

`gate-manifest.json`：

```jsonc
{
  "schemaVersion": 1,
  "current": "v2",
  "requiredChecks": [
    "contracts:CURRENT",          // 结构合同校验
    "suites:current:manifest",
    "suites:current:behavior",
    "identity:pack-git-binding",  // 独立 Git 对象绑定
    "hygiene:docs-links"          // 可选：仅文档卫生，不视为发布资格
  ],
  "archived": [
    { "id": "suites:archived:legacy-v1", "supersededBy": "decisions.md#D-2026-08-19", "reason": "..." }
  ]
}
```

- `check-release-version.mjs` 读 manifest 展开检查列表；新增/归档检查只改 manifest。
- 门禁输出：release report（版本、digest、每个 required check 的通过状态）+ 版本边界结论。
- **版本边界是独立判定**：shipped content 变化时必须回答"版本是否升/能否作为 unpublished candidate"；
  该判定属于 release decision，不由检查脚本自动通过。

---

## 8. spec 层（决策记录）细则

- **brief.md**：目标、范围、授权阶段。回答 WHY。
- **decisions.md**：每个决策的编号、理由、备选与选择。**supersession 是头等模式**：
  新决策可声明"替代 D-nn"，旧条目保留为历史（先例：SPEC-027 的 D26 替代 D2/D4/D7）。
- **tasks.md**：阶段清单与授权门记录（Phase 0–N），每步标注事实与日期。
- **禁止**：spec 文档里写合同 WHAT 的副本（"必须返回 X 字段"这类句子应该只存在于
  contracts/ 的 schema 里）。文档可引用合同路径，不复制内容。
- 文档卫生（断链/拼写）最多作为独立 hygiene 检查，不进发布资格判定（P8）。

---

## 9. 换代流程（标准操作程序）

> 对应 SPEC-027 v2→v3 cutover 的教训模板化：换代不是"改断言"，而是"升版本 + 换指针 + 归档"。

1. **冻结**：当前合同版本目录不再修改；记录当前版本快照。
2. **新建**：`contracts/v<N+1>/`，迁移结构合同与行为合同；不兼容变更明确列出。
3. **换指针**：`CURRENT → v<N+1>`；runtime 与共享断言同步消费新版本。
4. **更新验收**：新建 `suites/current/`（对应新合同）；共享断言升级。
5. **归档**：旧套件移入 `suites/archived/`，在 `gate-manifest.json` 登记 + 写 supersession 决策。
6. **跑门禁**：确认只跑 current 检查，fail-closed 点全部是"当前合同下的真实问题"。
7. **授权**：promote / release 等关键动作由人类独立授权；验收 PASS 记录在 tasks.md，不等于发布授权。

**换代成功判据**：新合同下门禁的每一个 fail 都能对应到"当前现实问题"，
不存在"因为检查脚本是旧版本写的"而产生的失败。

---

## 10. 项目间取舍与变体

| 维度 | 轻量变体 | 标准 | 重量变体 |
|---|---|---|---|
| 规模 | 合并 contracts/ 与 verification/（小项目一份 schema + 一份套件） | 五层 | 多合同包（per 领域/服务） |
| 团队 | 文档精简，决策口头记录 | 决策记录完整 | 决策评审 + 监管留痕 |
| 合同纯度 | 结构合同为主，行为合同最少 | 结构 + 行为 | 行为合同深（安全/合规） |
| 文档参与度 | 不进 gate | 仅 hygiene | 文档即验收（需求追踪矩阵） |
| 授权粒度 | 里程碑授权 | Phase 化授权 | 每 PR 授权门 |
| 兼容历史 | 新项目直接采用 | —— | 现有项目渐进迁移（见下） |

**兼容历史（渐进迁移指南）**：

1. 先抽"最痛"的合同：envelope / 关系类型 / 版本绑定 → 建 `contracts/v1/`，runtime import 它。
2. 把按 spec 编号的验收脚本**重组**为按合同版本的套件（可先只搬 required 部分）。
3. 引入 `gate-manifest.json`，门禁改读 manifest（而不是改 evaluatorPaths 列表）。
4. 旧 spec 文档加 supersession 状态段（保留历史，不再作为活跃合同）。
5. 不要求一次到位；每次换代顺手迁移一步。

---

## 11. 验收清单（应用本规范自检）

- [ ] 每个"必须满足的事实"是否只有一个权威载体？（P1）
- [ ] runtime 与验收是否都 import 同一份合同？（P2/P5）
- [ ] 合同是否版本化且有 `CURRENT` 指针？代码里有没有硬编码版本号？（P3）
- [ ] 归档套件是否在 `gate-manifest.json` 登记并默认不运行？（P4/P6）
- [ ] gate 是否读 manifest，而不是硬编码检查列表？（P6）
- [ ] promote/release 是否有人类独立授权点？验收 PASS 是否没有被当成发布授权？（P7）
- [ ] spec 文档里有没有合同 WHAT 的副本？（P2/P8）
- [ ] 换代时是否走 §9 流程，门禁失败点是否全部是"当前现实问题"？

---

## 12. 术语表

| 术语 | 含义 |
|---|---|
| SDD | Spec-Driven Development：规范驱动开发，功能由分阶段授权的 spec 驱动 |
| Contract-First | 合同优先：可机器检查的 WHAT 以代码形式（schema/类型/行为测试）承载并作为唯一真相 |
| 结构合同 | 用 JSON Schema / TS 类型表达的合同（形状、枚举、必填） |
| 行为合同 | 用测试代码表达的合同（顺序、失败原子性、安全边界） |
| ADR | Architecture Decision Records：决策记录，spec 层 decisions.md 的工程学名称 |
| Contract Testing | 合同测试：按"消费者合同"验证实现的实践（此处借用为内部验收组织方式） |
| Structural Retirement | 结构性退休：归档内容在工具结构上默认不参与验证，而非靠约定 |
| Supersession | 取代：新决策/新版本声明替代旧条目，旧条目保留为历史 |
| Gate-Manifest | 门禁清单：声明 current/required/archived 检查的数据文件 |
| Release Decision | 发布决策：版本边界、shipped-content 确认等人类独立授权点 |
