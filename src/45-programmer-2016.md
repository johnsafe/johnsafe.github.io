## 11个热门物联网开发平台的比较



文/Miyuru Dayarathna

本文基于我们对物联网（IoT）供应商进行的详细分析，针对现有 IoT 软件平台做了一份综合调查。首先，我们制作了一个清单，列出了 IoT 软件平台的重要功能。然后我们对比了当前11个较为热门的 IoT 软件平台上在这些关键功能的开发程度。最后根据我们的观察对列表进行缩减，总结需要改进的功能。

### 简介

从1999年 Kevin Ashton 第一次提出这个概念以来，物联网已经历了迅速转变。随着近年来连接到物联网的设备在多样性和数量方面呈现指数式增长，物联网已经成为了一种主流技术，在推动现在社会的生活方式方面有着极大的潜力。

在物联网的技术与工程上，硬件与软件平台之间目前仍有明确的界限，其中大多数供应商都将精力放在硬件方面。只有极少数供应商提供物联网软件服务：例如，Mattermark 根据所获总投资排名的前100名物联网创业公司中，只有13家提供物联网软件服务。

本文针对现有物联网软件平台，基于我们对 IoT 供应商进行的详细分析做了一份综合调查。而本文最后选择的物联网供应商，完全是基于这样的标准：这些供应商是否提供软件解决方案，来处理从物联网设备/传感器获取的信息。注意：虽然我们希望尽可能全面，但本文中仍有可能漏掉了一些这些平台的最新改进。

### 物联网软件平台想要的重要功能

基于最近的几份调查，我们选出了物联网软件平台最关键的功能：设备管理、集成、安全性、数据收集协议、分析类型以及支持可视化，以便对样本功能进行比较。本文的后半部分会对这些特性进行简单介绍。

#### 设备管理与支持集成
设备管理是物联网软件平台所需的重要功能之一。物联网平台应当维护着一堆与之连接的设备，并跟踪这些设备的运行状态；还应当能够处理配置、固件（或其他软件）更新问题，并提供设备级的错误报告和处理方案。每天结束前，设备用户应当能够获得个人设备级的统计。

支持集成是物联网软件平台需要的另一个重要功能。需要从物联网平台上公布的重要操作和数据应当能通过 API 访问，REST API 常用于这一目的。

#### 信息安全
运营物联网软件平台所需的信息安全手段，比普通软件应用和服务所需的要求更高。数百万台设备与物联网平台连接，代表着我们需要处理的漏洞也是相应比例的。一般来讲，为了避免被窃听，物联网设备与物联网软件平台之间的网络连接需要通过强大的加密机制来保障。

然而，现代的物联网软件平台保障，大多低成本、低功率的设备都无法支持这样的高级访问控制措施。因此，物联网软件平台自身需要采取替代措施，以解决这类设备级的问题。例如：将物联网流量划分为专用网络，依靠云应用级的强大安全性，要求定期更新密码并支持验证更新固件，还有签名才能更新软件等等，这些手段都能加强物联网软件平台的安全级别。

#### 数据收集协议
需要注意的另一个重要方面，是物联网软件平台的各个组件之间用于数据通信的协议类型。物联网平台可能需要扩展到数百万甚至数十亿设备（节点）上。应当使用轻量级通信协议，以实现低能耗以及低带宽功能。

注意：虽然我们在本文中将协议作为概述性词汇，不过用以收集数据的协议可分为下面几类：比如应用、负载容器、信息传递和传统协议。

#### 数据分析
从连接到物联网平台的传感器中所收集的数据需要通过智能化手段进行分析，以获得有意义的见解。

物联网数据分析有四种主要类型：实时分析、批处理分析、预测分析与交互式分析。实时分析：对数据流执行在线（动态）分析。样本操作包括基于窗口的集成、筛选、转换等。批处理分析：对积累的数据集进行操作。这样，批处理操作会在预定时间段内运行，也许持续数小时或数日。预测分析：基于各类统计与机器学习技术，集中进行预测。交互式分析：对流数据和批数据执行多个探索性分析。最后一个就是实时分析，在任何软件平台都占据较重的份量。

### 当前的物联网软件平台

对当前的物联网软件平台进行仔细调查后，我们发现上面提到的每个功能都已实现，只是程度不同而已。我们在下面列出了相关的平台，并进行了功能总结对比，如表1所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d55e66cd7c8.jpg" alt="表1  相关平台功能总结对比（标着“未知”的栏目代表相关信息在可用文件中无法找到）" title="表1  相关平台功能总结对比（标着“未知”的栏目代表相关信息在可用文件中无法找到）" />

表1  相关平台功能总结对比（标着“未知”的栏目代表相关信息在可用文件中无法找到）

很明显，上面列举的物联网初创公司，其中很多可能还不具备设备管理功能。在这方面，还需要物联网软件平台供应商提供解决方案。

此外在分析生成的物联网数据时，在计算及可视化方面提供的支持相对较少。它们大多支持实时分析——这是任何物联网框架的必备功能。然而，只有极少数物联网软件平台为其他3种分析类型提供支持。而可视化界面大多表现为门户网站这样的简单模式，允许对物联网生态环境进行管理，不过很少提供可视化的数据分析功能。

在不同的物联网软件平台中，还有几个常见功能，包括基于集成的 REST API，支持用 MQTT 协议来收集数据，以及使用 SSL 进行链路加密。尽管在表1中没有提到，不过单 ParStream 公司就能达到300万到400万行/秒的吞吐量。

这表明大多数物联网软件平台在设计时并未太多考虑物联网部署的系统性能，而在真实情况下这是非常关键的。

### 需要改进的功能

很明显有若干地方需要改进。在本节中，我们首先提供了一张改进功能列表。在物联网软件平台供应商的努力下，其中一些项目已经实现，还有一些性能等待实现。之后我们提供了一张列表，包括现在尚未实现的这些新功能。

#### 现有功能

- 数据分析

现在物联网软件平台大多支持实时分析，不过批处理分析和交互式数据分析也许同样重要。

在这一点上，有人可能会争辩：在其他知名的处理平台中包括这类分析功能，想要配置用于分析场景的软件系统也很简单。不过，这谈何容易。用于实时分析（Storm、Samza 等）、用于批处理分析（Hadoop、Spark 等）、用于预测分析（Spark MLLIB 等）、用于交互式分析（Apache Drill 等）的知名数据处理系统，并不能直接用在物联网案例中。

- 基准

物联网软件平台需要有扩展性，还应包含描述和评估系统性能的设备。定义良好的性能指标需要：能够塑造与测量物联网系统的性能，并考虑到网络特性、能耗特点、系统吞吐率、计算资源消耗以及其他运行特征。

- 边缘分析

需要采取措施以减少传感器设备与物联网服务器之间的大量网络带宽损耗。解决方案之一是使用轻量级的通讯协议。另一个办法就是使用边缘分析法，以减少传输到物联网服务器上的原始数据总量。即便是在简单的硬件嵌入系统中（如 Arduino），也可以实现边缘分析法。

- 其他问题

应当注意：有多个与物联网软件平台相关的其他问题，比如伦理、道德和法律问题，在本文中并未涉及。尽管这些问题也很重要，但在本文中不作讨论。

#### 需要添加的功能

- 处理无序进程

在任何物联网应用中都有可能碰到无序事件，在传感器所发出的事件流中，元组顺序混乱可能是网络延迟、时钟偏移等原因所导致的。处理无序的物联网事件可能会导致系统故障。处理无序事件时，需要在结果准确性与延迟之间做出权衡。

有四项主要的处理技术：基于缓存（Buffer-based）、基于标点（Punctuation-based）、基于推测（Speculation-based）以及基于近似（Approximation-based）。在物联网解决方案中，应当使用其中的一项或多项来解决无序事件的问题。

- 支持物联网背景

背景主要由个体、其偏好或过去的行为构成。例如：在移动电话案例中，由于现代移动电话中有很多不同类型的传感器，因此我们能够获得丰富的背景信息。在物联网分析中，这些背景数据应当被纳入考虑。

### 结论

物联网模式的快速发展需要强大的物联网软件平台，能通过物联网用例满足出现的需求。本文中，我们调查了现有最先进的物联网软件平台的功能，调查集中在这些方面：设备管理、集成、安全性、数据收集协议、分析类型、可视化支持。从这项研究中，像设备管理、物联网数据分析、物联网软件系统可扩展性以及性能这样的领域明显需要物联网平台社区投入特别的关注。

>原作者 Miyuru Dayarathna（WSO2 高级技术主管）已授权《程序员》将本文翻译为中文并刊载，感谢孙薇的翻译