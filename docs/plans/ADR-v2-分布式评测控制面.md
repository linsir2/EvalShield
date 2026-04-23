# ADR：为什么 v2 是分布式评测控制面，而不是通用 agent 平台

状态：Accepted
生成时间：2026-04-22。

## 背景

随着 benchmark run 数量增多，单机执行将遇到：

- 排队变长
- replay / rejudge / divergence analysis 成本上升
- worker 故障恢复复杂
- artifact / evidence 管理困难
- run trust 需要扩展到 fleet trust

此时分布式执行成为合理下一步。

但分布式演进很容易滑向另一条危险路径：

- 通用 agent cluster platform
- 通用 multi-agent orchestration
- 泛化 microservice 平台

这会让项目失去原本的独特问题意识。

## 决策

EvalShield v2 不做通用 agent 平台。
EvalShield v2 只做：

> Trust-Aware Distributed Evaluation Plane

即：

- 面向 benchmark runs 的分布式执行控制面
- 重点解决 trust-preserving distributed execution
- 不解决一般 agent runtime 问题

## 原因

### 原因 1：保留问题锋利度

EvalShield 的独特性来自：

- benchmark trust
- replay
- evidence
- verdict

如果转成通用平台，这个问题会被稀释。

### 原因 2：避免平台化空转

“做大平台”对学生项目风险极高：

- scope 大
- 真问题弱
- 面试追问难支撑

把 v2 定义成分布式评测控制面，问题仍然具体。

### 原因 3：继承 v1 真相系统

v1 已定义：

- canonical bundle
- trust verdict
- replay bundle
- case record

v2 的价值是扩展执行层，不是重写 truth layer。

### 原因 4：技术复杂度可控

如果一开始做通用平台，会过早引入：

- actor vs 微服务大战
- 泛状态管理
- 任意 workflow 编排
- 任意 worker 模型
- 泛协议兼容

这些都不是 v2 当前要解决的问题。

## 后果

### 正向后果

- v2 仍然聚焦 benchmark trust
- 分布式能力有真实问题驱动
- 项目叙事连续
- 面试时更容易讲清 tradeoff

### 负向后果

- v2 不会看起来像“通用超大平台”
- 无法被包装成任意 ACO 产品
- 需要主动放弃一部分泛化野心

## 不接受的替代方案

### 方案 A：直接做通用 agent cluster platform

拒绝原因：

- scope 太大
- 问题不够尖
- 极易模板化
- 看起来大，但站不住

### 方案 B：actor runtime 为核心

拒绝原因：

- 过早进入 runtime research
- 偏离 benchmark trust 主线
- 个人项目负担过大

### 方案 C：hosted dashboard-first

拒绝原因：

- 把产品表面放在系统问题前面
- 容易掩盖真正架构价值

## 实施原则

如果未来启动 v2，实现必须遵守：

1. truth layer 继续沿用 v1
2. distributed execution 不改写 canonical contracts
3. replay 在 v2 中仍先以 structural replay 为主
4. 调度先做 classic scheduler，不做 LLM scheduler
5. 先逻辑分层，再物理拆服务

## 一句话结论

EvalShield v2 的价值不在“我们也做了分布式”。
而在：

> 我们把 benchmark trust 问题扩展到了 distributed execution 场景，而且没有把项目做成空泛平台。
