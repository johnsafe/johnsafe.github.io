## Mesos 高可用解决方案剖析



文/王勇桥

本文系作者根据自己对 Mesos 的高可用（High-Availability）设计方案的了解以及在 Mesos 社区贡献的经验，深度剖析了 Mesos 集群高可用的解决方案，以及对未来的展望。

### Mesos 高可用架构概述

首先，我们来参考 Mesos 官方给出的设计架构，如图1所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/577613c9d12cf.png" alt="图1  Mesos架构图" title="图1  Mesos架构图" />

图1  Mesos 架构图

Mesos 采用的也是现在分布式集群中比较流行的 Master／Slave 主从集群管理架构，Mesos master 节点是整个集群的中枢，它责管理和分配整个 Mesos 集群的计算资源，调度上层 Framework 提交的任务，管理和分发所有任务的状态。这种主从架构设计简单，能够满足大多数正常情况下的集群运作需求，目前仍然存在于很多分布式的系统中，比如 Hadoop、MySQL 集群等。但是这种简单的设计存在一个致命缺陷，就是 Mesos master 必须做为一个服务程序持续存在于集群中，它虽然孤立，但是地位举足轻重，不容有失。

在单个 Mesos master 节点的集群中，如果 Mesos master 节点故障，或者服务不可用，虽然在每一个 Slave 节点上的任务可以继续运行，但是集群中新的资源将无法再分配给上层 Framework，上层 Framework 将无法再利用已经收到的 offer 提交新任务，并且无法收到正在运行任务的状态更新。为了解决这个问题，提高 Mesos 集群的高可用性，减少 Mesos master 节点故障所带来的影响，Mesos 集群采用了传统的主备冗余模式（Active－Standby）来支持一个 Mesos 集群中部署多个 Mesos master 节点，借助于 ZooKeeper 进行 Leader 的选举。选举出的 Leader 负责将集群的资源以契约（offer）的形式发送给上层的每一个 Framework，并处理集群管理员与上层 Framework 的请求，另外几个 Mesos master 节点将作为 Follower 一直处于备用状态，并监控当前的状态，当 Mesos master 节点宕机，或服务中断之后，新 Leader 将会很快从 Follower 中选出来接管所有的服务，减少了 Mesos 集群服务的宕机时间，大大提高了集群的可用性。

### Mesos 高可用集群部署

在 Mesos 高可用设计中，引入了 ZooKeeper 集群来辅助 Leader 的选举，这在当前的分布式集群中比较流行，比如 Docker Swarm 高可用集群同时支持利用 consul、etcd、ZooKeeper 进行 Leader 的选举，Kubernetes 也采用了 etcd 等实现了自身的高可用。这种设计可以理解为大集群＋小集群，小集群也就是 ZooKeeper/etcd/consul 集群，它们为大集群服务，比如提供 Leader 的选举，为大集群提供配置数据的存储和服务发现等功能。在一个复杂的系统中，这个小集群可以为系统的多个服务组件同时提供服务。因此在部署高可用 Mesos 集群时，必须首先部署好一个 ZooKeeper 集群。

本文主要介绍 Mesos 的高可用，不会详细介绍 ZooKeeper 的相关知识你可以参考官方文档<https://ZooKeeper.apache.org/>来部署。为了使读者可以快速搭建它们自己的 Mesos 高可用集群，我们将使用 Docker 的方式在 zhost1.wyq.com(9.111.255.10)，zhost2.wyq.com(9.111.254.41)和 zhost3.wyq.com(9.111.255.50)机器上快速的搭建起一个具有三个节点的演示 ZooKeeper 集群，它们的服务端口都是默认的2181。

登录 zhost1.wyq.com 机器，执行如下命令启动第一个 server：

```
# docker run -d \
  -e MYID=1 \
  -e SERVERS=9.111.255.10,9.111.254.41,9.111.255.50 \
  --name=ZooKeeper \
  --net=host \
  --restart=always \
  mesoscloud/ZooKeeper
```

登录 zhost2.wyq.com 机器，执行如下命令启动第二个 server：

```
# docker run -d \
  -e MYID=2 \
  -e SERVERS=9.111.255.10,9.111.254.41,9.111.255.50 \
  --name=ZooKeeper \
  --net=host \
  --restart=always \
  mesoscloud/ZooKeeper
```

登录 zhost3.wyq.com 机器，执行如下命令启动第三个 server：

```
# docker run -d \
  -e MYID=3 \
  -e SERVERS=9.111.255.10,9.111.254.41,9.111.255.50 \
  --name=ZooKeeper \
  --net=host \
  --restart=always \
  mesoscloud/ZooKeeper
```

ZooKeeper 集群搭建好之后，执行以下命令，通过指定一个不存在的 znode 的路径/mesos 来启动所有的 Mesos master，Mesos slave 和 Framework。

- 登录每一个 Mesos master 机器，执行以下命令，在 Docker 中启动所有的 Mesos master：

```
# docker run -d \
   --name mesos-master \
   --net host mesosphere/mesos-master \
   --quorum=2 \
   --work_dir=/var/log/mesos \
   --zk= zk://zhost1.wyq.com:2181,zhost2.wyq.com:2181,zhost3.wyq.com:2181/mesos
```

- 登录每一个 Mesos Agent 机器，执行以下命令，在 Docker 中启动所有的 Mesos agent：

```
# docker run -d \
   --privileged \
   -v /var/run/docker.sock:/var/run/docker.sock \
   --name mesos-agent \
   --net host gradywang/mesos-agent \
   --work_dir=/var/log/mesos \
   --containerizers=mesos,docker \
   --master= zk://zhost1.wyq.com:2181,zhost2.wyq.com:2181,zhost3.wyq.com:2181/mesos
```

>注意：Mesosphere 官方所提供的 Mesos Agent 镜像 mesosphere/mesos-agent 不支持 Docker 的容器化，所以作者在官方镜像的基础至上创建了一个新的镜像 gradywang/mesos-agent 来同时支持 Mesos 和 Docker 的虚拟化技术。

- 使用相同的 Znode 路径来启动 Framework，例如我们利用 Docker 的方式来启动 Docker Swarm，让它运行在 Mesos 之上：

```
$ docker run -d \
--net=host  gradywang/swarm-mesos  \
--debug manage \
-c mesos-experimental \
--cluster-opt mesos.address=9.111.255.10 \
--cluster-opt mesos.tasktimeout=10m \
--cluster-opt mesos.user=root \
--cluster-opt mesos.offertimeout=1m \
--cluster-opt mesos.port=3375 \
--host=0.0.0.0:4375 zk://zhost1.wyq.com:2181,zhost2.wyq.com:2181,zhost3.wyq.com:2181/mesos
```

>注：mesos.address 和 mesos.port 是 Mesos scheduler 的监听的服务地址和端口，也就是你启动 Swarm 的机器的 IP 地址和一个可用的端口。个人感觉这个变量的命名不是很好，不能见名知意。

用上边的启动配置方式，所有的 Mesos master 节点会通过 ZooKeeper 进行 Leader 的选举，所有的 Mesos slave 节点和 Framework 都会和 ZooKeeper 进行通信，获取当前的 Mesos master Leader，并且会一直检测 Master 节点的变化。当 Leader 故障时，ZooKeeper 会第一时间选出新 Leader，然后所有的 Slave 节点和 Framework 都会获取到新 Leader 进行重新注册。

### ZooKeeper Leader 的选举机制

根据 ZooKeeper 官方推荐的 Leader 选举机制：首先指定一个 Znode，如上例中的/mesos（强烈建议指定一个不存在的 Znode 路径），然后使用 SEQUENCE 和 EPHEMERAL 标志为每一个要竞选的 client 创建一个 Znode，例如/mesos/guid\_n 来代表这个 client。

- 当为某个 Znode 节点设置 SEQUENCE 标志时，ZooKeeper 会在其名称后追加一个自增序号，这个序列号要比最近一次在同一个目录下加入的 znode 的序列号大。具体做法首先需要在 ZooKeeper 中创建一个父 Znode，比如上节中指定的/mesos，然后指定 SEQUENCE｜EPHEMERAL 标志为每一个 Mesos master 节点创建一个子的 Znode，比如/mesos/znode-index，并在名称之后追加自增的序列号。

- 当为某个 Znode 节点设置 EPHEMERAL 标志时，当这个节点所属的客户端和 ZooKeeper 之间的 seesion 断开之后，这个节点将会被 ZooKeeper 自动删除。

 ZooKeeper 的选举机制就是在父 Znode（比如/mesos）下的子 Znode 中选出序列号最小的作为 Leader。同时，ZooKeeper 提供了监视（watch）的机制，其他的非 master 节点会不断监视当前的 Leader 所对应的 Znode，如果它被删除，则触发新一轮的选举。大概有两种做法：
 
- 所有的非 Leader client 监视当前 Leader 对应的 Znode（也就是序列号最小的 Znode），当它被 ZooKeeper 删除的时候，所有监视它的客户端会立即收到通知，然后调用 API 查询所有在父目录（/mesos）下的子节点，如果它对应的序列号是最小的，则这个 client 会成为新的 Leader 对外提供服务，然后其他客户端继续监视这个新 Leader 对应的 Znode。这种方式会触发“羊群效应”，特别是在选举集群比较大的时候，在新一轮选举开始时，所有的客户端都会调用 ZooKeeper 的 API 查询所有的子 Znode 来决定谁是下一个 Leader，这个时候情况就更为明显。

- 为了避免“羊群效应”，ZooKeeper 建议每一个非 Leader 的 client 监视集群中对应的比自己节点序小一号的节点（也就是所有序号比自己小的节点中的序号最大的节点）。只有当某个 client 所设置的 watch 被触发时，它才进行 Leader 选举操作：查询所有的子节点，看自己是不是序号最小的，如果是，那么它将成为新的 Leader。如果不是，继续监视。此  Leader 选举操作的速度是很快的。因为每一次选举几乎只涉及单个 client 的操作。

### Mesos 高可用实现细节

Mesos 主要通过 contender 和 detector 两个模块来实现高可用，架构如图2所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/577615f1d608d.png" alt="图2  Mesos高可用架构图" title="图2  Mesos高可用架构图" />

图2  Mesos 高可用架构图

Contender 模块用来进行 Leader 选举，它负责把每个 Master 节点加入到选举的 Group 中（作为/mesos 目录下的一个子节点），组中每个节点都会有一个序列号，根据上文对 ZooKeeper 选举机制的介绍，组中序列号最小的节点将被选举为 Leader。

以上文例子为例（假设作者部署了三个节点的 Mesos master），可以查看 ZooKeeper 的存储来加以验证。

登录到 ZooKeeper 集群中的某一个节点，执行如下命令链接到集群中的某个节点，查看这个 Group：

```
# docker ps
CONTAINER ID        IMAGE                     COMMAND                  CREATED             STATUS              PORTS               NAMES
a6fab50e2689        mesoscloud/ZooKeeper      "/entrypoint.sh zkSer"   50 minutes ago      Up 49 minutes                           ZooKeeper
 
# docker exec -it a6fab50e2689 /bin/bash
# cd /opt/ZooKeeper/bin/
# ./zkCli.sh -server 9.111.255.10:2181
[zk: 9.111.255.10:2181(CONNECTED) 0] ls /mesos
[json.info_0000000003, json.info_0000000004, json.info_0000000002, log_replicas]
```

我们可以看到在/mesos 目录下有三个带有后缀序号的子节点，序号值最小的节点将作为 master 节点，查看序号最小的节点 json.info_0000000002的内容如下：

```
[zk: 9.111.255.10:2181(CONNECTED) 1] get /mesos/json.info_0000000002
{"address":{"hostname":"gradyhost1.eng.platformlab.ibm.com","ip":"9.111.255.10","port":5050},"hostname":"gradyhost1.eng.platformlab.ibm.com","id":"93519a55-4089-436c-bc07-f7154ec87c79","ip":184512265,"pid":"master@9.111.255.10:5050","port":5050,"version":"0.28.0"}
cZxid = 0x100000018
ctime = Sun May 22 08:42:10 UTC 2016
mZxid = 0x100000018
mtime = Sun May 22 08:42:10 UTC 2016
pZxid = 0x100000018
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x254d789b5c20003
dataLength = 264
numChildren = 0
```

根据查询结果，当前的 Mesos master 节点是 gradyhost1.eng.platformlab.ibm.com。

Detector 模块用来感知当前的 master 是谁，它主要利用 ZooKeeper 中的 watcher 的机制来监控选举的 Group（/mesos 目录下的子节点）的变化。ZooKeeper 提供了 getChildren() 应用程序接口，此接口可以用来监控一个目录下子节点的变化，如果一个新子节点加入或者原来的节点被删除，那么这个函数调用会立即返回当前目录下的所有节点，然后 Detector 模块可以挑选序号最小的作为 master 节点。

每一个 Mesos master 会同时使用 Contender 和 Detector 模块，用 Contender 进行 master 竞选。在 master 节点上使用 Detector 模块的原因是在 Mesos 的高可用集群中，你可以使用任意一个 master 节点的地址和端口来访问 Mesos 的 WebUI，当访问一个 Replica 节点时，Mesos 会把这个浏览器的链接请求自动转发到通过 Detector 模块探测到的 master 节点上。

其他的 Mesos 组件，例如 Mesos Agent，Framework scheduler driver 会使用 Detector 模块来获取当前的 Mesos master，然后向它注册，当 master 发送变化时，Detecor 模块会第一时间通知，它们会重新注册（re-register)。

由于 IBM 在 Mesos 社区的推动，在 MESOS-4610 项目中，Mesos Contender 和 Detector 已经可以支持以插件的方式进行加载。现在 Mesos 社区官方仅支持用 Zookeeper 集群进行 Leader 选举，在支持了插件的方式加载后，用户可以实现自己的插件，用另外的方式比如选择用 etcd（MESOS-1806）、consule（MESOS-3797）等集群进行 Leader 选举。

>注意：现在 Mesos 仅用 ZooKeeper 进行 Leader 的选举，并没有用它进行数据的共享。在 Mesos 中有一个 Replicated Log 模块，负责进行多个 master 之间的数据共享、同步等。可以参考 Mesos 的官方文档获取详细的设计<http://mesos.apache.org/documentation/latest/replicated-log-internals/>。同时为了使 Mesos 的高可用不依赖与一个第三方的集群，现在社区正在考虑用 Replicated log 替代第三方集群进行 Leader 选举，具体进度可以参考 MESOS-3574 项目。

### Mesos master recovery

在 Mesos 设计中，master 除了要在 Replicated log 中持久化一些集群配置信息（例如 Weights、Quota 等），0集群 maintenance 的状态和已经注册的 Agent 的信息外，基本上被设计为无状态的。master 发生 failover，新的 master 选举出来之后：

它会首先从 Replicated log 中恢复之前的状态，目前 Mesos master 会从 Replicated log 中 recover 以下信息。

- Mesos 集群的配置信息，例如 weights，quota 等。这些配置信息是 Mesos 集群的管理员通过 HTTP endpoints 来配置的。

- 集群的 Maintenance 信息。

- 之前注册的所有的 Agent 信息（SlaveInfo）。同时 master 会为 Agents  的重新注册（re-register）设置一个超时时间（这个参数通过 master 的slave\_reregister\_timeout flag进行配置，默认值为10分钟），如果某些 Agents 在这个时间内没有向新 master 重新注册，将会从 Replicated log 中删除，这些 Agents 将不能以原来的身份（相同的 SlaveId）重新注册到新的 Mesos master，其之前运行的任务将全部丢失。如果这个 Agent 想再次注册，必须以新的身份。同时为了对生产环境提供安全保证，避免在 failover 之后，大量的 Agents从Replicated log 中删除进而导致丢失重要的运行任务，Mesos master 提供了另外一个重要的flag配置recovery\_slave\_removal_limit，用来设置一个百分比的限制，默认值为100%，避免过多的 Agents 在 failover 之后被删除，如果将要删除的 Agents 超过了这个百分比，那么这个 Mesos master 将会自杀（一般的，在一个生产环境中，Mesos 的进程将会被 Systemd 或者其他进程管理程序进行监管，如果 Mesos 服务进程退出，那么这个监管程序会自动再次启动 Mesos 服务）。而不是把那些 Agents 从 Replicated log 中删除，这会触发下一次的 failover，多次 failover 不成功，就需要人为干预。

另外的，新的 Mesos master 选举出来之后，所有之前注册的 Mesos agents 会通过 detector 模块获取新的 master 信息，进而重新注册，同时上报它们的 checkpointed 资源，运行的 executors 和 tasks 信息，以及所有 tasks 完成的 Framework 信息，帮助新 master 恢复之前的运行时内存状态。同样的原理，之前注册的 Framework 也会通过 detector 模块获取到新的 Master 信息，向新 master 重新注册，成功之后，会获取之前运行任务的状态更新以及新的 offers。

注意：如果在 failover 之后，之前注册并且运行了任务的 Frameworks 没有重新注册，那么它之前运行的任务将会变成孤儿任务，特别对于哪些永久运行的任务，将会一直运行下去，Mesos 目前没有提供一种自动化的机制来处理这些孤儿任务，比如在等待一段时间之后，如果 Framework 没有重新注册，则把这些孤儿任务杀掉。现在社区向通过类似 Mesos Agents 的逻辑，来持久化 Framework info，同时设置一个超时的配置，来清除这些孤儿任务。具体可以参见 MESOS-1719。

### Mesos Agent 健康检查

Mesos master 现在通过两种机制来监控已经注册的 Mesos Agents 健康状况和可用性：

- Mesos master 会持久化和每个 Agent 之间的 TCP 的链接，如果某个 Agent 服务宕机，那么 master 会第一时间感知到，然后：

    - 把这个 Agent 设为休眠状态，Agent 上的资源将不会再 offer 给上层 Framework。

    - 触发 rescind offer，把这个 Agent 已经 offer 给上层 Framework 的 offer 撤销。

    - 触发 rescind inverse offer，把 inverse offer 撤销。

- 同时，Mesos master 会不断的向每一个 Mesos Agent 发送 ping 消息，如果在设定时间内（由 flag.slave\_ping\_timeout 配置，默认值为15 s）没有收到对应 Agent 的回复，并且达到了一定的次数（由 flag. max\_slave\_ping\_timeouts 配置，默认值为5），那么 Mesos master 会：

    - 把这个 Agent 从 master 中删除，这时资源将不会再 offer 给上层的 Framework。

    - 遍历这个 Agent 上运行的所有的任务，向对应的 Framework 发送 TASK\_LOST 状态更新，同时把这些任务从 master 删除。

    - 遍历 Agent 上的所有 executor，把这些 executor 删除。

    - 触发 rescind offer，把这个 Agent 上已经 offer 给上层 Framework 的 offer 撤销。

    - 触发 rescind inverse offer，把 inverse offer 撤销。
    - 把这个 Agent 从 master 的 Replicated log 中删除。

### Mesos Framework 健康检查

同样的原理，Mesos master 仍然会持久化和每一个 Mesos Framework scheculer 之间的 TCP 的连接，如果某一个 Mesos Framework 服务宕机，那么 master 会第一时间感知，然后：

- 把这个 Framework 设置为休眠状态，这时 Mesos master 将不会在把资源 offer 给这个 Framework 。

- 触发 rescind offer，把这个 Framework上 已经收到的 offer 撤销。

- 触发 rescind inverse offer，把这个 Framework 上已经收到的 inverse offer 撤销。

- 获取这个 Framework 注册的时候设置自己的 failover 时间（通过 Framework info 中的 failover_timeout 参数设置），创建一个定时器。如果在这个超时时间之内，Framework 没有重新注册，则 Mesos master 会把 Framework 删除：

    - 向所有注册的 Slave 发送删除此 Framework 的消息。

    - 清除 Framework 上还没有执行的 task 请求。

    - 遍历 Framework 提交的并且正在运行的任务，发送 TASK\_KILLED 消息，并且把 task 从 Mesos master 和对应的 Agent 上删除。

    - 触发 rescind offer，把这个 Framework 上已经收到的 offer 撤销。

    - 触发 rescind inverse offer，把 Framework 上已经收到的 inverse offer 撤销。

    - 清除 Framework 对应的 role。
    - 把 Framework 从 Mesos master 中删除。

### 未来展望

从我个人的角度看，Mesos 高可用这个功能应该做如下增强。

- 现在的设计中，Mesos 的高可用必须依赖一个外部的 ZooKeeper 集群，增加了部署和维护的复杂度，并且目前这个集群只是用来做 Leader 选举，并没有帮助 Mesos master 节点之间存储和共享配置信息，例如 Weights、Quota 等。社区现在已经发起了一个新项目 MESOS-3574，将研究和实现用 Replicated log 来替代 ZooKeeper，帮助 Mesos master 选举和发现 Leader。个人认为价值比较大，它实现之后，可以大大简化 Mesos 高可用架构的复杂度。

- 现在 ZooKeeper 作为搭建 Mesos 高可用集群的唯一选择，可能在比较大的集成系统中不合时宜，在 IBM 工程师的推动下，社区已经将和 Mesos 高可用的两个模块 Contender 和 Detector 插件化，用户可以实现自己的插件来进行 Mesos master 的选举和发现，已经实现了对 etcd 的支持。感兴趣的同学可以参考 MESOS-1806 项目。

- 另外，Mesos 现在这个高可用的设计采用了最简单的 Active-standby 模式，也就是说只有当前的 Mesos master 在工作，其他的 candidate 将不会做任何事情，这会导致资源的浪费。另外在特别大的 Mesos 集群中，master candidates 并不能提供负载均衡。未来是不是可以考虑将 Mesos 高可用修改为 Active-Active 模式，比如让 master candidates 可以帮助处理一些查询的请求，同时可以帮助转发一些写请求到当前的 master 上，来提高整个集群的性能和资源利用率。