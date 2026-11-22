- **标题：** StepFinder: A Temporal Semantic Framework for Failure Attribution in Multi-Agent Systems
- **会议：** ACM SIGKDD 2026
- **研究任务：** Multi-Agent Systems Failure Attribution
- **核心目标：** 从失败的多智能体执行轨迹中定位导致任务失败的根因步骤。
## Contribution
1. **将失败归因重新表述为结构化时序建模问题**We reformulate MAS failure attribution as a structured temporal modeling problem. This perspective allows us to shift the heavy reasoning burden from LLMs to a specialized deep learning architecture.
2. **出 StepFinder 时序语义归因框架**We propose StepFinder, a lightweight temporal semantic framework designed to identify the root cause step by assigning an explicit error score to each step. 
3. Experiments demonstrate that StepFinder achieves superior attribution precision and ranking quality while significantly reducing inference l
## Introduction
![[StepFinder-1795326025537.webp]]
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
### 4.1 Trajectory Encoding
定义每个执行步骤：$x_t=(c_t,r_t)$
其中：
- $c_t$：第 $t$步的执行内容。
- $r_t$：第 $t$步对应的 Agent 身份. 
这里的 $r_t$不是 Agent 的原始 ID，而是后续通过 Embedding 得到的身份向量。
使用 Qwen3 Embedding 将执行内容和 Agent 身份分别映射到向量空间。
内容编码：$\mathbf c_t=Enc(c_t)\in\mathbb R^{d_c}$
Agent 身份编码：$\mathbf r_t=Enc(r_t)\in\mathbb R^{d_r}$
对于一条长度为$T$ 的轨迹，构建两个特征矩阵：
$C\in\mathbb R^{T\times d_c}$                  $R\in\mathbb R^{T\times d_r}$
因此，整个轨迹被表示为：$X=(C,R)$
论文称其为 Temporal Semantic Sequence。

| 特征                   | 作用                              |
| -------------------- | ------------------------------- |
| Content Embedding$C$ | 主要时序信号（Primary Temporal Signal） |
| Agent Embedding $R$  | 辅助上下文信息（Auxiliary Context）      |
### 4.2 Temporal Feature Extraction
MAS 的执行轨迹具有两个特点：
1. 存在长距离依赖关系。
2. 早期的小偏差可能经过多次 Agent 交互逐渐放大，形成级联失败。
**4.2.2 第一步：Layer Normalization**
首先，对内容 Embedding 矩阵 $C$ 进行 Layer Normalization：$\hat C=LayerNorm(C)=\{\hat c_1,\hat c_2,\ldots,\hat c_T\}$
这里归一化的是内容特征，而不是 Agent 身份特征。
作者认为，这样可以：
- 缓解不同 Embedding 维度之间的数值差异；
- 减轻长序列训练中的梯度不稳定；
- 为后续模型提供更加稳定的输入。
 **4.2.3 第二步：BiLSTM**
作者使用双向 LSTM 处理归一化后的内容特征。
每个时间步的隐藏状态定义为：
$h_t=[\overrightarrow{LSTM}(\hat c_t);\overleftarrow{LSTM}(\hat c_t)]$
这里的分号表示向量拼接。
BiLSTM 包含两个方向。
**Forward LSTM（正向）**
正向 LSTM 按照执行顺序处理：
$\hat c_1\rightarrow\hat c_2\rightarrow\cdots\rightarrow\hat c_T$
在第 $t$ 步，其隐藏状态整合从任务开始到当前步骤的历史信息：
$\overrightarrow h_t=LSTM(\hat c_1,\ldots,\hat c_t)$
**Backward LSTM（反向）**
反向 LSTM 按照相反顺序处理：
$\hat c_T\rightarrow\hat c_{T-1}\rightarrow\cdots\rightarrow\hat c_1$
其隐藏状态能够利用当前步骤之后的上下文：
$\overleftarrow h_t=LSTM(\hat c_T,\ldots,\hat c_t)$
两者拼接后得到 $h_t$。
 **第三步：再次归一化**
所有步骤的隐藏状态组成：$H=\{h_t\}_{t=1}^{T}$
再进行一次 Layer Normalization：$H_{out}=LayerNorm(H)$ 得到最终的时序表示。
### 4.3 Agent-Aware Step Interaction（Agent 感知的步骤交互）
BiLSTM 主要通过链式状态传播获取前后文信息。
但是 MAS 中存在大量非相邻步骤之间的关联。
因此设计 Agent-Aware Step Interaction。
它包含两个主要机制：
1. Agent-Aware Bias；
2. Agent-Aware Gating。
**4.3.2 Global Attention（全局注意力）**
首先将 BiLSTM 输出的特征映射到多头注意力空间。
输入：$H_{out}\in\mathbb R^{T\times d}$
通过三个可学习矩阵得到：$Q=H_{out}W_Q$  $K=H_{out}W_K$     $V=H_{out}W_V$
分别对应：

|符号|含义|作用|
|---|---|---|
|Q|Query|当前步骤需要寻找什么信息|
|K|Key|其他步骤提供什么匹配信息|
|V|Value|其他步骤实际提供什么特征|
随后计算缩放点积交互矩阵：$E=\frac{QK^\top}{\sqrt{d_k}}$
其中：$E\in\mathbb R^{T\times T}$
这个矩阵表达任意两个步骤之间的语义交互分数。
例如，$E_{2,5}$ 表示 Step 2 与 Step 5 在注意力空间中的交互分数。
与 BiLSTM 不同，Attention 不要求两个步骤在时间上相邻。
因此，它可以直接建立长距离步骤之间的关联。
 **4.3.3 Agent-Aware Bias（Agent 身份感知偏置）**
普通 Attention 主要根据 Query 和 Key 的相似程度计算注意力。
作者认为，步骤之间的关系还应该考虑执行它们的 Agent 身份。
因此构建 Agent-Aware Bias 矩阵：$B\in\mathbb R^{T\times T}$
其中：$B_{ij}=\cos(r_i,r_j)$
这里：
- $r_i$：第 $i$步的 Agent Embedding；
- $r_j$：第 $j$ 步的 Agent Embedding；
- $\cos$：余弦相似度。
然后将 Agent 身份偏置加入注意力分数：$Attn=softmax(E+\alpha\cdot B)$
其中 $\alpha$ 用于控制 Agent 身份信息的影响程度。
**4.4 Step-Level Error Scoring（步骤级错误评分）**
前面已经完成了时序特征提取和 Agent 间交互建模。
现在模型需要回答：
**哪一个步骤最可能是 Decisive Error？**
作者将评分过程拆成三个部分：
1. Preliminary Error Scoring；
2. Multi-Scale Differencing；
3. Position Bias。
此外还设计了 Temporal Consistency Head，作为训练阶段的辅助任务。
 **4.4.1 Preliminary Error Scoring（初始错误评分）**
<mark style="background: #FFB8EBA6;"> MLP（Multi-Layer Perceptron，多层感知机是最基础的深度学习模型，也是全连接神经网络 (FNN)的典型代表，由多层感知机神经元堆叠而成，核心作用是拟合复杂的非线性映射关系，可完成分类、回归、特征学习等任务。</mark>
<mark style="background: #FFB8EBA6;">MLP 突破了单层感知机只能拟合**线性边界**的局限，通过隐藏层+非线性激活函数，实现对任意复杂非线性函数的逼近。</mark>
<mark style="background: #FFB8EBA6;">激活函数（Activation Function），就是在人工神经网络神经元上运行的，负责将神经元的输入映射到输出端，旨在帮助网络学习数据中的复杂模式。</mark>
首先，将最终特征输入一个轻量级 MLP：$\ell=W_2\cdot GELU(W_1H_{final}+b_1)+b_2$
最终得到：$\ell\in\mathbb R^T$
其中，每一个时间步对应一个初始错误分数：$\ell=(\ell_1,\ell_2,\ldots,\ell_T)$
但是作者认为，单纯依靠 MLP 分类还不够。
因为错误不仅可能表现为语义异常，还可能表现为隐藏状态在时间维度上的异常变化。
**4.4.2 Multi-Scale Differencing（多尺度差分）**
变化可能有两种形式：
- 突然变化：相邻步骤之间出现明显偏移。
- 累积变化：多个步骤之间逐渐产生偏差。
对于给定尺度 $s$，定义：$D_t^{(s)}=\left\|h_t-s\cdot h_{t-1}+(s-1)\cdot h_{t-s}\right\|_2$
**第一种情况：当** $s=1$
公式退化为：$D_t^{(1)}=\|h_t-h_{t-1}\|_2$这就是一阶时间差分。
它衡量相邻两个步骤的隐藏状态变化程度。
例如，假设隐藏状态简化为标量：
$h_{t-1}=2,\quad h_t=8$
则：$D_t^{(1)}=|8-2|=6$
说明相邻步骤之间出现较大的变化。
**第二种情况：当** $s=2$
公式变为：
$D_t^{(2)}=\|h_t-2h_{t-1}+h_{t-2}\|_2$
这是一种二阶差分形式。
它主要衡量变化趋势是否发生改变。
例如：
$h_{t-2}=2,\quad h_{t-1}=4,\quad h_t=6$
则：$D_t^{(2)}=|6-2\times4+2|=0$
说明三个状态按照线性趋势变化。
如果：$h_{t-2}=2,\quad h_{t-1}=4,\quad h_t=10$
则：$D_t^{(2)}=|10-2\times4+2|=4$说明变化趋势出现了明显偏移。
对于更大的尺度 $s$，作者使用式（7）中的加权组合衡量不同时间跨度上的变化。
**多尺度归一化**
不同尺度的差分结果可能具有不同的数值范围。
因此，作者对每个尺度的差分进行归一化：
$\widetilde D_t^{(s)}=\frac{D_t^{(s)}}{\frac{1}{T}\sum_{t=1}^{T}D_t^{(s)}+\epsilon}$
其中：
- $D_t^{(s)}$：当前步骤在尺度 $s$ 下的差分强度；
- $\epsilon$：防止分母为零的稳定项。
然后对所有尺度进行平均：
$\Delta_t=\frac{1}{|\mathcal S|}\sum_{s\in\mathcal S}\widetilde D_t^{(s)}$ （8）
其中 $\mathcal S$ 表示使用的多尺度集合。
最终得到每个步骤的多尺度时间变化分数。
**这一机制的本质：**
- $\ell_t$：模型从特征分类角度判断当前步骤是否错误。
- $\Delta_t$：模型从隐藏状态变化角度衡量当前步骤是否异常。
二者提供不同的信息。
需要注意，论文第四部分没有在这里明确列出尺度集合的具体取值，也没有详细说明序列开头无法计算 $h_{t-s}$ 时的边界处理方式，不能自行假定其实现细节。
**4.4.3 Position Bias（位置偏置）**
这是一个比较有意思的设计。
作者认为，在多智能体协作中，错误通常源于早期的关键决策，而后期异常可能是这些错误传播的结果。
但是，如果模型只依靠步骤特征，可能更倾向于选择后期表现更加明显的异常步骤。
因此引入位置偏置。
对于长度为 $T$的轨迹，第 $t$步的位置偏置定义为：$b_t=-\frac{t-1}{T-1},\quad t=1,2,\ldots,T$

 **最终步骤错误分数**
$\ell_t^{final}=\ell_t+\beta\cdot\Delta_t+\gamma\cdot b_t$
**4.4.4 Temporal Consistency Head（时序一致性预测头）**
这一模块容易和 Multi-Scale Differencing 混淆，但二者的目标不同。

|   |   |   |
|---|---|---|
|**比较**|**Multi-Scale Differencing**|**Temporal Consistency Head**|
|核心目的|衡量隐藏状态变化|学习隐藏状态的演化规律|
|计算方式|多尺度差分|下一步状态预测|
|使用阶段|参与最终评分|只用于训练|
|是否需要辅助预测网络|不需要|需要|
作者引入一个轻量级预测函数：$\hat h_{t+1}=f_\theta(h_t)$
其中：
- $h_t$：当前步骤的隐藏表示；
- $f_\theta$：可学习映射函数；
- $\hat h_{t+1}$：预测的下一步隐藏表示。
例如：$h_1\rightarrow\hat h_2$  $h_2\rightarrow\hat h_3$  $h_3\rightarrow\hat h_4$
模型通过当前隐藏状态预测下一步的隐藏状态，再将预测结果与真实隐藏状态进行比较。
作者希望借此学习 MAS 轨迹中的局部时序依赖，增强模型对逻辑偏离的敏感性。
这里需要区分：
Temporal Consistency Head 并不是直接使用预测误差作为最终异常分数，而是为模型训练提供辅助约束。

### 4.5 Loss Function（损失函数）
作者采用联合损失函数，包含：
1. Classification Loss；
2. Temporal Consistency Loss。
**4.5.1 Classification Loss（分类损失）**
这是模型的主要训练目标。
假设：
- $B$：一个 Batch 中的轨迹数量；
- $y_i$：第 $i$ 条轨迹中真实错误步骤的索引；
- $\mathcal T_i$：第 $i$ 条轨迹中有效步骤的集合；
- $\ell_{i,j}^{final}$：第 $i$ 条轨迹第 $j$ 步的最终错误分数。
作者定义交叉熵损失：$L_{CE}=-\frac{1}{B}\sum_{i=1}^{B}\log\frac{\exp(\ell_{i,y_i}^{final})}{\sum_{j\in\mathcal T_i}\exp(\ell_{i,j}^{final})}$ （11）
我们拆开来看。
分子：$\exp(\ell_{i,y_i}^{final})$ 表示真实错误步骤的得分经过指数变换。
分母：$\sum_{j\in\mathcal T_i}\exp(\ell_{i,j}^{final})$ 表示该轨迹中所有有效步骤的指数得分之和。
二者相除，相当于在整条轨迹内部进行 Softmax。
最终通过交叉熵，让真实错误步骤获得最高预测概率。
一个重要理解
这不是对每个步骤分别进行独立的二分类。
而是让一条轨迹中的所有有效步骤进行竞争，最终选出一个 Decisive Error。
因此，它本质上是：
Trajectory-level Multi-class Classification（轨迹内部的多分类定位任务）。
另外，论文明确指出，轨迹长度可能不同，因此计算分类损失时会排除 Padding Steps。
**4.5.2 Temporal Consistency Loss（时序一致性损失）**
作者认为，仅使用分类损失可能无法充分学习轨迹中隐藏状态的连续演化规律。
例如：
- 正常步骤之间可能存在相对稳定的状态演化；
- 错误步骤可能使后续隐藏状态偏离原有模式；
- 只关注错误标签可能忽略这种时序结构。
因此引入辅助损失。
定义： $L_{temporal}=\frac{1}{T-1}\sum_{t=2}^{T}\|\hat h_t-h_t\|_2^2$ （12）
其中： $\hat h_t=f_\theta(h_{t-1})$
也就是说：
1. 使用前一个时间步的隐藏表示；
2. 预测当前时间步的隐藏表示；
3. 将预测结果与真实隐藏表示比较；
4. 使用均方误差优化预测能力。
这是一种自监督辅助任务，因为它不需要额外的人工步骤标签，而是直接利用轨迹中已有的隐藏状态作为预测目标。
作者希望模型通过这个任务学习轨迹内部的时序演化规律。
不过需要注意：训练数据本身包含失败轨迹，因此不能简单地认为这个预测头只学习了正常行为模式。它实际上学习的是训练轨迹中的表示转移规律。
**4.5.3 Joint Training Objective（联合训练目标）**
最后，作者将两个损失组合：$L_{total}=L_{CE}+\lambda L_{temporal}$ （13）
其中：
- $L_{CE}$：主要监督损失；
- $L_{temporal}$：辅助时序一致性损失；
- $\lambda$：控制辅助损失权重的超参数。