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