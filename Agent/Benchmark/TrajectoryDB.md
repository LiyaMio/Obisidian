- [[#ABSTRACT|ABSTRACT]]
- [[#INTRODUCTION|INTRODUCTION]]

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
