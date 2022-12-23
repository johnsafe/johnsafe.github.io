## Jaguar，一种基于 YARN 的长时服务自动扩展架构

文/马松，马巍娜

长时服务（Long Running Service）是指在生产环境中7×24小时运行的服务，在现实生产环境中，长时服务通常在生产活动中扮演基础服务的角色，如使用 HBase 作为实时随机访问的存储引擎，Kafka 作为系统消息总线等。在实际生产环境中，长时服务使用的资源往往在 YARN 的资源管理范围之外，从而会形成多个小集群硬件资源无法共享的场景。

本文将为大家分享 Jaguar，一种长时服务自动扩展架构，它会通过 YARN 的资源调度能力将长时服务集群与批处理集群合并，并确保长时服务的 SLA（Service Level Agreement，服务级别协议)。

### 基于 YARN 的长时服务调度现状

#### YARN 的长时服务特性

作为 Hadoop 的集群调度器组件，YARN 脱胎自 Hadoop 1.0，在较早版本中调度器特性更加贴合基于批处理类型的短时任务，对长时服务的支持比较薄弱。为了改进对长时服务的支持，Hadoop 社区提交了一个 JIRA（YARN-896)从资源调度、资源隔离、日志处理、安全处理、接口等各个方面长期跟踪相关特性的进展，而从笔者来看，几个支持长时服务的核心特性仍然缺少支持，其中包括：

1. 缺少长时服务发布功能；
2. 缺少长时服务执行容器自动水平/垂直扩展能力；
3. 缺少 YARN 服务进程 RM、NM 出现故障时的高可用方案；
4. 缺少长时服务在 YARN 上的部署模版，不同的长时服务需要遵守 YARN 的应用编程规范。

当然，为了弥补这些不足，YARN 在较新版本中逐渐加入了更多长时服务支持所需要的特性。

1.服务注册与发布（YARN-913）

通过这个特性，客户端可以查询 YARN 集群中动态部署的服务信息。目前，YARN 允许用户将服务的信息以及服务中的 Container 运行时信息以 JSON 格式发布。而在目前的实现中，每个发布的记录映都会射为一个 Zookeeper 路径。在对应的文件中，描述了服务的发布信息。

2.YARN 执行容器资源动态调整（YARN-1197）

较早的 YARN 版本中，一旦容器资源分配完毕，用于执行该容器的资源大小便无法动态调整，除非停止容器并重新分配资源。YARN-1197 特性允许 YARN 的执行容器资源在运行时进行资源的垂直扩展，这样就实现了在长时服务运行过程中，不停止服务实例对实例使用资源大小进行动态调整。该特性目前已经完成了内存维度垂直扩展的开发。
  
3.RM／NM 高可用以及作业保留机制（YARN-128 YARN-556 YARN-1336）

如果长时服务运行在 YARN 上，就应该避免因为 YARN 角色（RM/NM）故障或者非故障导致的长时服务进程重启。

通过 YARN-128 特性，当一个应用被提交到 RM 后，RM 会将应用的上下文持久化到 ZooKeeper 或 HDFS 上；一旦发生 RM 重启，AM 和 NM 会不断进行连接重试，新的 RM 启动后，会向 NM 发送 Resync 命令，NM 杀死包括 AM 在内的所有的执行容器。该特性通过持久化应用上下文信息而解决了 Hadoop 1.0 中一旦 JobTracke r失败必须重新提交任务的问题，但是没有解决执行容器的重启问题；这个问题在 YARN-556 中解决。

YARN-556 解决了在 RM 发生重启后，AM 以及 NM 上的作业仍然保持运行问题；当收到 Resync 时，NM 不再杀死已经在运行的执行容器，而是向新的 RM 上报正在运行的执行容器状态；AM 发现 RM 重启后，也会向 RM 重新注册。

而 YARN-1336 解决了在 NM 重启时，执行容器的状态保持问题。NM 在作业保持配置打开后，会将 NM 以及执行容器的状态存入 LevelDB。当 NM 重新启动时，会从 LevelDB 中重新获取应用状态，执行容器状态，安全相关信息；对于仍然在运行的执行容器将重新附着到 NM 上；对于执行完毕的容器，将获取退出状态进行后续处理。

1.基于标签的调度（YARN-2253）

通过计算节点的标签机制，特定类型的任务会按照被调度器分布在不同的节点之上，这个特性可以将对计算资源有特殊需求的长时服务实例分布在其需要的节点之上。

2.基于 Slider 长时服务架构

Slider 在 Apache 孵化，现已逐渐成熟，提供了长时服务在 YARN 上的部署能力，并对长时服务的各类实例（如 HBASE 的 Region Server，Kafka 的 Broker）的生命期进行管理和监控。

Slider 可以看作是长时服务与 YARN 的适配层，它开放了长时服务的资源定义模版和应用描述模版给服务部署人员，通过与 YARN Resource Manager 的交互获取集群资源，利用这些资源在集群中部署服务实例，并管理这些实例的生命周期。通过统一的 CLI/API，Slider 提供了对长时服务部署、管理、监控等工作，提供了在 YARN 上部署服务的“轮子”。从功能角度，Slider 包含三个角色：

Slider Application Master：支持 YARN AM 与 RM 之间的资源申请协议以及 AM 与 NM 之间的执行容器管理协议，并在早期版本提供了服务发布功能。
 
Slider Agent。Agent 部署在各个计算节点上；基于 Python 实现，Agent 与 AM 保持心跳，接收并执行服务实例的配置、起停、端口动态分配等操作；监控服务实例状态并返回给 AM。

Slider Client。Slider 客户端，支持 CLI 以及长时服务实例的部署功能。一个长时服务的部署、管理、监控，就是在以上三个角色的配合过程中完成。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56a9c4d9a7656.jpg" 
alt="图1  Slider的长时作业提交" title="图1  Slider的长时作业提交" />

图1  Slider 的长时作业提交

1. 客户端通过 Slider CLI 提交长时服务，YARN RM 为 Slider AM 分配资源，并启动 AM；  
2. AM 启动后，根据服务的资源描述文件向 RM 申请资源；  
3. AM 获得资源后，首先启动 Slider Agent，并启动长时服务实例；
4. Agent 在 HDFS 上获取长时服务的定义文件； 
5. Agent 向 AM 注册；
6. AM 生成启动命令； 
7. Agent 向 AM 报告实例状态和配置等信息；
8. AM 对长时服务进行服务注册，发布服务信息。

类似于传统的分布式架构，Slider 也采用了 Master-Slave 的架构，AM 作为 Master 进行资源管理和决策管理；Agent 作为 slave 进行执行。而长时服务的任务描述和资源描述开放给用户描述，一旦确定，会持久化在 HDFS 上。

### 基于策略的长时服务自动扩展架构

虽然社区已经完成了大量将长时服务部署在 YARN 上的工作，但是在生产环境中，长时服务的调度仍然需要管理员的手动介入，例如：通过 Slider 部署的 HBase 集群，当查询负载情况改变后，管理员手工调整（Flex）Region Server 的数量来保证服务的 SLA。当然，在对 Region Server 实例数量进行手工调整的时候，需要对 Region Server 管理的 Region 进行 Rebalance/Demission 等处理。

对于部署在 YARN 上的7×24的长时服务，依靠集群管理员手动管理长时服务的成本非常高且及时性差。亚信橘云在其版本与 Teraproc 合作推出的 Jaguar，提出了基于策略的长时服务自动扩展框架，保证长时服务的 SLA。结合实际运维，Jaguar 的最主要设计目标确定为：

1. 根据 SLA 策略，对长时服务实例动态进行水平/垂直的扩容/减容；
2. 基于时间动态调整 SLA 策略；
3. 根据 SLA 目标，对 SLA 策略进行自动修正。

图2展示了 Jaguar 基本模型。虚线中的部分为 Jaguar 内的主要实体，Jaguar 通过规则引擎进行长时服务实例的动态扩展：

1. 用户定义 SLA 策略后，提交给 Jaguar， Jaguar 会将 SLA 策略加载至规则引擎 Rule Engine。
2. Jaguar 通过部署在工作节点上的指标收集器 Metrics sink 收集各个实例的运行时指标，并通过指标引擎 Metrics Engine 进行指标聚合。
3. 规则引擎策略将转化成规则表达式，当规则表达式满足后，会生成事件告警，在连续触发告警至一定阈值（threshold）后，经过冷却期，规则引擎形成长时服务实例的水平/垂直的扩容/减容。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56a9c551a5551.jpg" alt="图2  Jaguar结构图" title="图2  Jaguar结构图" />

图2  Jaguar 结构图

规则引擎在输出动作（Action）后，会将动作历史转储在 Action DB，作为历史信息；SLA 的分析引擎（Analyze Engine）通过收集的指标和历史信息，以 SLA 为目标，对用户提供的策略进行修正和调整。

![](https://i.imgur.com/PMIn74V.jpg)

图3 基于规则的行为控制

Jaguar 通过控制模块和集群调度器交互，这样保证了 Jaguar 在设计上与实际调度框架解耦。在 YARN 框架下，Jaguar 通过 Hadoop/Slider 接口与 YARN、Slider 交互。在实际中，YARN 以及 Slider 的版本需要支持本文第一部分描述的 YARN 的特性。从长时服务本身角度，与 Jaguar 集成需要一定的集成工作：

1. 需要为长时服务设计 SLA 策略，这点与长时服务的业务相关。
2. 需要为长时服务制作 Slide r的部署包。

下面以 HBase 为例，设计一个简单的 Scale out/Scale up SLA 策略。在实际生产中，SLA 的设计根据集群生产用途不同而不同。在本例中，SLA 简单定义为 HBase 访问时间，SLA 需要保证 HBase 查询请求的平均响应速度。首先需要通过特性的 HBase 指标来设置 SLA 目标，HBase 指标选取如表1所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56a9c5b84006f.jpg" alt="表1  HBase指标" title="表1  HBase指标" />

表1  HBase指标

#### Scale-up 策略

根据约定的 SLA，如果在一段时间内集群部署的 RegionServer 的平均请求处理过慢且内存使用率高，触发垂直扩展动作。

#### 预置门限

1. 策略检查间隔 Interval；
2. 最大请求时间 max_requestProcess_time(MT)；
3. 最大内存使用率 max_memUsage(MMU)；
4. 策略条件连续触发次数 n。

#### SLA策略条件

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56a9c61776d3d.jpg" alt="" title="" />

#### 动作

提高 Region Server JVM 的 Heap 内存数量。

#### Scale out 策略

根据约定的 SLA，如果在一段时间内大部分的 RegionServer 已经触发了垂直扩展策略，但是 HBase 请求处理仍然不能达到 SLA，则触发水平扩展。

#### 预置门限

1. 策略检查间隔 Interval，此值应该数倍于 Scale up 的检查 Interval；
2. 最大请求时间 max_requestProcess_time(MT)；
3. 最大内存使用率 max_memUsage(MMU)；
4. 策略条件连续触发次数 n；
5. 已经触发了 Scale up 的 Region Server 门限(P)；
6. 集群中 Region Server 的数量 m。

#### SLA 策略条件

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56a9c67e23086.jpg" alt="" title="" />

#### 动作

1. 在数量上，以一定步长（如3个）在新的节点上启动 Region Server
2. 稳定后，触发 Region Rebalance
在实现层面，一个实际的 Scale up 的策略描述：

```
{
  "name": "scaleUpHBase",
  "description": "scale up HBase regionserver",
  "enabled": true,
  "interval": 60,
  "timezone": "UTC",
  "corn": "? * MON-FRI",
  "startTime": "9:00:00",
  "duration": "10H00M00S",
  "alert": {
    "successiveIntervals": 2,
    "and": [
      {
        "condition": {
          "componentName": "HBASE_REGIONSERVER",
          "evalMethod": "PERCENT",
          "threshold": ">80",
          "expression": "((last(QueueCallTime_num_ops)*last(QueueCallTime_mean)-             first(QueueCallTime_num_ops)*first(QueueCallTime_mean))/(last(QueueCallTime_num_ops)-first(ProcessCallTime_num_ops)))+ (last(ProcessCallTime_mean)*last(ProcessCallTime_num_ops)-first(ProcessCallTime_mean)*first(ProcessCallTime_num_ops))/(last(ProcessCallTime_num_ops)-first(ProcessCallTime_num_ops)))< 10"
        }
      },
      {
        "condition": {
          "componentName": "HBASE_REGIONSERVER",
          "evalMethod": "AGGREGATE",
          "expression": "MemHeapUsedM > 1900"
        }
      }
    ]
  },
  "actions": [
    {
      "componentName": "HBASE_REGIONSERVER",
      "adjustmentType": "DELTA_COUNT",
      "scalingAdjustment": {
        "COUNT": {
          "min": 1,
          "max": 4,
          "adjustment": 256M
        }
      }
    }
  ]
}

```

从上述的策略描述中，可以看到 Scale up 策略的 SLA 参数：

1. 策略检查周期 interval 60秒；
2. 策略生效时间 UTC 时间 周一至周五09:00向后10小时；
3. 策略触发条件参考 Scale up 策略，监控指标的采集间隔为60s；
4. 当规则满足后，Scale up 的行为是以256M内存为步长，提高 Region Server 的内存使用量。

可以看出，根据规则引擎输出的动作，依赖于 YARN 和 Slider 的调度能力。目前，社区版本的 YARN 仅支持基于 CPU 核数和内存的调度，而不支持 IO 的调度。这对类似 HBase 会出现 I/O 密集操作的长时服务调度会形成调度短板，但相信社区在未来一定能完成这方面特性，形成完整的技术栈。

### 结束语

本文描述了一种基于 YARN 的长时服务自动扩展架构，提供了利用策略定义来保证长时服务 SLA 的方法，在实际生产中，这种方法降低了管理员对集群的人工干预度以及运维成本，并且可以讲长时服务可以和批处理任务等集群合并部署，提高了集群整体资源利用率。