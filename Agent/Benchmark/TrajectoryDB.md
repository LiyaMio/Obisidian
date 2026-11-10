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
需求：不同workload对结构化轨迹的粒度需求不一样
Session retrieval：当派生子Agent时，轨迹不再是线性的结构。（Fig1 left）
Step retrieval：错误诊断通常在step粒度的轨迹上进行。
Role retrieval：微调等操作通常使用基于角色的信息。
![[TrajectoryDB-1793929418482.webp|339x364]]
**Sequence retrieval**
Many trajectory workloads identify patterns over *ordered events* rather than individual records.
edit → failed test → edit the same file may indicate an ineffective strategy
search → read → edit → passing test suggests a successful one
### LLMs as UDFs
UDF （User-Defined Function）
Exposing LLM-based analysis as a general UDF allows the same semantic primitive to be reused across workloads and composed with other query operators.
**Composing semantic and relational operators**
semantic operators(LLM calls) need to compose with relational and order-sensitive operators throughout the query plan
**Incremental semantic evaluation over growing trajectories**
增量语义评估：Semantic predicates often operate on growing inputs, so their results must be updated statefully to remain valid as new evidence arrives.
Semantic evaluation must build on previous state and incorporate new evidence as the trajectory grows.
### Lineage Query
**Context reconstruction**
Long-running agents often *compact their context* by replacing earlier messages with summaries.
**Preserving lineage for semantic operator**
TrajectoryDB should preserve not only each semantic operator result, but also its provenance: the evaluated input, supporting trajectory records, judge model, evaluation criterion, and other information needed to reconstruct how the result was derived.
This lineage serves two purposes. 
允许无效：First, it enables invalidation: if the source trajectory changes, the memory becomes stale, or the judge model or criterion is updated, the system can identify which judgments need to be reevaluated. 
支持来源查询：：Second, it supports provenance queries that ask what produced a judgment and which trajectory evidence supports it. 
Semantic judgments therefore become *reusable derived data* rather than outputs of one-time LLM calls.
## CHALLENGES AND OPPORTUNITIES
![[TrajectoryDB-1793932107486.webp]]
### Structure Reconstruction and Maintenance
Trajectory structure is scattered across timestamped requests. Therefore, TrajectoryDB should reconstruct this structure and organize it both logically and physically.
**Reconstructing Request Hierarchy.**
<mark style="background: #FFB86CA6;">Challenge: Missing structural links.</mark>
Design question: Which structures should be reconstructed eagerly?
考虑到性能：TrajectoryDB must therefore determine which structural relationships to materialize during ingestion and which to derive lazily.
<mark style="background: #FFB86CA6;">Challenge: Requests can be separated by long gaps.</mark>
连续请求间隔较长。
![[TrajectoryDB-1793933499689.webp]]
Design question: How to maintain state for long-lived streaming trajectories?
Existing stream-processing abstractions therefore do not naturally support incremental reconstruction of long-lived agent trajectories without assuming a bounded or predictable completion window.
**Maintaining Graph Structure over Relational Data.**
<mark style="background: #FFB86CA6;">Challenge: Trajectory relationships form graphs, not flat rows.</mark>
Subagent execution creates *tree-shaped branches*, while lineage introduces labeled *many-to-many edges*: memories are derived from evidence, judgments are supported by or inconsistent with earlier records, and failures may be caused by or repeat earlier failures. Together, these relationships form *DAGs* over trajectory records.
Design question: How to efficiently support both graph traversal and relational analytics?
遍历需要递归，成本高。
Agent trajectories are more specialized: they are append-only, mostly forest-shaped, and naturally clustered by session. Exploiting these properties may enable tighter integration of graph traversal with relational and ordered access.
### Data Layout and Storage
**Reconciling Conflicting Physical Layouts.**
<mark style="background: #FFB86CA6;">Challenge: Use cases favor conflicting physical layouts.</mark>
physical layouts 磁盘中的物理存储方式

| Physical Layout                                                     | How Data Is Arranged on Disk                                                                                                                                           | Workloads That Need It                                                                                                    | Why It Conflicts                                                                                                                              |
| ------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| **Session-Contiguous Layout** _(Row/Session Locality)_              | Stores all messages, turns, and steps of a single session together in chronological sequence.                                                                          | **Replay, context reconstruction, sequence matching** (e.g., reading a full session history to replay or extract memory). | If you only want to analyze a single field (like error rates) across millions of sessions, reading entire sessions causes massive wasted I/O. |
| **Columnar Layout** _(Attribute Locality)_                          | Stores values of the same field contiguously across millions of records (e.g., all `latency` values together, all `tool_name` values together).                        | **Telemetry analysis, cost monitor**ing, role filtering (e.g., aggregating token counts or finding all `tool` errors).    | If you need to reconstruct an entire conversation, gathering fields scattered across columns requires expensive multi-column reconstruction.  |
| **Graph / Lineage-Linked Layout** _(Pointer/Relationship Locality)_ | Stores data clustered by parent–child or lineage relationships (e.g., grouping a derived memory together with the specific tool result and message that justified it). | **Root-cause provenance, memory verification, invalidation queries**.                                                     | Records related by lineage often span different turns, steps, or sessions, breaking both chronological session-order and column order.        |
Design question: How should trajectories be physically organized for diverse access patterns?
TrajectoryDB must therefore determine the primary representation, which auxiliary layouts or indexes to maintain, and how to keep them consistent as trajectories continuously grow.
**Storing Large but Redundant Text Payload.**
<mark style="background: #FFB86CA6;">Challenge: Trajectories are requests with cumulative context.</mark>
上下文累积
Design question: Should the text payload be stored inline or separately?
TrajectoryDB therefore needs a layout that keeps large text efficient to scan without letting it dominate the record storage.
<mark style="background: #FFB86CA6;">Challenge: Context is highly redundant, but not always prefix-aligned.</mark>
Each request typically carries all the messages from the same session as accumulated context, often referred to as a *prefix*.
In production, we observe that 80% of the sessions have context mutation: compaction, truncation of tool results, or edits to earlier context break prefix sharing.This makes it impossible to store only the newly appended turn.
Design question: How to support prefix-aware text compression and deduplication?
The storage engine should exploit this known sharing directly rather than rediscovering redundancy from raw bytes or literals within physical data blocks.
### Query Language and Optimization
**Enabling Order-Sensitive Query Processing.**
<mark style="background: #FFB86CA6;">Challenge: Trajectory queries are often order-sensitive.</mark>
A trajectory query model must therefore support both branch-local order and partial order(偏序) across branches.
Design question: How to preserve order and optimize the query plan?
A trajectory-aware engine should therefore track order through operators, reuse existing order rather than recompute it, propagate it through lineage traversal, and explicitly represent partial order across concurrent branches.
**Co-optimization Across Relational and Semantic Operators.**
<mark style="background: #FFB86CA6;">Challenge: Semantic operators consume and produce relational data.  </mark>
Pushing a cheap telemetry filter before a semantic judge can reduce LLM invocations, whereas an early, highly selective semantic filter may avoid an expensive traversal.
Design question: How to define a declarative framework to integrate semantic operators into the relational plan?
**Caching Semantic Results.**
<mark style="background: #FFB86CA6;">Challenge: Semantic results are expensive and ephemeral.</mark>
Design question: When can a cached semantic result be safely reused?
Existing semantic caches such as GPTCache and MeanCache  reuse LLM outputs by matching semantically similar inputs. This is insufficient for trajectory queries, where validity *depends not only on input similarity but also on the underlying trajectory state*.
## RELATED WORK
Semantic operators.
Data lineage.
## Conclusion

