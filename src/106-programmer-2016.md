## 基于 Mesos 和 Docker 构建企业级 SaaS 应用——Elasticsearch as a Service

文/马文，徐磊，吕晓旭

Mesos 从0.23版本开始提供动态预留和持久化卷功能，可利用本地磁盘创建数据卷并挂载到 Docker 容器内，同时配合动态预留功能，保证容器、数据卷、机器三者的绑定，解决了 Docker 一直以来数据落地的问题。本文介绍了去哪儿网的 OPS 团队，利用 Mesos/Docker 开发 Elasticsearch 平台（ESAAS）的建设过程，以及实践过程中的经验总结。


### 背景

从2015年底开始, 去哪儿网（以下简称 Qunar）的 Elasticsearch（以下简称 ES）需求量暴增，传统 ES 部署方式大部分以 kvm 虚机为 ES 节点，需要提前创建好虚拟机，然后在每台机器上部署 ES 环境，费时费力，并且最大的问题是集群扩容和多个集群的管理需要大量的人力成本。随着 ES 节点/集群越来越多，这种方式已经严重加大了业务线的使用/维护成本，同时也不符合 OPS 的运维标准。因此我们开始考虑，是不是能将 ES 做成一个云化的服务，能够自动化运维，即申请即用，有条件地快速扩容/收缩资源。

近几年 Apache Mesos 和 Docker 都是大家讨论最多的话题，尤其对于做运维的人来说，这些开源软件的出现，使机器集群管理、资源管理、服务调度管理都大大节省了成本，也使我们的预想成为可能。借助 Mesos 平台使我们提供的 ES 集群服务化、产品化。总的来说，我们的目标有这几点：

1. 快速集群构建速度
 
2. 快速扩容和快速迁移能力

3. ES 使用/运维标准化

4. 良好的用户交互界面


### 技术调研/选型


我们调研了目前的三个产品/框架，从中获取一些功能方面的参考：

1. Elastic Cloud
 
2. Amazon Elasticsearch

3. Elasticsearch on Mesos

前两者收费并且没有开源，我们从它们的文档和博客中借鉴了许多实用的功能点，可以解决目前阶段的问题。我们根据这两款商用的 ES 服务，结合内部运维体系整理了需要实现的功能点：

1. 集群统一管理
 
2. 集群资源 Quota 限定

3. 数据持久化存储

4. 数据节点和节点资源快速水平/垂直扩容 easily scale out

5. 完整的集群监控和报警

6. 完整的平台监控和报警

7. 统一的集群详细信息和配置中心

8. 集群行为和外围插件的自助配置

9. 集群的发布和配置管理

第三个是开源 Mesos 框架，它的 executor 相当于 ES Node 节点，一个框架服务就是一个 ES 集群，在框架内部处理节点间的相互发现。它支持自动化部署、集群水平扩容、集群状态实时监控以及集群信息查看，如 shard/indices/node/cluster 等，有 Web UI。 此外还支持数据多副本容错等，但也有局限，比如集群维度的数据持久化存储，不能做到数据持久化到固定节点。由于调度的策略，我们事先无法预知节点在重启后会被调度到集群中的哪些机器上去，因此数据的持久化显得尤为重要。例如在集群重启情况下，如若不能保证 ES 节点在原先节点上启动，那对于集群数据和集群服务都是毁灭性的，因为重启之后， ES 节点都“漂移”了，一致性就得不到任何保证。第二个局限是无法完全自助化 ES 集群配置， 比如无法配置不同角色的节点、无法自定义数据存储行为（包括索引分片、副本数量之类）、无法自定义安装插件、无法自助 Script 的使用等，不满足业务线的使用需求。除此之外，还有一些，比如 ES 节点纵向扩容（为某个实例添加更多的 CPU 和内存）、没有 Quota 管理等局限。这就和我们预期的功能有些差距，不能完全满足需求。像这种自行实现一个调度框架的组织方式，缺少灵活性，反而增加了开发的复杂程度，也使我们的服务将会有较大局限，比如：将来可能的服务扩展、随着新业务出现的新功能和组织方式。
 
考虑到上述因素，我们决定利用 Marathon 部署 ES 集群。相较于前三者的局限，Marathon 是一个比较成熟通用的资源调度框架，并且对二次开发非常友好。尤其是1.0版本之后，增加了动态预留和对持久化卷的功能，这些都为 ES 节点数据的持久化存储提供了强有力的支持。鉴于这些特性，我们决定采用 Mesos + Marathon + Docker 方式构建自己的 ESAAS 服务。


### 实施过程


图1是系统的总体结构图，关于 Mesos/Marathon 不在这里过多介绍。

<img src="http://ipad-cms.csdn.net/cms/attachment/201611/58183702b8a63.png" alt="图1  总体结构图" title="图1  总体结构图" />

图1  总体结构图

从总体结构图中我们首先看看如何解决 Quota 分配和集群隔离问题。可以看到，整个系统分了好多层，最底层是一个个物理机，由上层的 Mesos 来管理，它们形成一个集群，也就是一个资源池，Mesos 管理着这些资源，包括 CPU、内存、磁盘和端口等。Mesos 上层是 Marathon 框架，主要负责任务调度，同样也是 ESAAS 系统最重要的一层，我们称之为 Root Marathon。它主要负责调度更上层的 Sub Marathon。这样做主要是考虑功能的隔离，以及为了后续实现 Quota 等功能。

在构建 ESAAS 过程中，我们的工作主要围绕着以下几个核心问题：

1. 数据可靠性
 
2. Quota 分配

3. 集群的隔离

4. 服务发现

5. 监控与报警
 
6. 部署自动化

7. ESAAS Console

### 如何解决 Quota 分配


Mesos 中有 Role（角色）的概念，Role 是资源限制的最小单位，Mesos 根据它来设定每个特定的 Role 能使用的最大资源，其实就是限定资源的 Quota。我们借助这一特性实现资源的隔离和分配。这里不仅涉及了静态资源分配，即通过 Resources 配置，还涉及到了 Mesos 0.27 中的动态申请 Quota 功能。
Root Marathon 不做资源限制，即可以使用整个集群的资源。同时，我们会为每个 Role 划定 Quota，并用 Root-Marathon 为每个 Role 构建一个独享的 Marathon，即图2中的 Sub Marathon。这样，每个 Role 可以使用自有 Sub Marathon 在 Quota 限定范围内的资源，并且享有逻辑上隔离的命名空间和路由策略，更不用担心某一个 Sub Marathon 无节制地使用整个平台资源。

<img src="http://ipad-cms.csdn.net/cms/attachment/201611/581839c2a1910.png" alt="图2  Marathon嵌套示意图" title="图2  Marathon嵌套示意图" />

图2  Marathon 嵌套示意图

### 如何解决集群的隔离


使用嵌套结构来组织 Marathon，除了方便根据 Role 来设置 Quota 之外，还有一个作用就是实现 ES 集群逻辑上的隔离。

我们的每一个 Sub Marathon 都负责维护一个或多个完整的 ES 集群，为每一个 Sub Marathon 分配一个 Role/Quota，等同于每一个 ES 集群的 Group 动态划分资源池，系统的 ES Group 根据业务线来划定。Group 与 ES 集群的关系是一对多的关系，即每个 ES Group 内包含了多套相互隔离的 ES 集群。ESAAS 最终的服务单元就是这一个个 Sub Marathon 所承载的 Elasticsearch Clusters Group 服务。基于 Mesos + Marathon 体系，我们所有的组件都是跑在 Docker 容器里面。

<img src="http://ipad-cms.csdn.net/cms/attachment/201611/581839fbea0c0.png" alt="图3  线上Marathon环境截图" title="图3  线上Marathon环境截图" />

图3  线上 Marathon 环境截图

图4是 ESAAS 系统中一个单台物理机的结构快照。一台物理机运行多个 ES 节点实例，使用不同的端口来隔离同一台机器上不同 ES 集群实例间的通讯，利用 Mesos 的持久化卷隔离 ES 落地数据。除了 ES 节点之外，机器上还有一些其他组件，这些组件和 ES 共同协作来保证服务的稳定可靠。

<img src="http://ipad-cms.csdn.net/cms/attachment/201611/58183a214dd3c.png" alt="图4  物理机部署模块示意图" title="图4  物理机部署模块示意图" />

图4  物理机部署模块示意图

每一个 Sub Marathon 内部结构如图5，包含了 ES Masternode、ES Slavenode、Bamboo/Haproxy、ES2graphite、Pyadvisor 以及 StatsD，构成一个完整的集群模块。

<img src="http://ipad-cms.csdn.net/cms/attachment/201611/58183a4fba63d.png" alt="图5  ESAAS集群模块关系图" title="图5  ESAAS集群模块关系图" />

图5  ESAAS 集群模块关系图

### 如何解决服务发现

下面看看我们是如何解决 Elasticsearch 集群内服务发现问题的。首先说明一下，我们选择了 ES 的单播模式来构建集群，主要目的是降低建群之间相互污染的可能。我们 ES 集群的标准配置是6个 ES 节点：3个 Masternode 节点，3个 Datanode 节点。它们会作为两个不同的 Marathon App 存在。每个 App 有三个 Task, 每个 Task 是一个 ES node（无论是 Masternode 还是 Datanode）。之所以要将这两种节点分开来作为两个 App，主要的原因就是，这两种角色的节点会有不同的配置方式和不同的资源分配方式，Master 节点不做数据存储，起到集群 HA 的作用，所以我们为它分配了最小的 CPU 和内存资源。Datanode 节点真正的存储数据在配置上，也会有不同，因此将这两种节点分开来，方便管理。

两种类型的 ES 节点，是通过 Bamboo + Haproxy 相互连接成一个集群的。Bamboo 是一个用于服务发现的开源工具，能根据 Marathon 的 Callback 信息动态的 Reload Haproxy，从而实现服务自动发现。它的基础原理就是注册 Marathon 的 Callback 来获取 Marathon 事件。ES 节点间的互连之所以要使用 Bamboo 来进行服务发现，主要原因就是我们无法在节点被发布前预知节点被分配到了哪些 Mesos Slave 机器上。因为节点的调度结果是由 Marathon 内部根据 Slave 可用的 CPU、内存、端口等资源数量来决策，无法提前预知。这样就有一个问题，在发布的时候， 三个 Masternode 之间就无法预知对方的地址。为了解决这个问题，我们使用 Bamboo + Haproxy 来动态发现 Masternode 地址实现节点之间的互连。

Bamboo + Haproxy 能做到动态服务发现， 那么它和 ES 节点间是怎么协作的呢？为了能保证集群节点之间正确的互连，在 ES 两种类型的节点没有部署之前，首先需要部署一个 Bamboo + Haproxy 工具用作服务发现。Bamboo 会注册 Marathon Callback 从而监听 Marathon 事件。Haproxy 也会启动并监听一个前端端口。在开始部署 Masternode 时，由于有新的 App 部署，Marathon 就会产生一个事件从而 Callback 所有注册进入 Marathon 的接口，这时 Bamboo 会根据这个 Callback 调用 Marathon API，根据配置的规则去获取指定 App 中 Task 的机器和端口信息，也就是 Masternode 的机器和端口信息。这些信息得到之后，Bamboo 会去修改 Haproxy 的配置并 Reload Haproxy，将 Task 机器和端口信息作为后端 Server，使得 Haproxy 前端所监听的端口可以正确地转发到后端这些机器端口上去。这里，我们使用4层 TCP 协议。

<img src="http://ipad-cms.csdn.net/cms/attachment/201611/58183ab57a84a.png" alt="图6  端口转发示意图" title="图6  端口转发示意图" />

图6  端口转发示意图

至此，服务发现的过程就已经完成。在 Bamboo 服务部署完成之后，Haproxy 服务就已经启动，所以，Haproxy 这个前端的端口是一直存在的。因此，在 Masternode 启动时，可以先调用Marathon API 获取到 Haproxy 这个前端的端口和机器信息，并将这个信息写入 Masternode 配置文件中，节点在启动时，就会以单播的方式去连接这个 Haproxy 的前端端口，进而将请求转发到后端三台 Masternode 节点上去。可以看到，Masternode 的互相发现其实就是通过 Haproxy 再去连接自己，是一个环形连接。对于 Datanode，也是同样的方式，唯一区别于Masternode 的就是 Datanode 不会去自己连接自己。

### 如何保证数据可靠


ES 是一个带状态的服务，在软件层面提供了数据的冗余及分片，理论上是允许集群中部分节点下线并继续对外提供服务。我们要尽量保证底层的 Mesos/Marathon 能够充分利用 ES 的数据冗余能力，提高数据可靠性。低版本的 Mesos/Marathon 不支持动态预留和持久化卷，数据只能通过 Mount Volume 方式驻留在宿主机的某个目录，不同集群间的数据目录需要人工管理。而且默认 Failover 功能可能会导致 Datanode 被调度到其他机器上，变成空节点。现在 Mesos 的动态预留和持久化卷功能可以解决这一问题，保证容器与数据绑定，“钉”在某个节点，保证实例重启后直接本地恢复数据。同时为了尽最大可能保证数据不会丢失，我们做了以下规定：

1. 每个索引至少有一个副本（index.number_of_replicas ≥1）；
 
2. 每个宿主上同一个集群的节点等于 Replica 个数。如果某个集群 A 的配置为 index.number_of_replicas= 2，那么每个宿主上可以为 A 启动2个节点实例；
 
3. index.routing.allocation.total_shards_per_node=2，禁止多个主 shard 集中同一实例。

除了数据层面的多份冗余外，我们还默认提供了 Snapshot 服务，业务线可以选择定时/手工 Snapshot 集群的数据到 HDFS 上。最后就是集群级的冗余，ESAAS 允许申请热备集群，数据双写到在线/热备集群中，当在线集群出现故障时切换到热备集群提供服务。


### 监控与报警

从上面可以看到，Masternode、Datanode、Bamboo + Haproxy 这三个是组成一个 ES 集群必不可少的组件。除此之外，我们还有一些用于监控的组件，如 Sub Marathon 结构图所示的那样， 我们提供了两个维度的监控：一个是集群维度的监控，一个是容器维度的监控。 

我们选择了 ES2graphite（一个 py 脚本）来收集 ES 集群指标，其主要原理就是通过 ES 的 API 获取 ES 内部各项指标，将收集的指标打到后端监控系统——Watcher 中（Watcher 是 Qunar OPSDEV 基于 Graphite+Grafana+Nagios 等开发的一套监控/报警系统），最后通过监控系统的指标看板展示集群当前状态。如图7所示，为某个 ES 集群在 Watcher 上的监控看板。

<img src="http://ipad-cms.csdn.net/cms/attachment/201611/58183b5fb65cb.png" alt="图7  ES监控截图" title="图7  ES监控截图" />

图7  ES 监控截图

为了监控容器的运行状态，我们自主开发了容器指标收集工具——Pyadvisor。Pyadvisor 会收集容器的 CPU，内存和 I/O 等信息，并将其发往后端的监控系统 Watcher。图8为容器的监控看板截图。 

<img src="http://ipad-cms.csdn.net/cms/attachment/201611/58183b8880485.png" alt="图8  容器监控截图" title="图8  容器监控截图" />

图8  容器监控截图

监控的重要性不言而喻，它就相当于维护者的眼睛，通过这些监控，我们可以定位集群问题。除了监控，我们还提供了两个集群的基础报警，一个是集群状态报警，一个是集群的 GC 时间报警。ES 集群状态有 Green、Yellow、Red 三种状态。我们的报警条件是非 Green 就报警，这样尽早发现可能出现的异常情况。GC 时间报警指的是一分钟内集群节点 Full GC 的时间，报警阈值我们会根据集群的使用场景去调节。GC 时间指标在一定程度上往往标识着集群当前的 Health 程度，过高的GC时间可能导致集群停止对外界请求的一切响应，这对于一个线上服务往往是致命的。


### 自动化部署

ESAAS 系统设计初衷有快速构建的要求，否则我们避免不了繁重的人力成本。整个系统的组件几乎全部依赖于 Docker，Docker 容器的平台无关特性使得我们可以提前将环境和可执行体打包成一个镜像，这在很大程度上节约了我们为部署环境差异而付出的人力成本。快速的构建和发布是一个 SaaS 系统必须具备的特性。我们使用 Jenkins 来完成 ESAAS 系统的构建和发布工作。

对于 ESAAS 系统来说，快速的构建就是快速的生成配置，发布就是按照规则顺序创建 Marathon APP，图9是一个 Jenkins 构建发布的流程图。

<img src="http://ipad-cms.csdn.net/cms/attachment/201611/58183bce50d9f.png" alt="图9  部署流程图" title="图9  部署流程图" />

图9  部署流程图

ES 集群的发布，由两个 Jenkins 任务完成。首先，第一个 Jenkins 任务会调用集群初始化脚本，生成 Marathon 和 ES 的配置，并将这些配置提交 Gitlab 做归档管理。然后，第二个 Jenkins 任务（如图10）会通过 Gitlab API 读取配置，并使用 Marathon API 创建集群中的各组件（Marathon App）。图10是我们线上一个 Jenkins 发布任务的截图（二次开发过的任务面板）。

<img src="http://ipad-cms.csdn.net/cms/attachment/201611/58183c01e12f3.png" alt="图10  Jenkins Job截图" title="图10  Jenkins Job截图" />

图10  Jenkins Job 截图

### ESAAS Console


为了提供一个标准化的 ES 产品，统一用户界面，我们模仿 ES Cloud 开发了 ESAAS Console（以下简称 Console）。ESAAS Console 包括了集群概况展示、集群配置、操作日志查询等功能。

集群概况页汇总了节点地址、端口信息、配置 Git 地址、监控地址以及常用插件等信息（如图11），方便用户接入 ES 集群和插件工具的使用。 

<img src="http://ipad-cms.csdn.net/cms/attachment/201611/58183c4275dbf.png" alt="图11  ESAAS Console 截图" title="图11  ESAAS Console 截图" />

图11  ESAAS Console 截图

同时，我们也将配置管理和插件管理整合到了 Console 中（如图12所示）。用户可以通过使用 ES API 修改集群配置。另外，ES 插件多种多样，相互搭配使用可以提高集群管理的灵活度，我们提供一键安装插件功能，免去了手动为每一个 ES 节点安装插件的麻烦。集群配置页还提供了 Kibana 一键安装功能（Kibana 是一个 ES 数据可视化看板）。这些功能都大大降低了用户使用 ES 的成本。

<img src="http://ipad-cms.csdn.net/cms/attachment/201611/58183c7680b78.png" alt="图12  ESAAS Console 截图" title="图12  ESAAS Console 截图" />

图12  ESAAS Console 截图

所有对集群的操作都会记录下来（如图13），方便回溯问题。我们这些操作信息后端存储也是使用的 ESAAS，一定程度地做到了服务闭环。

<img src="http://ipad-cms.csdn.net/cms/attachment/201611/58183c9f6092b.png" alt="图13  ESAAS Console截图" title="图13  ESAAS Console截图" />

图13  ESAAS Console 截图


### 总结


截止到写这篇文章时,  ESAAS 已经稳定运行了半年有余，为业务线提供了44组 ES 集群，涵盖了离线/热备/在线集群。并且和我们 OPS 提供的实时日志系统完成了功能整合，支持线上日志导入，双写以及 ETL 等，为业务线提供数据/服务闭环的平台。目前 ESAAS 服务的几个规模指标如下：

1. ESAAS 集群机器数量：77台服务器

2. Datanode 数据节点机器数量：66台服务器

3. 当前托管的集群数量：44个集群
 
4. 当前数据存储总量量：120TB 左右
 
5. 当前覆盖业务线：30 个
 
6. 最大的集群数据量：25.6 T


### 遇到的问题和解决方法


1. Mesos Role 不能动态地创建

Mesos 的 Role 不能动态地创建, 而我们 ESAAS 的资源隔离是根据 Mesos 的 Role 来完成。我们提前创建了 Role 来解决这一问题。
 
2. Mesos Slave 重启之后重新加入集群，原先跑在这台 Slave 上的 Task 将不可恢复

Mesos Slave 节点由于某些原因机器需要重新启动的时候，Mesos 会将机器识别为新的 Slave，这就导致了机器重启之后，原来跑在该节点上的 Task 不可恢复。我们在研究了 Mesos 内部机制之后解决了这一问题，即持久化 boot_id 文件。

### 下一步计划

我们的 ESAAS 系统还处于探索阶段，仍然存在不少待解决的问题。比如：

1. ESAAS 服务的计费
 
2. ES 和平台日志的收集 

3. 独立 ES 集群间的数据迁移服务
 
4. 独立 ES 集群间的 I/O及CPU 优先级管理 

<img src="http://ipad-cms.csdn.net/cms/attachment/201611/58183d350c587.png" alt="" title="" />