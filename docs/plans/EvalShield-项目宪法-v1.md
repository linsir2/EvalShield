# EvalShield 项目宪法 v1

生成时间：2026-04-21。

## 一句话定义

`EvalShield` 是面向 PR-review-agent benchmark 的 run-trust 基础设施。

它不负责提升 agent 能力。
它负责判断一次 benchmark run 是否足够可信，能否用于工程或研究决策。

## 核心问题

我们不问：

- 这次 agent 分数高不高？

我们问：

- 这次 run 是否被污染？
- 这次 run 是否利用了 harness 结构或隐藏元数据？
- 这次 run 在 replay 下是否仍然成立？
- 这个分数是否足够可信，能用于系统比较和路线决策？

## v1 主楔子

v1 只面向：

- PR review benchmark
- PR-review-agent / code-review-agent 的评测 run

v1 不面向：

- browser / computer-use benchmark
- 通用 coding-agent 任务
- bug-fix benchmark
- multimodal 或 tool-use agent benchmark

## 接口形态

v1 提供：

- CLI
- Python library

v1 的优先级是：

- research-grade trust analysis

而不是：

- product completeness
- hosted product surface

CLI 用于：

- demo
- 离线审计
- casebook 生成
- 简历与展示型 artifact 输出

Python library 用于：

- benchmark harness 集成
- 研究工程师工作流
- 未来的 CI / eval pipeline 接入

## 输入策略

允许两类输入模式，但实现顺序固定：

### Phase 1

- 先做 artifact ingest

### Phase 2

- 再做 live hooks

外部 benchmark 格式永远不是事实真相源。
所有输入都必须先归一化为 EvalShield 的 canonical contracts，然后 detector 才能运行。

## v1 规范范围

v1 只覆盖三类 trust-failure：

### 1. Leakage

- hidden answer exposure
- fixture leakage
- hidden benchmark metadata exposure
- context access outside declared boundary

### 2. Exploit

- harness metadata exploitation
- scaffold shortcut exploitation
- benchmark-structure exploitation
- unintended helper path usage

### 3. Replay mismatch

- recorded run 与 replay 结果偏离
- 无法根据记录证据复现实验
- replay 后 findings 与原 findings 有实质差异

## 最小不可再删能力集

v1 必须只包含以下四个能力块：

### A. Run capture

- ingest run artifacts
- normalize trace、metadata、output、context refs
- 产出 canonical `RunArtifact`

### B. Detectors

- leakage detector family
- exploit detector family
- replay mismatch detector family

### C. Trust engine

- 聚合 findings
- 产出 machine-readable trust verdict
- 附带 reason codes 和 evidence refs

### D. Replay + evidence pack

- replay bundle
- evidence summary
- report projection
- run diff artifact

任何新功能，如果不能直接增强这四块之一，就不属于 v1。

## 输出边界

事实真相源：

- machine-readable verdict

投影视图：

- human-readable trust report

每次审计 run 至少要能输出：

- raw benchmark score
- trust verdict：`trusted | suspect | invalid`
- primary reason codes
- evidence references
- replay bundle
- run diff report

machine verdict 只有在以下条件同时满足时才是权威输出：

- 来自有效 canonical bundle
- 合同版本明确
- detector-set 版本明确
- verdict-policy 版本明确

## 核心契约

v1 围绕以下稳定对象构建：

- `RunArtifact`
- `DetectorFinding`
- `TrustVerdict`
- `EvidenceRef`
- `ReplayBundle`
- `CaseRecord`

这些契约必须先于 detector 实现被定义。
所有 canonical contracts 都必须 versioned + digest-addressed。
所有 detector 输出和 verdict policy 都必须在持久化 artifact 中带版本信息。

## Detector 视野边界

v1 detector 允许查看：

- benchmark task metadata
- PR diff
- repo context references
- run trace
- tool trace
- model output / review output
- replay trace

v1 detector 不允许查看：

- long-term memory
- approval history
- org-wide telemetry
- generic runtime session history
- unrelated external system data

这样可以保证 EvalShield 始终聚焦 benchmark run trust，而不是滑向通用 agent forensics。

## Casebook 模型

EvalShield 生态结构是：

- core repo
- casebook / corpus sidecar

casebook 原子单位：

- 一次 suspect 或 invalid run，附带证据和解释

casebook 必须支持两种视图：

- taxonomy view
  - leakage
  - exploit
  - replay mismatch
- benchmark view
  - 按 benchmark / adapter source 聚合 case

casebook record 必须结构化、带版本、并且能从 `EvidenceRef` pointer 复原。
在 v1 中，`casebook` 是逻辑层名，`cases/` 是磁盘真相目录。

## v1 成功标准

只有当以下内容全部存在时，v1 才算成功：

- CLI
- Python library
- canonical JSON bundle schema
- 一个 PR review benchmark adapter
- 三个 detector family 的最小实现
- machine verdict schema
- human report projection
- replay bundle format
- 至少 3 到 5 个有意义的 casebook case

如果项目只交付 framework 代码、schema 或 dashboard，而没有真实 case，则不算成功。

## 明确不做

以下内容在 v1 中明确不在范围内：

- 通用 agent security 平台
- 通用 benchmark 平台
- benchmark authoring studio
- dashboard / workspace 产品
- browser / computer-use benchmark
- 通用 coding-agent benchmark
- MCP / tool supply-chain scanner
- approval system
- policy engine
- memory poisoning system
- plugin marketplace
- 通用框架兼容层
- auto-remediation engine
- organization-level analytics

## 生态形态

EvalShield 只允许沿四个方向增长：

- capture
- detect
- explain
- integrate

未来可以桥接到：

- CI hooks
- benchmark family expansion
- policy systems
- 更大的 exploit corpus

但这些桥接不会改变核心定义。

EvalShield 不允许漂移成：

- generic security scanning
- generic observability
- benchmark authoring
- runtime orchestration
- agent identity / auth 平台

## 长期愿景

v1 回答 "这次 run 有没有作弊"（Run Integrity）。
长期目标是回答 "这个分数能不能用来做模型比较和路线决策"。
这需要完整信任链：Raw Scores → Output Calibration → Run Integrity → Benchmark Validity。

信任层与执行层（v1/v1.1/v2）正交。信任层用描述性名称，实施顺序用 Phase 1/2/3 表示。
详见 `EvalShield-架构演进路线图-v0.md`。
本文件不改变 v1 的任何边界。

## 反漂移门

任何新增提议都必须回答下面五个问题：

1. 它是否直接提升 run-trust 判断能力？
2. 它是否作用于 PR-review eval runs，而不是更宽对象？
3. 它是否属于 `capture / detect / explain / integrate` 四类之一？
4. 去掉它之后 v1 是否仍然成立？如果成立，就先 defer。
5. 它是否会把项目拉向 scanner / platform / runtime / marketplace？如果会，就拒绝。

如果有两个答案站不稳，这个功能就不能进入 core。

## 工程顺序

实现顺序固定：

1. 定义 canonical contracts
2. 写 contract-spec docs
3. 实现 artifact ingest 和 normalization
4. 实现 detector families
5. 定义 verdict policy matrix
6. 实现 trust engine
7. 定义 replay contract
8. 实现 replay / evidence 输出
9. 实现 casebook 和 curated cases
10. 最后才允许 live hooks 和 CI bridges

禁止事项：

- detector 直接消费外部原始 artifact 格式
- report layer 自己决定 verdict
- casebook entry 没有结构化 evidence refs
- replay engine 在没有 replay contract 前先实现

## 参考架构

```text
                    +---------------------------+
                    |   PR Review Benchmarks    |
                    +-------------+-------------+
                                  |
                                  v
+------------------+   +-------------------------+   +----------------------+
| Eval Integrations|-->|     EvalShield Core     |-->| Trust Artifacts      |
| CLI / library    |   |-------------------------|   | JSON verdict         |
| future CI hooks  |   | run capture             |   | report               |
+------------------+   | detectors               |   | replay bundle        |
                       | trust engine            |   | diff / evidence      |
                       | replay + evidence       |   +----------------------+
                       +-------------+-----------+
                                     |
                    +----------------+----------------+
                    |                                 |
                    v                                 v
         +-------------------------+       +--------------------------+
         | Detector Families       |       | Casebook / Corpus        |
         | leakage / exploit /     |       | case-based archive       |
         | replay mismatch         |       | taxonomy + benchmark view|
         +-------------------------+       +--------------------------+

Peripheral, bridge-only, not core:
- CI gates
- policy bridges
- other benchmark families
- generic agent security systems
```

## 最终规则

只要未来任何改动还可以被一句话准确描述，EvalShield 就是健康的：

`EvalShield` 告诉我们一次 PR-review benchmark run 是否可信，并展示原因。
