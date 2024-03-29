## 超融合架构与容器超融合

文/刘爱贵

从辩证角度看，任何事物都不可能是完美的，超融合是不是也存在一些问题或局限性呢？超融合有适用场景，但肯定不是普遍适用的。因此，这篇文章我想换一个角度来看超融合，找找茬，梳理一下超融合，也算是为大家选择超融合架构方案提供一个参考。容器这么火，不谈好像也不大好，最后简单胡侃一下容器超融合。

### 什么是超融合？

超融合（Hyper-Converged）概念最早由谁提出无从考证，目前还没有严格的标准定义，各厂商和机构都有各自的定义，这也佐证了超融合仍然处于快速发展演变当中，并未形成统一的标准规范。

超融合中“超”是什么含义？我最开始以为是超人的 Super，结果差点闹出笑话。实际上，“超”对应英文“Hyper-Converged”中的“Hyper”，特指虚拟化，对应虚拟化计算架构，比如 ESXi/KVM/XEN/Hyper-V。这一概念最早源自 Nutanix 等存储初创厂商将 Google/Facebook 等互联网厂商采用的计算存储融合架构用于虚拟化环境，为企业客户提供一种基于 x86 硬件平台的计算存储融合产品或解决方案。不难看出，超融合架构中最根本的变化是存储，由原先的集中共享式存储（SAN/NAS）转向软件定义存储，特别是分布式存储（包括 Object/Block/File 存储），比如 NDFS/VSAN/ScaleIO/SSAN。因此基于这个“超”，数据库一体机和大数据一体机都不能划为超融合的范畴，除非 RAC/Hadoop 等应用跑在虚拟化之上。还有一点，超融合中的软件定义存储通常是分布式存储，ZFS 虽然属于 SDS 范畴，但基于 ZFS 构建的计算存储融合系统，严格意义上不能称为超融合架构。

我们再来看“融合”又是什么含义？超融合这个概念当前似乎被神化了。简单地讲，融合就是将两个或多个组件组合到一个单元中，组件可以是硬件或软件。就虚拟化和私有云而言，按照是否完全以虚拟化为中心，我个人把融合分为物理融合和超融合两种。超融合是融合的一个子集，融合是指计算和存储部署在同一个节点上，相当于多个组件部署在一个系统中，同时提供计算和存储能力。物理融合系统中，计算和存储仍然可以是两个独立的组件，没有直接的相互依赖关系。比如，SSAN+oVirt 方案，一个节点的 Redhat/CentOS 系统上 SSAN 和 oVirt 物理融合，共享主机的物理资源。超融合与物理融合不同在于，重点以虚拟化计算为中心，计算和存储紧密相关，存储由虚拟机而非物理机 CVM（Controller VM）来控制并将分散的存储资源形成统一的存储池，而后再提供给 Hypervisor 用于创建应用虚拟机。比如 Nutanix 和 SSAN+vShpere 超融合方案。这里狭义的定义才是真正意义上的超融合，Nutanix 首次提出这种架构并申请了专利。按照这里的定义，OpenStack+Ceph 只是物理融合而非超融合。值得注意的是，出于性能考虑，超融合架构通常都需要将主机物理设备透传（Pass Through）给控制虚机 CVM。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/5779d06a23709.png" alt="图1   融合的两种形态：物理融合与超融合" title="图1   融合的两种形态：物理融合与超融合" />

图1   融合的两种形态：物理融合与超融合

下面我们再看看几个具有代表性的超融合定义。

Nutanix：超融合架构（简称“HCI”）是指在同一套单元设备中不仅具备计算、网络、存储和服务器虚拟化等资源和技术，而且还包括备份软件、快照技术、重复数据删除、在线数据压缩等元素，而多套单元设备可以通过网络聚合起来，实现模块化的无缝横向扩展，形成统一的资源池。HCI 是实现“软件定义数据中心”的终极技术途径。

Gartner：HCI 是一种以软件为中心的体系结构，将计算、存储、网络和虚拟化资源（以及可能的其他技术）紧密集成在单一的供应商提供的一台硬件设备中。

IDC：超融合系统是一种新兴的集成系统，其本身将核心存储、计算和存储网络功能整合到单一的软件解决方案或设备中。该定义与集成基础设施和平台那些由供应商或经销商在出厂时将自主计算，存储和网络系统集成的产品有所不同。

总结这些定义：超融合架构是基于标准通用的硬件平台，通过软件定义实现计算、存储、网络融合，实现以虚拟化为中心的软件定义数据中心的技术架构。如何评判一个系统是否为超融合？我大胆给出一个简单标准。

1. 完全软件定义。独立于硬件，采用商业通用标准硬件平台（如 x86），完全采用软件实现计算、存储、网络等功能。

2. 完全虚拟化。以虚拟化计算为中心，计算、存储、网络均由虚拟化引擎统一管理和调度，软件定义存储由虚拟机控制器 CVM 进行管理。

3. 完全分布式。横向扩展的分布式系统，计算、存储、网络按需进行动态扩展，系统不存在任意单点故障，采用分布式存储。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/5779d0cabb2be.png" alt="图2   Nutanix超融合架构" title="图2   Nutanix超融合架构" />

图2   Nutanix 超融合架构

### 超融合完美吗？

#### 新的信息孤岛
几乎所有的超融合方案都不支持数据中心中原有的外部存储，大多数企业也不可能在短期内更换整个数据中心基础设施，结果数据中心又分裂成两个彼此独立分散的基础设施孤岛。对于大的数据中心，出于不同业务需求和平衡考量，很有可能会同时部署不同的超融合架构，不同 HCI 之间无法整合和互操作，结果就是又多了几个新的信息孤岛。新的信息孤岛带来了资源利用效率和统一管理的问题。

#### 性能一致性问题
数据中心中存储的性能至关重要，而且期望性能是可以预测并保持一致性的，包括延迟、IOPS 和带宽，这一点尤其对核心业务系统很关键。对于超融合架构而言，恰恰是很大的挑战。原因主要有两点，一是超融合架构“共享一切”。计算和存储会争抢 CPU/内存/网络等物理资源，而且计算和存储又相互依赖，一旦一方资源需求骤升就导致另一方资源枯竭，进而影响性能并在整个基础架构中产生涟漪效应。虽然可以采用 cgroup 或容器技术进行资源隔离限制，但和非超融合架构的效果还是不同。二是超融合架构“一切分布式和软件定义”，集群规模较大后，网络、硬盘、服务器发生故障的概率都会增大，数据重删/压缩/加密/纠删码等功能都用软件实现，故障的自修复和数据功能实现都会消耗一定的系统资源，导致性能下降和抖动。自修复的流控，数据功能旁路到硬件模块处理，这些方法会缓解性能一致性问题，但似乎又与超融合的理念相背离。

#### 横向扩展之殃
超融合架构关键特征之一就是易于扩展，最小部署，按需扩容。超融合架构厂商宣称最大集群规模也差别很大，从数十到数千节点不等，通常从3节点起配。超融合中计算能力、存储性能和容量是同步扩容的，无法满足现实中单项能力的扩展，有些厂商还对扩容最小单元有要求，扩展灵活性会受到限制。集群达到一定规模后，系统架构复杂性就会非线性增加，集群管理变得更加困难，硬件故障和自修复发生的概率也会大大增加。因此，我们是不建议构建大集群的，如果业务允许尽量构建多个适当规模的较小集群，或者采用大集群中构建故障域或子资源池，光大是不行的。集群扩展还面临一个棘手问题，就是容量均衡。如果存储集群容量很大，均衡是一个非常漫长而痛苦的过程，同时还会对正常的业务负载产生较大的影响。

#### 系统复杂性
超融合简化了 IT 架构，极大降低了数据中心设计的复杂性，实现了快速交付，并极大简化了运维管理。不过，这都是基于用户角度的，从产品研发角度而言，超融合实际上使得内部的软件复杂度更高了。前面我们已经阐述，超融合架构需要采用 CVM 虚拟机控制器，并且需要将主机物理设备透传给控制虚机，增加了部署配置管理的复杂度。计算和存储对硬件平台的要求都不同，融合后也会一定程度上增加兼容性验证的复杂度。超融合架构下，管理、计算、存储、高可用通常都需要配置独立的虚拟网络，网络配置也会更加复杂。同时，共享物理资源的分配、隔离、调度，这也是额外增加的复杂度。还有一点，如果出现故障，问题的跟踪调试和分析诊断也变得更加困难。

#### SSD 分层存储
SSD 基本成为超融合架构中必不可少的元素，消除了计算和存储的巨大鸿沟，解决了 I/O 性能瓶颈问题，尤其是随机读写能力。目前闪存的价格相对 HDD 磁盘还是要高出许多，迫于成本因素，全闪超融合方案应用仍然较少，多数应用以 SSD 混合存储配置为主，从而获得较高的性价比。通常情况下，我们假设热点数据占10-20%，配置相应比例的 SSD 存储，采用 Cache 加速或 Tier 分层模式将热点数据存储在 SSD 存储中，一旦热点数据超过预先设置阈值或触发迁移策略，则按相应淘汰算法将较冷数据迁移回 HDD 磁盘存储，从而期望在性能和容量方面达到整体平衡。看上去很完美是吧？SSD 擅长的随机读写，带宽并不是它的强项，对于带宽型应用，SSD 对性能并没有帮助。SSD 混合存储并非理想模式，实际中我们推荐根据应用场景采用全闪 SSD 或全磁盘 HDD 配置，从而获得一致性的性能表现。如果真的无法全用 SSD，还有另外一种应用方式，同时创建一个全 SSD 和一个全 HDD 存储池，人为按照性能需求将虚拟机分配到不同存储池中。

#### 企业级数据功能
目前在大多数超融合系统以及 SDS 系统都具备了核心的企业级功能，包括数据冗余、自动精简配置、快照、克隆、SSD Cache/Tier、数据自动重建、高可用/多路径等数据功能，有些甚至还提供了重复数据删除、加密、压缩等高级数据功能。然而，相对于高端存储系统，如果超融合架构要承载核心关键应用，还有很大的差距，包括但不限于 QoS 控制、数据保护、数据迁移、备份容灾、一致性的高性能。核心存储系统应该遵循 RAS-P 原则，先做好稳定可靠，其次是企业数据功能完备，最后才是高性能，这个顺序不能乱，光有高性能是不行的。比如 Ceph，企业级数据功能列表多而全，功能规格参数非常诱人，但真正稳定而且能够实际生产部署应用的功能其实并不多。目前，核心关键业务系统还不太敢全面往超融合架构上迁移，主要还是从非核心业务开始检验，毕竟超融合出现还比较短，需要更多的时间和实践验证 RAS-P 特性。但是，未来超融合必定是核心关键业务的主流架构。

#### 物理环境应用
目前普遍公认的适合应用场景是桌面云、服务器虚拟化、OpenStack 私有云、大数据分析等新型应用。理论上超融合系统可以适用于 IT 环境的所有应用类型，需要注意的是，超融合系统管理虚拟化环境，而更多的传统 IT 应用仍然运行在物理服务器和传统存储系统之上。我们可以乐观地认为没有哪一种应用程序不能被部署在超融合基础架构上，但是考虑到运行效率、硬件依赖以及和虚拟化环境兼容性等因素，很多 IT 应用最好还是继续保持运行在物理硬件架构，比如数据库应用、实时控制系统以及大量遗留 IT 系统。

#### 异构虚拟化环境
目前超融合方案通常是仅支持一种虚拟化环境，Nutanix 可以支持多种虚拟化环境，但是对于一套超融合架构部署，实际上也仅支持一种虚拟化环境。每种虚拟化环境都有各自的优势，很多企业可能需要同时运行几种虚拟化环境，比如 VMware、KVM、Hyper-V、XEN，因为超融合不支持异构虚拟化环境，需要部署多套超融合架构，这就是新的信息孤岛。客户非常希望看到支持异构虚拟化环境的超融合架构方案。

#### 超融合数据共享
超融合架构采用软件定义存储替换传统的共享式存储解决了虚拟化存储问题，这里的 SDS 实际上主要是指 ServerSAN，提供分布式块存储。然而无论是虚拟机还是物理机，实际 IT 应用都有着数据共享需求，需要分布式文件系统或 NAS 存储系统。这是目前超融合普遍缺失的，现实还是依赖外部独立部署的 NAS 或集群 NAS 存储系统，比如 GlusterFS、ZFS。从技术架构和实现来说，一个 SDS 系统很好地统一支持 Object/Block/File 存储，这个非常难以实现。比如 Ceph，它的 CephFS 一直没有达到生产环境部署标准，更别提性能。因此，超融合架构中可以采用相同方式同时部署两套 SDS 存储，分别提供分布式块存储和文件系统文件共享存储，比如 SSAN 和 GlusterFS，不必非得要求分布式统一存储。

### 容器要不要超融合？

容器是个神奇的东西，火热程度毫不逊色于超融合，它正在引领一场云化数据中心架构的新变革，而被革命的对象是目前还没有大行其道的超融合架构，许多企业已经或正在将应用从虚拟机迁移到容器上。那么什么是容器？简单讲容器就是主机上被隔离的进程，它运行在沙盒之中，借助 CGroup/Namspace 技术限定和隔离所使用的主机物理资源。为什么容器这么受热捧？虚拟机管理程序对整个设备进行抽象处理，通常对系统要求很高，而容器只是对操作系统内核进行抽象处理，使用共享的操作系统，可以更有效地使用系统资源，相同硬件可以创建的容器数量是虚拟机的4-6倍。这可以为数据中心节省大量成本，同时可以快速构建随处运行的容器化应用，并简化部署和管理。“最小部署，按需扩容”，这是云计算要解决的弹性扩展问题。容器相对虚拟机非常轻量，能解决的根本问题就是提升效率和速度，从而实现秒级扩展（包括缩容）。正是这些显著优势，容器非常有潜力替换虚拟机成为云计算的基础架构，这么火热自然就可以理解了。值得一提的是，并不是所有应用都要容器化，关键要看业务是否适合高度弹性计算的微服务，不能盲目推崇。

容器是用来承载应用的，其设计就是为了应用的运行环境打包、启动、迁移、弹性扩展，容器的一个重要特性就是无状态，可以根据需要动态创建和销毁。然而，并不是所有应用都是无状态的，对于有状态的容器怎么办？容器中需要持久化的数据即状态，是不能随便丢弃的。如何持久化保存容器的数据，这是自容器诞生之日起就一直存在的问题。普遍的看法是不应该把数据放到容器中，最好保证所有容器都是无状态的，但还是要提供保存状态的内部机制。容器唯一与状态有关的概念是 volume，容器访问外部应用数据接口，完全脱离容器的管制。Volume 解决了容器的数据持久化存储问题，但它仅仅是一个数据接口，容器本身并不负责持久化数据的管理。这个问题几乎被所有容器厂商忽略，主要依靠外部存储来解决，其中一种解决方案是把容器数据持久保存在可靠的分布式存储中，比如 GlusterFS、Ceph，管理员不用再考虑容器数据的迁移问题。

容器里面直接跑的应用，天生比虚拟机 VM 更接近应用，能通过应用感知对存储的深层次需求，从而动态配置不同的存储策略。因此，为容器提供状态持久化的外部存储系统，应该是面向应用的存储系统，它针对不同类型应用的容器提供精细的存储策略，并进行动态智能应用感知。那么，容器计算＋应用感知存储要不要超融合？这里我们定义一下容器超融合：采用分布式存储，完全容器化，存储控制器也容器化。容器天然应该是无状态的，它的职能是实现敏捷的弹性计算，如果容器是有状态的，这个优势会被极大减弱。分布式存储都是比较重的系统，需要管理大量的磁盘和网络资源，本身并不适合容器化，它们更适合直接运行在原生的物理机操作系统中。分布式存储对状态要求很严格，如果把应用容器和存储控制器容器融合在一个，存储状态的变化会严重影响应用。因此，我个人理解容器本质上是不需要做所谓的超融合，容器重点做好弹性的云计算架构，分布式存储则重点做好可容器感知的应用存储，需要有状态则由独立的外部专业存储来负责，通过容器提供的存储机制进行数据访问，如 Rancher Convoy 或者 Flocker 容器存储驱动。实际上，有状态的容器需求非常少，这也是为什么容器存储被忽视的重要原因。最后，容器与虚拟机 VM 最大区别就是它不是 VM，容器超融合这个说法也不正确。