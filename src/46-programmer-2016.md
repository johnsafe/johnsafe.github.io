## 物联网大数据平台 TIZA STAR 架构解析



文/孙杰

>随着传感技术、通信技术以及大数据处理技术的发展，物联网已在公共事务管理、公共社会服务和经济发展建设等领域中遍地开花，涉及的行业越来越多，如交通管理、节能环保、物流零售等。本文解析物联网大数据平台的典型架构。

### 背景
万物互联的时代正逐步到来，据权威报告预测，2020年全球物联网连接的终端数将达到500亿，数据呈现爆发式增长。这个数据究竟有多大呢？举个典型的例子：

某个工程机械集团，拥有10万台工程机械设备，每台设备上的采集终端每隔10秒上传一次数据，每条数据大小 1KB，一天的总数据量为：10万\*24时\*60分\*60秒/10秒 = 8.6亿，一年的数据量为365天\*8.6亿=3000亿，一年占用的存储为3000亿\*1K=300TB。

我们通常为了保证高可用，会对数据做3份冗余存储，也就是说，这样一个企业一年的数据存储量就可以达到 1PB，而 1PB 就相当于50个国家图书馆的总信息量。

物联网的数据中蕴含的价值也是非常高的，比如：利用车载终端上传的数据，可以提前预测群体出行的时间、方式和路线，可以为城市车辆调度提供决策帮助；在工业设备上安装的传感器，实时收集工业设备的使用状况，可以进行设备诊断、能耗分析、质量事故分析等；通过各种环保传感器采集的数据，可以辅助空气质量改善、水污染控制等。

### 传统架构的瓶颈

面对如此海量的物联网数据，传统技术架构已经难以应对。

首先，传统架构都严重依赖于关系型数据库，而关系型数据库的索引结构基本上都是类 B+树，随着终端数量增大，数据库读写压力剧增，读写延迟增大，数据库面临崩溃。

其次，难以支持海量数据的存储，传统数据库无法水平扩展，所以扩容成本非常高，难以满足 PB/EB 级数据的读取和写入。

最后，难以进行大规模数据处理，传统情况下依赖数据库的 SQL 或者存储过程来进行数据分析的模式，无法对数据做分布式处理，经常一个 SQL 能把整个数据库拖垮。

为了能够把众多行业的物联网大数据价值发挥出来，天泽信息推出了一个企业级的物联网大数据平台（TIZA STAR）。

### TIZA STAR 架构解析

那么 TIZA STAR 到底是什么呢？它是一个面向物联网的开放的数据处理平台，涵盖了数据接入、计算、存储、交换和管理。用户基于这个平台，可以很轻松地打造自己的物联网解决方案。图1可以展现出 TIZA STAR 的定位。

<img src="http://ipad-cms.csdn.net/cms/attachment/201610/57f88edbd5ffe.png" alt="图1  TIZA STAR的定位" title="图1  TIZA STAR的定位" />

图1  TIZA STAR 的定位

TIZA STAR 主要是为了解决什么问题呢？下面的这几个都是典型的物联网应用场景：

1. 物联网安全，解决了从数据接入到最终展现给用户的每个环节的安全防护；
2. 实时接入，解决了百万级别的物联网终端，以很高的频率发送的数据能够实时的接入到系统中；
3. 当前状态，解决了在百万级别的物联网终端中，快速地获取到某个终端的当前的状态；
4. 历史状态，解决了在百万级别的物联网终端中，快速地获取到某个终端在过去的某个时间段内的状态参数；
5. 下发指令，解决了如何给一个或者多个物联网终端下发指令，从而可以实现远程控制和参数调校等。

#### 架构图
最底层是设备层，可以全部采用 X86 通用服务器，无需采购小型机等昂贵的计算设备和高端存储设备，从整体上可以大幅拉低成本。

最上层是业务层，主要是物联网的各种应用和第三方系统。业务层无需对数据进行处理和分析，只是通过查询接口获取到平台层中已经处理过的数据并在界面进行展示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201610/57f88ffd59fcd.png" alt="图2  TIZA STAR架构图" title="图2  TIZA STAR架构图" />

图2  TIZA STAR 架构图

中间的这一层就是 TIZA STAR，包含了数据接入服务、数据存储服务、数据计算服务（包括实时计算和离线计算）、监控报警服务、平台管理服务、数据交换服务。下面会对每个模块详细介绍。

#### 数据接入
数据接入时，传感器或者采集终端通过无线或者有线的方式发送到平台端，平台端通过软负载均衡（LVS）或者硬负载均衡（F5 等）将流量均匀的负载到各个可水平扩展的网关，每个网关都是基于 netty 实现的高性能的网络接入程序。

数据接入协议分两个层次，在通讯层次上，支持 TCP、UDP、HTTP 和 WEBSOCKET 等通讯协议；在数据协议层次上，支持 MQTT、JSON、SOAP 和自定义二进制协议。通过这两个层次的互相搭配，可以轻松实现任何物联网终端、任何协议的数据接入。

网关接收到数据，并完成解包之后，将数据包发送到消息中间件中，可以有效地应对“井喷流量”和下游服务短暂不可用的问题。在消息中间件的选择上，我们比较了 Kafka、RabbitMQ 和 ActiveMQ，最后选择了 Kafka，因为在分布式环境下 Kafka 的吞吐性能非常优秀，并且其持久化和订阅/发布的功能与物联网的场景非常匹配。

<img src="http://ipad-cms.csdn.net/cms/attachment/201610/57f8902c74192.png" alt="图3  TIZA STAR数据接入协议" title="图3  TIZA STAR数据接入协议" />

图3  TIZA STAR 数据接入协议

#### 数据存储
TIZA STAR 综合使用了多种存储引擎，包括 HDFS、HBase、RDBMS 和 Redis。

其中 HDFS 非常适合于非结构化数据的存储，支持数据的备份、恢复和迁移，在系统中主要用于存储原始数据和需要进行离线分析的数据。

HBase 适合于存储半结构化的数据，可以很好的支持海量物联网终端的历史数据的查询，在系统中主要用于存储终端的历史轨迹和状态等体量比较大的数据。

RDBMS 适合于存储结构化的数据，通常根据具体的数据库采用不同的高可用部署方案，在系统中主要用来存储终端基础数据、字典数据和数据分析的结果等。

Redis 是基于内存的 KV 数据库，在系统中通常用来缓存需要频繁更新和访问的数据，比如物联网终端的当前状态等。在多种 KV 数据库中我们最后选择了 Redis，主要是看重 Redis 为多种数据类型以及多种数据操作提供了很好的内嵌支持。

#### 数据处理
数据处理包括实时计算和离线计算两种。

实时计算我们比较了 Storm 和 Spark-streaming，最后选择了 Storm，主要考虑两点：一方面是因为 Storm 的实时性更好一些，另一方面是因为在物联网的场景中需要支持对终端数据的全局分组，而 Spark-streaming 只能在每个 RDD 中做分组。所以最后我们选择了 Storm 作为我们的实时处理引擎，在它的基础上我们包装了自己的实时计算服务，可以支持应用层的调度和管理。基于实时计算服务可以很容易实现对物联网数据的清洗、解析、报警等实时的处理。

离线计算目前支持 MapReduce 和 Hive，对 Spark 的支持也正在进行中，主要用于对物联网数据做日/周/月/年等多个时间维度做报表分析和数据挖掘，并将结果输出到关系数据库中。

#### 数据交换接口
数据交换接口主要是为了简化应用层与平台层之间的数据访问而抽象了一层访问接口，有了这层接口，应用层就不需要直接调用 Hadoop、HBase 等原生 API，可以快速地进行应用开发。

目前数据交换接口支持：SQL、Restful、Thrift 和 Java API，用户可以根据实际情况灵活选择数据交换的方式。

数据交换的内容包括：物联网终端的当前状态、物联网终端的历史状态/轨迹、指令下发、数据订阅与发布等等。

#### 平台管理
平台管理包括监控报警和管理 UI。

监控报警我们是用 Ganglia 和 Nagios 配合来做的，从三个级别来做：硬件级别（服务器、cpu、内存、磁盘等）、进程级别（进程不存在、端口监听异常等）、关键业务指标（中间队列的元素数、网关建立的 tcp 连接数等）。

管理 UI 包括界面化安装部署、用户管理、终端管理、集群管理、数据接入管理、实时和离线计算任务界面化管理。

#### 平台 SDK
平台 SDK 是为了方便企业用户基于 TIZA STAR 定制自己的物联网应用，我们提供了三个 SDK：GW-sdk、RP-sdk、OP-sdk。

其中基于 GW-sdk 可以快速新增一种新的物联网终端协议的接入。

基于 RP-sdk 可以快速开发一整条实时处理链，也可以快速开发处理链中的某个模块。

基于 OP-sdk 可以快速开发一个可周期性调度的 MapReduce/Spark 任务。

#### 平台安全
物联网安全也日益重要，前段时间发生的私家车被恶意远程控制的事件就体现出了物联网安全的重要性。

TIZA STAR 从链路安全、接入安全、网络安全、存储安全和数据防篡改这几个方面来保证物联网安全。

1. 通过 SSL 和 TLS 保证链路安全；
2. 通过秘钥鉴权对数据的访问有效进行控制；
3. 通过防火墙等硬件设备防止网络攻击；
4. 通过副本冗余保证数据的存储安全；
5. 通过每512字节进行 CRC 校验的机制保证数据的防篡改。

### 应用案例

#### 数据流
接下来我们以车联网为例，看一下 TIZA STAR 的数据流。

<img src="http://ipad-cms.csdn.net/cms/attachment/201610/57f8905fc11e4.png" alt="图4  车联网数据流" title="图4  车联网数据流" />

图4  车联网数据流

如图4所示，物联网终端通过无线/有线网络发送到 TIZA STAR 平台，经过一系列的处理后存入到各种存储引擎中，业务可以通过数据交换接口来访问处理后的数据。具体流程如下：

1. 车载设备或者传感器设备通过网络经过 LVS/F5 负载均衡将数据发送至网关；
2. 网关接收到数据后进行公共协议解析，然后把解析后的数据发给 Kafka，存放在原始数据 Topic；
3. 实时计算任务从原始数据 Topic 中读取数据经过数据清洗后发送至原始数据解析模块；
4. 原始数据解析模块将解析出来的车辆的参数发送至 Kafka 解析数据 Topic。然后将解析后的数据发送至报警判断模块；
5. 报警判断模块根据已有规则进行预警，并将产生的结果分别发送至 Kafka 的报警数据 Topic，同时把解析后的数据发送至当前状态分析模块；
6. 当前状态分析模块对车辆当前状态进行分析，如果状态有变化则更新至 Redis；
7. 数据导入模块异步的将 Kafka 中的数据分别导入 HBase 和 HDFS；
8. 离线计算则周期性地从 HDFS 中读取数据进行各种报表分析和数据挖掘；
9. 用户业务平台和管理平台可通过数据交换接口访问 TIZA STAR 平台数据。

#### 性能对比
表1是天泽信息为某个大型工程机械集团做的平台升级前后的性能指标对比情况，其中老平台是传统的基于 IOE 的解决方案，硬件环境包括 IBM 的小型机、EMC 的存储和 Oracle RAC，新平台是基于 TIZA STAR 的解决方案，使用的硬件是30台左右 X86 架构服务器，服务器中自带存储。

<img src="http://ipad-cms.csdn.net/cms/attachment/201610/57f8909940451.png" alt="表1  性能指标对比" title="表1  性能指标对比" />

表1  性能指标对比

该工程机械集团注册终端数接近15万，每个终端分布在全国各地，每隔30秒发送一条数据到平台，上传的数据包含工程机械设备的位置、工况等信息，该企业会对上报的数据进行分析，用于生产、经营的改善。

### 结束语

从研发到正式商用，耗时一年半， TIZA STAR 经历了三个阶段：

第一个阶段是封闭研发阶段。TIZA STAR 平台最初的研发缘于公司一个客户的迫切需求，原有的数据平台随着业务量扩大，性能捉襟见肘，经常出现宕机的情况，已经无法支撑正常的业务需求。为了更好的为客户服务，我们决定对老平台进行升级，目标是用业界最新的大数据技术来解决老平台的性能问题。经过半年多的时间，我们发布了新平台，为了不影响客户业务的正常开展，决定新老平台同时运行。几个月后，经过对比，新平台在性能、高可用性、可运维性等方面都完胜老平台。

第二个阶段是拓展阶段。TIZA STAR 平台在第一个客户成功上线后，公司开始在物联网的诸多领域推广这个产品，但推广的过程不是很顺利。一方面，由于目前物联网领域还没有一个统一的标准，不同企业、不同物联网终端都有自己的非标协议，我们在数据接入、协议解析、存储方面都要进行不同程度的定制。另一方面，随着上线的案例越来越多，我们原来的平台架构也暴露出很多问题，针对这些问题，我们做了很多调整，比如将网络通讯组件由 Mina 替换成 Netty，引入了 KV 数据库，对 Hadoop、Hbase 和 Kafka 等进行了大版本的升级。在这个阶段，虽然 TIZA STAR 平台在不同的行业都有成功案例，但团队做的还是很辛苦。于是把物联网平台进行封装以满足大多数场景的需求就显得迫在眉睫，TIZA STAR 的产品化就此应需而生。

第三个阶段是产品化阶段。在此阶段，我们对数据接入、计算、存储、交换等各个环节进行了封装；累计开发了超过100种行业协议；抽象出了3个 SDK，便于用户基于平台定制自己的新协议和业务处理模块；完善了监控和报警，形成一个完整的运维闭环；实现了安装部署、管理和运维的界面化操作；提供了标准版和简化版来满足不同规模的客户需求。经历了半年多努力，TIZA STAR 平台终于实现了产品化。

在产品迭代的过程中，我们经历过产品初次上线成功的喜悦，也经历过产品拓展过程中的迷茫。项目组也从最初的五六人，到几十个人初具规模的研发团队。目前，TIZA STAR 平台正式注册了商标，申请了软件著作权和四项物联网大数据领域的发明专利，产品在各个方面都有非常大的提升，希望今后可以给更多的企业提供物联网平台端解决方案。