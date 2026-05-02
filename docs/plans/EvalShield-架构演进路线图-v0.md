# EvalShield 架构演进路线图 v0

生成时间：2026-04-30（修订：2026-04-30，去掉 L 编号，统一用描述性名称）。
适用范围：EvalShield 长期演进方向。
权威边界：v1 项目宪法仍以 `EvalShield-项目宪法-v1.md` 为准，本文件不覆盖 v1 冻结边界。

## 1. 为什么需要这份路线图

v1 只解决 "这次 run 有没有作弊"（Run Integrity）。
但 LLM 评测的信任问题不止这一层。

2026 年 3 月，斯坦福李飞飞团队发表《MIRAGE: The Illusion of Visual Understanding》，发现：
多模态模型在**没有图像输入**的情况下，仍然能在 MMMU-Pro 等视觉 benchmark 上拿到有图时 70%-80% 的准确率。
一个 3B 参数的纯文本模型在胸部 X 光问答上超过了所有前沿多模态模型和放射科医生。

这说明：即使一次 run 没有作弊（EvalShield v1 判为 trusted），benchmark 本身可能在测错误的东西。

本文件定义 EvalShield 从 Run Integrity 向上扩展到完整信任链的演进路径。

## 2. 双轨版本线

EvalShield 的演进沿两个正交维度展开：

```
执行层（Execution Layer）—— ADR-v2 定义
  v1   → 单机单次 run
  v1.1 → 增强
  v2   → 分布式执行

信任层（Trust Layer）—— 本路线图定义，用描述性名称
  Run Integrity         → 这次 run 有没有作弊（v1 已覆盖）
  Output Calibration    → 模型的置信度是否准确
  Benchmark Validity    → 题目是否测了它声称要测的能力
```

两个维度正交：
- Output Calibration 不依赖分布式执行（可以在 v1 上做）
- v2 不自动带来 Output Calibration（分布式不等于校准）
- Benchmark Validity 不依赖其他信任层（可以独立做）

### 实施顺序用 Phase 表示

实施顺序用 Phase 1/2/3 表示，不跟架构层的逻辑位置混用：

```
Phase 1 → Run Integrity（v1 已覆盖）
Phase 2 → Output Calibration（v1 稳定后开始）
Phase 3 → Benchmark Validity（Phase 2 完成后）
```

Phase 编号只表达 "什么时候做"，不表达 "哪个更重要" 或 "哪个更基础"。

## 3. 两种心智模型

四层信任链有两种读法，都是对的，回答不同的问题：

### 逻辑依赖金字塔（"什么必须为真"）

从下往上读。底层是上层的前提：

```
        ┌─────────────────────┐
        │  Benchmark Validity │ ← 如果题目无效，上面全没意义
        ├─────────────────────┤
        │  Run Integrity      │ ← 如果考试作弊，分数不可信
        ├─────────────────────┤
        │  Output Calibration │ ← 如果置信度失准，解读无效
        ├─────────────────────┤
        │  Raw Scores         │ ← 原始分数，需要上面三层才有意义
        └─────────────────────┘
```

### 评测流水线（"系统按什么顺序运行"）

从下往上读。底层是输入，顶层是最终验证：

```
        ┌─────────────────────┐
        │  Benchmark Validity │ ← 最后验证：题目本身有没有效
        ├─────────────────────┤
        │  Run Integrity      │ ← 检查：这次 run 有没有作弊
        ├─────────────────────┤
        │  Output Calibration │ ← 检查：置信度是否校准
        ├─────────────────────┤
        │  Raw Scores         │ ← 输入：评测框架产出的原始分数
        └─────────────────────┘
```

### 为什么流水线顺序跟逻辑依赖顺序不同

Benchmark Validity 在逻辑上是最基础的前提（题目无效则一切无意义），但在流水线中是最后一步，因为它**依赖前面步骤的数据**：

- B-Clean 需要多个模型的 raw scores 才能计算排名不稳定性
- 模态消融需要分别跑有图/无图两轮评测
- VDS 计算需要 Output Calibration 的置信度数据

你必须先有 raw scores，才能做 benchmark validity 检查。这是实际数据依赖，不是纯逻辑依赖。

## 4. 各层详细定义

### 4.1 Raw Scores

现状：所有评测框架都在做这一层。
输出：accuracy、pass rate、F1、BLEU 等原始指标。

问题：原始分数本身不告诉你任何关于可信度的信息。

### 4.2 Output Calibration（输出校准）

核心问题：模型对自己答案的置信度是否准确？

背景（Google Research 2026）：
- 所有 25 个被测 LLM 都存在系统性过度自信
- 人类意见五五开的争议性问题，模型会以 90%+ 的置信度给出单一答案
- 模型声称的 "低冲动性" 与实际行为不一致（自述 vs 揭示行为的 gap）

需要检测的问题：

| 检测项 | 说明 |
|--------|------|
| 置信度过高 | 模型在不确定的问题上给出过高置信度 |
| 分布失真 | 争议性话题上，模型不表达 "人类共识度低" |
| 校准漂移 | 同一模型在不同 prompt 下校准程度差异大 |
| 揭示行为不一致 | 模型声称的能力与实际表现不匹配 |

输出对象（新增）：

```python
class CalibrationReport(BaseModel):
    """单次 run 的输出校准报告。"""
    model_id: str
    benchmark_id: str
    expected_calibration_error: float  # ECE
    overconfidence_rate: float
    per_question_calibration: list[QuestionCalibration]
    contested_topic_flag_rate: float
    rater_count: int
    statistical_power: float

class QuestionCalibration(BaseModel):
    """单题校准结果。"""
    question_id: str
    model_confidence: float
    actual_correct: bool
    human_agreement_rate: float
    is_contested: bool
    calibration_gap: float
```

与 Run Integrity 的关系：
- Run Integrity 的 verdict 是 binary 的（trusted / suspect / invalid）
- Output Calibration 增加一个连续维度：校准分数（calibration score）
- 一个 run 可以是 Run Integrity trusted 但 calibration 低（run 没作弊，但模型过度自信）

### 4.3 Run Integrity（运行完整性）

这就是 EvalShield v1 已经在做的层。

覆盖：
- Leakage Detection
- Exploit Detection
- Replay Mismatch Detection
- System Provenance

详见 `EvalShield-项目宪法-v1.md` 和 `ADR-v2-分布式评测控制面.md`。

### 4.4 Benchmark Validity（评测有效性）

核心问题：这个 benchmark 测的东西是不是它声称要测的？

背景（MIRAGE 2026）：
- MMMU-Pro 声称测 "视觉理解"，但 70%-80% 的题不需要看图就能答对
- 医学 benchmark（VQA-Rad、MicroVQA）的非视觉推理易感性高达 60%-99%
- 一个 3B 纯文本模型在胸部 X 光 QA 上超过了所有前沿多模态模型
- B-Clean 清洗后，MMMU-Pro 75% 的题被剔除，模型排名发生变化

需要检测的问题：

| 检测项 | 说明 |
|--------|------|
| 模态冗余 | 声称需要视觉输入的题目，去掉图像后模型仍能答对 |
| 文本线索泄露 | 题目文本中包含足够回答问题的线索 |
| 统计先验利用 | 模型靠训练数据中的统计模式答题，不是靠理解 |
| 标注员偏差 | 标注员的个人偏好影响了题目和答案 |
| 题目分布偏斜 | 某些子类别题目过多或过少 |

检测方法：

```
模态消融测试（Modality Ablation）：
  对 benchmark 中的每道题：
    有图条件：正常评测，记录准确率
    无图条件：悄悄移除图像，不告知模型，记录准确率
    计算 delta = 有图准确率 - 无图准确率

  delta ≈ 0 的题目 → 标记为 "non-visual"

B-Clean 基准清洗：
  1. 所有候选模型在无图条件下评测
  2. 剔除所有模型无图答对的题目的并集
  3. 用清洗后的子集重新计算排名

Visual Dependency Score：
  VDS = (有图准确率 - 无图准确率) / 有图准确率
  VDS ≈ 0：benchmark 完全不依赖视觉 → 无效
  VDS ≈ 1：benchmark 完全依赖视觉 → 有效
```

输出对象（新增）：

```python
class BenchmarkValidityReport(BaseModel):
    """单个 benchmark 的有效性报告。"""
    benchmark_id: str
    evaluated_models: list[str]
    modality_ablation_results: list[ModalityAblationResult]
    bclean_results: BCleanResult
    visual_dependency_score: float
    non_visual_question_rate: float
    ranking_instability: float
    validity_verdict: Literal["valid", "degraded", "invalid"]
    validity_reasons: list[str]
```

与 Run Integrity 的关系：
- Run Integrity 检测单次 run 的过程可信度
- Benchmark Validity 检测 benchmark 本身的评测有效性
- 两者独立运行，但结果互补：一个 run 可以是 Run Integrity trusted 但 benchmark invalid

## 5. 实施时序

信任层的实施不绑定执行层版本（v1/v1.1/v2）：

```
执行层时间线：
  v1   → 现在
  v1.1 → v1 完成后
  v2   → v1.1 完成后

信任层时间线（Phase）：
  Phase 1: Run Integrity      → v1 已覆盖
  Phase 2: Output Calibration → v1 稳定后即可开始，不等 v2
  Phase 3: Benchmark Validity → Phase 2 完成后

可能的组合：
  v1 + Run Integrity           → 当前目标
  v1 + Output Calibration      → v1 稳定后可做（不需要分布式）
  v2 + Run Integrity           → v2 的默认配置（分布式但不加校准）
  v2 + Output Calibration      → v2 + 校准
  v2 + 全部三层                 → 完整信任链（终极目标）
```

### Phase 2 阶段

#### Phase 2a: 输出校准基础版

范围：
- 对单次 run 的模型输出做置信度分析
- 检测 overconfidence 模式
- 输出 CalibrationReport（简化版）

不做的事：
- 不做 cross-model calibration comparison
- 不做 human agreement ground truth 采集
- 不做实时校准（post-hoc 分析）

#### Phase 2b: 校准完善

范围：
- 完整的 CalibrationReport
- 争议性话题检测
- 与 Run Integrity verdict 联合判定

### Phase 3 阶段

#### Phase 3a: 模态消融

范围：
- 模态消融测试框架（有图 vs 无图）
- 单个 benchmark 的 Visual Dependency Score
- 与 Run Integrity verdict 联合判定

不做的事：
- 不做 B-Clean 自动清洗（需要访问所有候选模型）
- 不做 benchmark 重排名

#### Phase 3b: 完整 Benchmark Validity

范围：
- B-Clean 基准清洗 pipeline
- Benchmark Validity Report
- 完整信任链联合判定
- 跨 benchmark 有效性比较

## 6. 各层之间的关系

```
独立性：
  每一层可以独立运行，不依赖其他层
  Run Integrity 不需要 Output Calibration 就能工作
  Benchmark Validity 不需要 Run Integrity 就能工作

正交性：
  信任层和执行层正交
  Output Calibration 不依赖 v2（分布式）
  v2 不自动带来 Output Calibration

互补性：
  完整信任链联合比任何单层更可靠
  例：一个 run 可能是：
    - Run Integrity trusted（没作弊）
    - 但 calibration low（模型过度自信）
    - 且 benchmark VDS ≈ 0（题目不测视觉理解）
    → 综合判定：分数不可用于 "视觉能力" 的结论

优先级：
  Run Integrity 是 Phase 1 → v1 已覆盖
  Output Calibration 是 Phase 2 → v1 稳定后开始
  Benchmark Validity 是 Phase 3 → Phase 2 完成后做
```

## 7. 与 MIRAGE 论文的关系

MIRAGE 论文的核心贡献：
1. 区分了 "幻觉"（hallucination）和 "海市蜃楼推理"（mirage reasoning）
2. 提出了 Phantom-0 基准（200 道无图视觉问题）
3. 提出了 B-Clean 基准清洗框架
4. 提出了模态消融测试方法

EvalShield Benchmark Validity 直接消费 MIRAGE 的方法论：
- 模态消融测试 → 核心检测方法
- B-Clean → 基准清洗 pipeline
- Visual Dependency Score → 有效性指标

但 EvalShield 超越 MIRAGE 的地方：
- MIRAGE 只做了视觉模态消融 → EvalShield 推广到任意模态（音频、视频、代码）
- MIRAGE 是一次性研究 → EvalShield 是持续运行的自动化 pipeline
- MIRAGE 只输出论文 → EvalShield 输出 machine-readable validity report，可集成到 CI

## 8. 技术依赖

### Output Calibration 依赖

```
必需：
  - 模型输出的 token-level 概率（logprobs）
  - 或模型通过 API 返回的置信度信号
  - 多次采样（同一题跑 N 次，看一致性）

可选：
  - 人类标注员的 ground truth（用于计算 ECE）
  - 争议性话题标注数据集
```

### Benchmark Validity 依赖

```
必需：
  - benchmark 题目的模态可控（能移除图像/音频）
  - 多个候选模型的 API 访问（用于 B-Clean）
  - 评测框架支持无图模式

可选：
  - 人类标注员对 "这道题需不需要看图" 的判断
  - 题目设计者的 intent 标注（"这道题想测什么"）
```

## 9. 对 EvalShield 项目宪法的影响

本路线图**不改变** v1 项目宪法的任何条款。

v1 项目宪法中 "明确不做" 的列表仍然有效：
- 通用 agent security 平台 → 不做
- 通用 benchmark 平台 → 不做
- benchmark authoring studio → 不做

Output Calibration 和 Benchmark Validity 是 EvalShield 信任能力的扩展，不是 v1 scope 的膨胀。
它们会在各自启动时有自己的宪法和 ADR。

## 10. 一句话总结

EvalShield 的终极目标不只是 "这次考试有没有作弊"，而是：

> 这个分数能不能用来做模型比较和路线决策。

要回答这个问题，需要全部信任层通过：
题目出得对（Benchmark Validity）+ 答案置信度准确（Output Calibration）+ 考试没作弊（Run Integrity）+ 分数可靠（Raw Scores）。
