- **标题：** StepFinder: A Temporal Semantic Framework for Failure Attribution in Multi-Agent Systems
- **会议：** ACM SIGKDD 2026
- **研究任务：** Multi-Agent Systems Failure Attribution
- **核心目标：** 从失败的多智能体执行轨迹中定位导致任务失败的根因步骤。
## Contribution
1. **将失败归因重新表述为结构化时序建模问题**We reformulate MAS failure attribution as a structured temporal modeling problem. This perspective allows us to shift the heavy reasoning burden from LLMs to a specialized deep learning architecture.
2. **出 StepFinder 时序语义归因框架**We propose StepFinder, a lightweight temporal semantic framework designed to identify the root cause step by assigning an explicit error score to each step. 
3. Experiments demonstrate that StepFinder achieves superior attribution precision and ranking quality while significantly reducing inference l
## Introduction
MAS 虽然具备较强的协作能力，但内部存在明显的脆弱性（Internal Fragility）。
在代表性基准测试中，MAS 的失败率可以达到 41%–86.7%。
- **Error Occurrence**：错误最初发生的位置。
- **Error Propagation**：错误向后续步骤传播。
- **Failure Manifestation**：错误最终表现为任务失败。
已有 LLM 自动诊断方法虽然取得了一定进展，但通常只能取得较好的 Agent-level 诊断效果，缺乏足够准确的 Step-level 定位能力。
**第一阶段：Trajectory Encoding**
将每个执行步骤中的两类信息编码：$x_t=(c_t,r_t)$
其中：
- $c_t$：Agent 在当前步骤产生的执行内容；
- $r_t$：执行该步骤的 Agent 身份。
然后通过 Embedding 模型得到语义向量。
**第二阶段：Temporal and Interaction Modeling**
作者采用混合结构：
- BiLSTM：提取执行轨迹中的时序依赖；
- Agent-Aware Attention：建模不同步骤之间的交互关系；
- Agent-Aware Gating：利用整体 Agent 信息调节特征。
**第三阶段：Step-Level Error Scoring**
给每个步骤计算错误分数，再通过：
- Multi-Scale Differencing；
- Position Bias；
调整最终得分
## Related Work
**2.1：LLM-based MAS**
**2.2：Fragility and Error Patterns in MAS**

|错误类别|含义|示例|
|---|---|---|
|Cognitive and Reasoning Mistakes|认知与推理错误|数学推理错误、事实判断错误|
|Interaction and Collaboration Conflicts|交互与协作冲突|Agent 间信息不一致、协作冲突|
|Planning and Execution Deviations|规划与执行偏差|规划错误、工具执行偏离|
|System and Validation Failures|系统与验证失败|系统故障、结果验证失败|
**2.3：Failure Attribution for MAS**
1. LLM-based Diagnostic Methods
2. Decoupled Semantic Extraction and Diagnostic Logic

| 方法      | 主要机制          |
| ------- | ------------- |
| FAMAS   | 多次执行 + 统计谱分析  |
| CDC-MAS | 因果推断 + 依赖关系建模 |
## Preliminaries
轨迹：$\tau=\langle s_0,a_0,s_1,a_1,\ldots,s_T\rangle$，系统执行任务的过程被表示为一串交替出现的状态与动作。
## Methodology
(1)轨迹编码 Trajectory Encoding. We encode both the content and the agent of each step in MAS trajectories into embeddings, thereby constructing the entire execution trajectory as a temporal semantic sequence for subsequent model processing. 
(2) 时序特征与跨步骤交互建模Model Architecture. To capture both temporal dependencies and cross-step interactions, we design a network architecture that combines temporal modeling and agent-aware interaction modeling to learn representations of the temporal semantic sequences. For each step, an initial error score is computed, which is then adjusted using multi-scale differences and position bias to obtain the final anomaly score. 
(3)模型训练目标 Loss Function. The model is primarily trained with a supervised classification loss, supplemented by a self-supervised auxiliary loss that predicts the embedding of each step. This enhances trajectory pattern representation and improves the stability and generalization of the failure attribution model.
![[StepFinder-1795316006149.webp]]
