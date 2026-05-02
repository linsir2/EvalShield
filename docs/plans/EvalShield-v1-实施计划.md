# EvalShield v1 实施计划

说明：本文件只回答三件事：先实现什么、按什么顺序实现、如何验证实现结果。需求范围与非目标以 `EvalShield-需求文档-v1.md` 为准。模块职责边界以 `EvalShield-模块分工与职责.md` 为准。

## 1. 验收标准

### 产品层
1. CLI 能对单次 PR-review benchmark artifact bundle 输出：raw score、verdict、primary reasons、evidence refs、replay bundle 路径、report 路径。
2. Python library 能返回同样的结构化 verdict 对象。
3. 至少实现 1 套 PR-review benchmark adapter。
3a. 只预留上游评测系统 provenance 字段；One-Eval adapter design spike 不阻塞 v1，且不进入 v1 必交付。
4. 至少沉淀 3-5 个 curated cases。

### 工程层
5. 所有 detector 只能消费 canonical `RunArtifact`。
6. 所有 detector 只能输出 canonical `DetectorFinding`。
7. verdict 生成只能走一条 trust-engine 路径。
8. 每个非 `trusted` verdict 至少包含一个结构化 `EvidenceRef`。
9. replay bundle 对 `replay_unavailable` 必须显式处理。
10. curated cases 必须有 golden tests。
11. canonical contracts 必须 versioned 且 digest-addressed。
12. 所有持久化输出都必须包含 `detector_set_version` 与 `verdict_policy_version`。
13. replay mode 必须显式标注为 `structural` 或 `behavioral`。
14. 每个非 `trusted` verdict 至少包含一个可解析的 `EvidenceRef.pointer`。
15. CLI 必须遵守 verdict-aware exit codes。

## 2. 模块级实施承诺

设计阶段只冻结模块，不冻结最终文件名。

- `contracts/`
- `ingest/`
- `detectors/`
- `engine/`
- `replay/`
- `reporting/`
- `casebook/`
- `api/`
- `cli/`
- `cases/`
- `tests/`

## 3. 实施步骤

### Step 1. Contracts
目标：建立 canonical truth objects 与验证规则。
验证：contract tests 通过。

### Step 2. Ingest + Normalization
目标：支持 exemplar -> canonical bundle，并保留上游评测系统 provenance refs。
验证：known-good exemplar 成功，malformed exemplar fail loud；缺失上游 raw evidence 时有明确 ingest note。

### Step 3. Detector Runtime
目标：建立 detector protocol，实现三类 trust-failure family 的最小规则，并支持 system provenance findings。
验证：detector 只接受 canonical `RunArtifact`，输出合法 `DetectorFinding`。

### Step 4. Trust Engine
目标：实现 verdict aggregation、reason 选择、单条审计主路径。
验证：冲突 finding 仍能得到 deterministic verdict。

### Step 5. Replay + Reporting
目标：实现 structural replay、replay bundle、report projection。
验证：replay unavailable 明确可见，report 带 evidence refs。

### Step 6. Casebook
目标：生成结构化 case records，并提供 taxonomy / benchmark 两种视图。
验证：3-5 个 curated cases 稳定产出 verdict + reason codes。

### Step 7. CLI / Python Library
目标：暴露可调用接口。
验证：CLI 有 summary 与路径，Python API 返回结构化对象。

## 4. 测试计划

### Unit Tests
- contract validation and serialization
- detector family behavior on curated mini-fixtures
- reason code ranking / verdict aggregation
- replay divergence formatting
- case record writing / index generation
- evidence pointer parsing / validation
- upstream provenance field validation

### Integration Tests
- external artifact -> normalized `RunArtifact`
- upstream workflow provenance -> canonical refs / ingest notes
- `RunArtifact` -> findings -> verdict
- verdict -> evidence pack / report
- report + replay bundle -> casebook record
- CLI end-to-end against curated fixture
- CLI exit code mapping against verdict categories

### Golden Tests
至少覆盖：
1. leakage case
2. exploit case
3. replay mismatch case
4. mixed-case with conflicting findings
5. clean trusted run
6. upstream report-only suspect run
7. upstream human override unaudited suspect run

## 5. 风险与缓解

### Risk 1. Contract drift
缓解：contracts 先锁，detector 统一 typed against `RunArtifact`。

### Risk 2. Scope drift
缓解：一切新增先过宪法与决策台账。

### Risk 3. Replay 过度承诺
缓解：v1 只承诺 structural replay。

### Risk 4. Casebook 退化成文档堆
缓解：`CaseRecord` 永远是 truth source。

### Risk 5. Detector 虚假信心
缓解：强制 evidence refs、confidence、pointer 解析。

## 6. 并行计划

| Lane | 范围 | 依赖 |
|---|---|---|
| A | contracts + reason codes + package scaffold | — |
| B | artifact ingest + adapter + normalizer | A |
| C | detector implementations | A, B |
| D | trust engine + report + replay bundle | A, C |
| E | casebook + curated cases + golden tests + docs | A, C, D |

## 7. 与启动清单的关系

本文件定义总体实施顺序。
真正开工前，仍需执行 `实现前启动清单-v1.md`。
启动清单负责冻结第一波实现范围、第一波顺序与第一批测试目标。
