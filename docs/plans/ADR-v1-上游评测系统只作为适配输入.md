# ADR-v1: 上游评测系统只作为适配输入

## Status
Accepted

## Date
2026-04-25

## Context

EvalShield v1 的核心定位是面向 PR-review benchmark runs 的 benchmark trust layer。它回答的是一次 run 的结果是否可信，而不是如何生成 benchmark、如何执行模型、如何推荐 metric。

One-Eval 这类系统说明 eval 正在从静态脚本演化为 agentic eval workflow：自然语言需求可能经过 query understanding、benchmark selection、dataset mapping、metric recommendation、model inference 和 report generation 等步骤，最终生成 raw score 与 human report。

这类上游系统对 EvalShield 很有价值，因为它们会成为 run artifacts 的来源。但如果 EvalShield 直接集成这些 runner 或把它们的 report 当作 truth source，项目会滑向通用 eval 平台，破坏 v1 的 PR-review run trust 边界。

## Decision

EvalShield v1 只把 One-Eval、lm-eval-harness、OpenCompass、内部 PR-review harness 等系统视为 upstream eval system。

上游系统只能通过 adapter 输入 EvalShield canonical contracts。Adapter 负责 artifact validation、source digest、canonical mapping 和 provenance refs，不负责执行 benchmark、推荐 metric、选择 benchmark 或决定 `TrustVerdict`。

上游 workflow provenance 可以记录在 `RunArtifact.provenance` 中，例如：

- `upstream_runner_name`
- `upstream_runner_version`
- `upstream_workflow_trace_ref`
- `upstream_workflow_trace_digest`
- `upstream_schema_mapping_ref`
- `upstream_metric_plan_ref`
- `upstream_human_overrides_ref`
- `upstream_report_ref`

Upstream provenance 风险属于 `system` family 下的 provenance completeness / artifact integrity 风险。它可以使用 `upstream_*` reason codes，但不是 v1 的第四类 trust-failure family。

`TrustVerdict` 必须继续来自 canonical `RunArtifact`、structured `DetectorFinding` 和 rule-based verdict policy。LLM 可以辅助 human report projection，但不能决定 `TrustVerdict`。

## Alternatives Considered

### 直接集成 One-Eval 作为 v1 runner

- Pros: 可以快速展示一个完整 eval workflow。
- Cons: 会把 EvalShield 从 trust layer 推向 eval runner；引入外部 runtime / workflow 复杂度；阻塞 PR-review 最小闭环。
- Rejected: v1 必须先完成 single-run trust chain，不能依赖 One-Eval 完整集成。

### 把 One-Eval report 当作 truth source

- Pros: 接入成本低，用户容易理解。
- Cons: report 是展示投影，通常无法完整表达 raw artifacts、node trace、人工 override 和 score 计算路径。
- Rejected: EvalShield 的 truth source 必须是 canonical bundle，不是上游 report。

### 把 upstream provenance 建成第四类 detector family

- Pros: 命名直观，和 leakage / exploit / replay 并列。
- Cons: 会和项目宪法中 v1 三类 trust-failure 冲突；容易把上游 workflow 风险泛化成通用 eval 平台能力。
- Rejected: upstream provenance 风险归入 `system` family，使用 `upstream_*` reason codes。

### 不记录上游 workflow provenance

- Pros: v1 contract 更小。
- Cons: 对 agentic eval workflow 缺少审计能力，无法解释 metric drift、human override 或 report-only 风险。
- Rejected: 预留 provenance refs 是 additive contract extension，且不要求 v1 实现完整 One-Eval adapter。

## Consequences

- v1 保持 PR-review benchmark run trust layer 定位。
- One-Eval 仍是重要设计参照，但不是 v1 硬依赖。
- `RunArtifact.provenance` 会包含上游 workflow refs。
- `upstream_*` reason codes 属于 `system` family。
- 缺失 raw evidence、缺失 workflow trace、未审计 human override 可以影响 verdict，但不会成为第四条主判定轴。
- 未来可以实现 One-Eval output adapter，但它不能改变 EvalShield 的 truth source 边界。
