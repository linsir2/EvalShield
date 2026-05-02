# EvalShield v1 -> v2 演进边界

生成时间：2026-04-22。
说明：本文件定义 v1、v1.1、v2 的功能分界，防止 v1 被提前平台化。

## 1. v1 是什么

v1 是：

- 单次 PR-review benchmark run 的 trust layer
- offline / artifact-first
- canonical truth chain
- machine verdict + human report
- casebook seed

一句话：

> v1 解决单次 run 是否可信。

## 2. v1.1 是什么

v1.1 仍然留在 v1 边界内，只做增强，不改本质。

允许：

- richer replay diagnostics
- richer exploit corpus
- 更完整 adapter 文档
- upstream provenance diagnostics，不新增第四类 trust-failure family
- 更细错误分类
- 更强 casebook 组织能力

不允许：

- 第二 benchmark family
- 分布式执行
- hosted dashboard
- policy / approval 系统

一句话：

> v1.1 解决“单次 run trust 判断更强”，不是“系统更大”。

## 3. v2 是什么

v2 才进入：

- distributed run execution
- control plane
- worker leases
- retry / fault recovery
- replay workers
- distributed artifact / evidence store
- fleet-level observability

一句话：

> v2 解决大量 run 在集群中执行时是否仍可信。

## 4. v1 和 v2 的根本区别

### v1 核心问题

- 这一次 run 是否可信

### v2 核心问题

- 大量 run 在集群里执行时，整条评测链是否仍可信

### v1 计算模型

- 单机 / 单次 run / artifact-first

### v2 计算模型

- 多 worker / 多 run / distributed evaluation control plane

## 5. 为什么不能把 v2 内容提前塞进 v1

如果把 v2 内容提前塞进 v1，会导致：

1. scope 爆炸
2. 分布式复杂度压过 trust 问题本身
3. 项目从独特问题变成泛平台
4. 学生项目典型失败：技术词很多，但核心问题不清

## 6. v1 输出给 v2 的稳定资产

v2 必须直接继承 v1 的这些真相层：

- canonical bundles
- trust verdicts
- replay bundles
- case records
- reason-code taxonomy
- verdict policy

v2 不能重新发明一套 truth system。

## 7. v2 第一阶段应做什么

v2a 只做：

- 1 个 control plane
- 2 类 worker：
  - execution
  - replay
- 1 个 artifact store
- 1 套 run state machine
- 1 套 retry / lease 机制

先不做：

- actor runtime
- LLM scheduler
- multi-region
- dashboard-first
- 通用平台抽象

## 8. 超越 v2：双轨版本线

v1 和 v2 都只在 "Run Integrity" 层。完整信任链需要往上扩展，但信任层的演进与执行层正交：

```
执行层（Execution Layer）：
  v1   → 单机单次 run
  v1.1 → 增强
  v2   → 分布式执行

信任层（Trust Layer）—— 用描述性名称，不用 L 编号：
  Run Integrity         → 这次 run 有没有作弊（v1 已覆盖）
  Output Calibration    → 模型的置信度是否准确（不依赖分布式）
  Benchmark Validity    → 题目是否测了它声称要测的能力（不依赖分布式）
```

两个维度正交：
- Output Calibration 不依赖 v2（你可以在单机上做置信度校准）
- v2 不自动带来 Output Calibration（分布式不等于校准）
- Benchmark Validity 不依赖 v2 或 Output Calibration（可以独立做）

### 实施顺序用 Phase 表示

```
Phase 1 → Run Integrity（v1 已覆盖）
Phase 2 → Output Calibration（v1 稳定后即可开始，不等 v2）
Phase 3 → Benchmark Validity（Phase 2 完成后）
```

详见 `EvalShield-架构演进路线图-v0.md`。
```

详见 `EvalShield-架构演进路线图-v0.md`。

## 9. 一句话总结

v1 建立 run integrity truth。
v2 把这套 truth 扩展到 distributed execution。
Output Calibration 和 Benchmark Validity 是信任能力的独立扩展，与执行层版本正交。
