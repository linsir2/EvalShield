# EvalShield 整体架构设计

说明：本文件回答系统整体怎么工作、核心对象如何流动、为什么这样设计。

## 1. 设计目标

EvalShield 的整体架构必须满足五个目标：

1. 把外部 benchmark 输出收口为统一真相源
2. 把 trust 判断和展示输出分离
3. 让 replay 成为独立能力，而不是散在各处的小逻辑
4. 保留上游 eval workflow provenance，让 score 生成路径可审计
5. 让未来 v2 可以扩展到分布式执行，而不推翻 v1

## 2. 系统总览

```text
外部 Benchmark Run Artifacts
 / 上游 Eval Workflow Outputs
           |
           v
      Artifact Ingest
           |
           v
     Exemplar / Adapter 映射
           |
           v
    Canonical Bundle
           |
    +------+------+----------------+------+
    |      |      |                |
    v      v      v                v
Leakage  Exploit  System          Replay
Detector Detector Provenance      Checks
    \      |      |                /
     \     |      |               /
      \    |      |              /
       \   |      |             /
        v  v      v            v
        Detector Findings
                |
                v
           Trust Engine
                |
       +--------+--------+
       |                 |
       v                 v
  Machine Verdict   Human Report
       |
       v
  Replay Bundle / Case Record
```

## 3. 核心设计原则

### 3.1 Canonical Bundle 是真相源
外部 benchmark 原始格式不可靠，也不稳定。必须先经过：
1. 外部 artifacts -> exemplar
2. exemplar -> canonical bundle

### 3.2 单次 run 是主对象
v1 只面向单次 PR-review benchmark run，不提前引入 cluster / scheduler / workflow bus。

### 3.3 判断与展示分离
- `TrustVerdict` 是 truth object
- report 只是投影
- casebook 只是结构化沉淀，不是 verdict 源头

### 3.4 Replay 是独立链
Replay 必须有自己的输入要求、状态、输出对象，不能散在 ingest / detector / report 中。

### 3.5 上游 workflow provenance 是审计输入
One-Eval 这类系统说明评测可能由多节点 workflow 产生，而不只是静态脚本。

EvalShield 必须允许 adapter 保留 benchmark selection、schema mapping、metric plan、human override 和 workflow trace 的引用。

这些 provenance refs 可以触发 `system` family findings，并使用 `upstream_*` reason codes，但不能直接替代 canonical truth。

## 4. 数据流设计

### 4.1 输入流
`外部 artifacts / upstream workflow outputs -> ingest 校验 -> exemplar / adapter 映射 -> canonical bundle`

### 4.2 检测流
`canonical bundle -> leakage / exploit / replay / system provenance checks -> detector findings`

### 4.3 判定流
`detector findings -> verdict policy matrix -> trust engine -> TrustVerdict`

### 4.4 输出流
`TrustVerdict + EvidenceRefs + ReplayBundle -> Human Report / Case Record`

## 5. 核心对象关系

说明：这里只说明对象在系统中的作用与关系，不重复定义字段。字段级真相以 `标准真相包契约-v0.md` 为准。

- `RunArtifact`：单次 run 的标准化真相包，detector 唯一合法输入
- `DetectorFinding`：结构化发现，detector 唯一合法输出
- `TrustVerdict`：系统级结论，machine-authoritative output
- `EvidenceRef`：证据定位引用，连接 finding、verdict、replay、case record
- `ReplayBundle`：replay 尝试结果对象，不等于 verdict，但会影响 verdict
- `CaseRecord`：案例原子单位，是 casebook 的结构化真相记录
- `upstream provenance refs`：上游 eval workflow 的 trace、schema mapping、metric plan、human override 等引用，只作为审计输入

## 6. 关键 tradeoff

### 6.1 为什么不用外部 benchmark 格式直接做 truth source
如果直接使用外部格式：
- detector 会围外部格式写逻辑
- replay 会跟着外部变化漂
- casebook 会失去稳定结构

所以必须 canonicalize。

### 6.2 为什么 v1 不做 behavioral replay 强承诺
LLM 场景天然有非确定性。v1 一开始就承诺 behavioral replay，会制造高噪声和误伤，因此 v1 只承诺 structural replay。

### 6.3 为什么不把 One-Eval 这类上游 report 当 truth source
上游 report 是展示层，通常无法完整表达 raw artifacts、node trace、人工修改和 score 计算路径。

如果把上游 report 直接当 truth source：
- detector 会被 report 版式绑定
- 缺失 raw evidence 时无法复核
- 人工 override / metric drift 可能被掩盖

所以 One-Eval 等系统只能通过 adapter 进入 canonical bundle。

### 6.4 为什么 v1 不直接做分布式
当前核心问题是单次 run 是否可信，不是多 run 如何调度。过早做分布式会让复杂度压过问题本身。

## 7. v1 和 v2 的架构边界

### v1
- single-run trust chain
- offline artifact-first
- canonical truth objects
- upstream workflow provenance refs
- no distributed scheduler

### v2
- distributed evaluation control plane
- worker leases
- replay workers
- fleet-level execution state

关键点：v2 不重写 v1 truth layer，只在其上增加 distributed execution。

## 8. 失败路径设计

至少显式处理：
- malformed artifact
- unsupported version
- digest mismatch
- detector failure
- replay unavailable
- replay failed
- conflicting findings
- case record 生成失败
- upstream raw evidence missing
- upstream workflow trace missing
- unaudited human override

原则：fail loud，不静默降级，不让坏结果伪装成 trusted。

## 9. 一句话总结

EvalShield 的整体架构，不是为了做大平台，而是为了把“这次 benchmark run 到底能不能信”做成一条清晰、可审计、可演进的系统链路。
