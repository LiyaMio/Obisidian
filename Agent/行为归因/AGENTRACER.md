AGENTRACER: WHO IS INDUCING FAILURE IN THE LLM AGENTIC SYSTEMS?
ICLR 2026
## ABSTRACT
Pinpointing the specific agent or step responsible for an error within long execution traces defines the task of *agentic system failure attribution*.
dataset TracerTraj :  counterfactual replay and programmed fault injection
multi-granular（多粒度） reinforcement learning
## Introduction
LLM 多任务表现出色，但是依赖单模型存在固有局限。
MAS 出现，性能优于单模型，但是多个自主代理与辅助模块（e.g, 外部工具）的集成加剧了系统的脆弱性。举例说明，系统失败率高。
A natural response to this challenge is *failure attribution*, i.e., the precise identification of faulty components within the system upon task failure.
错误归因的重要性，也可以说是引用
(I) system debugging, by enabling agentic systems to perform more effective self-debugging across iterative attempts, thereby enhancing performance; 
(II) data efficiency, by leveraging failed agent trajectories to construct more informative training data; 
(III) grounded self-improvement, by facilitating grounded self-correction through precise agent credit assignment. （代理信用分配）
问题：需要人工分析轨迹日志，自动化方法准确率低。
研究空白：
 训练资源：training resources, concerning the automated construction of large-scale annotated multi-agent trajectories; 
 训练方法：methodology, on developing swift and accurate multi-agent failure localizer, which we also refer to as a tracer.
**Present Framework.**
(1) <mark style="background: #FFB8EBA6;">AgenTracer</mark>, a fully automated pipeline for constructing well-annotated multi-agent failure trajectories,
(2)<mark style="background: #FFB8EBA6;"> AgenTracer-8B</mark>, a lightweight failure attributor for multi-agent systems. （反事实重放与程序化出错误注入）
QWEN3-8B 微调，多粒度奖励机制
**主要贡献**
❶ <mark style="background: #FFB8EBA6;">Automated Pipeline</mark>. We propose <mark style="background: #FFB8EBA6;">AgenTracer</mark>, the first automated pipeline for annotating  multi-agent system failures, curating over 2,000 high-fidelity failure trajectories across seven datasets through counterfactual replay and programmatic fault injection.  
❷<mark style="background: #FFB8EBA6;"> Failure Tracer</mark>. We develop <mark style="background: #FFB8EBA6;">AgenTracer-8B</mark>, a lightweight failure tracer dedicated for LLM  agentic systems, trained via a multi-granular reinforcement learning to ensure accurate attribution at both the step and agent levels, facilitating automatic agentic system debugging. 
❸ Empirical Evaluation. Experiments show that AgenTracer-8B facilitates (I) accurate attribution, outperforming giant LLMs like DEEPSEEK-R1 by ∼ 12.21% and GEMINI-2.5-PRO by ∼ 18.18% on Who&When benchmark; and (II) self-evolution, enabling off-the-shelf agentic systems to improve performance by 4.8 ∼ 14.2%.
## Related Work
**LLM-based Multi-Agent System.**
根据自动化程度分成三类
Handcrafted. 手动指定项目配置
Partially Automated.  自动化特定的系统组件
Fully Automated.  系统内所有模块都是自动设计和进化
Our method is designed to provide precise error tracing across this entire spectrum of agentic systems. Accordingly, we curate trajectories from all three categories.
**Failure Attribution for Agents.**
**LLM-as-a-Judge & Credit Assignment**
Credit Assignment is a long-standing topic in multi-agent reinforcement learning (MARL), linking individual actions to long-term outcomes (Pignatelli et al., 2023; Arjona-Medina et al., 2019; Yin et al., 2023).
## Preliminary
多智能体定义：$M = <I, S, A, Ψ, μ>$ Consider a LLM-based multi-agent system *M*, tasked with collaboratively resolving a user-issued query *Q*（用户给出的任务）. The system consists of *N* agents, indexed by *$I = {1, 2, . . . , N }$*, operating in discrete time under a turn-based protocol, *S* 系统状态，*A* 动作空间, *μ(t)* 在时间t时采取行动的智能体, *Ψ(st+1 | st, at, μ(t))* 状态转移,
At each step t, the active agent μ(t) selects an action *$a_t ∈ A_{μ(t)}$* according to its policy $π_{μ(t)}$, conditioned on the current state $s_t$, the query Q, and a subset of the prior interaction history $H_t$:
$a_t = π_{μ(t)}(s_t, H_t, Q), H_t ⊆ \{a_0, a_1, . . . , a_{t−1}\}.$
The full execution trajectory of the system is denoted by: $τ = (s_0, a_0, s_1, a_1, . . . , s_T ),$
**Objective Formulation.**
decisive error : the earliest action in the trajectory whose correction is sufficient to steer the system from failure to success.
Formally, let $Ω(τ ) ∈ {0, 1}$ be a binary evaluation function indicating failure $(Ω(τ ) = 0)$ or success $(Ω(τ ) = 1)$, and $R(τ, t, a^′_t)$ be an oracle rectification operator（校正算子）, which represents an idealized process where the original action at is replaced by a theoretically  perfect, oracle-provided action a′t, and all subsequent steps are re-simulated. Naturally, the set of all decisive agent-step pairs C(τ ) can be expressed as :
$C(τ) = \{(i, t)\ |\ i = μ(t) \ \ and \ \ Ω(τ) = 0\ ∧\ Ω (R(τ, t, a^′_t)) = 1\} ,$
$Ω(τ) = 0$ 任务失败；$R(τ, t, a^′_t)$ 反事实修正算子；
可能有多个不同步骤，单独修正后都能够让系统成功
所以论文接着提出公式（5）：$(i^*,t^*)=arg min_{(i,t) \in C(τ)}t$ 从所有能够通过单步修正使任务成功的候选步骤中，选择时间最早的那一个。
## Methodology
![[AGENTRACER-1794186415918.webp]]
### AgenTracer: AUTOMATIC TRAJECTORY ANNOTATION
**Collection**
从M=6的多Agent系统中进行轨迹提取，对于每个MAS进行一系列的查询，采集查询后的成功 $Ω(τ) = 1$与失败轨迹 $Ω(τ) = 0$。
**Locating Decisive Errors**
