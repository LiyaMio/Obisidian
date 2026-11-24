| 项目   | 内容                                                                                        |
| ---- | ----------------------------------------------------------------------------------------- |
| 论文简称 | ProvP4                                                                                    |
| 发表会议 | ACM CCS 2026                                                                              |
| 研究领域 | 网络安全、系统溯源、运行时可观测性                                                                         |
| 研究对象 | P4 可编程数据平面                                                                                |
| 核心技术 | 程序分析、代码插桩、溯源图、状态版本化                                                                       |
| 开源代码 | [GUSecLab/ProvP4](https://github.com/GUSecLab/ProvP4?utm_source=chatgpt.com)（论文提供）        |
| DOI  | [10.1145/3830454.3832594](https://doi.org/10.1145/3830454.3832594?utm_source=chatgpt.com) |
## Preliminary
为打破传统的固定封装模式，充分解放数据平面的编程能力，[Nick McKeown](https://zhida.zhihu.com/search?content_id=249061917&content_type=Article&match_order=1&q=Nick+McKeown&zhida_source=entity)领导的斯坦福大学研究团队于2014年提出可编程处理语言P4。借助P4的数据平面编程能力，用户可在网卡、交换机、路由器等网络设备上实现包括VXLAN、MPLS等在内的各种网络协议。
P4语言具有如下三点特性，同时也是P4语言的设计目标。
- 协议无关性：P4语言不与特定网络协议绑定。用户只需根据现网协议支持情况和P4语法要素来实现转发逻辑自定义，可按需加入新协议，也可删去冗余协议。
- 平台无关性：P4语言无需关注底层平台细节。用户能够独立于底层硬件对报文处理功能编程。用户不需要关心底层硬件的细节就可实现对数据包的处理方式的编程描述。前端编译器将P4高级语言程序转换成中间表示IR，后端编译器再将IR编译成目标设备配置。
- 可重构性：P4语言支持在不更换硬件的情况下动态修改数据包处理方式。允许“朝令夕改”，同一台设备可根据现网需求随时重新配置。
**传统网络设备如何工作？**
传统路由器或交换机内部包含两类核心功能：
- **控制功能**：决定数据包应该被发送到哪里。
- **转发功能**：根据已有规则实际处理和转发数据包。
控制平面已经通过路由协议计算出对应的转发规则，数据平面只需要查询转发表，就可以将数据包从 Port 2 转发出去，控制平面不需要为每个数据包重新计算路由。
**SDN（Software-Defined Networking，软件定义网络）：**
将网络控制逻辑与实际的数据转发功能解耦，使网络能够通过软件进行统一管理和编程。
![[ProgrammableDataPlaneAttacks-1795485986569.webp]]
SDN 控制器可以管理多台交换机，统一制定网络策略，并向交换机下发规则。这里的集中是逻辑上的集中，不一定意味着只能使用一台物理服务器。

| 比较维度  | 传统网络      | SDN      |
| ----- | --------- | -------- |
| 控制逻辑  | 分布于网络设备   | 逻辑集中于控制器 |
| 网络视图  | 设备维护各自的视图 | 可以维护全局视图 |
| 策略管理  | 分布式配置     | 集中式软件管理  |
| 数据转发  | 网络设备执行    | 网络设备执行   |
| 控制与转发 | 相对紧密耦合    | 解耦       |
**为什么有了 SDN 还需要 P4？**
传统 OpenFlow SDN 虽然实现了控制平面与数据平面的分离，但交换机所支持的数据包处理能力仍然受到预定义功能的限制。如果我们希望交换机实现一种新的网络协议，或者根据历史数据包的交互状态进行更加复杂的判断，传统固定功能的数据平面就会受到限制。
P4（Programming Protocol-independent Packet Processors）是一种专门用于描述数据包处理逻辑的领域特定语言。
它允许程序员定义：
- 如何解析数据包；
- 如何匹配数据包中的字段；
- 如何修改、转发或丢弃数据包；
- 如何读取和更新内部状态；
- 如何与控制平面交互
![[ProgrammableDataPlaneAttacks-1795486308440.webp]]
SDN 实现了网络控制逻辑的软件化，而 P4 进一步增强了数据平面本身的可编程能力。
## Contribution
(1) the design of a *programmable data plane provenance model*  that captures P4 semantics;
(2) the design and implementation of *ProvP4*, a tool to observe  and reason about security attacks against programmable data planes; 
(3) a comprehensive security and performance evaluations demonstrating the efficacy of our approach.
## 1 Introduction
Programming Protocol-independent Packet Processors (P4), a data plane programming language that enables developers to specify how packets should be parsed, processed, and forwarded.一种数据平面编程语言，使开发人员能够指定如何解析、处理和转发数据包。
现有网络安全分析机制存在三个关键问题。

|问题|具体表现|
|---|---|
|数据平面黑盒化|无法观察内部状态以及状态如何影响决策|
|跨平面交互不透明|无法准确追踪控制平面与数据平面之间的状态依赖|
|传统可观测机制难以扩展|全量记录运行时事件会产生巨大的性能和存储开销|
传统日志能够记录系统做了什么，但未必能够解释系统为什么执行该行为。尤其是在具有持久化内部状态的系统中，当前行为可能受到很早之前发生的状态修改影响。
## 2 Background and Motivation

![[ProgrammableDataPlaneAttacks-1795485386612.webp]]

|过程|涉及组件|作用|
|---|---|---|
|编译与部署|Architecture、P4 Program、P4C|定义交换机能够执行的处理逻辑|
|网络控制|Event Processor、Runtime Client、Runtime Server|管理和调整数据平面|
|数据包处理|Parser、Ingress、Buffer、Egress、Deparser|实际处理网络流量|

|组件|位置|职责|
|---|---|---|
|P4 Runtime Client|控制器|发起管理请求|
|P4 Runtime Server|目标设备侧|接收并执行管理请求，处理相关上报|
**P4 Pipeline.**
为了指定数据平面的数据包处理行为，程序员在程序中指定了三个组件：解析器、控件和去解析器。
*解析器*组件提取数据包的报头字段，*控制块*指定如何使用匹配动作表（MAT）、内存和算术逻辑单元（ALU）处理数据包，*去解析器*指定如何重建数据包报头并将其发送到网络。
