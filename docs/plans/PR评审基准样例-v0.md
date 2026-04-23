# PR评审基准样例 v0

生成时间：2026-04-21。
适用范围：EvalShield v1。

## 目的

这份文档定义 EvalShield 在 PR-review benchmark 支持上的第一套 exemplar fixture 形态。

v1 只使用一套 EvalShield-native exemplar fixture。
任何外部 benchmark source，都必须先映射到这套 exemplar，再进入 canonical normalization。

## Exemplar 目录结构

```text
fixtures/pr_review_generic_v0/
  task.json
  run.json
  review.txt
  trace.jsonl
  meta.json
```

## 文件职责

### task.json

包含：

- benchmark/task metadata
- task id
- benchmark name + version
- expected task framing
- 对 PR diff 与 repo context 的引用

### run.json

包含：

- raw score（若存在）
- structured harness outputs（若存在）
- benchmark 侧在本次 run 中产生的 metadata

### review.txt

包含：

- 原始 PR review agent 输出

### trace.jsonl

包含：

- tool/model/harness trace events，按 append-only 顺序记录

### meta.json

包含：

- model identifier
- adapter name + version
- environment fingerprint 相关输入
- artifact source notes

## v1 规则

EvalShield v1 的第一类 benchmark 支持，必须建立在这套 exemplar 上。

任何外部 benchmark 都必须走：

```text
external benchmark output
        ->
adapter
        ->
pr_review_generic_v0 exemplar
        ->
canonical RunArtifact
```

## 为什么必须有这份文档

如果没有固定 exemplar，v1 很快会过早漂移成 generic adapter framework。

这套 exemplar 的作用是：

- 把第一 adapter 压窄
- 让它可测
- 让不同 case 可比较

## 明确不做

- browser benchmark fixtures
- bug-fix benchmark fixtures
- generic multi-family fixture abstraction
