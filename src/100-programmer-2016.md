## Swarm 和 Mesos 集成指南——资源利用率优化实践

文/王勇桥

>Apache Mesos 作为一个非常优秀的分布式资源管理和调度系统，如何高效的管理和分配资源，必然成为它研究和努力的主要方向之一。本文是基于 IBM Platform DCOS Team 在资源调度领域的经验，以及他们在 Mesos 社区提升 Mesos 资源利用率的实践，深度剖析了如何在 Mesos 中提供 Revocable 资源来提高 Mesos 数据中心的资源利用率并结合作者在 Docker Swarm 社区的贡献经验，重点讲解了在 Docker Swarm 如何支持 Mesos 的 Revocable 资源。

### Mesos 集群资源调度策略

Apache Mesos 能够成为数据中心资源调度的内核，一个非常重要的原因就是它能够支持大多数 framework 的接入，那么 Mesos 要考虑的一个主要的问题就是如何为众多的 framework 分配资源。在 Mesos 中，这个功能是由 Master 中的一个单独的模块（Allocation module）来实现的，因为 Mesos 设计用来管理异构环境中的资源，所以它实现了一个可插拔的资源分配模块架构，用户可以根据自己的需求来实现定制化的分配策略和算法。 

本章节将主要介绍官方默认给出了一个资源调度的流程：

DRF(Dominant Resource Fairness) Allocator，它的目标是确保每一个上层的 Framework 能够接收到它最需资源的公平份额。下边我们先简单介绍一下 DRF 算法。在 DRF 中一个很重要的概念就是主导资源（dominant resource），例如对于运行计算密集型任务的 framework，它的主导资源是 CPU，对于运行大量依赖内存的任务的 framework，它的主导资源是内存。DRF Allocator 在分配资源之前，首先会按照资源类型计算每个 framework 占有（已经使用的资源＋已经以 offer 的形式发给 framework 但是还没有使用的资源）份额百分比，占有比最高的为这个 framework 的主导资源，然后比较每个 framework 主导资源的百分比，哪个值小，则将这个机器上资源发给对应的 framework。你可以参考 Benjamin Hindman 的这篇论文 https://www.cs.berkeley.edu/~alig/papers/drf.pdf 来了解 DR F算法的详细说明。

另外，Mesos 集群管理员可以通过以下几种方式提供更多的资源分配策略：

给某个 framework 设置权重（weight）来使某个 framework 获取更多的资源。

可以使用 static/dynamic reservation 指定将某一个 Slave 节点上的资源发送给特定 role 下的 framework。

可以为某个 role 配置 quota 来保证这个 role 下的 framework 在整个 Mesos 集群中获取资源的最小份额。

Framework 收到资源之后，可以创建持久化存贮卷，保证这些存储资源不能被其他的 framework使用。

### 使用 Revocable 资源来提高资源利用率

Mesos 默认的资源调度模块 DRF Allocator。在资源分配方面可能存在以下几个方面的问题：

1. 资源碎片的问题。由于 DRF 总是追求公平，它会把剩余资源公平的分配给所有的 framework，可能会导致所有的 framework 在一个分配的周期中都没有足够的资源来运行它的任务。 
2. 本性贪婪的 framework 为了提高性能，有时候总试图去 hold offer，即便当前没有需要执行的任务，这些 resource 也不能分配给另外的 framework 使用。
3. 对于 gang-scheduing（同时申请一组资源）类型的应用，由于 Mesos 在一个时刻只能提供集群的部分资源，那么这种类型的应用可能需要等待比较长的时间才可以运行。

为了适当的提高 Mesos 的资源利用率，Mesos 社区现在引入了 Revocable 类型的资源。这种资源是一种可以被 Mesos 自行回收的资源，也就是说如果上层的 framework 使用这种资源来运行的它们的任务，那么这些任务就有随时被杀掉的可能。它被主要设计用来运行一些可靠性要求比较低的、无状态任务。也就是说现在 Mesos 主要为上层的 framework 提供两种资源：

Non-revocable resources：Mesos 提供的最常见的资源类型，这种资源一旦发给某个 framework，可以永远保证对这个 framework 可用，framework 一旦用这种资源运行它的任务，那么这个任务所占用的资源不能被 Mesos 收回，除非 framework 主动杀掉任务或者任务结束。

Revocable resources：这种资源是 framework 没有充分利用的资源，Mesos 会把这部分资源再发送给其他的 framework 使用，如果其他 framework 用这种资源运行任务，那么这个任务有可能在某个时候被 Mesos 杀掉来回收资源。这种资源主要运来执行 best-effort 任务，例如后台分析、视频图像处理、芯片模拟等一些低优先级的任务。

首先 Mesos 在 MESOS-354 项目中初次引入了 Revocable 类型的资源，它是我们所说的其中一种 Revocable 类型的资源，它主要考虑的应用场景是：在 Mesos 集群中，Mesos 对资源的分配率一般可以达到80%以上，但实际的资源利用率却低于20%。在这个项目中，它提供了一种资源的再分配机制，就是把临时没有使用的资源（已分配给某些任务，而且这些任务正在运行，但没有完全使用已经分配给它们的资源）再分配给其他的任务。这种类型的 revocable 资源被称为 Usage Slack 的资源，其生命周期如下：

第一步首先要确定哪些资源是 Usage slack 的 revocable 资源。每个 Mesos Salve 节点上的 Resource estimator 会定期和 Resource Monitor 模块交互来计算 Usage slack，如果前后两次计算的值不同，则把这些 revocable 的资源上报给 Mesos master。

Mesos allocator 会收到每个 Slave 上报的 Usage slack 的 revocable resource，然后把这些 revocable resource 发送给上层的 framework。注：默认情况下 framework 是不接收 revocable resource 的，如果
framework 在注册时候指定 REVOCABLE_RESOURCES capability，那么它将可以收到 revocable resource。

Framework 可以选择使用这些 revocable resource 运行任务，这些 revocable 任务会像正常的任务一样被运行在对应的节点上。

每个 Slave 上的 Monitor 会监视原来 task 的运行状况，如果它的运行负载提高，则 Qos controller 模块会杀掉对应的 revocable task 来释放资源。

另外的一种 Revocable 资源则在 MESOS-4967 项目中引入，它主要由 IBM Platform DCOS Team 的工程师主导设计开发，目前基本的设计和编码已经完成，正在 review 阶段，很快会在最近的 Mesos 版本中支持。它主要是考虑到在 Mesos 中有些资源某些 role reserve，但是没有被使用的情况（Allocation Slack），我们在后续的文章中会对其进行详细介绍。

对于 Revocable 类型的资源，需要注意以下几点：

Revocable 资源不能动态的 reserve。

Revocable 资源不能用来创建存储卷。

运行任务的时候，对同一种资源，revocable 资源和 non-revocable 资源不能混用，但是不同类型的资源可以混用。比如 CPU 用 revocable 资源，Memory 可以使用 non-revocable 资源。

如果一个 task 使用了 revocable 资源或者它对应的 executor 使用了 revocable 资源，则整个 container 都被视为 revocable task，可以被 Qos controller 杀掉。

### 在 Swarm 中使用 Mesos 的 Revocable 资源

Docker Swarm 目前的版本（v1.1.3）不支持接收 Mesos 的 revocable resource。IBM Platform DCOS Team 对 Swarm 做了改进，使 Swarm 可以接收 revocable resource，并且 Swarm 的终端用户可以选择使用 revocable 资源来运行他们低优先级的任务，或者使用non-revocable 资源来运行他们的高优先级任务。这样的话，Swarm 和其他的 framework 共享 Mesos 集群资源时，其他 framework 的未充分利用的资源将可以被 Swarm 使用，这样的话 Mesos 集群整体的资源利用率将得到提升。注：这个新的改进可能会在 Swarm 的v1.1.3之后的某个版本 release，如果哪位读者需要尝试这个新的特性，可以关注这个 https://github.com/docker/swarm/pull/1946 pull request。

这个特性主要支持以下几种使用场景：

Swarm 集群的管理员可以配置 Swarm 集群来接收 Mesos 的 revocable resource。

Swarm 的终端用户可以只选择非 revocable 资源来运行他们的高优先级任务。

Swarm 的终端用户可以优先选择非 revocable 资源来运行他们的中优先级任务，如果没有非 revocable 资源可用，也可以选择 revocable 资源。

Swarm 的终端用户可以只选择 revocable 资源来运行他们的低优先级任务。

Swarm 的终端用户可以优先考虑选择 revocable 资源来运行他们的低优先级任务。如果没有 revocable 资源可用，也可以选择非 revocable 资源来运行。

Swarm 的终端用户可以查看他们的 container 使用什么资源来运行的。

Swarm 支持 Revocable resource 主要做了以下几个方面的改进：

在 Swarm manage 命令中新增一个 cluster 参数选项 mesos.enablerevocable，通过将这个参数设置为 true 来使 Swarm 可以接收 Mesos 发送的 revocable 资源。当然按照 Swarm 之前的设计原则，你可以 export SWARM_MESOS_ENABLE_REVOCABLE 环境变量来替代那个参数选项。

Mesos 会定期的发送 offer 给 Swarm，当 Swarm 接收到这些 offer 之后，需要修改对应 Docker engine 的资源类型，现在支持的有三种类型：Regular（表示这个节点上只有非 revocable 资源），Revocable（表示这个节点上只有 revocable 资源），Any（表示这个节点上即有非 revocable 资源，也有 revocable 资源），Unknown（表示这个节点的资源类型不确定，也就是这个节点上暂时没有可用的资源，这类节点如果没有正在运行的任务，它默认会被 Swarm 很快地删除）。

根据 Swarm 所提供的 constraint 来使用 revocable 或者非 revocable 资源来创建任务。最常见的 constraint 写法有以下几种：

1. constraint:res-type==regular：表示只使用非 revocable 资源来运行任务，如果没有非 revocable 资源，那么这个任务将会被放到任务队列中等待更多的非 revocable 资源，直到超时。 
2. constraint:res-type==~regular：表示优先使用非 revocable 资源来运行任务，如果没有非 revocable 资源，那么选择使用 revocable 资源来执行。如果这两种资源都不够，任务将会被放到任务队列中等待 Mesos 发送更多的 offer，直到超时。
3. constraint:res-type==revocable：表示只使用 revocable 资源来运行任务，如果没有 revocable 资源，那么这个任务将会被放到任务队列中等待更多的 revocable 资源，直到超时。

constraint:res-type==~revocable：表示优先使用 revocable 资源来运行任务，如果没有 revocable 资源，那么选择使用非 revocable 资源来执行。如果这两种资源都不够，任务将会被放到任务队列中等待 Mesos 发送更多的 offer，直到超时。

constraint:res-type==＊：表示可以使用任何类型的资源来运行任务，如果 revocable 资源和非 revocable 资源都不够，那么这个任务将会被放到任务队列中等待 Mesos 发送更多的 offer，直到超时。

几个细节需要特别注意：

1. 由于 Swarm 的 constraint 支持正则表达式，当指定的正则表达式同时满足使用 revocable 和非 revocable 的资源，默认优先使用非 revocable 的资源来运行任务，如果非 revocable 的资源不够用，我们才会使用 revocable 资源运行任务。 
2. 在 Swarm 对使用 revocable 资源的设计中，同一个任务不支持即使用 revocable 资源又使用非 revocable 资源。但是 Mesos 的这种限制只是局限于同一类型的资源，也就是说在 Mesos 中，不同类型的资源是可以混用的。个人认为 Mesos 的这种设计不是很合理，原因是某个任务使用了 revocable 资源，那么它就会被当作 revocable 任务，可以被 Qos 控制器杀掉，根本就不会考虑它所使用的非 revocable 资源。
3. 在 Swarm 中创建 container 的时候，同一个 constraint 是可以指定多次的，Swarm 会把它当作各自不同的 constraint 来对待，一个一个进行过滤。但是在此设计中，我们不允许 Swarm 用户指定多个 res-type，如果指定多个，创建 container 将会失败。
4. Mesos 现在的设计是将非 revocable 资源和 revocable 资源放在了同一个 offer 中发给上层 framework 的，也就是说当 Swarm 收到 offer 之后，需要遍历 offer 中的每个资源来查看它的类型，来更新对应 Docker engine 的资源类型。在创建容器的时候，需要遍历某个节点上的所有的 offer 来计算所需要的资源，这可能会带来性能上的问题。比较庆幸的是， Mesos 正在考虑对这个行为作出改进，也就是把 revocable 资源和非 revocable 资源用不同的 offer 来发送，也就说到时候 framework 收到的 offer 要么全是 revocable 资源，要么全是非 revocable 资源，这样的话就可以提高 Swarm 计算资源的性能。顺便说一下，Mesos 做这个改进的出发点是考虑到 rescind offer 的机制，因为 rescind offer 是以整个 offer 为单位回收资源的，如果我们把 revocable 资源和非 revocable 资源放到一个 offer 进行发送，那么回收 revocable 资源的时候，不可避免的会同时回收非 revocable 资源。

下文将带领大家进行实战，用 Swarm 使用 Mesos 的 revocable 资源。在进行之前，我建议读者先阅读和这篇文章相关的前两篇《Swarm 和 Mesos 集成指南－原理剖析》和《Swarm 和 Mesos 集成指南－实战部署》。

为了简化环境部署的复杂度，使读者可以轻松地在自己的环境上体验以下这个新的特性，我提供了三个 Docker Image，读者可以很轻松的使用这三个 Image 搭建起体验环境。

1.准备三台机器，配置好网络和 DNS 服务，如果没有 DNS 服务，可以直接配置/etc/hosts，使这三台机器可以使用机器名相互访问。我使用的是三台 Ubuntu 14.04的机器：

```
wangyongqiao-master.grady.com
wangyongqiao-slave1.grady.com
wangyongqiao-slave2.grady.com
```

2.在 wangyongqiao-master.grady.com 上执行以下 Docker Run 命令启动 Zookeeper 节点，用作 Mesos 集群的服务发现，在这里我们直接使用 mesoscloud 提供的 zookeeper 镜像。

```
＃ docker run --privileged --net=host -d mesoscloud/zookeeper
```

其实这一步不是必须的，但是如果你的 Mesos 集群不使用 Zookeeper，在 Swarm 使用的依赖库 mesos-go 有一个问题会导致 Mesos Master 重启或者 failover 之后，Swarm manage 不会向 Mesos 重新注册（re-register），需要用户重新启动 swarm manage 节点,这会导致它之前运行的所有任务都变成了孤儿任务。我在 Swarm 社区已经创建了一个 issue 来跟踪这个问题，具体的情况和重现步骤可以参考：https://github.com/docker/swarm/issues/1730。

3.在 wangyongqiao-master.grady.com 上执行以下 Docker Run 命令启动 Mesos Master 服务。这里使用我提供一个 Mesos Master 的 Image：

```
# docker run -d \
-v /opt/mesosinstall:/opt/mesosinstall \
--privileged \
--name mesos-master --net host gradywang/mesos-master \
--zk=zk://wangyongqiao-master.grady.com:2181/mesos
```

对于从事 Mesos 开发的人来说，经常性地修改 Mesos 的代码是不可避免的事情，所以为了避免每次改完代码都要重新 build Docker 镜像，我提供的镜像中并不包含 Mesos 的 bin 包，你在使用这个镜像之前必须先在本地 build 你的 Mesos，然后执行 make install 把你的 Mesos 按照到某个空的目录，然后把这个目录 mount 到你的 Mesos master 容器中的/opt/mesosinstall 目录。 具体的步骤可以参考《Swarm 和 Mesos 集成指南－实战部署》。

本例中，我把 Mesos build 完成之后安装在了本地的/opt/mesosinstall 目录：

```
# make install DESTDIR=/opt/mesosinstall
# ll /opt/mesosinstall/usr/local/
total 36
drwxr-xr-x 9 root root 4096 Jan 15 20:02 ./
drwxr-xr-x 3 root root 4096 Jan 15 19:44 ../
drwxr-xr-x 2 root root 4096 Mar 18 13:30 bin/
drwxr-xr-x 3 root root 4096 Jan 15 20:02 etc/
drwxr-xr-x 5 root root 4096 Mar 18 13:30 include/
drwxr-xr-x 5 root root 4096 Mar 18 13:31 lib/
drwxr-xr-x 3 root root 4096 Jan 15 20:02 libexec/
drwxr-xr-x 2 root root 4096 Mar 18 13:31 sbin/
drwxr-xr-x 3 root root 4096 Jan 15 20:02 share/
```

4.在 wangyongqiao-slave1.grady.com 和 wangyongqiao-slave2.grady.com 上执行以下命令，启动 Mesos Salve 服务。这里使用我提供了一个 Mesos Slave 的 Image：

```
# docker run -d \
-v /opt/mesosinstall:/opt/mesosinstall \
--privileged \
-v /var/run/docker.sock:/var/run/docker.sock \
--name mesos-slave --net host gradywang/mesos-slave \
--master=zk://wangyongqiao-master.grady.com:2181/mesos \
--resource_estimator="org_apache_mesos_FixedResourceEstimator" \
--modules='{
      "libraries": {
        "file": "/opt/mesosinstall/usr/local/lib/libfixed_resource_estimator-0.29.0.so",
        "modules": {
          "name": "org_apache_mesos_FixedResourceEstimator",
          "parameters": {
            "key": "resources",
            "value": "cpus:4"
           }
        }
     }
  }'
```

在使用 gradywang/mesos-slave 这个镜像之前，必须注意以下这几点：

像上一步一样，我们假设你已经手动 build，并安装了你的 Mesos 在/opt/mesosinstall 目录下，当然你可以只需要在 wangyongqiao-master.grady.com 这个机器上构建你的 Mesos，然后在 wangyongqiao-master.grady.com 机器上按照 NFS 服务，把安装目录/opt/mesosinstall 挂载到其他的两个计算节点上，具体的步骤可以参考《Swarm 和 Mesos 集成指南－实战部署》。

因为我们要演示怎么用在 Swarm 上使用 Mesos 提供的 revocable 资源，所以我们使用了 Mesos 默认提供的 Fixed estimator，它是 Mesos 提供用来模拟计算 revocable 资源的一个工具，可以配置系统中固定提供的 revocable 资源。上例中我们默认在每个机器上配置4个 CPU 作为这个机器的 revocable 资源。

5.在 wangyongqiao-master.grady.com 上执行以下 Docker Run 命令启动 Docker Swarm mange 服务。这里使用我提供了一个 Swarm 的 Image：

```
# docker run -d \
  --net=host  gradywang/swarm-mesos  \
  --debug manage \
  -c mesos-experimental \
  --cluster-opt mesos.address=192.168.0.2 \
  --cluster-opt mesos.tasktimeout=10s \
  --cluster-opt mesos.user=root \
  --cluster-opt mesos.offertimeout=10m \
  --cluster-opt mesos.port=3375 \
  --cluster-opt mesos.enablerevocable=true \
  --host=0.0.0.0:4375 zk://wangyongqiao-master.grady.com:2181/mesos
```

注意：

1. 在启动 Swarm Manage 的时候，我们使用了 mesos.enablerevocable=true cluster 选项，这个选项是在这个 patch 中新增的，设置为 true，表示 Swarm 愿意使用 mesos 的 revocable 资源。为了考虑向后的兼容性，如果你不指定这个选项，启动 Docker Swarm 的时候，Swarm 将默认不会接收 Mesos 的 revocable 资源。当然了，你可以像其他的参数一样，通过 export 这个变量 SWARM_MESOS_ENABLE_REVOCABLE 来设置。
 
2. 192.168.0.2是 wangyongqiao-master.grady.com 机器的 IP 地址，Swarm 会检查 mesos.address 是不是一个合法的 IP 地址，所以不能使用机器名。

gradywang/swarm-mesos 这个镜像在 Docker Swarm v1.1.3版本之上主要包含了四个新的特性：

支持了 Mesos rescind Offer 机制，相关的 pull request 是：https://github.com/docker/swarm/pull/1866。这 Mesos 集群中的某个计算节点宕机之后，它上边的 offer 在 Swarm 中立即会被删除，不会在 Swarm info 中显示不可用的 offer 信息。

支持 Swarm 用一个特定的 role 来注册 Mesos。在之前的版本中，Swarm 只能使用 Mesos 默认的 role（＊）来注册。相关的 pull request 是：https://github.com/docker/swarm/pull/1890。

支持 Swarm 使用 reserved 资源来创建容器。在 Swarm 支持了用特定的 role 注册之后，我们就可以使用 Mesos 的 static／dynamic reservation 来给 Swarm 提前预定一些资源。这些资源只能给 Swarm 使用。现在的行为是，当 Swarm 的用户创建容器的时候，我们会默认的优先使用 reserved 资源，当 reserved 资源不够的时候，我们会用部分的 unreserved 资源弥补。相关的 pull request 是：https://github.com/docker/swarm/pull/1890。

支持 Swarm 使用 revocable 资源。这将是本文重点演示的特性。相关的 pull request 是：https://github.com/docker/swarm/pull/1946。

6.查看 Docker Info：

```
# docker -H wangyongqiao-master.grady.com:4375 info
Containers: 0
 Running: 0
 Paused: 0
 Stopped: 0
Images: 6
Server Version: swarm/1.1.3
Role: primary
Strategy: spread
Filters: health, port, dependency, affinity, constraint
Offers: 2
  Offer: eba7f1e4-36ca-4a2f-9e24-e8cffce7393a-S0>>eba7f1e4-36ca-4a2f-9e24-e8cffce7393a-O0
   └ cpus: 2
   └ mem: 2.851 GiB
   └ disk: 29.79 GiB
   └ ports: 31000-32000
   └ cpus(Revocable): 4
   └ Labels: executiondriver=native-0.2, kernelversion=3.13.0-32-generic, operatingsystem=Ubuntu 14.04.1 LTS, res-type=any, storagedriver=aufs
  Offer: eba7f1e4-36ca-4a2f-9e24-e8cffce7393a-S1>>eba7f1e4-36ca-4a2f-9e24-e8cffce7393a-O1
   └ cpus: 2
   └ mem: 2.851 GiB
   └ disk: 29.79 GiB
   └ ports: 31000-32000
   └ cpus(Revocable): 4
   └ Labels: executiondriver=native-0.2, kernelversion=3.13.0-32-generic, operatingsystem=Ubuntu 14.04.1 LTS, res-type=any, storagedriver=aufs
Plugins:
 Volume:
Network:
Kernel Version: 3.13.0-32-generic
Operating System: linux
Architecture: amd64
CPUs: 12
Total Memory: 5.701 GiB
Name: wangyongqiao-master.grady.com
```

可以观察到，每个机器上都收到了四个 revocable 的 CPU，每个机器的 res-type 都是 any，表示既有 regular 资源，又有 revocable 资源。

7.下面演示上文提到的那几个 user casees。

Swarm 的用户可以只选择 regular 的资源来运行他们的高优先级的任务：

```
＃ docker -H wangyongqiao-master.grady.com:4375 run -d -e constraint:res-type==regular --cpu-shares 2 ubuntu:14.04 sleep 100
```

可以用以上的命令连续创建三个 contianer，每个 container 使用两个 regular 的 CPU，当创建第三个时，由于所有的 regular CPU 资源都被前两个使用，会因为缺乏可用的 regular资源而失败（执行超时）。

Swarm 的用户可以优先选择 regular 的资源来运行中优先级的任务，如果没有足够的 regular 资源，也可以用 revocable 资源来运行这个任务：

```
＃ docker -H wangyongqiao-master.grady.com:4375 run -d -e constraint:res-type==～regular --cpu-shares 2 ubuntu:14.04 sleep 100
```

可以用以上的命令连续创建三个 contianer，每个 container 使用两个 regular 的 CPU，当创建第三个时，由于所有的 regular CPU 资源都被前两个使用，第三个任务会使用 revocable 资源来创建。

Swarm 的用户可以只选择 revocable 的资源来运行他们的低优先级的任务，把 regular 的资源预留来执行后续的高优先级任务：

```
＃ docker -H wangyongqiao-master.grady.com:4375 run -d -e constraint:res-type==revocable --cpu-shares 2 ubuntu:14.04 sleep 100
```

本章节将主要介绍官方默认给出了一个资源调度的流程：

DRF(Dominant Resource Fairness) Allocator，它的目标是确保每一个上层的 Framework 能够接收到它最需资源的公平份额。下边我们先简单介绍一下 DRF 算法。在 DRF 中一个很重要的概念就是主导资源（dominant resource），例如对于运行计算密集型任务的 framework，它的主导资源是 CPU，对于运行大量依赖内存的任务的 framework，它的主导资源是内存。DRF Allocator 在分配资源之前，首先会按照资源类型计算每个 framework 占有（已经使用的资源＋已经以 offer 的形式发给 framework 但是还没有使用的资源）份额百分比，占有比最高的为这个 framework 的主导资源，然后比较每个 framework 主导资源的百分比，哪个值小，则将这个机器上资源发给对应的 framework。你可以参考 Benjamin Hindman 的这篇论文  https://www.cs.berkeley.edu/~alig/papers/drf.pdf 来了解 DRF 算法的详细说明。

另外，Mesos 集群管理员可以通过以下几种方式提供更多的资源分配策略：

给某个 framework 设置权重（weight）来使某个 framework 获取更多的资源。

可以使用 static/dynamic reservation 指定将某一个 Slave 节点上的资源发送给特定 role 下的 framework。

可以为某个 role 配置 quota 来保证这个 role 下的 framework 在整个 Mesos 集群中获取资源的最小份额。

Framework 收到资源之后，可以创建持久化存贮卷，保证这些存储资源不能被其他的 framework 使用。

可以用以上的命令连续创建五个 contianer，每个 container 使用两个 regular 的 CPU，当创建第五个的时候，由于所有的 revocable CPU 资源都被前四个使用，所以第五个会因为缺乏可用的 revocable 资源而失败（执行超时）。

Swarm 的用户可以优先选择 revocable 的资源来运行中优先级的任务，如果没有足够的 revocable 资源，也可以用 regular 资源来运行这个任务：

```
＃ docker -H wangyongqiao-master.grady.com:4375 run -d -e constraint:res-type==～revocable --cpu-shares 2 ubuntu:14.04 sleep 100
```

可以用以上的命令连续创建五个 contianer，每个 container 使用两个 regular 的 CPU，当创建第五个时，由于所有的 revocable CPU 资源都被前四个使用，第五个会使用 regular 的资源来创建。

Swarm 的终端用户可以通过 docker inspect 来查看他们之前运行的 contianer 使用的是 revocable 资源还是 regular 资源：

```
# docker -H gradyhost1.eng.platformlab.ibm.com:4375 inspect --format "{{json .Config.Labels}}"  22a8a06ccf5e
{"com.docker.swarm.constraints":"[\"res-type==revocable\"]","com.docker.swarm.mesos.detach":"true","com.docker.swarm.mesos.name":"","com.docker.swarm.mesos.resourceType":"Revocable","com.docker.swarm.mesos.task":"e93593367025"}
```