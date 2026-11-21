Intention-behavior consistency-based automated failure attribution for LLM-driven multi-agent systems
Contribution
- **基于意图–行为一致性的故障归因框架**：An attribution framework for intention-behavior consistency: We introduce a novel attribution framework that explicitly characterizes and aligns the expected intention with the actual behavior via core collaborating agents without requiring task-specific supervision or prior failure annotations. This provides a new perspective for automated failure attribution. 
- **ToM 与 Cohen–Levesque 理性意图理论的结合**：Integration of ToM（Theory of Mind） and the Cohen-Levesque theory of rational intention: Building on ToM’s ability to extract an agent’s interpretable intention, the Cohen-Levesque theory provides the normative conditions for intention’s formation, maintenance, and termination. The synergy of these two components converts the consistency between expected intention and actual behavior into a testable standard. 
- **提出具有泛化性的意图–行为一致性视角**：Generalizable perspective on intention–behavior consistency: We introduce a generalizable theoretical perspective that formalizes intention–behavior consistency as structured and interpretable relations within a shared semantic space. This formulation enables the alignment and comparison of behavioral semantics across heterogeneous agents and coordination settings, providing a principled foundation for analyzing how failures propagate and generalize across MAS. 
- **实验验证**：Comprehensive experimental validation: Experimental results show that our framework achieves substantially higher step-level attribution accuracy than all baseline methods, allowing it to pinpoint the earliest failure point. At the same time, the framework maintains competitive or superior agent-level attribution performance, confirming its reliability.
## 1. Introduction
The current state-ofthe-art approaches for failure attribution in these systems remain predominantly manual, requiring developers to meticulously analyze hundreds or thousands of lines of interaction logs.
This fundamental limitation stems from the fact that current methods treat failure attribution as a pattern recognition task over long conversation logs, rather than addressing the core challenge of understanding the causal relationships between agent intentions and observed behaviors.
## 2. RelatedWork
**2.1 Characteristics of Runtime Failures in LLM-driven MAS**
MAS 的故障具有三个特点：
- **概率性（Probabilistic Nature）**：LLM 输出具有非确定性，相同输入也可能产生不同结果。
- **交互耦合（Tight Coupling）**：Agent 之间相互依赖，一个 Agent 的错误可能影响多个下游 Agent。
- **复杂任务流（Complex Task Flows）**：任务执行中存在隐式依赖、反馈循环和涌现行为。
2.1.1 Cascading Amplification（级联放大）
定义：上游 Agent 产生的错误输出，被下游 Agent 持续采用和传播，最终导致错误影响不断扩大。
三种常见诱因：
1. 指令解析错误、单位转换错误、维度理解错误。
2. LLM 幻觉产生自洽但不真实的信息。
3. 下游 Agent 对上游输出过度信任，缺乏有效验证。
![[IBC-1795238897763.webp]]
2.1.2 Goal Misalignment and Coordination Failure（目标错位与协调失败）
级联放大主要关注错误信息如何传播，而目标错位主要关注任务分解和协调机制本身存在什么问题。
**局部合理性不代表全局合理性。**
Agent 并不一定执行了明显错误的动作，问题可能在于上游任务分解没有提供足够的约束条件。
两种故障的区别

| 维度   | Cascading Amplification | Goal Misalignment |
| ---- | ----------------------- | ----------------- |
| 核心问题 | 错误传播和放大                 | 目标分解与协调冲突         |
| 主要来源 | 错误信息、幻觉、过度信任            | 不完整或冲突的子目标        |
| 下游表现 | 继承错误信息                  | 局部合理但整体冲突         |
| 典型机制 | 错误结果被反复采用               | 多个 Agent 独立优化     |
**2.2 Existing Automated Failure Attribution Methods**
1. All-at-Once Attribution：将完整的故障日志和任务描述一次性输入 LLM，让它判断哪个 Agent、哪个步骤出现了错误。
2. Step-by-Step Attribution：按照时间顺序逐步提供日志，让 LLM 判断当前步骤是否存在错误。
3. Binary Search Attribution：将行为序列不断划分为不同区间，通过迭代搜索缩小可疑步骤的范围。
![[IBC-1795240018654.webp]]
**2.3 ToM and Rational Intention**
 1. Theory of Mind（ToM）：**一个主体能否理解其他主体的心理状态，并据此推断其行为。**
 2. Cohen–Levesque 理性意图理论：
	 - An agent’s Beliefs encompass its cognition of the external world, other agents, and its own state. （一个主体的信念包括它对外部世界、其他主体和自身状态的认知。）
	 - Its Desires correspond to the sub-goals assigned to it. （它的愿望与分配给它的子目标相对应。）
	 - Its Intentions are its sustained commitments made to achieve those desires.（它的意图是它为实现这些愿望而作出的持续承诺。）
ToM：即使 LLM 能够推断意图，如何判断推断出来的意图是合理的？
Cohen–Levesque 理性意图理论：能够提供意图形成、维持和终止的规范性约束，使 ToM 推断出来的意图具有可以检查的合理性标准。

| ToM            | Cohen–Levesque |     |
| -------------- | -------------- | --- |
| 推断 Agent 的心理状态 | 约束合理意图的形成与维持   |     |
| 回答 Agent 想做什么  | 判断其意图是否合理      |     |
| 提供推断能力         | 提供规范性标准        |     |
## 3. Problem formulation
经典马尔可夫理论
轨迹：$\tau=\langle s_0,a_0,s_1,a_1,\ldots,s_T\rangle$，系统执行任务的过程被表示为一串交替出现的状态与动作。
论文假定每个离散时间步只有一个活跃 Agent 执行动作。
**Attribution Objective**：在失败轨迹中，定位对系统最终失败具有决定性作用的错误，并分析局部动作如何影响全局结果。
## 4. Methodology
![[IBC-1795240564563.webp]]
The core idea is to explicitly model each agent’s internal cognitive state (intention) and compare it with its externally observable behavior to detect inconsistencies that signify potential failure.（如何从多智能体运行日志中重建 Agent 的意图，并利用意图与实际行为之间的一致性定位故障？）
IBC（Intention–Behavior Consistency）框架由两个核心模块组成
- IRA（Intention Reconstruction Agent）：意图重建。
- AAA（Attribution Analysis Agent）：归因分析。
传统方法：$\text{Behavior Logs}\rightarrow\text{Anomaly Detection}$
IBC：$\text{Behavior Logs} \rightarrow \text{Intention Reconstruction} \rightarrow \text{Consistency Verification} \rightarrow \text{Attribution}$
**4.2 Intention Reconstruction：从日志到意图**
IRA 利用 LLM 的 ToM 能力，从原始运行日志中推断每个 Agent 行为背后的预期意图。
限制：
 - 意图必须对应实现目标的方法或计划。Intention poses a problem: The agent must determine a method or plan to achieve the intention. 
 - 新意图不能与 Agent 已有的意图或信念发生冲突。Intention provides a filter: The adoption of a new intention must not conflict with the agent’s existing intentions or beliefs. 
 - 意图应当保持一定的持续性，除非目标已经完成，或者 Agent 主动撤销意图。Intention maintains persistence: The intention should persist unless it is satisfied or actively rescinded, avoiding frequent, unjustified changes.
 - Agent 不应当形成自己认为无法实现的意图。The Agent believes it’s intention is feasible: The agent will not intend to do something it believes is impossible.
 - 如果 Agent 相信自己的意图无法实现，就应当放弃该意图。The Agent does not believe it will fail to achieve it’s intention: If the agent believes its intention will not occur, it must drop that intention.
**输入定义**：
**原始日志** $\mathcal L$：$\mathcal L=\langle l_1,l_2,\ldots,l_j\rangle$ .  $l_j$表示目标 Agent 的一条原始日志记录。
**任务规范** $\mathcal T$：Task Specification 描述 Agent 应当完成的任务。任务规范为意图重建提供外部目标依据。
**理性约束集合** $\mathcal C$：这是从 Cohen–Levesque 理性意图理论中引入的五项约束。限制 IRA 推断出来的意图，使其符合一定的合理性要求。
**输出定义**：
IRA 输出一个结构化意图序列：$\mathcal I_{\mathrm{eval}} = \langle i_{\mathrm{eval},1}, i_{\mathrm{eval},2}, \ldots, i_{\mathrm{eval},j} \rangle$ ，$i_{\mathrm{eval},j}$表示与日志记录 \(l_j\) 对应的重建意图。
完整过程定义为：$\mathcal I_{\mathrm{eval}} = \mathcal F_{\mathrm{eval}} (\mathcal L,\mathcal T,\mathcal C)$
$mathcal F_{\mathrm{eval}}$：基于 LLM 的理性意图推断过程。
$\mathcal L$：原始行为日志。
$\mathcal T$：任务规范。
$mathcal C$：理性约束集合。
$\mathcal I_{\mathrm{eval}}$：重建后的意图序列。
这里的意图是经过任务上下文和理性约束重建的预期意图，并不是直接观测到的真实心理状态。
**4.3 Attribution Analysis：意图与行为的一致性验证**
Agent 实际执行的动作，是否真正实现了重建出来的合理意图？
一致性判断函数:$mathcal C(i,l) $
其中：
$i$：重建出来的意图。
$l$：对应的实际行为日志。
$C$：基于 LLM 的语义一致性判断。
函数输出：$\mathcal C(i,l)\in\{0,0.5,1\}$

|分数|含义|判断|
|---|---|---|
|1|Full Consistency|行为完全符合意图|
|0.5|Partial Consistency|行为部分符合意图|
|0|Inconsistency|行为与意图不一致|

**如何确定故障步骤？**
Deviation Point：$j^*= \min\left\{ j\mid \mathcal C(i_{\mathrm{eval},j},l_j)<0.5 \right\}$
$j$：当前日志步骤。
$i_{\mathrm{eval},j}$：该步骤对应的预期意图。
$l_j$：该步骤实际执行的行为。
$\mathcal C$：一致性评分。
$\min$：寻找最早出现偏差的位置。