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

## 8. 一句话总结

v1 建立 trust truth。
v2 把这套 truth 扩展到 distributed execution。
