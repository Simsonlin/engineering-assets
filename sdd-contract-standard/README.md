# SDD + Contract 工程规范（sdd-contract-standard）

一套把 **Spec-Driven Development（规范驱动开发）** 与 **Contract-First Verification（合同优先验证）**
结合的工程规范，面向长期演进、分阶段授权、强控制与可追溯要求的项目。

## 内容

| 文档 | 作用 |
|---|---|
| [sdd-contract-standard.md](./sdd-contract-standard.md) | 规范本体：核心原则、五层架构、目录模板、合同/验收/门禁/spec 层细则、换代流程、项目间取舍 |
| [background-and-rationale.md](./background-and-rationale.md) | 背景与过程问题：一份真实 cutover（SPEC-027 v2→v3 正式切换）暴露的症状、根因、处理过程与新方案推导 |

## 怎么用

1. 新项目：按规范 §4 模板建目录骨架，先定 `contracts/` 的结构合同，再写 spec 决策记录。
2. 已有项目：按 §10"兼容历史"渐进迁移，不要求一步重构。
3. 应用时按项目规模/团队/监管要求调整，取舍点见规范 §9–§10。

## 维护说明

- 本文档是"活文档"：随项目实践更新，不追求一次性完备。
- 规范的授权与迭代本身也应走文档内记录的决策模式（记录 WHY，保持可追溯）。
