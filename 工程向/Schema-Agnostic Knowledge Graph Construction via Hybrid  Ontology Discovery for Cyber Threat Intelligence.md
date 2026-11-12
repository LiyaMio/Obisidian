keywords: Cyber threat intelligence（网络威胁情报）, ontology（本体）, knowledge graph, large language models
## Abstract

## Preliminary
**"An ontology is a formal, explicit specification of a shared conceptualization." (本体是对概念化的形式的、明确的、共享的规范说明)**

|关键词|含义|通俗解释|
|---|---|---|
|**Formal(形式的)**|机器可读、可推理|不能只靠人脑理解,必须用计算机能处理的语言写下来|
|**Explicit(明确的)**|概念、约束被显式陈述|不靠隐含默契,"客户"的定义必须白纸黑字写清楚|
|**Shared(共享的)**|经群体共识认可|不是个人笔记,而是组织/社区内人人遵守的"官方词典"|
|**Conceptualization(概念化)**|抽象模型,而非具体数据|描述"世界上有哪些类型的事物",而不是"张三上周买了什么"|
一个完整的知识本体通常包含以下组成部分：

|要素|说明|示例（电影领域）|
|---|---|---|
|**Class（类）**|概念的集合|Movie, Person, Genre|
|**Property（属性）**|描述特征的数据|title, releaseDate, rating|
|**Relation（关系）**|类之间的关联|directedBy, actedIn, belongsTo|
|**Instance（实例）**|具体的个体|"Inception", "Christopher Nolan"|
|**Axiom（公理/规则）**|约束和推理规则|一部电影至少有一个导演|

| 概念                        | 回答的问题                          | 结构      | 例子                                        |
| ------------------------- | ------------------------------ | ------- | ----------------------------------------- |
| **Taxonomy(分类法)**         | "这是什么类的东西?归在哪个上级之下?"           | 树状层级    | 生物分类学:界门纲目科属种                             |
| **Ontology(本体)**          | "世界上存在哪些东西?它们之间允许有什么关系?能推出什么?" | 网状,含规则  | "客户可以下单,订单包含商品,取消的订单不能发货"                 |
| **Knowledge Graph(知识图谱)** | "具体到每个实例,事实是什么?"               | 图(节点+边) | "张三(客户)于 9 月 1 日下单了订单 #1024,内含 iPhone 一台" |
| **Schema(模式)**            | "数据库的表怎么建?"                    | 表结构     | 关系数据库的 ER 图                               |
**Taxonomy 回答"是什么、归谁管";Ontology 回答"有什么、能做什么、能推出什么";Knowledge Graph 则是"用真实实例填充后的 Ontology"**。AI Agent 需要全部三层:用 Taxonomy 分类、用 Ontology 推理、用 Knowledge Graph 行动。
为了让 Ontology "机器可读、可推理",W3C 建立了一套标准栈:

|标准|角色|通俗理解|
|---|---|---|
|**RDF**|数据模型|万物皆"三元组":主语-谓语-宾语(张三-就职于-字节跳动)|
|**RDFS / OWL**|本体语言|定义类、属性、层级,以及更复杂的公理(如"互斥""传递")|
|**SPARQL**|查询语言|图谱版 SQL|
|**SHACL**|约束校验|用"形状(Shape)"验证数据是否合规,类似数据质量的 JSON Schema|

| 标准             | 角色   | 通俗理解                                     |     |
| -------------- | ---- | ---------------------------------------- | --- |
| **RDF**        | 数据模型 | 万物皆"三元组":主语-谓语-宾语(张三-就职于-字节跳动)           |     |
| **RDFS / OWL** | 本体语言 | 定义类、属性、层级,以及更复杂的公理(如"互斥""传递")            |     |
| **SPARQL**     | 查询语言 | 图谱版 SQL                                  |     |
| **SHACL**      | 约束校验 | 用"形状(Shape)"验证数据是否合规,类似数据质量的 JSON Schema |     |
## I. Introuduction
indicators of compromise(IoC) : 入侵指标
Cyber threat intelligence (CTI) ：网络威胁情报
structured threat information expression (STIX)：结构化信息表达
malware information sharing platform (MISP)：恶意软件信息共享平台
OWL：定义本体类别、属性和层级结构
Schema-Agnostic：不将系统绑定到某一个固定本体
Ontology Typing：将实体或关系映射到正式本体元素
完整的网络攻击活动分析不能只依赖孤立的 IOC。
现有方法捕获孤立的数据点，丢弃了攻击步骤之前的而英国关系，以及将他们联系起来的因果推理。
Notable efforts include the unified cyber ontology (UCO)(统一多个网络安全标准的综合性本体) [7], STUCCO(网络安全知识图谱本体) [8], and MALOnt(面向恶意软件威胁情报的轻量级本体) [9].
不同组织、不同应用场景可能使用不同的本体，没有一个统一 Schema 能够覆盖所有需求。
Schema Dependency-->Challenge 1) How can we extract ontology-aligned knowledge without being tied to a specific schema?(何在不与特定模式绑定的情况下提取与本体对齐的知识？)
decouple extraction logic from schema definitions.
Scalability-->Challenge 2) How can we handle large-scale ontologies that exceed the capacity of prompt-based schema inclusion?(如何处理超过基于提示的模式包含能力的大规模本体？)
dynamically discover and retrieve only task-relevant ontology fragments(仅动态发现和检索与任务相关的本体片段)
Privacy-->Challenge 3) How can we support privacy-preserving CTI analysis with local LLMs?
本地部署开源大模型，减少敏感数据向企业API传输
To this end, we propose ANCHOR (Adaptive Navigation for Cybersecurity Hybrid Ontology Reasoning)（网络安全自适应导航混合本体推理）, a schema-agnostic system that builds a structured threat knowledge graph from unstructured CTI reports.
Contributions：
• We propose *ANCHOR*, a schema-agnostic CTI knowledge graph construction framework that decouples the extraction pipeline from any specific ontology schema, supporting arbitrary OWL/SHACL ontologies at runtime without manual reconfiguration. 
• We introduce *hybrid ontology discovery*, which combines embedding-based semantic search with LLM-guided recursive navigation to retrieve only task-relevant ontologies, integrated with closed-loop SHACL validation to ensure schema-compliant knowledge graphs. 
• We demonstrate privacy-preserving CTI knowledge graph construction with locally deployed open-source LLMs, retaining 99.2% (entity) and 97.8% (predicate) of the best enterprise LLM’s performance.
## II. BACKGROUND
A. CTI Data Formats and Ontologies
CTI records adversary **t**actics（战术）, **t**echniques, and **p**rocedures（过程） (TTPs) and IoCs（入侵指标）.
