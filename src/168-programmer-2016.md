## 基于 ROS 的无人驾驶系统

文/刘少山

>本文是无人驾驶技术系列的第二篇，着重介绍基于机器人操作系统 ROS 的无人驾驶系统。文中将介绍 ROS 以及它在无人驾驶场景中的优缺点，并讨论如何在 ROS 的基础上提升无人驾驶系统的可靠性、通信性能和安全性。


### 无人驾驶：多种技术的集成

无人驾驶技术是多个技术的集成，如图1所示，一个无人驾驶系统包含了多个传感器，包括长距雷达、激光雷达、短距雷达、摄像头、超声波、GPS、陀螺仪等。每个传感器在运行时都不断产生数据，而且系统对每个传感器产生的数据都有很强的实时处理要求。比如摄像头需要达到 60FPS 的帧率，意味着留给每帧的处理时间只有16毫秒。但当数据量增大之后，分配系统资源便成了一个难题。例如，当大量的激光雷达点云数据进入系统，占满 CPU 资源，就很可能使得摄像头的数据无法及时处理，导致无人驾驶系统错过交通灯的识别，造成严重后果。

<img src="http://ipad-cms.csdn.net/cms/attachment/201605/57281f2853bac.jpg" alt="图1 无人驾驶系统范例" title="图1 无人驾驶系统范例" />

图1 无人驾驶系统范例

如图2所示，无人驾驶系统整合了多个软件模块（包括路径规划、避障、导航、交通信号监测等）和多个硬件模块（包括计算、控制、传感器模块等），如何有效调配软硬件资源也是一个挑战。具体包括三个问题：第一，当软硬件模块数据增加，运行期间难免有些模块会出现异常退出的问题，甚至导致系统崩溃，此时如何为提供系统自修复能力？第二，由于模块之间有很强的联系，如何管理模块间的有效通信（关键模块间的通信，信息不可丢失，不可有过大的延时）？第三，每个功能模块间如何进行资源隔离？如何分配计算与内存资源？当资源不足时如何确认更高的优先级执行？

<img src="http://ipad-cms.csdn.net/cms/attachment/201605/57281f43a3a6f.jpg" alt="图2 无人驾驶软硬件整合" title="图2 无人驾驶软硬件整合" />

图2 无人驾驶软硬件整合

简单的嵌入式系统并不能满足无人驾驶系统的上述需求，我们需要一个成熟、稳定、高性能的操作系统去管理各个模块。在详细调研后，我们觉得机器人操作系统 ROS 比较适合无人驾驶场景。下文将介绍 ROS 的优缺点，以及如何改进 ROS 使之更适用于无人驾驶系统。

### 机器人操作系统（ROS）简介

ROS 是一个强大而灵活的机器人编程框架，从软件构架的角度说，它是一种基于消息传递通信的分布式多进程框架。ROS 很早就被机器人行业使用，很多知名的机器人开源库，比如基于 quaternion 的坐标转换、3D 点云处理驱动、定位算法 SLAM 等都是开源贡献者基于 ROS 开发的。因为 ROS 本身是基于消息机制的，开发者可以根据功能把软件拆分成为各个模块，每个模块只是负责读取和分发消息，模块间通过消息关联。如图3所示，最左边的节点可能会负责从硬件驱动读取数据（比如 Kinect），读出的数据会以消息的方式打包，ROS 底层会识别这个消息的使用者，然后把消息数据分发给他们。

<img src="http://ipad-cms.csdn.net/cms/attachment/201605/57281f5a36fd5.png" alt="图3 ROS系统" title="图3 ROS系统" />

图3 ROS 系统

#### ROS 1.0

ROS 1.0 起源于 Willow Garage 的 PR2 项目，主要组件包括 ROS Master、ROS Node 和 ROS Service 三种。ROS Master 的主要功能是命名服务，它存储了启动时需要的运行时参数，消息发布上游节点和接收下游节点的连接名和连接方式，和已有 ROS 服务的连接名。ROS Node 节点是真正的执行模块，对收到的消息进行处理，并且发布新的消息给下游节点。ROS Service 是一种特殊的 ROS 节点，它相当于一个服务节点，接受请求并返回请求的结果。图4展示了 ROS 通信的流程顺序，首先节点会向 master advertise 或者 subscribe 感兴趣的 topic。当创建连接时，下游节点会向上游节点 TCP Server 发布连接请求，等连接创建后，上游节点的消息就会通过连接送至下游节点。

<img src="http://ipad-cms.csdn.net/cms/attachment/201605/57281f710f8b8.png" alt="图4 ROS Master Node通信" title="图4 ROS Master Node通信" />

图4 ROS Master Node 通信

#### ROS 2.0

ROS 2.0 的改进主要是为了让 ROS 能够符合工业级的运行标准，采用了 DDS（数据分发服务）这个工业级别的中间件来负责可靠通信，通信节点动态发现，并用 shared memory 方式使得通信效率更高。通过使用 DDS，所有节点的通信拓扑结构都依赖于动态 P2P 的自发现模式，所以也就去掉 ROS Master 这个中心节点。如图5所示，RTI Context、PrismTech OpenSplice 和 Twin Oaks 都是 DDS 的中间件提供商，上层通过 DDS API 封装，这样 DDS 的实现对于 ROS Client 透明。设计上 ROS 主页详细讨论了用 DDS 的原因，详情参见 http://design.ros2.org/articles/ros_on_dds.html。

<img src="http://ipad-cms.csdn.net/cms/attachment/201605/57281f8a1506b.jpg" alt="图5 ROS 2.0 DDS" title="图5 ROS 2.0 DDS" />

图5 ROS 2.0 DDS

在无人车驾驶系统中, 我们选择依旧基于 ROS 1.0 开发，而不是 ROS 2.0，主要有以下几点考虑:

1. ROS 2.0 是一个开发中的框架，很多功能还不是很完整，有待更多的测试与验证。而在无人驾驶环境中，稳定性与安全性是至关重要的，我们需要基于一个经过验证的稳定系统来保证系统的稳定性、安全性和性能，以达到无人车的要求。

2. DDS 本身的耗费。我们测试了直接在 ROS 1.0 上使用 DDS 中间件，其中国防科技大学有一个开源项目 MicROS 已经做了相关的尝试，详情参见 https://github.com/cyberdb/micROS-drt。但是实验发现在一般的 ROS 通信场景中（100K 发送者接收者通信），ROS on DDS 的吞吐率并不及 ROS 1.0，主要原因是 DDS 框架本身的耗费要比 ROS 多一些，同时用了 DDS 以后 CPU 占用率有明显提高。但是我们也确认了使用 DDS 之后，ROS 的 QoS 高优先级的吞吐率和组播能力有了大幅提升。我们的测试基于 PrismTech OpenSplice 的社区版，在它的企业版中有针对单机的优化，比如使用了共享内存的优化，我们暂未测试。

3. DDS 接口的复杂性。DDS 本身就是一套庞大的系统，其接口的定义极其复杂，同时文档支持较薄弱。

### 系统可靠性

如上文所述，系统可靠性是无人驾驶系统最重要的特性。试想几个场景：第一，系统运行时 ROSMaster 出错退出，导致系统崩溃；第二，其中一个 ROS 节点出错，导致系统部分功能缺失。以上任何一个场景在无人驾驶环境中都可能造成严重的后果。对于 ROS 而言，其在工业领域的应用可靠性是非常重要的设计考量，但是目前的 ROS 设计对这方面考虑得比较少。下面就讨论实时系统的可靠性涉及的一些要素。

#### 去中心化

ROS 重要节点需要热备份，以便宕机时可以随时切换。在 ROS 1.0 的设计中，主节点维护了系统运行所需的连接、参数和主题信息，如果 ROS Master 宕机了，整个系统就有可能无法正常运行。去中心化的解决方案有很多，如图6所示，我们可以采用主从节点的方式（类似 ZooKeeper），同时主节点的写入信息随时备份，主节点宕机后，备份节点被切换为主节点，并且用备份的主节点完成信息初始化。

<img src="http://ipad-cms.csdn.net/cms/attachment/201605/57281fa374dc7.png" alt="图6 基于ZooKeeper的监控和报警" title="图6 基于ZooKeeper的监控和报警" />

图6 基于 ZooKeeper 的监控和报警

#### 实时监控和报警

对于运行的节点实时监控其运行数据，并检测到严重的错误信息时报警。目前 ROS 并没有针对监控做太多的构架考虑，然而这块方面恰恰是最重要的。对于运行时的节点，监控其运行数据，比如应用层统计信息、运行状态等，对将来的调试、错误追踪都有很多好处。如图7所示，实时监控从软件构架来说主要分成3部分：ROS 节点层的监控数据 API，让开发者能够设置所需的统计信息，通过统一的 API 进行记录；监控服务端定期从节点获取监控数据（对于紧急的报警信息，节点可以把消息推送给监控服务端）；获取到监控数据后，监控服务端对数据进行整合、分析、记录，在察觉到异常信息后报警。

<img src="http://ipad-cms.csdn.net/cms/attachment/201605/57281fbd01833.png" alt="图7 基于ZooKeeper的监控和报警" title="图7 基于ZooKeeper的监控和报警" />

图7 基于 ZooKeeper 的监控和报警

#### 节点宕机状态恢复

节点宕机的时候，需要通过重启的机制恢复节点，这个重启可以是无状态的，但有些时候也必须是有状态的，因此状态的备份格外重要。节点的宕机检测也是非常重要的，如果察觉到节点宕机，必须很快地使用备份的数据重启。这个功能我们也已经在 ZooKeeper 框架下实现了。

### 系统通信性能提升

由于无人驾驶系统模块很多，模块间的信息交互很频繁，提升系统通信性能会对整个系统性能提升的作用很大。我们主要从以下三个方面来提高性能：

第一，目前同一个机器上的 ROS 节点间的通信使用网络栈的 loop-back 机制，也就是说每一个数据包都需要经过多层软件栈处理，这将造成不必要的延时（每次20微秒左右）与资源消耗。为了解决这个问题，我们可以使用共享内存的方法把数据 memory-map 到内存中，然后只传递数据的地址与大小信息，从而把数据传输延时控制在20微秒内，并且节省了许多 CPU 资源。

第二，现在 ROS 做数据 broadcast 的时候，底层实现其实是使用 multiple unicast，也就是多个点对点的发送。假如要把数据传给5个节点，那么同样的数据会被拷贝5份。这造成了很大的资源浪费，特别是内存资源的浪费。另外，这样也会对通信系统的吞吐量造成很大压力。为了解决这个问题，我们使用了组播 multicast 机制：在发送节点和每一接收节点之间实现点对多点的网络连接。如果一个发送节点同时给多个接收节点传输相同的数据，只需复制一份相同的数据包。组播机制提高了数据传送效率，减少了骨干网络出现拥塞的可能性。图8对比了原有的通信机制（灰线）与组播机制（橙色）的性能，随着接收节点数量增加（X 轴），原有的通信机制的数据吞吐量急剧下降，而组播机制的数据吞吐量则比较平稳，没有受到严重影响。

<img src="http://ipad-cms.csdn.net/cms/attachment/201605/57281fd71361e.png" alt="图8 Multicast性能提升" title="图8 Multicast性能提升" />

图8 Multicast 性能提升

第三，对 ROS 的通信栈研究，我们发现通信延时很大的损耗是在数据的序列化与反序列化的过程。序列化将内存里对象的状态信息转换为可以存储或传输的形式。在序列化期间，对象将其当前状态写入到临时或持久性存储区。之后，可以通过从存储区中读取或反序列化对象的状态，重新创建该对象。为了解决这个问题，我们使用了轻量级的序列化程序，将序列化的延时降低了50%。

### 系统资源管理与安全性

如何解决资源分配与安全问题是无人驾驶技术的一个大课题。想象两个简单的攻击场景：第一，其中一个 ROS 的节点被劫持，然后不断地分配内存，导致系统内存消耗殆尽，造成系统 OOM 而开始关闭不同的 ROS 节点进程，从而整个无人驾驶系统崩溃。第二，ROS 的 topic 或者 service 被劫持, ROS 节点之间传递的信息被伪造，导致无人驾驶系统行为异常。

我们选择的方法是使用 Linux Container（LXC）来管理每一个 ROS 节点进程。简单来说，LXC 提供轻量级的虚拟化以便隔离进程和资源，而且不需要提供指令解释机制以及全虚拟化等其他复杂功能，相当于 C++ 中的 NameSpace。LXC 有效地将单个操作系统管理的资源划分到孤立的群组中，以更好地在孤立的群组之间平衡有冲突的资源使用需求。对于无人驾驶场景来说，LXC最大的好处是性能损耗小。我们测试发现，在运行时 LXC 只造成了5%左右的 CPU 损耗。

除了资源限制外，LXC 也提供了沙盒支持，使得系统可以限制 ROS 节点进程的权限。为了避免有危险性的 ROS 节点进程可能破坏其他 ROS 节点进程的运行，沙盒技术可以限制可能有危险性的 ROS 节点访问磁盘、内存以及网络资源。另外为了防止节点中的通信被劫持，我们还实现了节点中通信的轻量级加密解密机制，使黑客不能回放或更改通信内容。

#### 结论

要保证一个复杂的系统稳定、高效地运行，每个模块都能发挥出最大的潜能，需要一个成熟有效的管理机制。在无人驾驶场景中，ROS 提供了这样一个管理机制，使得系统中的每个软硬件模块都能有效地进行互动。原生的 ROS 提供了许多必要的功能，但是这些功能并不能满足无人驾驶的所有需求，因此我们在 ROS 之上进一步地提高了系统的性能与可靠性，完成了有效的资源管理及隔离。我们相信随着无人驾驶技术的发展，更多的系统需求会被提出，比如车车互联、车与城市交通系统互联、云车互联、异构计算硬件加速等，我们也将会持续优化这个系统，力求让它变成无人驾驶的标准系统。