## ABSTRACT
A trajectory combines hierarchical execution structure, large volumes of text whose analysis often requires semantic reasoning, and rich dependencies and lineage among events, intermediate states, and derived artifacts.
## INTRODUCTION
An agent places an LLM in a runtime loop: when the model response contains a tool call, the runtime executes it in the environment and feeds the result into the next model invocation .
轨迹定义：A trajectory records the messages sent to the LLM, model responses, tool calls, tool results, and associated runtime state.
相关工作：
轨迹碎片化管理
Research systems commonly store trajectories as JSONL files and process them using task-specific Python pipelines.
结构化存储：Production memory systems such as Mem0 [14] and Letta [32] take the opposite approach: they persist extracted memories using <mark style="background: #FFB8EBA6;">structured storage, similarity search, and graph retrieval</mark>, but typically expose only task-specific derived state rather than the complete relationship between that state and its source trajectory.
日志存储：Security and observability systems ingest agent activity into mature log stores.轨迹重建于语义回溯依赖特定的应用程序。
挑战：
嵌套有序执行结构：First, trajectories have nested and ordered execution structure: a session contains turns, model invocations, tool calls, messages, and potentially branches, retries, and subagents.
free-text：Second, their dominant payload is free-form text, so many important predicates require semantic evaluation rather than exact matching.
高度冗余：Third, trajectories are highly redundant: successive model invocations often repeat much of the same context, causing large volumes of duplicated text to be stored and processed.
派生与谱系：Finally, trajectory processing creates rich derived state and lineage. A memory may depend on several messages, a failure judgment on a particular tool result and model response, and a training label on the context evaluated by a specific judge model and criterion.
主要贡献 ：
提出 TrajectoryDB，将轨迹作为一种原生数据进行管理。
We therefore envision TrajectoryDB, a trajectory-native data management system that co-designs ingestion, storage, and query processing to efficiently manage and analyze agent execution trajectories.
## TRAJECTORY AS A NEW DATA TYPE
### Agent Trajectory
The basic unit of communication between an agent and the model is<mark style="background: #FF5582A6;"> a message, which contains a role and its associated content</mark>.
角色划分（four roles）：
System：*system* refers to the **system prompt**, which specifies instructions that always apply, user provides user input, assistant records model responses, and tool captures results returned by external tools.
Step：We refer to one such model request and its resulting execution as a *step*.
Turn：A user input can trigger multiple steps before the agent returns a response to the user, which forms a *turn*.
Session：A task often spans multiple turns, and the resulting sequence of turns forms a *session*.
<mark style="background: #FF5582A6;">Agent trajectory</mark>：We call the sequence of requests 1, messages, and tool executions produced while completing a task as an *agent trajectory*.
Execution configuration：
A trajectory often also records the components and settings that shaped its execution, including the model, available tools and their versions, and sampling parameters such as temperature and random seed.
**Operational telemetry：**
A trajectory may also include systemlevel measurements associated with its execution.
Time to first token (TTFT) and per-step latency can support service-level agreement (SLA) and user-experience (UX) monitoring.
Input and output token counts, together with cache-hit ratios.
Retries, rate-limit events, timeouts, and error code.
Tool call resource usage.
### Diverse Use Cases
轨迹常用分类：We group six representative use cases into three categories: (A) learning from past experience, (B) analyzing and controlling agent behavior, and (C) monitoring and optimizing runtime execution.
**A.1 Extracting reusable memory：**
Agents frequently extract memory from past trajectories.利用提取的信息来指导之后的任务，不需要轨迹重放。
**A.2 Fine-tuning from successful trajectories：**
轨迹可以做训练任务数据，帮助模型调优。
**B.1 Root-cause analysis：**
Trajectories support failure diagnosis and anomaly detection.
For example, *TRAIL* identifies reasoning, planning, and system-execution errors in the trajectory; *AgentRx* checks each step against constraints derived from tool schemas and domain policies, then uses the resulting violations to localize the first unrecoverable failure.
**B.2 Enforcing safety and security：**
Trajectories help to reveal unsafe behavior.
Tool-level checks and LLM monitors.
A sequence can be unsafe even when each action appears acceptable in isolation.
**C.1 Monitoring runtime health.**
Operational telemetry supports performance and cost monitoring.
Systems such as *LangSmith*, *Langfuse*, and *Datadog record* model and tool executions together with latency, token usage, cost, and errors.
**C.2 System resource usage and scheduling optimizations.**
Agents frequently block on LLM inference and tool execution, making the *scheduling order* of LLM requests and tool calls a key determinant of end-to-end performance.
## TRAJECTORY-DB REQUIREMENTS
Despite the diverse use cases, these workloads repeatedly access, filter, and aggregate overlapping parts of a trajectory. We next summarize common data operations into three requirements that a database engine should support,
### Structure-aware Retrieval
**Session-structure retrieval**
