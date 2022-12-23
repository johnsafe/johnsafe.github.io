## Apache Eagle: 分布式实时大数据性能和安全监控平台

文/张勇

大数据系统在最近几年发展非常迅速，各种以 Hadoop 为基础的大数据产品层出不穷，典型的有存储层 HDFS, 计算层 MapReduce, 查询层 Hive、Impala、Kylin, 数据库存储层 HBase 等等，另外新兴的大数据分析系统如 Spark 也迅猛发展。

时下，大数据技术带来的价值已不容质疑，在 eBay 我们使用 Hadoop 来存储上亿的买家和卖家信息，物品上架信息和用户搜索信息，这些海量数据已被用于一些关键的应用，如搜索引擎的索引创建、个性化的搜索体验、关联广告投放和用户点击流分析等等，这些数据的存量已达到几百 Petabytes，并且每天都在迅速增加。对于大型互联网企业来说，对服务器、网络等硬件设备的监控平台已经相当成熟，而对于大数据的监控，如应用性能监控和用户访问数据的安全监控却相对滞后很多。为了监控大数据应用性能和安全，企业暂时使用一些原本针对于服务器的监控产品。而 eBay 正是认识到这些明显的差距，专门基于大数据系统对于监控的特殊要求而设计了 Apache Eagle（以下简称 Eagle）。

对比传统的服务器监控平台（如 Zabbix、Ganglia）和 APM(Application Performance Monitoring)监控平台（如 New Relic 等等），Eagle 是一个实时的分布式大数据性能和安全监控平台，旨在迎合大数据系统的一些特点，如海量的日志数据需要在流式处理中分区以便并行处理，而判断系统的性能问题需要运行复杂的计算（如长时间的滑动窗口聚合），甚至需要集成机器学习的模型来判断异常等等。所有这些业务需求，要求 Apache Eagle 监控平台必须设计成支持高度水平可伸缩、功能可扩展、具有声明式的告警和聚合计算、容错、数据源快速接入等等。
Eagle 起始于 Hadoop 集群的数据访问安全实时监控、重要 Hadoop 服务如 Namenode 的性能监控、MapReduce 作业性能监控、datanode 节点异常监控等等，但现在的框架已逐渐支持对于任意数据源的监控。

### Eagle 的设计目标

#### 监控系统面临的挑战

一个好的监控系统不仅仅是产生一些实时的报表，更要能够通过复杂的规则或机器学习来发现异常。当监控系统判断一个监控对象是否存在问题时，往往需要复杂的规则，这些规则不仅仅是简单地对单条日志或指标进行规则匹配，很多有实际意义的规则都是基于时间窗口或空间窗口，甚至会用到模式匹配和序列模式等，因而可以使用CEP引擎来处理。另外，监控系统面临的是海量的流动数据，需要在流动的数据中应用复杂的规则或算法，因而需要用流式引擎来计算。

在现在企业中，大多数监控系统会由运营部门来管理，通常会有很多脚本分布在各个服务器，也有一些企业是用 DevOps 团队来开发监控系统，但是如果要从零架设全套系统，在流引擎的基础上分布式地运行 CEP 引擎是一个非常困难的任务。

Eagle 不仅仅是将 CEP 引擎运行在流引擎上，更重要的是在实现了监控系统的功能需求之外，同时也在框架级别支持很多非功能性的系统级需求。比如 Eagle 是分布式策略引擎，它将策略分布到不同的物理机器上执行，而分布的算法是可以定制的。分布式处理涉及的不仅仅是数据的分布，还有计算的分布。另外，框架要保证数据或计算分布的均衡性以最大化利用系统资源。Eagle 实时搜集数据分布和计算分布的统计信息，从而让用户了解数据和计算的权重，方便 policy(注: 本文中 policy 或策略泛指规则或更复杂的统计算法、机器学习算法等)计算的平衡。

另外，Eagle 要能够处理容错(Fault Tolerance)，比如有些策略是针对很大的时间窗口，而在分布式处理中，一台机器失败的概率是比较大，在机器失败的情况，Eagle 能保证这些基于时间窗口的 policy 计算能做到 failover.

#### Eagle 的设计目标

告警计算的高度可伸缩性(High Scalability)。对于海量的数据源和大量的规则计算，Eagle 在数据和计算两个维度上进行分布，同时提供基于统计的算法将数据和计算进行动态平衡，从而最大化资源的使用。

元数据驱动 (Metadata Driven)。元数据设计是 Eagle 各个重要组件的基础。元数据包括告警数据流 schema 的描述、告警 policy 的描述、数据源的描述、数据计算拓扑图的描述、数据聚合的描述和 UI 页面功能的描述等等。

监控数据源的快速接入 (Usability)。监控的数据源是多样化的，Eagle 提供高度抽象的通用编程接口来支持开发人员对各种数据源的接入，另外也将支持用户编辑数据处理拓扑图，由框架编译并执行。

容错支持 (Fault Tolerance)。Eagle 支持对复杂 policy 计算的中间状态进行快照，并记录之后的日志，在宕机后可以通过最近的快照和增量日志进行状态恢复。这个功能对于长时间滑动窗口等规则计算尤其关键。

可扩展性(Extensibility)。告警计算是 Eagle 的核心，而基于 CEP(Complex Event Processing)的规则计算只是默认支持的计算引擎。Eagle 社区正在加入更多的计算引擎来丰富 Eagle 的功能。另外，对数据源的可扩展性，可以让 Eagle 应用在更多的监控场景。

### Eagle 整体架构

Eagle 使用典型的流式处理架构，由数据搜集和缓存层，数据流处理层，数据存储层等组成。对于 Eagle 来说，由于整体架构采用元数据管理，而 Eagle Service(API)是存取元数据的服务，所以 Eagle Service 是联系各层之间的纽带，如图1所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56ab2c7cd6972.jpg" alt="图1  Apache Eagle整体架构图" title="图1  Apache Eagle整体架构图" />

图1  Apache Eagle 整体架构图

### 分布式策略引擎

Eagle 数据处理的核心是对进入的数据流进行处理和验证。Eagle 处理的数据流有几个特征，一是数据量比较庞大；一是处理逻辑会有一定的复杂性和多样性但不是特别复杂；另外对于数据流的处理验证逻辑可能会随着监控的需求进行动态变化，这就要求 Eagle 在对监控数据流进行监控时还必须保证可伸缩性(Scalability)和可扩展性(Extensibility)。

Eagle 提供了一个可扩展的分布式策略引擎框架。用户可以把 CEP 逻辑以元数据的方式进行定义。由 Eagle 框架进行调用并自动进行分布式处理。分布式策略引擎的数据流程图如下图。目前在 Eagle 的引擎实现中，基于开源 CEP 引擎 Siddhi（https://github.com/wso2/siddhi）提供了类 SQL 的描述性策略处理支持。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56ab2cebca42c.jpg" alt="图2  Apache Eagle分布式引擎" title="图2  Apache Eagle分布式引擎" />

图2  Apache Eagle 分布式引擎

在 Eagle 中，policy 由元数据管理器进行生命周期管理。元数据管理器提供 policy 的读写操作(并由 UI 提供可视化操作，见图2)。而 Ealge 的 policy 引擎会在部署程序时读取策略定义，生成多个策略引擎节点进行数据处理(AlertExecutor_{1-N})。一个典型的 Eagle 策略如图3所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56ab2cdbacbe0.jpg" alt="图3  Eagle Policy定义" title="图3  Eagle Policy定义" />

图3  Eagle Policy 定义

基于 Siddhi 提供的 CEP 支持，Eagle 可以支持多种类型的监控处理操作。

单 event。单 event 的 CEP 支持可用于对某指标的阈值判断，典型的用法是对于磁盘这种缓慢增加的指标。

event 窗口（基于时间的或者 event 数量的滑动窗口和批量窗口）。滑动窗口和批量窗口可以提供窗口内某个值的 avg/min/max 的阈值判断。在动态变化频繁地指标中使用窗口比单 event 能更好地避免 false alert。

多 stream join。Stream 的 join让Eagle 有能力组合不同的 stream 进行操作。比如不同 stream 之间关系的表述和 pattern match。

Eagle 将 WSO2 的 Siddhi CEP 技术无缝引入到流式框架中，使得用户可以在分布式环境中使用 CEP 来做告警 policy 的计算，这是 Eagle 的一个创新。另一个有趣的创新是 Eagle 对数据在流式聚合也是基于 Siddhi CEP 技术，并且采用类似告警策略的架构实现聚合计算的分布式。

流式数据的聚合是指在数据流动过程中计算数据的统计值，对于理解数据的统计意义和快速查看历史数据很有帮助，如对于 Gauge 类型的 metric 进行 avg、min、max 聚合，而对 count 类型的 metric 进行 sum 等等。而聚合的维度可以是空间(Spatial)的，也可以是时间(Downsampling)的。

Eagle 支持数据的动态聚合，默认是基于 CEP 引擎来计算，因此可以像定义告警 policy 一样，来声明聚合 policy 的表达式。这样带来极大的灵活性，用户可以动态添加新的聚合表达式，而不用重新写代码发布。用户甚至可以修改原来的聚合表达式，有了 trial-and-error 的灵活性。

声明式聚合的潜在好处是，如果将聚合的结果实时输出到报表，由于可以动态增加、删除、修改聚合表达式，用户就能动态监控各种实时数据。聚合的分布式计算也非常重要，Eagle 框架需要能够分析用户的聚合表达式，并且能够根据表达式对 graph 进行重写，在某些情况下，需要对数据进行 map 然后再做 reduce，这也是 Eagle 下一步需要开发的重要功能。Eagle 策略引擎具备两个主要的特性：

#### 可伸缩性

Eagle 的可伸缩性来自于两个部分，一个是底层的 Storm 引擎可以将数据进行分区，从而把数据分布在不同的节点处理，另一方面则来自 Eagle 提供的 PolicyPartitioner 将 policy 的执行在物理上分开。应用可以配置 Eagle 策略引擎把 policy 的计算分布在不同的策略引擎节点上，以提供水平可伸缩性。图4中每个 alert executor 对应于一个 storm bolt，纵向是对数据进行分区，横向是对计算进行分区 。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56ab2d12f2db7.jpg" alt="图4 Apache Eagle对于策略计算的可伸缩性" title="图4 Apache Eagle对于策略计算的可伸缩性" />

图4 Apache Eagle 对于策略计算的可伸缩性

一个典型的对策略计算进行分布处理的配置：

```
“AlertExecutorConfigs” : {
   “hdfsAuditLogAlertExecutor” : {
         “parallelism” : 1,
         “partitioner” : “org.apache.eagle.policy.DefaultPolicyPartitioner”
    }
}
```

下文会讲述如何通过 Eagle 提供的 policy partitioner 和 Storm 提供的 CustomizedGrouping 来解决数据和计算的分布倾斜。

#### 可扩展性

Eagle 策略引擎通过 plugin 的方式可以增加不同的策略实现。在 Eagle 的实现中，Siddhi CEP 引擎是默认的 CEP 实现，除此以外 Eagle 还以插件的方式提供了一些机器学习算法的 policy 引擎实现。

### Eagle 动态流式拓扑 DSL

为了保证海量数据流监控的实时性和可伸缩性，Eagle 支持将 policy 计算运行在分布式 streaming 引擎之上，使用这些引擎在提供强大分布式计算能力的同时，引入了更多的复杂性，特别当 Eagle 引入了策略计算和聚合计算的自动分布处理，基于统计的动态改善数据分布的均匀性等等高级功能之后，Eagle 应用的使用者会面临复杂的编程 API。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56ab2ebe75f74.jpg" alt="图5  Eagle高度抽象的面向领域的API" title="图5  Eagle高度抽象的面向领域的API" />

图5  Eagle 高度抽象的面向领域的 API

因此， Eagle 提供了针对实时监控的高度抽象描述性领域特定接口 DSL，不仅支持通用的流式处理语义如 filter、map、flatmap 等，同时也支持 Eagle 特有的策略计算、聚合、数据分区定制等多种语义，使得 Eagle 开发者可以通过非常简单明了的方式组装流式处理的逻辑单元，如过滤、转化、外部 Join 或者策略执行等，而且这些语义可以非常方便被扩展。

Eagle DSL 是平台独立的，通过 Eagle DSL 定义的流式处理会转化为对应的“逻辑有向无环图(DAG)” 或者称之为“逻辑拓扑”，逻辑拓扑只包含逻辑的功能，运行时会根据特定的执行环境转化并优化为具体的物理执行实例，称为物理拓扑 。Eagle DSL 默认支持通过 StormCompiler 将逻辑拓扑翻译成可运行在 Apache Storm(http://storm.apache.org/) 之上的拓扑结构，但也可以通过增加其他类似 StormCompiler 的编译器来将逻辑拓扑翻译成可运行在其他流处理引擎比如 Samza 等。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56ab2ef0add81.jpg" alt="图6  Eagle将逻辑拓扑图转化并优化为流引擎支持的物理拓扑图" title="图6  Eagle将逻辑拓扑图转化并优化为流引擎支持的物理拓扑图" />

图6  Eagle 将逻辑拓扑图转化并优化为流引擎支持的物理拓扑图

在 Eagle 框架内部会将每个复杂的 DSL 操作转化为一个或者多个原子流处理操作，每个操作会分别运行在物理 DAG 的一个处理单元中，这个过程称之为 Graph Expansion。Eagle 内置支持多种不同类型的 Expansion，包括但不限于 union expansion、group-by expansion、external join expansion 等，同时也支持自定义扩展新的 expansion。如图7所示，展示了开发者如何简单地使用 Eagle DSL。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56ab2f20a9c9e.jpg" alt="图7  逻辑拓扑和物理拓扑转化的简单用例" title="图7  逻辑拓扑和物理拓扑转化的简单用例" />

图7  逻辑拓扑和物理拓扑转化的简单用例

Eagle 目前支持多种典型的流操作，包括但不限于 map、flatmap、filter、alert、union、externalJoin 等。每个 Stream 通常会连接两个或多个 PE。告警处理单元是 Eagle 的核心处理单元之一，提供针对数据流的实时告警功能。目前在需要通过 Stream 连接上游处理单元和告警单元时，Eagle 要求提供数据流结构(Schema)。数据流的结构是通过 Eagle 的元数据 API 集中管理的，开发者可以通过 API 声明数据流包含哪些属性、分别什么类型以及如何在运行时动态解析属性值等，用户则可以根据这些元数据更方便的定义告警策略。

作为一个典型的示例，假如开发者希望定义如下数据流处理逻辑：首先对输入数据流进行转化，然后与外部元数据进行 Join，最后执行告警，利用 Eagle 动态流 DSL 可以非常简单的定义如下：

```
x.flatmap(task1).externalJoin(task2).alert(task3)
```

转化过程称为物理拓扑的过程如图8所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56ab2f5103842.jpg" alt="图8  逻辑拓扑和物理拓扑的复杂用例" title="图8  逻辑拓扑和物理拓扑的复杂用例" />

图8  逻辑拓扑和物理拓扑的复杂用例

### 数据/计算倾斜(skew)问题

在 Eagle 监控的一些用例，如用户行为监控中，保证事件的时间顺序很重要，一些检测策略会根据同一个用户行为发生的先后顺序来判定其是否是一个异常。在用户行为监控中，用户 ID 就是一个需要保证时间序的键值。

以一个典型的案例来说明，HDFS 审计日志的监控中，我们将 namenode 的审计日志导入到 Kafka 中，Storm Spout 从 Kafka 中读取日志流，送给下游 bolt 进行处理，若要保证整个数据流的事件顺序，则每个上游到下游的数据传递过程都要保证。在 Kafka 中，我们将用户 ID 做为键值来分区，如果我们采取用户随机分区，会出现很大的倾斜，这是由于不同用户产生的审计日志量相差巨大，一些 Batch 用户产生的日志数量比普通用户多几个数量级。假设 #user=N，#partition=M，用户 i 在给定时间内产生的 log 数量为 Wi，将 N 个 user 分到 M 个 group，目标是数据分布要尽可能均衡，也就是每个 group 中的数据量趋于相等，而 Wi 会随时间不断变化，当 N、M 较大时，性能会受到一定影响，我们的做法是先按照 Wi 将用户进行排序, Wi越大排序越前，再将用户逐个分配到当前权重最小的分区中，直到所有用户都分配完。从实际运行情况来看，设 #partition=8，Greedy 算法能够很大程度上缓解了数据倾斜问题，算法权重最大的分组与权重最小的分组比为0.3200/0.0837=3.82， 而 Random 算法的为0.3535/0.024=14.729，可见 Greey 算法与 Random 算法相比要好很多。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56ab2b137c3f8.jpg" alt="表1  随机算法和贪婪算法对于数据均衡的不同结果" title="表1  随机算法和贪婪算法对于数据均衡的不同结果" />

表1  随机算法和贪婪算法对于数据均衡的不同结果

除了数据倾斜问题外，我们还要处理计算倾斜的问题。为了做到可扩展，Eagle 不但对数据进行分区同时还要对 policy 进行分区，有的 policy 很轻，例如简单的 filter，有的 policy 则比较重，如 slide window。Eagle 默认使用 DefaultPartitioner 来分布 policy，但这种通过 hash 进行随机分布的算法，会导致某些计算量大的 policy 聚集在一起，从而导致个别物理节点无法正常运行。

为了对策略计算进行平衡，从而最大化资源利用效率，Eagle 实时搜集 policy 运行时刻的信息，主要是 policy 执行的 wall time 和 policy 处理中缓存的 event 的数量等等。这些统计值将成为对计算进行均衡的基础。Eagle 同样使用 Greedy 算法来做 policy 的动态均衡，这部分算法正在开发中。

### 容错支持

Eagle 可以支持具有非常复杂语义的 policy 如用户最近5分钟，1个小时甚至一个月的访问统计，而这些有意义的 policy 往往是有状态的。这里有状态只指 Eagle 的 policy 引擎需要维护较长时间窗口的数据集合，如果某个时刻计算节点出了故障，这个 policy 对应的窗口数据就丢失了。在分布式监控系统中，大量 policy 的计算是分布在多个节点上的，而由于偶发的机器故障或者系统升级造成的机器停机是常见的，所以状态管理对于分布式监控是不可或缺的。而正因为 policy 的状态是分布式的，要同时达到高一致性和高效率需要很好的状态管理设计。

Eagle 的 policy 计算引擎建立在 Storm 之上，而 Storm 框架并不支持节点状态的持久化。即使是 Samza、Flink 或者 Spark 这些流计算引擎支持用户状态持久化，但是对于 policy 状态的持久化和恢复依然需要 Eagle 的框架来支持。Eagle 支持对用户透明的 policy 状态持久化和状态恢复，图9是状态管理的逻辑。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56ab2f8999de6.jpg" alt="图9  Eagle policy状态管理的基本架构" title="图9  Eagle policy状态管理的基本架构" />

图9  Eagle policy 状态管理的基本架构

Eagle 状态管理基本分为两个部分

1. 状态持久化

   Eagle 框架会保存 T1 时刻的 policy 状态和在 T1 之后所有的 event 或者 transaction log(tx log)，其中 T1 时的 policy 状态包括状态快照(snapshot)和起始 event Id(txid)。Policy 状态快照和起始 event Id 默认会保存在 Eagle Service 中，而 event 默认会保存在 Kafka 中。

   支持状态容错的计算节点(executor)需要实现 Snapshotable 接口，而 Eagle 框架会启动一个线程定时对这类 Executor 的状态进行快照，并保存到 Eagle Service 中，同时框架会将快照之后的第一个 event Id 保存到 Eagle Service 中。这个起始 event Id 会作为在状态恢复时在 Kafka 中的起始 offset。对状态进行快照之后，框架会将每一个之后的 event 持久化到 Kafka。虽然 Kafka 可以支持很高的写吞吐量，但为了降低 Kafka 负荷，这里框架会设计一个优化，并不是每一个 event 都和 Kafka 服务器交互并且等待 Ack，而是批量地发送给 Kafka 并等待 Ack，在收到 Kafka 的 Ack 之后，再批量对 Storm 框架进行 Ack。这样保证了引入状态管理之后，对流式处理的效率并没有很大的降低。
   
   Storm 框架支持 at-least once 的语义，所以 Eagle 的状态管理框架并不会丢失数据。又因为对 Storm 框架的 Ack 是在收到 Kafka 的 Ack 之后，所以能保证所有存储在 Kafka 里面的增量 event 至少能够被状态恢复阶段处理一次。如果在收到 Kafka 的 Ack 之后，对 Storm 框架的 Ack 过程失败，则 Storm 框架会重新发送这些 event，那么状态恢复可能会对这些 event 处理2次。这是 Storm 不支持 exactly once 语义造成的，未来 Eagle 框架会对这部分进行改进。如果把对 Storm 框架的 Ack 放在 Kafka 数据写和 ack 之前，这时 Kafka 数据写失败，则状态恢复时，会丢失那些还没有来得及写到 Kafka 的数据。
   
   由于 Eagle 使用 Siddhi 作为主要的规则计算引擎，而 Siddhi 在3.0之后已经支持状态的持久化和恢复，这一点确保了可以在流式框架中实现透明的状态持久化和状态恢复，提高了Eagle 监控平台的可靠性(Reliability)。

2. 状态恢复

   状态恢复是状态持久化的反向过程，首先框架会从 Eagle Service 读取最近的状态快照和起始 Event Id，然后从 Kafka 中重播(replay)这些 events，直到最近的 offset。

### Eagle Service 和 UI

Eagle 监控框架是基于元数据驱动的，所以元数据的存储和读写接口至关重要。另外，作为监控系统，将会有海量的原始数据保存下来以供离线分析，Eagle 必须能够支持灵活和复杂的查询，如过滤、聚合、histogram、排序、TopN 、算术表达式、分页等等。而 Eagle Service 正是管理元数据和原始海量数据的服务，同时也是 Eagle 与外部系统及外部数据集成的纽带，它由两大部分组成，一是 API 接口，二是 UI 框架。

#### 类 SQL 的 Restful API

作为一个综合的监控平台，Eagle 提供了一个易用、高性能的 Restful API 来管理 policy，dashboard，甚至是原始监控数据。Eagle API 的易用性是通过类 SQL 的查询接口来表达数据的过滤、投影、聚合、排序等等，而高性能是通过支持 HBase 作为默认存储，并且充分利用 HBase 的分布式计算的功能（如 HBase Co-processor)。当然 Eagle 也支持将关系型数据库作为存储，但建议只存储 metadata，而不是海量的原始监控数据。一个典型的 Eagle 类 SQL 的查询语句如下：

```
/eagle-service/rest/list?query=JobExecutionService[@site="cluster4UT" AND EXP{(@endTime-@startTime)/60/1000}>100]<@normJobName>{sum(EXP{numTotalMaps+numTotalReduces})}&pageSize=100&startTime=0000-00-00 00:00:00&endTime=9999-12-31 23:59:59
```

通过以上的 Restful API, 可以查询到指定时间段的 MapReduce 作业执行的 map 和 reduce 的任务总量，并且这些作业的执行时间大于100分钟，结果按照 normJobName 字段进行分组。

#### 模块化 UI 框架

Eagle UI 由模块化组件构成，因此可以很方便地拓展 UI 功能。UI 由多个 Application 组成，每个 Application 由数个 Feature 组成。Feature 是 UI 中最基本的功能单元，如图10所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56ab2fc09aced.jpg" alt="图10  Eagle UI框架" title="图10  Eagle UI框架" />

图10  Eagle UI 框架

Application。Application 层通过 Eagle API 存储 application 对应的配置信息，feature 通过读取当前所在的 application 配置信息进行 UI 交互。application 间可以共用(common)feature，Application 作为抽象层，不涉及代码编写。Eagle UI 管理员页面提供了 Application 管理页面，可以直接在管理页面对 Application 进行配置。

Feature。Eagle UI 中，每个 Feature 放置于独立的文件夹中，开发应该保证 Feature 间没有直接调用，需要相互协作的功能应该放置于同一个 Feature 之中。

### 数据源接入

Eagle 现在已经支持众多的预定义数据源，比如 HDFS 审计日志、Hadoop JMX metrics、Map/Reduce 作业、Hive 查询语句、JVM GC 日志等等。而 Eagle 要成长为通用的监控平台，必须对任意的数据源进行支持。这正是 Eagle 设计 DSL 抽象层的一个原因，有了 DSL 来屏蔽用户的数据 schema，使 Eagle 支持任意数据源成为可能。 一般化的 metric 接入，比如 Mongdb 数据库的众多 metric，用户也只需要两步即可轻松完成：在 Eagle Web 上通过添加新 stream 定制数据流 schema；将待接入数据解析成定义好的格式导入 Kafka 中。

### 结论

Eagle 是功能强大的现代监控平台，它融合了业界最新的在实时流式数据处理上的重大进展，并对传统的监控系统进行大量的提升优化，特别是支持了 policy 分布式计算、功能可扩展、容错、抽象 DSL 等功能特性，使得平台具有高性能和可扩展性。目前社区正在设计开发基于 Eagle 框架的各类监控应用，同时 Eagle 框架本身也只是处于初步阶段，需要社区一起探索工业级的开源监控平台。

Eagle 在2015年10月进入 Apache 孵化器(Incubator)之后迅速得到了社区的大量关注，目前有多名来自 eBay、Paypal、Dataguise、Blueton、Hortonworks、Cloudera 等公司的贡献者，社区非常活跃。欢迎对流式计算、开放监控平台感兴趣的架构师、软件工程师和运维工程师参加到 Apache Eagle 社区的讨论和代码贡献中，可通过以下链接进行注册。http://eagle.incubator.apache.org/docs/community.html 

【参考资料】
Apache Eagle 文档: http://eagle.incubator.apache.org
Apache Eagle 源码：http://github.com/apache/incubator-eagle 

Apache Eagle PMC 成员陈浩、蒋吉麟、赵晴雯、苏良飞、孙立斌对本文亦有所贡献。