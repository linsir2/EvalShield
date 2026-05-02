# EvalShield 需求文档 v1

说明：本文件聚焦需求与功能，不重定义架构契约真相。

## 1. 项目名称

EvalShield

## 2. 一句话定义

`EvalShield` 是面向 PR-review benchmark runs 的 benchmark trust layer，负责判断一次 benchmark run 是否值得信，并给出原因与证据。

## 3. 背景

当前多数 agent benchmark 只输出 raw score、pass rate、success rate。团队真正缺的不是更多分数，而是对分数的信任判断。

主要问题：
1. 高分可能来自 benchmark 泄露
2. 高分可能来自 harness / metadata 被利用
3. 高分可能不可复现
4. benchmark 更新后，团队无法区分“agent 变强”和“benchmark 变脏”
5. 结果排查依赖人工读 trace，慢、主观、不可规模化

## 4. 核心问题

EvalShield 要回答的问题是：

> 这次 PR-review benchmark run 的结果，是否足够可信，能拿来比较模型、流程和路线？

## 5. 目标用户

### 主用户
- 做 agent benchmark / eval 的研究工程师

### 辅用户
- 将 reviewer bot 接入 CI / PR 流程的 DevEx / Infra 工程师

## 6. 用户场景

### 场景 1：比较模型
研究工程师跑完一批 PR-review benchmark 后，不只想知道谁分高，还想知道：
- 高分是否来自污染
- replay 后是否仍成立
- 是否利用了 harness 结构

### 场景 2：benchmark 更新后回归
团队更新 benchmark 后，发现分数变化明显，需要判断：
- 是 benchmark 本身变化
- 是 artifact 结构变化
- 还是模型确实变强

### 场景 3：评测流水线集成
团队希望 benchmark run 自动产出“可信度层”，而不是每次人工审计。

## 7. 产品目标

### v1 目标
把单次 PR-review benchmark run 从“只有 raw score”升级成“带 trust verdict 和证据包的评测结果”。

## 8. v1 功能清单

### v1 核心功能
1. Artifact Ingest
2. Canonical Normalization
3. Leakage Detection
4. Exploit Detection
5. Replay Mismatch Detection
6. Trust Verdict Generation
7. Evidence Pack
8. Replay Bundle
9. Case Record Generation
10. CLI
11. Python Library

### v1 支撑能力
- versioned contracts
- digest-addressed artifacts
- reason-code taxonomy
- verdict policy matrix
- evidence pointer spec
- exemplar fixture schema
- golden case tests

## 9. 非功能要求

- 可解释性：所有非 `trusted` verdict 必须可解释
- 一致性：所有 detector 必须只消费 canonical contracts
- 可复核性：结果必须支持 replay 与 evidence 定位
- 结构化输出：verdict、bundle、case record 都必须 machine-readable
- 边界稳定：v1 只服务 PR-review benchmark runs，不引入第二 benchmark family

## 10. v1 非目标

详见 `EvalShield-项目宪法-v1.md`。需求侧简版如下：
- browser/computer-use benchmark
- 通用 benchmark 平台
- 通用 agent security 平台
- hosted service / dashboard
- policy / approval / memory 系统
- MCP scanner
- 第二 benchmark family

## 11. 成功标准

v1 成功的最小判断标准：
1. 对单次 run 输出 raw score + verdict + reasons + evidence + replay bundle
2. 支持至少 1 套 PR-review benchmark adapter
3. 拥有 3-5 个 curated cases
4. 能明确说明为什么这次分数可信 / 不可信

## 12. 长期演进方向

v1 聚焦 Run Integrity。完整信任链还需要：

- Output Calibration — 检测模型置信度是否校准（Phase 2）
- Benchmark Validity — 检测 benchmark 是否在测它声称要测的能力（Phase 3）

信任层与执行层（v1/v1.1/v2）正交，不互相绑定。信任层用描述性名称，实施顺序用 Phase 1/2/3 表示。

详见 `EvalShield-架构演进路线图-v0.md`。
