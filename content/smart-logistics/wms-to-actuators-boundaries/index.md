---
title: "从 WMS 到执行机构：自动化立库的职责边界，以及统一平台的真正变化"
description: "理解自动化立库的职责边界：谁有权作决定，谁负责记录事实，什么证据足以证明物理结果。"
date: 2026-10-30
draft: false
isCJKLanguage: true
categories: ["智能物流"]
tags: ["WMS", "WCS", "PLC", "WES", "虚拟 PLC"]
showTableOfContents: true
showReadingTime: true
showSummary: true
---

> 公开资料核对日期：2026-10-24。文中的托盘、四向车和提升机都是通用假设，不对应真实客户或项目。厂商资料用于说明特定产品的实践与限制；职责划分、通信契约和实施路线中的建议，均为本文的架构判断，不冒充统一行业标准。

自动化立库最容易被误解的地方，是把“软件已经决定”当成“现场已经发生”。

WMS 分配了库位，托盘还可能停在入库口；执行系统接受了搬运任务，提升机未必可用；PLC 报告某个动作完成，也不能脱离身份、位置和交接条件，直接解释成库存入账成功。

对熟悉 Java 的工程师来说，这里仍然有对象、状态机、消息和事务。但程序最终面对的是带惯性、摩擦、间隙和故障的机械系统。理解边界，要分别回答三个问题：**谁有权作决定，谁负责记录事实，什么证据足以证明物理结果。**

## 一、先看职责，再看 WMS、WCS、WES 和 MFS 的名称

WMS 通常管理仓库库存账及物流目标：这批货属于谁，批次是什么，当前质量或可用状态如何，可以分配给什么订单，应当去哪个业务位置。这里的“管理”不意味着货主、物料和质量结论都由 WMS 创造；它们可能来自主数据、ERP 或质量系统，WMS 保存关联和仓储业务所需的有效状态。

SAP EWM 的 Quant 文档将货主、批次、库存类型等作为库存区分因素，也区分仓库内的位置移动与数量变化。这支持“库存不只是一个 SKU 数字”的理解，但它是 EWM 9.0 的产品文档，不能当成所有 WMS 的数据库标准。[SAP Quant 文档](https://help.sap.com/saphelp_ewm900/helpdata/en/4b/c49585921f4c88e10000000a42189e/content.htm?no_cache=true)。

WCS 常承担路径、资源冲突和执行编排：当前哪些通道可用，任务怎样拆分，先服务哪块托盘，何时可以交给下一段设备。PLC 则靠近现场，运行周期任务或事件任务，维护动作序列、输入输出、设备状态和联锁。局部跟踪、占用维护乃至局部路由，也可以被分配给控制器；具体有没有这些能力，要看控制程序与接口契约。

这种划分描述的是逻辑责任，不要求存在三个独立软件包。SAP EWM 的 MFS 能直接集成控制层而不另设独立 WCS，其文档同时描述 PLC 对搬运单元的跟踪；在其箱式输送机方案中，路由决策又明确由 EWM 承担。因此，不能借这份资料进一步断言“PLC 普遍自主选路”。[SAP MFS 官方课程](https://learning.sap.com/courses/resource-management-in-sap-ewm/integrating-a-material-flow-system)。

名称也有厂商差异。SSI SCHAEFER 在 WAMAS 页面中，将其欧洲语境下的 WCS 对应到北美语境下的 WES，将 MFS 对应到北美语境下的 WCS；Dematic 则把 WES 描述为协调订单、人员、自动化与资源的执行能力。这些是各自产品体系中的用法，不是 WCS、WES、MFS 可以无条件互换的证明。[WAMAS WCS](https://www.ssi-schaefer.com/en-us/software/wamas-software-solutions/wcs-warehouse-control-system)；[Dematic 软件](https://www.dematic.com/en-us/software/)。

下图将信息链与能量链放在一起。它是职责示意，不是接线图，也不代表每台设备都有全部环节。

{{< mermaid >}}
flowchart TD
    W["WMS：库存与物流目标"] --> C["执行协调：任务与资源"]
    C --> P["PLC：序列与联锁"]
    P -->|控制信号| D["驱动器或阀"]
    E["电源或压缩空气"] -->|能量| D
    D --> M["电机、气缸与负载"]
    M --> S["传感器与编码器"]
    S --> I["I/O 与采集"]
    I -->|测量反馈| P
    S -.->|部分轴反馈| D
    P -.->|执行报告| C
    C -.->|交接结果| W
{{< /mermaid >}}

## 二、从一条数据到一次运动，中间发生了什么

### 2.1 传感器提供证据，不提供全知视角

光电开关检测遮挡，编码器测转角或位移，压力传感器把压力转换为电信号。信号还可能经过采样、滤波、转换、总线传输和软件映射。每一环都会影响数值、延迟或有效性。

“入口光电被遮挡”不等于“托盘 P 已到达”；电机轴转到了目标角度，也不自动证明托盘已准确落位。后者还取决于机械传动、打滑、间隙及测量点的位置。Beckhoff 的测量文档明确讨论不确定度及其条件；具体 ELM300x 文档则展示滤波和时间戳之间的关系，不能外推为所有传感器都有相同补偿机制。[测量不确定度](https://infosys.beckhoff.com/content/1033/elm3xxx/1678264203.html)；[ELM300x 技术数据](https://infosys.beckhoff.com/content/1033/elm3xxx/3642134923.html)。

因此，上层应接收“值、来源、质量、时间语义”，而不只是一个布尔值。时间字段还要说明是采样时间、变化时间还是接收时间。OPC UA 的 DataValue 区分源时间戳、服务器时间戳与状态码；源时间戳不能一律解释成采样时刻，值长期不变也不能仅凭旧源时间戳判断通信已中断。数据质量良好，同样不意味着测量没有误差。[OPC UA DataValue，1.05.07](https://reference.opcfoundation.org/specs/OPC-10000-4/7.11)。

本文建议同时保留生产者标识、启动代次和事件序号。序号用于同一生产者、同一代次内排序；随机生成的启动标识不能按大小判断先后。跨设备因果关系应靠命令关联和交接事件表达，即使时钟同步，也不能单凭时间接近证明因果。

### 2.2 PLC 扫描不是不断重复一次 HTTP 请求

以周期任务为例，程序读取当次可见的输入状态，推进状态机，计算输出，再等待后续周期确认动作结果。等待气缸到位可以是一个持续若干周期的状态，并不需要把控制任务阻塞在一次远程调用里。

这只是常见设计方式。PLC 也有事件任务；逻辑任务和总线交换未必同频。CODESYS 文档明确区分任务类型，并说明总线周期任务与其他任务所读取的缓冲区之间的关系。[任务配置](https://content.helpme-codesys.com/en/CODESYS%20Development%20System/_cds_f_reference_task.html)；[Bus Cycle Task](https://content.helpme-codesys.com/en/CODESYS%20Development%20System/_cds_buscycle_task.html)。

Java 同样能编写异步状态机。差别主要在运行时、调度保证、I/O 路径及故障响应的工程约束，而不在“哪种语言会循环”。数据库事务可以保护本地记录的一致性，却不能把已经移动的托盘回滚到原位。

### 2.3 执行机构改变能量，闭环可能分布在不同位置

电气运动链通常是电源、驱动器、电机、机械传动和负载。驱动器调节送入电机的电能，电机产生转矩，传动机构转换为实际运动；PLC 发出的是控制信息，不直接提供搬运负载的功率。

伺服系统常采用分层闭环，但位置环不必都放在 PLC 用户程序中。Beckhoff AX5000 文档给出位置控制位于驱动器或控制器 NC 侧的不同模式，说明闭环位置取决于产品和控制方式。[AX5000 控制模式](https://infosys.beckhoff.com/content/1033/ax5000_system_doku_hw2/648069771.html)。

气动链则由压缩空气、阀、气缸和负载组成。开关阀加端点传感器的方案，通常确认的是伸出、缩回等离散状态；不能因此说气缸无法连续位置控制。Festo 的伺服气动方案明确包含位移编码器、比例方向阀和位置控制器。[Festo 气缸原理](https://www.festo.com/us/en/c/products/actuators-and-drives/pneumatic-cylinders/piston-rod-cylinders-id_pim215)；[伺服气动定位](https://www.festo.com/us/en/c/products/actuators-and-drives/servopneumatic-positioning-systems-id_pim185)。

制动器还要区分静态保持、动态制动及安全相关用途。不能因为电机停转、输出关闭或程序进入 STOP，就直接推断负载已经被可靠保持。

## 三、沿着一块托盘走完入库

设想一个通用立库：托盘在入库口完成识别和检查，随后由四向车搬运；跨层时，四向车携带托盘进入载车提升机，到目标层后驶出，再完成货位交接。以下不涉及具体设备动作参数。

**第一步，建立身份关联。** 扫码结果、入口位置和当次到达事件要能够对应同一搬运单元。重复扫码、无法识别和物理存在但身份未知，应有不同的处理状态。光电只能提供存在证据，不能替代身份识别。

**第二步，做业务校验。** WMS 检查收货依据、货主、物料、批次和质量要求。收货与上架完成是不同业务事件：库存可能在收货时已建立，位置暂为收货区，后续上架改变位置或状态，不能一概说到达货架才“增加库存”。

**第三步，决定业务目标。** WMS 或项目指定的仓储决策模块分配目标库位和任务。业务库位预留表示它暂不再分给其他货物，不证明该位置此刻物理上为空；有效占用、异常遗留货物和数据陈旧仍需检查。

**第四步，形成可执行路径。** 执行协调层把业务目标映射到交接点、通道和资源，按当前设备能力安排搬运。WMS 任务、运输任务与设备命令应能追溯关联，但不必强制成为所有项目都遵守的一棵固定三级任务树。

**第五步，完成现场交接。** 局部控制器依据设备状态与联锁推进动作。上层收到“完成”时，还应知道完成的是哪条命令、哪个交接点及什么判据。Beckhoff 的 MC_MoveAbsolute 文档区分 Busy、Active 和 Done，Done 的产生还受位置监视配置影响；这些信号不能直接翻译成“托盘已经正确入账”。[MC_MoveAbsolute](https://infosys.beckhoff.com/content/1033/tcplclib_tc2_mc2/70094731.html)。

**第六步，确认业务结果。** 执行协调层结合身份、目标交接和异常信息生成可核查的搬运结果，WMS 按业务规则确认上架。不能确认时，应保留待核实状态及后续处理入口，而不是用一个超时错误覆盖所有物理可能性。

以下是本文为该案例建议的权威记录划分：

| 信息 | 建议的权威记录方 | 其他系统所持信息 |
| --- | --- | --- |
| 仓储库存、业务位置、有效可用状态 | WMS；引用主数据和质量域的有效结论 | 执行侧保存必要副本 |
| 原始位置、存在、压力等观测 | 对应传感器、驱动器或采集源 | 上层收到的测量副本 |
| 位置估计、搬运单元跟踪 | 明确指定的定位或区域控制模块 | 其他模块保存带时效的估计 |
| 各层任务及其状态 | 各自任务管理者 | 通过关联标识形成全程视图 |
| 资源预约 | 指定的资源仲裁者 | 设备侧获取获准使用的范围 |
| 实际占用判定 | 指定的区域控制器，结合现场证据维护 | 调度器保存占用副本并处理不确定性 |
| 设备内部状态 | 对应控制器或驱动器 | 平台保存报告及其时间、质量 |

“权威”指谁负责记录与裁决，不意味着它永不出错。实际物理状态也不是一个数据库写入者。Swisslog 对子系统库存和仓库级库存的区分，恰好说明记录范围必须讲清楚，不能只凭系统缩写选唯一真相。[Swisslog 软件说明](https://www.swisslog.com/en-us/products-systems-solutions/software-inventory-management)。

## 四、共享提升机：预约为什么不是占用

### 4.1 先把设备假设说清楚

本例的提升机承载“车与托盘”的整体，每次只服务一个这样的搬运单元；各层等待区和接收区容量有限。它不同于只搬托盘、车辆留在原楼层的方案。车辆调度、轿厢占用和交接边界都随这一假设改变。

预约是对未来使用机会的承诺；准入是当前是否允许进入；占用则表示设备或载荷已经进入受控空间。这三者不能压成同一个锁字段。

本文建议把轿厢、入口交界和目标接收空间一起纳入编排。调度器可以提出跨层计划，但是否满足载荷、对接、防护及相关联锁条件，必须由设计指定的现场控制链确认。具体条件及安全属性来自设备设计和风险评估，不由本文的通信模型替代。

### 4.2 握手要覆盖交接边界，而不只是“我完成了”

一个完整的交接契约，应表达请求对象、目标层、许可范围、控制权代次，以及进入、接收、退出和清空的确认。传输方式可以是消息或映射后的控制信号，语义不能因此丢失。

目标接收空间应在准入前纳入可行性判断和必要预留；到层后仍要复核现场条件。前者减少“车已进轿厢却无处可去”，后者防止把过时预约当成当前事实。

<div style="overflow-x: auto;" tabindex="0" role="region" aria-label="流程图，可横向滚动">
<div style="min-width: 480px;">

{{< mermaid >}}
flowchart TD
    A["申请跨层任务"] --> B["预约轿厢与接收空间"]
    B --> C{"现场允许进入？"}
    C -->|是| D["进入握手与占用确认"]
    D --> E["跨层运输"]
    E --> F{"到层后允许交接？"}
    F -->|是| G["退出与目标接收确认"]
    G --> H{"轿厢及交界已清空？"}
    H -->|是| I["释放已清空的轿厢与交界"]
    C -->|否| X["等待或恢复：保留已知状态"]
    F -->|否| X
    H -->|否| X
{{< /mermaid >}}

</div>
</div>

这是一张建议性的协作状态图，不是设备操作步骤。遇到状态不明，平台可以暂停受影响区域的新分配；现有动作如何继续、停止或保持，由现场控制和安全设计决定。车驶出轿厢时也可能仍跨占两个区域，不能在消息发送瞬间机械地转移占用记录。目标接收区在车辆驶入后转为实际占用，仍按自身的离开与清空判据维护，不能随轿厢释放而一起清空。

### 4.3 死锁、饥饿与利用率是同一个问题的不同侧面

假设车辆 A 占着接收口等待提升机，车辆 B 占着提升机等待这个接收口，此时又没有可用退避位或替代交接路径，双方就可能陷入互相等待：各自需要的下一项资源，都被对方占着。

可选策略包括统一资源申请顺序、在进入前获得关键后继空间、限制同时进入瓶颈区域的任务，或保留恢复通道。代价可能是提前占用容量、降低并行度或增加等待。等待图中的环是重要风险信号，但在多容量资源或存在替代路径时，不能仅凭有环就证明死锁；还要分析剩余容量与可完成动作。

优先级老化、配额和等待时间约束可以缓解饥饿，但不能让故障阻断或物理上不可行的任务获得进展。长期超出瓶颈能力的任务流，应通过任务释放和上游背压管理，不能靠不断提高优先级解决。

### 4.4 租约结束，不会让轿厢里的车消失

软件预约可以有期限，实际占用不能因租约到期被盲目清除。即使尚未进入，也必须确认旧准入许可已失效、在途或已排队的旧命令不会随后触发进入，才能把相关使用权重新分配。

控制权切换可以引入单调的控制代次，要求执行边界拒绝旧代次请求。但它只有在所有相关控制路径都检查、重启后规则仍可靠时才有效；新代次不能撤销已经接受或正在进行的动作。随机启动标识与控制权代次也不是同一概念。

故障恢复需要同时对账任务、许可、占用和现场观测。无法建立可信对应关系时，应把资源标为待核实，交由经设计的恢复流程处理，而不是创建一条新命令试试看。

## 五、命令接受，不等于执行完成

### 5.1 先定义状态语义，再选消息中间件

本文建议每条搬运命令具有稳定的 `commandId`，并明确其唯一性范围、不可变请求内容、执行目标及记录保留期限。同一标识再次出现，应关联同一个执行实例；相同标识却带不同目标，应产生请求冲突，而不是悄悄修改原动作。

状态含义可以约定为：

- **Accepted**：执行方已按契约受理。是否已持久保存、是否已排队，必须明确；它不表示动作开始。
- **Started**：达到契约定义的开始边界，不一定等同于负载刚刚移动。
- **Completed**：满足约定的完成判据，不是消息已收到，也不自动等于库存业务完成。
- **Rejected**：本次请求未被接受。重复请求被拒绝，不证明同一标识的原命令未执行。
- **Failed**：执行未按目标成功结束，但可能已经造成部分移动和占用。
- **Canceled**：取消已成立，并报告实际终止结果。它不保证返回原位或资源为空。

取消请求被接受，只表示取消正在处理。它与完成存在竞争：请求到达时，原动作可能已经完成。接口必须允许返回真实完成结果，不能为了迎合调用方而把成功动作改写为“已取消”。

<div style="overflow-x: auto;" tabindex="0" role="region" aria-label="流程图，可横向滚动">
<div style="min-width: 650px;">

{{< mermaid >}}
flowchart TD
    R["提交命令"] --> A["Accepted"]
    R --> J["Rejected：本次请求"]
    A --> S["Started"]
    S --> D["Completed"]
    A -->|取消成立| C["Canceled"]
    S -->|取消成立| C
    A --> F["Failed"]
    S --> F
    R -.-> U["观测超时：结果未知"]
    A -.-> U
    S -.-> U
    U --> Q["查询、对账与恢复"]
{{< /mermaid >}}

</div>
</div>

图同时展示请求受理结果与命令执行状态。Rejected 仅针对本次请求，不代表同一标识的原命令进入拒绝终态。虚线表示观察方失去确定信息，超时也可能发生在收到受理回执之前。取消转移是否可用及成立条件，由设备能力和契约规定；观察方不一定依次看见所有中间状态。

### 5.2 超时回答的是通信问题，不是物理问题

Java 服务调用超时，可能是请求没到、仍在排队、正在执行，也可能是动作已完成但回执丢失。OPC UA 的 Call 服务规范甚至明确指出：会话终止时，结果可能被丢弃，而服务器上的任务是否已经执行是独立问题。[OPC UA Call，1.05.07](https://reference.opcfoundation.org/specs/OPC-10000-4/5.12.2)。

因此，“查询显示尚未开始，就可以重发一条新命令”仍然不充分。查询之后原命令可能开始；重启后的 NotFound 也可能只是历史记录丢失。

按同一标识重放请求，只有在执行方的去重范围、保留期限、持久化和重启恢复契约仍成立时，才能用于返回原执行状态或继续关联同一实例。若这些条件不明，应先对账。要创建新的执行命令，则必须解决旧命令是否还可能执行，并重新检查当前物理前提。

### 5.3 消息幂等解决不了全部物理一致性

消息去重可以避免重复记账；若去重仅发生在上层数据库，下游仍可能重复接受动作。即使执行方记住标识，保存记录、产生输出和机械响应之间，也不能自动形成数据库意义上的原子事务。

Outbox 可以缩小业务提交与消息发送之间的缺口，不能让数据库与真实运动共同提交。物理动作“恰好一次”不能从消息中间件的承诺直接推出。

恢复记录至少应能关联命令、启动代次、事件序号、执行阶段和交接结果。去重记录的保留期限要覆盖可能的重投与恢复窗口；不能一边清理历史，一边继续承诺无限期幂等。对账证实原命令满足完成判据，且托盘身份、目标位置与交接证据一致后，才能按业务规则补齐确认。仅发现目标位置有货，不足以完成原任务或更新可用库存。位置不明时应先恢复可信事实；需要搬回原处时，那是新的受约束动作，不是数据库回滚。

## 六、普通联锁与功能安全属于不同工程责任

“通道占用时不再派车”可以是普通控制逻辑；如果防碰撞被分配了降低人员风险的责任，它又可能成为安全相关功能的一部分。区分依据是风险评估与功能分配，不能仅按软件层次或“保护人、保护设备”二分。

安全功能涉及检测、通信、逻辑求解、输出和最终执行元件的完整链路，并受到响应时间、故障模式、诊断能力和配置的约束。认证组件只有在适用版本、用途和集成条件内才有意义，也不意味着链路中每个零件都必须各有一张同类证书。Siemens 的功能安全管理指南将风险评估、设计及验证放在完整生命周期中讨论，它不是任何具体立库的符合性证明。[Siemens 功能安全管理指南，V1.0](https://assets.new.siemens.com/siemens/assets/api/uuid:bed06495-8796-4a94-a11c-e1f50a85447c/guideline-functional-safety-management.pdf)。

提升机尤其不能混淆 STO、机械制动和悬挂负载保持。Beckhoff AX5805/AX5806 手册明确指出，STO 使驱动失去产生转矩的能力，并不对电机实施制动；惯性可能使其继续运动，悬挂负载还可能使其加速，需要相应外部安全措施。安装一个制动器，也不能跳过适用性和完整功能验证。[AX5805/AX5806 手册，2.0.0](https://download.beckhoff.com/download/document/automation/twinsafe/ax5805_ax5806en.pdf)。

普通 PLC 程序、云服务或 AI 输出，都不会因“运行正常”而自动取得安全资格；软件和虚拟化也并非原则上不能用于安全。判断必须落到具体产品、认证范围及完整系统验证上。

## 七、统一平台真正改变的是能力的组织方式

统一平台值得做的原因，是让跨设备、跨项目的共同问题形成可复用能力。收益来自一致语义与可验证边界，不来自把所有程序合成一个进程或数据库。

### 7.1 统一身份、布局和拓扑：让同一对象能够被不同系统认出

托盘标识、车号、PLC 地址、设备名称和库位编码，可能来自不同命名空间。平台应建立稳定身份及映射，说明哪个系统负责创建、变更和停用。

布局还要区分几何位置、可达拓扑与资源冲突。画面上相邻，不等于能够直接通行；线路相交，也不一定处于同一高度。拓扑边应表达方向、设备能力、容量和适用条件，并与布局版本关联。

收益是路径、看板和仿真可以复用同一套对象关系。代价是持续治理主数据、映射和变更；成立条件是明确变更权威，并防止一张过期布局悄悄成为控制依据。图形编辑完成不能直接等同于现场配置已生效。

### 7.2 统一命令和事件：从碎片日志走向可对账事实

共同契约要统一身份、目标、版本、状态、错误分类和关联关系，同时保留各设备能力差异。原始报告、经过校验的交接事实和上层推断，应分别表达；纠错事件应能追溯原记录，不能修改历史后假装从未发生异常。

这样才能回答“一块托盘为何在这里”，而不只回答“某个服务打印了什么”。成本是事件模式演进、数据保存、时间质量管理与对账逻辑。成立条件是各方对同一状态词具有可检验的理解，而不只是使用同一 JSON 字段。

### 7.3 可插拔适配器：把对接程序变成明确的能力边界

适配器除了翻译协议，还要声明设备是否支持查询命令结果、取消、重启识别、占用报告和重复请求关联。

缺少取消能力，就不能伪造 Cancel 成功；设备不能保留历史，也不能由适配器对外承诺跨重启的无限期幂等。某些能力可以通过额外存储和握手补足，但必须说明补足范围及仍存在的窗口。

收益是新设备接入时复用上层业务与调度；成本是兼容矩阵、协议版本和异常行为的持续维护。只有能力被测试并纳入契约，接口统一才具有工程价值。

### 7.4 调度或 WES：从局部忙碌走向约束下的协调

让每台车都满负荷运行，可能把提升机入口堵死。统一调度可以同时观察目的层容量、瓶颈资源和业务优先级，通过任务释放、批次组织和背压，减少无效等待。

这里的目标是改善选定指标，并满足容量、公平性与恢复要求，不是宣称实时获得全局最优解。收益成立需要状态足够可信、模型覆盖关键约束，且现场能执行计划；代价是计算复杂度、模型维护和中心故障影响。通信中断后允许哪些本地动作继续，也应预先定义。

厂商将这类能力命名为 WES、WCS 或 MFS，并不改变必须回答这些问题的事实。

### 7.5 统一可视化：让人看见信息的可信程度

统一画面应把库存、任务、命令、设备和告警关联起来，同时区别实测、推断、动画插值与未知状态。若画面只是按计划播放车辆运动，它展示的是预测，不是实时定位。

收益是缩短定位问题与核实交接的路径；成本是采集、存储、关联规则及访问控制。看板一致不等于现场一致，关键还在于每个结论能否追溯到证据。

### 7.6 版本化配置与数字孪生：复用之前，先定义可验证的模型

把布局、设备能力、规则和适配版本变成受控配置，可以减少复制项目代码的分叉。前提是变化可以在模型中准确表达，而不是把大量例外隐藏进配置脚本。上线与回退还要处理在途任务、旧命令和占用，不能仅回滚数据库版本。

仿真可以用于比较任务释放策略、缓冲容量或瓶颈变化，但“仿真跑出了多少”是模型输出。本文建议用实际到达分布、动作时间、故障与恢复数据校准，再用未参与校准的观测验证；缺少实测时，应公开假设并分析敏感性。

验证范围要明确：验证了物流流程和排队，不等于验证了机械碰撞、制动距离或安全功能。数字孪生也需要持续的数据映射与模型维护。它能帮助提出更好的决策，不能把预测吞吐自动升级为实测成绩，更不能凭产品宣传推导 ROI。

## 八、渐进实施：每一步都留下验收依据

以下是建议的实施与验收顺序，不是本文已经实施的测试。

1. **梳理权威来源和边界。** 为库存、位置、任务、许可、占用和设备状态确定记录者与恢复责任。验收依据是正常、断网、重启和人工介入场景都能找到裁决方，不存在两套系统同时独立授予同一排他使用权。
2. **统一身份与最小契约。** 先贯通一类托盘任务，明确命令和事件语义。检查重复标识、内容冲突、回执乱序、保留期届满和旧代次消息的处理，不只看正常路径跑通。
3. **建立观测与对账。** 先利用现有执行系统建立证据链。验收关注能否区分已完成、未完成与未知，能否定位账实差异；不能用“日志都收到了”代替。
4. **逐步接入适配器并交接控制权。** 切换前明确原控制者、在途命令、准入许可和占用的处置，验证新旧路径不会对同一排他控制边界产生未经协调的并行命令权，并确认已接受的命令、许可和占用按交接规则延续。部署新服务本身不构成控制权交接。
5. **从一个稀缺资源开始调度。** 例如本例提升机，检查互斥、有限容量、长期等待和故障恢复。特别验证实际占用不会因超时消失，旧许可不会在重新分配后突然生效。
6. **最后引入仿真和优化。** 先形成可信基线，再在相同统计边界和业务条件下比较结果。报告模型误差、排除场景与观测范围，不凭一次演示推算长期收益。

我的判断是：先把事实讲清楚，再让平台做更多决定。事实和契约不可靠时，更复杂的优化只会更快地产生无法执行的计划。

## 九、独立扩展：从 PLC 到虚拟 PLC

### 9.1 逻辑、运行时、硬件与 I/O 是四件事

PLC 逻辑描述控制行为；运行时负责执行任务、管理状态及接口；计算硬件提供处理器和存储；I/O 与现场网络连接传感器、驱动器和其他设备。它们可以封装在一台设备里，也可以部分分离。

| 形态 | 通常关注的部署方式 | 不能据名称推断的能力 |
| --- | --- | --- |
| 传统硬 PLC | 厂商控制器中预集成硬件、运行时与接口 | 任意型号都具备相同实时、运动或安全能力 |
| PC-based／软 PLC | 控制运行时部署在工业 PC 或受支持操作系统上 | 普通 PC 配置天然满足确定性要求 |
| 虚拟 PLC | 运行时以虚拟机、容器等形式部署和管理，定义随厂商而变 | 支持任意宿主机、热迁移或无中断恢复 |
| 仿真 PLC | 为程序调试、测试和联合仿真模拟控制器 | 自动具备真实生产控制的支持资格 |

这些分类有交集。虚拟 PLC 仍然是软件运行时，也可能属于 PC-based 控制；“仿真”主要说明用途，并不由是否运行在 PC 上决定。Siemens 将 S7-PLCSIM Advanced 用于控制程序测试及与设备模型联合仿真，这不能替代生产运行时的适用性证明。[S7-PLCSIM Advanced](https://developer.siemens.com/s7-plcsim-advanced/overview.html)。

本轮核对时，CODESYS 三个相关商店页面显示当前版本均为 **4.23.0.0**，但支持范围不同：

- **Control for Linux SL** 被描述为软实时控制器，页面明确未发布支持容器或虚拟机；许可证可绑定设备的软件许可容器，或存储在硬件密钥中。
- **Virtual Control SL** 面向容器部署，要求受支持的 Linux 环境及相应容器、实时和许可条件；许可证服务器可位于本机或局域网。
- **Virtual Safe Control SL** 是另一项安全产品，不能把它的资格附会到普通 Virtual Control SL。

对应官方页面：[Control for Linux SL](https://store.codesys.com/en/codesys-control-linux-sl-1.html)、[Virtual Control SL](https://store.codesys.com/en/codesys-virtual-control.html)、[Virtual Safe Control SL](https://store.codesys.com/en/codesys-virtual-safe-control.html)。版本号只是本次访问快照，部署仍需核对具体组合。

### 9.2 收益来自生命周期管理，条件落在整个平台

运行时与专用控制器硬件解耦后，可以统一分发版本、管理配置，并在满足隔离条件时复用计算资源。代价是操作系统、实时配置、网络、许可服务和平台运维也进入控制系统责任范围。

实时性的核心是任务在规定时限内得到执行并产生有效结果。抖动是实际周期或响应时间相对目标的偏差、波动；平均 CPU 利用率低，不代表最坏响应满足要求。输入数据年龄、任务调度、总线交换和驱动器处理，都可能影响端到端响应。

产品条件必须逐项看：

- **Siemens CPU 151xV(F) 2.2.1** 文档要求相关 Industrial Edge 环境具备资源管理能力，为控制应用隔离 CPU 核；其虚拟设备的 ESXi 配置还规定 CPU 资源预留、客户机内存预留及延迟敏感度等条件。文档明确：应用能部署成功，不等于已经获得实时控制能力。[实时配置要求](https://docs.tia.siemens.cloud/r/en-us/2.2.1/s7-1500-virtual-controller-cpu-151xv-f/required-settings/hypervisor-settings-for-real-time-computation-and-communication/setting-the-configuration-required-for-deterministic-real-time-behavior)。
- **CODESYS 的 Linux 性能指南** 要求针对实时需求使用 RT Preempt，并指出普通低延迟内核不足以保证确定性。这不意味着装上内核补丁后任意产品和硬件组合都获得支持。[性能优化指南](https://content.helpme-codesys.com/en/CODESYS%20Control/_rtsl_performance_optimization_linux.html)。
- **Beckhoff TwinCAT 3** 的系统要求强调硬件、BIOS、操作系统、驱动与软件的整体条件，并列出 Hyper-V 对相关实时运行配置的限制；用户模式运行也不能自动解释成实时执行。[TwinCAT 3 系统要求](https://infosys.beckhoff.com/content/1033/tc3_overview/6162419083.html)。

容器提供打包与隔离机制，本身不保证硬实时。验收应在支持配置及代表性的负载、通信和故障情形下检查响应与抖动；有限测试中观察到的最大值，也不等于数学意义上的最坏情况保证。

### 9.3 I/O 与运动能力不会随程序一起自动迁移

程序能够运行，不代表原现场总线、接口卡、同步机制和运动功能都能沿用。必须核对运行时、驱动、网卡、I/O 设备及固件组合。

Siemens 2.2.1 文档的 **CPU 151xV** 功能比较表明确列出不支持 PROFINET IRT、中央 I/O 与可插拔接口，以及 Basic／Standard／Advanced motion control 等项目。不能因为文档总标题带有“(F)”，就把表中结论任意扩展到其他型号或版本；也不能把这些限制改写成“不能向任何独立运动设备发送命令”。[功能比较表](https://docs.tia.siemens.cloud/r/en-us/2.2.1/s7-1500-virtual-controller-cpu-151xv-f/product-limitations/comparison-to-simatic-s7-1500-hardware-cpus?contentId=xQFb~858L_GUY2nJgTEd8Q)。

支持矩阵还包括工程软件、Edge 环境与基础平台版本。Siemens 的兼容性页面给出的参考平台结果具有明确测试条件，不能当成任意服务器的性能保证。[兼容性说明](https://docs.tia.siemens.cloud/r/en-us/2.2.1/s7-1500-virtual-controller-cpu-151xv-f/installing-and-getting-started/compatibility-information)。

### 9.4 多实例、冗余与热迁移是不同命题

同机容器通常共享宿主内核；虚拟机具有各自的客户机内核。二者仍可能共享物理主机、电源路径、宿主系统或虚拟化管理层等故障来源。网卡和权限是否隔离取决于配置，不能笼统说所有实例都共用一块网卡或同一权限。

把两个实例放在同一服务器，并不自动获得独立故障域。容器重启也不等于控制冗余。CODESYS Redundancy 文档要求受支持设备、控制器间直接连接，并在周期内完成同步，说明冗余需要专门的运行时与通信设计。[CODESYS Redundancy](https://www.codesys.com/products/runtime/redundancy/)。

热迁移还涉及暂停时间、设备绑定、网络身份、输出控制权和许可，没有相应产品支持及验证，不能从虚拟机平台的迁移能力推导 PLC 可不停机迁移。Siemens 2.2.1 文档列出的 Edge 设备更新、应用更新、设备重启及网络配置变更会使 CPU 151xV 进入 STOP，也提醒我们管理便利不等于无中断维护。[STOP 场景](https://docs.tia.siemens.cloud/r/en-us/2.2.1/s7-1500-virtual-controller-cpu-151xv-f/installing-and-getting-started/stop-mode-scenarios?contentId=6YI~oMJTp2XaZgYmMHfp8g)。

### 9.5 崩溃后恢复的是记录，现场可能已经继续变化

Siemens 2.2.1 的循环保持数据备份，从最近成功写入的数据恢复。文档不仅指出最后一个 OB1 周期的数据可能缺失，也指出写入跳过时，恢复数据可能落后多个周期；其他 OB 或通信并发修改，还可能影响数据一致性。[崩溃后的行为](https://docs.tia.siemens.cloud/r/en-us/2.2.1/s7-1500-virtual-controller-cpu-151xv-f/additional-features/cyclic-retentive-backup/behavior-after-system-crashes?contentId=V_VebsWUCk5HBXiQBe8a1g)；[备份限制](https://docs.tia.siemens.cloud/r/en-us/2.2.1/s7-1500-virtual-controller-cpu-151xv-f/additional-features/cyclic-retentive-backup/limitations-to-cyclic-retentive-backup?contentId=MmBgREplsC9y_HQcWg6jAg)。

因此，恢复一个“动作未完成”的变量，不证明动作没有完成；恢复一个“资源空闲”的变量，也不能覆盖现场占用。前文的身份、代次、命令关联和对账机制，在虚拟 PLC 中仍然需要。

### 9.6 离线自治要同时考虑控制、许可与管理依赖

本地运行时可以被设计为在上层业务网络中断后继续有限范围的现场控制，但“断网”必须区分 WMS 链路、管理网络、许可链路和现场总线。前者中断后的自治能力，不代表其他链路都可无限期缺失。

以 **Siemens CPU 151xV(F) 2.2.1 的许可文档**为例：

- 下载配置后若尚未获得许可，可进入 RUN 两小时；条件仍不满足时进入 STOP，在成功取得许可前不能重新进入 RUN。
- 曾获得许可、随后授权连接失败，才适用七天宽限；按实际经过的时间计时，PLC 不在 RUN 也不暂停。到期仍未恢复授权条件，原处于 RUN 的 PLC 会进入 STOP，恢复许可授予条件前不能重新进入 RUN。
- 宽限期间改变运行中 CPU 的硬件配置，会失去七天扩展宽限，回到默认两小时规则。

这组限制说明该版本的离线能力受到授权基础设施约束，不能推广成所有 Siemens PLC 的规则。[许可要求及宽限条件](https://docs.tia.siemens.cloud/r/en-us/2.2.1/s7-1500-virtual-controller-cpu-151xv-f/licensing/licensing-requirements-for-the-virtual-controller?contentId=OQU_OdUzCXsfGLdplJjP9g)。

CODESYS Virtual Control SL 可使用本机或局域网许可服务器，同样需要设计许可服务故障与恢复行为，不能仅凭“本地服务器”宣布永久离线可用。

网络安全方面，宿主机、镜像、运行时、管理接口和账号权限都进入维护范围。网络隔离、访问控制、补丁与受控变更，应和控制可用性一起设计；平台管理员的权限也可能影响多个控制实例。[Siemens Industrial Edge App 安全要求](https://docs.tia.siemens.cloud/r/en-us/2.2.1/s7-1500-virtual-controller-cpu-151xv-f/introduction/security-information-for-the-industrial-edge-app?contentId=_OehZswJu01asQKkPAu7UQ)。

### 9.7 虚拟化不取消安全边界，也不改变物流职责

CODESYS Virtual Safe Control SL 产品页声明其安全功能基于经过认证的 Safe Control Core，并列出容器、实时系统及独立时间源等条件；官方帮助同时要求依据随产品交付的 `H1 – SafetyManual.pdf` 判断安全与认证限制。[产品条件](https://store.codesys.com/en/codesys-virtual-safe-control.html)；[官方安全文档入口](https://content.helpme-codesys.com/en/CODESYS%20Control/_rtsl_codesys_virtual_safe_for_linux_control_sl.html)。

本次未取得该安全手册及对应认证附件，因而只引用产品方的声明与文档要求，不对具体配置或完整安全链路作符合性判断。

虚拟 PLC 改变的是控制运行时的承载、部署和管理方式。库存权威、资源仲裁、现场联锁和安全责任仍需明确分工；是否由 Java 服务承担某个逻辑职责，取决于接口、实时和故障边界，而不是 PLC 运行在机箱里还是服务器上。

统一平台成熟的标志，是同一块托盘在业务账、执行任务和现场证据之间始终能够解释、对账与恢复。把控制运行时虚拟化，可以成为这条演变路线的一部分，但不能替代这条路线本身。

## 来源与主张边界

下列链接均于 **2026-10-24** 打开核对；访问日期不代表页面发布日或认证生效日。未标明确切发布版本的网页按访问时内容引用。厂商概览只用于说明其产品定位，不用于证明性能、ROI 或行业统一标准。

| 主张 | 官方来源 | 支持范围、版本与局限 |
| --- | --- | --- |
| 库存属性与仓内移动有不同语义 | [SAP Quant](https://help.sap.com/saphelp_ewm900/helpdata/en/4b/c49585921f4c88e10000000a42189e/content.htm?no_cache=true) | EWM 9.0 归档；产品实例，不是统一数据模型 |
| 可集成 MFS，不必另设独立 WCS；控制层可跟踪搬运单元 | [SAP MFS](https://learning.sap.com/courses/resource-management-in-sap-ewm/integrating-a-material-flow-system) | SAP 方案；不据此证明 PLC 普遍自主路由 |
| WCS／WES／MFS 命名与职责范围存在厂商差异 | [SSI SCHAEFER](https://www.ssi-schaefer.com/en-us/software/wamas-software-solutions/wcs-warehouse-control-system)、[Dematic](https://www.dematic.com/en-us/software/)、[Swisslog](https://www.swisslog.com/en-us/products-systems-solutions/software-inventory-management) | 产品概览；不证明通用术语等价或效果 |
| 测量有不确定度，滤波与时间语义受产品条件影响 | [Beckhoff 不确定度](https://infosys.beckhoff.com/content/1033/elm3xxx/1678264203.html)、[ELM300x](https://infosys.beckhoff.com/content/1033/elm3xxx/3642134923.html) | 一般测量解释及具体端子资料；不外推全部传感器 |
| 值、状态码和时间戳应区分；调用失联不能证明未执行 | [OPC UA DataValue](https://reference.opcfoundation.org/specs/OPC-10000-4/7.11)、[Call](https://reference.opcfoundation.org/specs/OPC-10000-4/5.12.2) | Part 4，1.05.07；不提供本文搬运命令的幂等契约 |
| 任务周期与 I/O 交换不能简单等同 | [CODESYS Task](https://content.helpme-codesys.com/en/CODESYS%20Development%20System/_cds_f_reference_task.html)、[Bus Cycle](https://content.helpme-codesys.com/en/CODESYS%20Development%20System/_cds_buscycle_task.html) | 在线帮助；具体设备和总线仍需核对 |
| 位置闭环可在不同层，完成信号有配置条件 | [AX5000](https://infosys.beckhoff.com/content/1033/ax5000_system_doku_hw2/648069771.html)、[Tc2_MC2](https://infosys.beckhoff.com/content/1033/tcplclib_tc2_mc2/70094731.html) | 特定驱动及功能块；不直接证明托盘交接完成 |
| 气缸能量链及伺服气动闭环 | [Festo 气缸](https://www.festo.com/us/en/c/products/actuators-and-drives/pneumatic-cylinders/piston-rod-cylinders-id_pim215)、[伺服气动](https://www.festo.com/us/en/c/products/actuators-and-drives/servopneumatic-positioning-systems-id_pim185) | 产品原理；不推导具体立库选型 |
| 功能安全需完整生命周期；STO 不保持悬挂负载 | [Siemens FSM](https://assets.new.siemens.com/siemens/assets/api/uuid:bed06495-8796-4a94-a11c-e1f50a85447c/guideline-functional-safety-management.pdf)、[Beckhoff 手册](https://download.beckhoff.com/download/document/automation/twinsafe/ax5805_ax5806en.pdf) | FSM V1.0，2020-09；AX5805/AX5806 2.0.0，2023-03-23；非本例安全符合性证明 |
| 仿真 PLC 用于测试与联合仿真 | [S7-PLCSIM Advanced](https://developer.siemens.com/s7-plcsim-advanced/overview.html) | 产品概览；不证明生产 I/O、实时或安全适用性 |
| CODESYS 不同运行时具有不同部署和许可条件 | [Linux SL](https://store.codesys.com/en/codesys-control-linux-sl-1.html)、[Virtual SL](https://store.codesys.com/en/codesys-virtual-control.html) | 页面当前版本均为 4.23.0.0；不可跨产品套用 |
| 实时能力需要底层配置 | [Siemens 实时要求](https://docs.tia.siemens.cloud/r/en-us/2.2.1/s7-1500-virtual-controller-cpu-151xv-f/required-settings/hypervisor-settings-for-real-time-computation-and-communication/setting-the-configuration-required-for-deterministic-real-time-behavior)、[CODESYS Linux 指南](https://content.helpme-codesys.com/en/CODESYS%20Control/_rtsl_performance_optimization_linux.html)、[TwinCAT 系统要求](https://infosys.beckhoff.com/content/1033/tc3_overview/6162419083.html) | Siemens 文档 2.2.1；其余按页面配置范围，非任意硬件保证 |
| CPU 151xV 存在 I/O、运动与平台限制 | [功能比较](https://docs.tia.siemens.cloud/r/en-us/2.2.1/s7-1500-virtual-controller-cpu-151xv-f/product-limitations/comparison-to-simatic-s7-1500-hardware-cpus?contentId=xQFb~858L_GUY2nJgTEd8Q)、[兼容性](https://docs.tia.siemens.cloud/r/en-us/2.2.1/s7-1500-virtual-controller-cpu-151xv-f/installing-and-getting-started/compatibility-information) | 2.2.1；按表中具体型号与支持组合解释 |
| 专用冗余与普通实例重启不同；维护可能导致 STOP | [CODESYS Redundancy](https://www.codesys.com/products/runtime/redundancy/)、[Siemens STOP](https://docs.tia.siemens.cloud/r/en-us/2.2.1/s7-1500-virtual-controller-cpu-151xv-f/installing-and-getting-started/stop-mode-scenarios?contentId=6YI~oMJTp2XaZgYmMHfp8g) | 前者受设备与周期同步约束；后者为 2.2.1；均不证明任意热迁移 |
| 保持数据恢复可能滞后现场 | [崩溃恢复](https://docs.tia.siemens.cloud/r/en-us/2.2.1/s7-1500-virtual-controller-cpu-151xv-f/additional-features/cyclic-retentive-backup/behavior-after-system-crashes?contentId=V_VebsWUCk5HBXiQBe8a1g)、[备份限制](https://docs.tia.siemens.cloud/r/en-us/2.2.1/s7-1500-virtual-controller-cpu-151xv-f/additional-features/cyclic-retentive-backup/limitations-to-cyclic-retentive-backup?contentId=MmBgREplsC9y_HQcWg6jAg) | CPU 151xV(F) 文档 2.2.1；不代表其他 PLC 的保持机制 |
| 许可宽限影响离线运行；宿主与管理环境需要安全维护 | [Siemens 许可](https://docs.tia.siemens.cloud/r/en-us/2.2.1/s7-1500-virtual-controller-cpu-151xv-f/licensing/licensing-requirements-for-the-virtual-controller?contentId=OQU_OdUzCXsfGLdplJjP9g)、[安全要求](https://docs.tia.siemens.cloud/r/en-us/2.2.1/s7-1500-virtual-controller-cpu-151xv-f/introduction/security-information-for-the-industrial-edge-app?contentId=_OehZswJu01asQKkPAu7UQ) | 2.2.1；宽限须满足各自触发条件，不是通用 PLC 规则 |
| 虚拟安全产品存在，但具体安全资格须按完整资料判断 | [Virtual Safe SL](https://store.codesys.com/en/codesys-virtual-safe-control.html)、[安全帮助](https://content.helpme-codesys.com/en/CODESYS%20Control/_rtsl_codesys_virtual_safe_for_linux_control_sl.html) | 页面当前版本 4.23.0.0；未取得 H1 安全手册及认证附件，不作配置符合性承诺 |
| 权威划分、提升机契约、控制权交接、平台路线与仿真验收 | 本文基于上述边界提出的架构建议 | 通用假设与设计判断；未进行设备、运行时、吞吐或站点构建测试 |
