# One-Eval 适配分析 v0

生成时间：2026-04-25。
适用范围：EvalShield v1 设计补充。
文档类型：Explanation。

## 目的

这份文档记录 OpenDCAI/One-Eval 对 EvalShield 的参考价值、适配边界与暂不实施事项。

One-Eval 不改变 EvalShield 的定位。
它作为一种上游 eval workflow runner，帮助 EvalShield 明确未来如何审计 agentic evaluation workflow 的输出可信度。

## 一句话判断

One-Eval 负责把评测需求自动转成可执行评测流程；EvalShield 负责判断一次评测结果是否可信。

两者关系是上下游，不是替代关系：

```text
One-Eval / 其他 eval runner
        -> run artifacts / reports / workflow traces
        -> EvalShield ingest adapter
        -> canonical RunArtifact
        -> trust verdict / evidence pack / replay bundle
```

## One-Eval 的核心流程

根据公开 README 与代码结构，One-Eval 的主流程可以概括为：

```text
Natural language eval request
        -> query understanding
        -> benchmark search / benchmark resolve
        -> dataset structure understanding
        -> metric recommendation
        -> score calculation
        -> report generation
```

其实现倾向于 graph/node/state 风格，把完整评测链路拆成多个可追踪步骤。

## 对 EvalShield 最有价值的设计点

### 1. Agentic eval workflow 正在成为上游事实

One-Eval 说明评测不再只是固定脚本，而可能是一个由 LLM、registry、dataset resolver、metric recommender 和 report generator 共同组成的 workflow。

EvalShield 因此不能只关心最终 score，还要关心 score 生成路径。

### 2. Workflow trace 本身应成为 trust 输入

如果上游评测系统包含 benchmark selection、dataset mapping、metric recommendation、人类审核或 rollback，这些过程都可能影响最终分数。

EvalShield 应把以下内容视为可选但重要的上游证据：

- workflow node sequence
- node input / output digest
- benchmark selection decision
- dataset schema mapping decision
- metric plan / metric recommendation decision
- human interrupt / override events
- rollback events
- final report artifact

### 3. Registry 模式可迁移到 detector 设计

One-Eval 的 metric registry / recommendation 思路，可作为 EvalShield detector registry 的参考。

但两者权威边界不同：

- One-Eval 可以让 LLM 辅助推荐 metric
- EvalShield 不应让 LLM 决定 `TrustVerdict`

EvalShield 可以使用 LLM 生成解释、摘要或 report projection，但 `TrustVerdict` 必须由 structured findings 与 rule-based policy matrix 决定。

### 4. Human-in-the-loop 是信号，不是豁免

One-Eval 支持人工审核、修改或回滚评测流程。

在 EvalShield 中，人工介入不能自动提升可信度。它应该被记录为 provenance signal，并在证据不足时触发 suspect 级 reason code。

## One-Eval 到 RunArtifact 的潜在映射

| One-Eval artifact / concept | EvalShield canonical target | 说明 |
|---|---|---|
| eval request | `input.task_metadata_ref` 或 upstream workflow trace | 记录评测意图，不作为 PR-review truth source |
| benchmark selection | `provenance.upstream_workflow_trace_ref` | 用于判断 benchmark drift / selection bias |
| dataset structure mapping | `provenance.upstream_schema_mapping_ref` | 用于判断 schema mapping 是否可复核 |
| metric recommendation / metric plan | `provenance.upstream_metric_plan_ref` | 用于判断 metric 是否临时漂移 |
| model inference output | `output.raw_output_ref` / `output.review_text_ref` | 进入 canonical output |
| score calculation | `output.raw_score` | score 只是被审计对象，不是 trust verdict |
| report generation | `provenance.artifact_sources` | report 是投影，不是真相源 |
| checkpoint / state trace | `provenance.upstream_workflow_trace_ref` | 用于 digest 与 evidence pointer |
| human interrupt / override | `provenance.upstream_human_overrides_ref` | 用于人工介入审计 |

## 建议的 OneEvalAdapter 边界

v1 可以预留 `OneEvalAdapter` 设计，但不把完整 One-Eval 集成作为 v1 必须项。

最小 adapter 目标：

```text
One-Eval output bundle
        -> validate available artifacts
        -> extract score / output / report refs
        -> extract upstream workflow provenance when available
        -> produce EvalShield canonical RunArtifact
```

`OneEvalAdapter` 不负责：

- 运行 One-Eval
- 生成 benchmark
- 推荐 metric
- 修改 One-Eval workflow
- 把 One-Eval report 当作 EvalShield truth source

## 适合新增的上游风险 reason codes

One-Eval 启发 EvalShield 需要一组 `upstream_*` reason codes：

- `upstream_schema_mapping_unverified`
- `upstream_metric_recommendation_unverified`
- `upstream_human_override_unaudited`
- `upstream_benchmark_selection_drift`
- `upstream_dataset_split_fallback`
- `upstream_report_only_no_raw_evidence`
- `upstream_workflow_trace_missing`

这些 code 不应把 EvalShield 扩展成通用 eval 平台。它们只表示上游评测流程产生的 system-family provenance 风险，不是第四类 trust-failure family。

## 与 PR-review v1 的关系

EvalShield v1 仍然只服务 PR-review benchmark runs。

One-Eval 当前更偏通用 LLM eval workflow，其公开文档中 PR-review / SWE-style / agentic sandbox 能力不是 v1 成熟入口。

因此：

- One-Eval 是重要上游参照
- One-Eval adapter 是合理预留
- PR-review benchmark adapter 仍是 EvalShield v1 的第一实现目标

## 明确不做

- 不把 EvalShield 做成自然语言 eval 生成器
- 不把 EvalShield 做成通用 benchmark runner
- 不在 v1 引入 LangGraph / DataFlow 作为核心依赖
- 不让 LLM 决定 trust verdict
- 不把 One-Eval report 当作 canonical truth source
- 不因为支持 One-Eval 而扩大到所有 benchmark family

## 设计结论

One-Eval 证明 eval 系统正在从静态脚本演化为 agentic workflow。

EvalShield 应抓住这个趋势，成为 agentic eval workflow 后面的 benchmark trust layer：审计 score 生成路径、artifact provenance、workflow trace、人工介入和 replay 证据，而不是重复建设 eval workflow 生成器。
