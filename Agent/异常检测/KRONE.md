**KRONE: Hierarchical and Modular Log Anomaly Detection**，发表于 ICDE 2026。
https://github.com/LeiMa0324/Krone_official
## Contribution
1. 提出可自适应提取的日志层次抽象。Propose a hierarchical abstraction for log data, which  can be adaptively extracted for multi-scale log anomaly detection from any new application data set. 
2. 自动构建层次结构，将日志拆分为 KRONE Seqs。Develop a streamlined (optionally LLM-assisted) pipeline  that instantiates the hierarchy dynamically and that way decomposes logs into modular KRONE Seqs, achieving up to 117.3× data-space reduction (Figure 4). 
3. 设计 Local-Context 和 Nested-Aware 混合检测。Design hybrid KRONE Seq-level modular detection with  adaptive routing between a lightweight Local-Context detector of pattern matching and a Nested-Aware detector using LLM reasoning, to balance efficiency and accuracy. 
4. 利用缓存复用和层次调度降低成本.Develop resource-optimized KRONE detection framework  that executes the modular detection tasks along the hierarchy. Experiments show that even with a simple detector, hierarchical execution brings significant F-1 improvement (42.49% → 87.98%) over non-hierarchical detection (Table V), with resource savings up to 43.7×(Figure 7). 
5. 在三个公开数据集和一个工业数据集上验证.,valuate KRONE on three public benchmarks and one  Bytedance industrial dataset—revealing at which hierarchical abstraction levels anomalies manifest across datasets (Figure 6(b)), showcasing three level-specific anomalies that KRONE found with LLM-generated explanations in our supplement1, and achieving significant F-1 improvement (82.76% → 92.83%) over SOTA methods, while reducing LLM usage to 1.1–3.3% of test sequences.
## Introduction
![[KRONE-1795247588816.webp]]
研究背景：
传统日志异常检测存在一个结构性问题：真实的软件系统具有层次化的执行结构，但日志通常被处理成扁平序列。
传统方法主要使用 LSTM、RNN、Transformer 等模型学习日志序列中的正常模式。
这里产生两个问题：
1. 如果模型试图学习长距离依赖，计算成本会比较高。
2. 如果采用滑动窗口切分序列，又可能丢失跨窗口的依赖关系。
作者将这种问题称为 **Flattened-Dependency Dilemma**
本质上是： 扁平化的日志表示，使检测模型难以区分真正的执行依赖与无关事件之间的关联。
层次结构的价值：
第一：模块化、可复用的模式学习。
- 每个系统组件通常只有少量重复出现的原子行为模式。
- 整条日志序列的多样性，很大程度上来自这些局部模式的不同组合。
第二：多层次异常解释。
分类：组件内部异常和组件之间的执行异常。
这两个异常分别发生在不同的抽象层次。
如果只在高层检测，可能发现模块跳转异常，却无法发现组件内部的具体问题。
如果只在低层检测，又可能忽略跨组件的执行关系。
Challenge 1：Effective Hierarchy Abstraction of Black-Box Systems 黑盒系统的有效层次抽象。
现实系统通常是黑盒，检测者无法直接获得：
- 系统模块结构；- 组件之间的调用关系；- 具体的执行边界。
因此，不能依赖人工提供完整的执行层次。
Challenge 2：Lightweight Adaptive Modeling under Massive Logs 海量日志下的轻量级自适应建模。
如果通过人工规则构建层次结构，会遇到
- 不同应用需要不同规则；- 日志模板变化需要人工维护；- 海量日志使规则构建成本增加。
Challenge 3：Detection Efficiency with Nested Semantics 嵌套语义下的检测效率。
假设两个执行序列具有相同的高层模块顺序：`[Session, Auth, Comm]`但它们在各个组件内部执行了不同的动作。
只检查高层模块顺序，检测效率高，但无法发现细粒度异常。
检查所有低层动作和状态，检测能力更强，但计算开销也更高。
## RelatedWork
传统规则方法与深度学习日志异常检测
早期主要采用：- Keyword Search - Rule Matching

|类型|特点|代表性思路|
|---|---|---|
|Supervised|需要正常与异常标签|二分类|
|Unsupervised|不需要大量异常标签|学习正常日志行为|
Hierarchical Log Representation

| 研究思路                     | 结构形式                  | 作者指出的局限                 |
| ------------------------ | --------------------- | ----------------------- |
| 多尺度文本表示                  | Multi-scale Text      | 仍然侧重文本表示                |
| Hierarchical Transformer | 分层 Transformer        | 没有真正恢复执行组件语义            |
| 浅层文本结构                   | Word → Log → Sequence | 层次较浅                    |
| JSON-style Fields        | JSON 字段组织             | 没有体现组件执行层次              |
| DeepTraLog               | Span Events           | 依赖 TrainTicket 系统的硬编码规则 |
LLM-based Log Analysis
方向一：LLM-based Log Parsing
LLMParser LibreLog
主要利用生成式 LLM 和语义聚类提高日志解析效果。
方向二：LLM-based Anomaly Detection
LogPrompt LogGPT RAG-based Log Detection

|方法|问题|
|---|---|
|逐条日志调用 LLM|缺乏足够上下文，难以识别序列级异常|
|对整条序列调用 LLM|计算成本高，长上下文还可能出现 Lost-in-the-Middle|
## Preliminaries
**Log Parsing：日志解析**
日志解析的目标是从程序生成的格式化字符串中稳定的文本模板：
Definition 1：$k \longleftrightarrow t$
- $k$：Log Key，日志模板的唯一标识。
- $t$：Log Template，日志模板。
Log Key 不是某次执行的唯一 ID，而是某种日志模板的标识。
**Log Partitioning：日志划分**
哪些日志应该被划分到同一条日志序列中？
把海量、连续的日志流组织成相对独立的执行序列，让后续异常检测能够在有意义的上下文中分析日志之间的关系。
A 和 B 是两个不同节点，它们的行为可能相互独立。
如果直接学习这条混合序列，模型可能错误地学习到：
- A_Start 后面通常是 B_Start；
- B_Start 后面通常是 A_Read；
- A_Read 后面通常是 B_Write。
