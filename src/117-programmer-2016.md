## OpenStack 云端的资源调度和优化剖析

文/胡晓亮

>本文将会从虚拟机初始调度策略、实时监控和优化策略、用户自定义 OpenStack Filter、虚拟机调度失败的 Trouble Shooting Report 和基于拓扑结构调度等方面概括介绍 PRS 的主要功能和使用场景，之后将有一系列文章对每个主题展开深入介绍。

### OpenStack 简介

OpenStack 是旨在为公有及私有云的建设与管理提供软件的一个开源项目，采用 Apache 授权协议，它的核心任务是简化云系统的部署过程，并且赋予其良好的可扩展性和可管理性。它已经在当前的基础设施即服务（IaaS）资源管理领域占据领导地位，成为公有云、私有云及混合云管理的“云操作系统”事实上的标准，在政府、电信、金融、制造、能源、零售、医疗、交通等行业成为企业创新的利器。OpenStack 基于开放的架构，支持多种主流的虚拟化技术，许多重量级的科技公司如 RedHat、AT&T、IBM、HP、SUSE、Intel、AMD、Cisco、Microsoft、Citrix、Dell 等参与贡献设计和实现，更加推动了 OpenStack 的高速成长， 解决了云服务被单一厂商绑定的问题并降低了云平台部署的成本。

### OpenStack 资源调度和优化现状

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d53a3033988.png" alt="图1 OpenStack调度workflow" title="图1 OpenStack调度workflow" />

图1 OpenStack 调度 workflow

OpenStack 的虚拟机调度策略主要是由 FilterScheduler 和 ChanceScheduler 实现的，其中 FilterScheduler 作为默认的调度引擎实现了基于主机过滤（filtering）和权值计算（weighing）的调度算法，而 ChanceScheduler 则是基于随机算法来选择可用主机的简单调度引擎。如图1是 FilterScheduler 的虚拟机调度过程，它支持多种 built-in 的 filter 和 weigher 来满足一些常见的业务场景。在设计上，OpenStack 基于 filter 和 weigher 支持第三方扩展，因此用户可以通过自定义 filter 和 weigher，或者使用 JSON 资源选择表达式来影响虚拟机的调度策略从而满足不同的业务需求。

**Built-in 的 filter（部分）**：

- ComputeFilter 过滤计算节点 down 机的主机；

- CoreFilter 过滤 vcpu 不满足虚拟机请求的主机；

- DiskFilter 过滤 disk 不满足虚拟机请求的主机；

- RamFilter 过滤 ram 不满足虚拟机请求的主机；

- ImagePropertiesFilter 过滤 architecture、hypervisor type 不满足虚拟机请求的主机；

- SameHostFilter 过滤和指定虚拟机不在同一个主机上的主机；

- DifferentHostFilter 过滤和指定虚拟机在同一个主机上的主机；

- JsonFilter 过滤不满足 OpenStack 自定义的 JSON 资源选择表达式的主机：JSON 资源选择表达式形如 query='[">", "$cpus",4]'表示过滤掉 CPUs 小于等于4的主机。

**Built-in的weigher（部分）**：

- RAMWeigher 根据主机的可用 RAM 排序；

- IoOpsWeigher 根据主机的 IO 负载排序。

在一个复杂的云系统中，对云计算资源的监控和优化对于保证云系统的健康运行，提高 IT 管理的效率有重要的作用。最新版本的 OpenStack 也没有提供类似的功能，这可能是由于不同的用户对于云系统的监控对象和优化目标有不同的要求，难于形成统一实现和架构，但是 OpenStack 已经意识到这部分的重要性并且启动了2个项目来弥补这个短板，当前它们都处于孵化阶段：

- Watcher（https://github.com/openstack/watcher）：一个灵活的、可伸缩的多租户 OpenStack-based 云资源优化服务，通过智能的虚拟机迁移策略来减少数据中心的运营成本和增加能源的利用率。

- Congress（https://github.com/openstack/congress）：一个基于异构云环境的策略声明、监控、实施、审计的框架。

### PRS 简介

由于 OpenStack 开源的特性，直接投入商业使用可能面临后期升级，维护，定制化需求无法推进的问题，因此一些有技术实力的公司都基于 OpenStack 开发了自己商业化的版本，这些商业化版本的 OpenStack 都包含了一些独有的特性并和社区开源的 OpenStack 形成了差异化，比如完善了 OpenStack 虚拟机的调度和编排功能，加强了云系统的运行时监控和优化，弥补了云系统自动化灾难恢复的空缺，简化了云系统的安装和部署，引入了基于资源使用时长的账务费用系统等等。PRS（Platform Resource Scheduler）是 IBM Platform Computing 公司基于 OpenStack 的商业化资源调度、编排和优化的引擎，它基于对云计算资源的抽象和预先定义的调度和优化策略，为虚拟机的放置动态地分配和平衡计算容量，并且不间断地监控主机的健康状况，提高了主机的利用率并保持用户业务的持续性和稳定性，降低 IT 管理成本。PRS 采用可插拔式的无侵入设计100%兼容 OpenStack API，并且对外提供标准的接口，方便用户进行二次开发，以满足不同用户的业务需求。

### 虚拟机初始调度策略

虚拟机的初始放置策略指的是用户根据虚拟机对资源的要求决定虚拟机究竟应该创建在哪种类型的主机上，这种资源要求就是一些约束条件或者策略。例如，用户的虚拟机需要选择 CPU 或者内存大小满足一定要求的主机去放置，虚拟机是需要放置在北京的数据中心还是西安的数据中心，几个虚拟机是放在相同的主机上还是放置在不同的主机上等等。原生 OpenStack 调度框架在灵活地支持第三方的 filter 和 weigher 的同时也丧失了对调度策略的统一配置和管理，当前 PRS 支持如图2的初始放置策略，并且可以在运行时动态的修改放置策略。

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d53a3f3aac8.png" alt="图2 虚拟机初始放置策略" title="图2 虚拟机初始放置策略" />

图2 虚拟机初始放置策略

- Packing： 虚拟机尽量放置在含有虚拟机数量最多的主机上 ；

- Stripping：虚拟机尽量放置在含有虚拟机数量最少的主机上；

- CPU Load Balance：虚拟机尽量放在可用 core 最多的主机上；

- Memory Load Balance：虚拟机尽量放在可用 memory 最多的主机上；

- Affinity：多个虚拟机需要放置在相同的主机上 ；

- AntiAffinity：多个虚拟机需要放在在不同的主机上；

- CPU Utilization Load Balance：虚拟机尽量放在 CPU 利用率最低的主机上。

### 实时监控和优化策略

随着 OpenStack 云系统的持续运行，云系统中的计算资源由于虚拟机的放置会产生碎片或分配不均，虚拟机的运行效率由于主机 load 过载而降低，主机的 down 机会造成用户应用程序无法使用等一系列问题。用户可以通过人工干预的方式来排除这些问题。例如用户可以将 load 比较高的主机上的虚拟机 migrate 到其他主机上来降低该主机的 load，通过 rebuild 虚拟机从 down 掉的主机上到其它可用主机上解决用户应用程序高可用性的问题，但这需要消耗大量的 IT 维护成本，并且引入更多的人为的风险。PRS 针对这些问题提供了如图3的两种类型的运行时策略来持续的监控和优化云系统。

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d53a51c182e.png" alt="图3 监控和优化策略" title="图3 监控和优化策略" />

图3 监控和优化策略

- 基于虚拟机的 HA 策略：当主机 down 机后，主机上运行的虚拟机会自动 rebuild 到新的可用主机上。

- 基于主机的 Load Balance 策略：支持 Packing/Stripping/CPU Load Balance/Memory Load Balance/CPU Utilization Load Balance 策略，根据用户设置的阈值持续不断的平衡系统中主机上的计算资源。

用户可以根据业务需要定义相应的优化策略监控主机的健康状况并进行持续不断的优化。例如，用户定义的集群中主机运行时监控 Load Balance 策略是 CPU Utilization  Load Balance，并且阈值是70%，这就意味着当主机的 CPU 利用率超过70%的时候，这个主机上的虚拟机会被 PRS 在线迁移到别的 CPU 利用率小于70%的主机上，从而保证该主机始终处于健康的状态，并且平衡了集群中主机的计算资源。这两种运行时监控策略可以同时运行并且可以指定监控的范围：

- 整个集群：监控的策略作用于整个集群中所有的主机；

- Host aggregation：host aggregation 是 OpenStack 对一群具有相同主机属性的一个逻辑划分，这样用户可以根据业务需求对不同的 host aggregation 定义不同 Load Balance 策略，例如对 aggregation 1应用 Packing 策略， 对 aggregation 2应用 Stripping 策略。

### 用户自定义 OpenStack Filter

OpenStack 对虚拟机的调度是基于对主机的过滤和权值计算，PRS 也实现了相同的功能，并且为提供了更加优雅的接口方便用户定义出复杂的 filter 链，并且配合使用虚拟机初始调度策略从而动态的将用户自定义的虚拟机放置策略插入到虚拟机的调度过程中去满足业务的需求。

- PRS filter 支持定义 working scope：OpenStack 原生的 filter 会默认作用于虚拟机调度的整个生命周期，比如 create、live migrate、cold migrate、resize 等。而 PRS 为 filter 定义了 working scope，这样可以实现让某些 filter 在 create 虚拟机的时候生效，某些 filter 在虚拟机 migrate 的时候生效，并且还支持让一个 filter 工作在多个 working scope。

- PRS filter 支持定义 include hosts 和 exclude hosts：用户可以直接在 filter 中为虚拟机指定需要排除的主机列表或者需要放置的主机列表。

- PRS filter 支持定义 PRS 资源查询条件：用户也可以在 filter 中定义 PRS 资源查询条件，直接选择条件具备住主机列表，例如 select(vcpu>2 && memSize>1024)。

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d53a63e45e9.png" alt="图4 PRS filter workflow" title="图4 PRS filter workflow" />

图4 PRS filter workflow

### 虚拟机调度失败 Trouble Shooting Report

当虚拟机创建失败处于 Error 的时候，云系统应该提供足够的能力方便管理员 trouble shooting，从而尽快排除错误并保证云系统正常运行。造成虚拟机部署失败的原因主要有2种：第一种是调度失败，没有足够的计算资源或者合适的主机满足虚拟机虚拟机的请求；第二种是调度成功，但是在计算节点上部署虚拟机的时候失败。后者原因是多种多样的，比如 Libvirt 错误，Image 类型错误，创建虚拟机网络失败等。当前的 OpenStack 虚拟机的 Trouble Shooting 机制不能够清晰反映问题的原因，需要管理员大量的分析工作，这无疑增加了排除问题的难度和时间。

- 对于虚拟机调度失败，OpenStack 只提供 NoValidHost 的错误异常来表明没有可用的资源，用户无法通过 CLI(nova show $vm_uuid) 得到是哪个 filter 的约束条件造成调度失败。

- 对于部署失败，管理员需要 SSH 到失败的计算节点去检查日志文件分析失败原因。

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d53aa21f2cc.png" alt="图5 Trouble Shooting Report" title="图5 Trouble Shooting Report" />

图5 Trouble Shooting Report

PRS 提供了 trouble shooting report 统一的视图显示虚拟机整个生命周期（create/migrate/resize/等）操作失败的原因如图5，虚拟机 test_vm 在第一次创建的时候由于没有足够的计算资源或者合适的主机而失败（“Error Message”选项有失败原因，“Deployed Host”为空的列表）。由 trouble shooting report 的“Available Hosts"选项可以知道系统中有4台主机，绿色的方框表示系统中每一个 fillter 的资源要求和满足资源要求的主机列表。最终选择的主机应该被包含在所有 filter 主机列表中。由 ComputeFilter 选择的主机列表不包含主机“my-comp3"，可以得此主机的 nov-compute  service 可能被关闭，由 DiskFilter 的主机列表不包含“my-comp1”和主机“my-comp2”可以得知这些主机的可用 disk 资源不足（<1024MB），并且这两个 filter 选择的主机没有交集，因此调度失败，管理员可以根据这些信息能确定调度失败的原因从而轻易地排除错误。

### 基于拓扑结构的调度

OpenStack Heat 是虚拟机组的编排组件，它本身没有调度模块，它基于 Nova 的 FilterScheduler 作为调度的引擎对一组或多组虚拟机进行主机级别的扁平化调度和编排，但这种调度模型每次只能处理一个虚拟机请求，当部署多个虚拟机的时候，它不能根据资源请求进行统一的调度和回溯，将会造成调度结果不准确。PRS 不但支持主机级别的扁平化调度，还支持对一组同构虚拟机内部或者一组虚拟机和另一组虚拟机在一个树形拓扑结构上（region、zone、rack、host）上进行整体调度。基于拓扑结构的多个虚拟机整体调度可以得到一些显而易见的好处，比如在部署的时候， 为了拓扑结构上层级之间或虚拟机之间获得更好的通信性能，可以选择 Affinity 的策略；为了获得拓扑结构上层级之间或虚拟机之间的高可用性，可以选择 Anti-Affinity 策略。PRS 通过和 Heat 的深度集成实现了基于拓扑结构的整体调度。新的 Heat 资源类型 IBM::Policy::Group 用来描述这种一组或多组虚拟机在一个树形的拓扑结构上的部署需求。

- Affinity：用来描述一组虚拟机内部的在指定的拓扑结构层级上是 Affinity 的，或者一组虚拟机和另一组虚拟机在指定的拓扑结构层级上是 Affinity 的；

- Anti-Affinity：用来描述一组虚拟机内部的在指定的拓扑结构层级上是 Anti-Affinity 的或者一组虚拟机和另一组虚拟机在指定的拓扑结构层级上是 Anti-Affinity 的；

- MaxResourceLostPerNodeFailure：用来描述当拓扑结构指定层级发生单点故障时，用户的一组虚拟机在这个层级上的损失率不能高于一个阈值

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d53ce82dfd5.png" alt="图6 Affinity/Anti-Affinity策略" title="图6 Affinity/Anti-Affinity策略" />

图6 Affinity/Anti-Affinity 策略

案例1：如图6，用户定义了2个 auto scaling group tier1 和 tier2，每个 tier 都需要2个虚拟机，其中 tier1 需要虚拟机在 rack 节点上 Anti-Affinity，tier2 需要虚拟机在 rack 节点上 Affinity，并且 tier1 和 tier2 上的虚拟机之间需要满足 Affinity。这个场景类似于在生产环境上部署2组 web application，要求运行 database 的虚拟机 tier1 和运行 web 的虚拟机（tier2）在相同的主机上（方便 web 服务器和 database 服务器通信），并且2个运行 database 的虚拟机 tier1 和2个运行 web 的虚拟机 tier2 不能同时运行在一台主机上（rack 级别上 Anti-Affinity，担心单 rack 单点故障造成所有的 database 服务器或者 web 服务器都不可用）。图的左边是一个部署的结果，红色的虚拟机的是 web 服务器 tier1，黄色的虚拟机是 database 服务器 tier2， 这样 host1 上的 database 服务器直接为 host1 上的 web 服务器提供服务，host6 上的 database 服务器直接为 host6 上的 web 服务器提供，并且 rack1 或者 rack3 的单点故障，不会造成用户 web 服务的中断。
     
<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d53cff2ee12.png" alt="图7  MaxResourceLostPerNodeFailure策略" title="图7  MaxResourceLostPerNodeFailure策略" />

图7  MaxResourceLostPerNodeFailure 策略

案例2：如图7，用户定义了1个 auto scaling group tier1，这个 tier1 需要4个虚拟机，要求当 zone 发生单点故障的时候，用户的4个虚拟机的损失率不能大于50%。这个场景类似于在生产环境上部署一个 Nginx 服务器集群，当发生故障时，总有一半的 Nginx 服务器能够正常工作。图的左边是一个部署的结果，当 zone1 或者 zone2 中任何一个发生故障，用户的应用程序最多损失2个 Nginx 服务器，这样用户在部署的时候就通过整体优化的虚拟机放置策略实现应用程序的高可用性而不必等节点失败的时候通过 PRS HA 策略的监控策略亡羊补牢。