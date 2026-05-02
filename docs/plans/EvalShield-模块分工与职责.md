# EvalShield 模块分工与职责

说明：本文件回答系统分成哪些模块、每个模块负责什么、为什么这样拆。

## 1. 拆分原则

1. 单次 run 是核心对象
2. 真相源必须收口
3. 判断与展示分离
4. 上游评测系统只通过 adapter 进入 canonical contracts
5. 先逻辑分层，后物理拆分

## 2. 模块总览

v1 建议拆成七个核心模块：
- Contracts
- Ingest
- Detectors
- Trust Engine
- Replay
- Reporting
- Casebook

## 3. 模块职责

### 3.1 Contracts
负责：定义核心对象、字段规则、版本规则、digest 规则、上游 workflow provenance 字段。
不负责：业务逻辑、detector 逻辑、verdict 聚合、report 渲染。
复杂度：4 / 5

### 3.2 Ingest
负责：读取外部 benchmark run 产物、校验输入、映射 exemplar / upstream adapter、归一化 canonical bundle、记录 ingest notes 与 provenance refs。
不负责：detector findings、trust verdict、report、casebook。
复杂度：3 / 5

### 3.3 Detectors
负责：对 canonical run 做 leakage / exploit / replay mismatch 检测，并对上游 provenance 完整性输出 system findings。
不负责：最终 verdict、直接改 artifacts、直接生成 casebook。
复杂度：4 / 5

### 3.4 Trust Engine
负责：聚合 findings、应用 verdict policy、生成 `trusted / suspect / invalid`、选择 primary reasons；保证 system provenance 风险不会压过高危 leakage / exploit。
不负责：直接读 raw benchmark 格式、做 detector 细节判断、report 版式。
复杂度：3 / 5

### 3.5 Replay
负责：执行 replay、生成 replay bundle、生成 divergence summary、提供 replay 证据。
不负责：定义 replay 真相边界、最终 verdict、分布式调度。
复杂度：3.5 / 5

### 3.6 Reporting
负责：把 machine truth 投影成人能读的 trust report 与 evidence summary。
不负责：定义 truth、做隐藏规则判断、直接存储 case 真相。
复杂度：2.5 / 5

### 3.7 Casebook
负责：沉淀 suspect / invalid run 案例，提供 taxonomy 与 benchmark 两种视图。
不负责：代替 truth source、定义 contract 规则、决定是否 suspect / invalid。
复杂度：2.5 / 5

## 4. 模块边界总结表

| 模块 | 负责 | 不负责 | 复杂度 |
|---|---|---|---|
| Contracts | truth objects、字段规则、版本规则、upstream provenance 字段 | 检测、聚合、展示 | 4 / 5 |
| Ingest | 输入读取、校验、归一化、adapter provenance refs | detector、verdict、report | 3 / 5 |
| Detectors | 输出 leakage / exploit / replay findings 与 system provenance findings | 决定最终 verdict | 4 / 5 |
| Trust Engine | findings -> verdict | 直接读 raw artifacts | 3 / 5 |
| Replay | replay bundle、divergence | 最终结论 | 3.5 / 5 |
| Reporting | human-readable 输出 | 定义 truth | 2.5 / 5 |
| Casebook | 案例沉淀与视图 | 代替 truth source | 2.5 / 5 |

## 5. v1 先做什么，后做什么

先做：Contracts、Ingest、Detector 接口，包括 upstream provenance 字段预留。
后做：Trust Engine、Replay、Reporting、Casebook。
绝不能一开始就做：分布式调度、hosted service、dashboard、通用平台、One-Eval 完整 runner 集成。

## 6. 未来模块（Phase 2 / Phase 3）

以下模块在 v1 实现期间不创建，属于信任层演进方向：

### Phase 2: Output Calibration
- `calibration/` 模块 — 置信度分析、ECE 计算、overconfidence 检测
- 复杂度预估：3.5 / 5
- 依赖：模型 logprobs 或多次采样一致性

### Phase 3: Benchmark Validity
- `ablation/` 模块 — 模态消融测试框架
- `validity/` 模块 — B-Clean pipeline、VDS 计算、BenchmarkValidityReport
- 复杂度预估：4 / 5
- 依赖：benchmark 题目模态可控、多个模型 API 访问

这些模块与 v1 的七个核心模块独立，不修改 v1 模块边界。

详见 `EvalShield-架构演进路线图-v0.md`。

## 7. 一句话总结

EvalShield 的模块拆分不是为了显得复杂，而是为了把"单次 run 的可信判断"拆成一条清晰、可审计、可演进的系统链路。
