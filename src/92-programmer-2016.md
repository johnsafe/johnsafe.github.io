## 基于 OpenStack 和 Kubernetes 构建组合云平台——网络集成方案综述

文/王昕

一谈到云计算，大家都会自然想到三种云服务的模型：基础设施即服务（IaaS），平台即服务（PaaS）和软件即服务（SaaS）。OpenStack 已经成为私有云 IaaS 的标准，而 PaaS 层虽然有很多可选技术，但已经确定的是一定会基于容器技术，并且一定会架构在某种容器编排管理系统之上。在主流的容器编排管理系统 Kubernetes、Mesos 和 Swarm 中，Kubernetes 以它活跃的社区，完整强大的功能和社区领导者富有远见的设计而得到越来越多的企业青睐。我们基于 OpenStack 和 Kubernetes 研发了首个实现容器和虚拟机组合服务、统一管理的容器云平台。在本文将分享我们在集成 OpenStack 和 Kubernetes 过程中的网络集成方案经验总结。

### Kubernetes 的优秀设计

在我们介绍网络方案之前，先向大家简单介绍 Kubernetes 的基础构架。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/5779ccb935b70.png" alt="图 1   Kubernetes集群的架构" title="图 1   Kubernetes集群的架构" />

图 1   Kubernetes 集群的架构

一个 Kubernetes 集群是由分布式存储（etcd），服务节点（Minion）和控制节点（Master）构成的。所有的集群状态都保存在 etcd 中，Master 节点则运行集群的管理控制模块。Minion 节点是真正运行应用容器的主机节点，在每个 Minion 节点上都会运行一个 Kubelet 代理，控制该节点上的容器、镜像和存储卷等。

#### 支持多容器的微服务实例
Kubernetes 有很多基本概念，最重要也是最基础的是 Pod。Pod 是运行部署的最小单元，可以支持多容器。为什么要有多容器？比如你运行一个操作系统发行版的软件仓库，一个 Nginx 容器用来发布软件，另一个容器专门用来从源仓库做同步，这两个容器的镜像不太可能是一个团队开发的，但是它们一块儿工作才能提供一个微服务；这种情况下，不同的团队各自开发构建自己的容器镜像，在部署的时候组合成一个微服务对外提供服务。

#### 自动提供微服务的高可用
Kubernetes 集群自动提供微服务的高可用能力，由复制控制器（Replication Controller）即 RC 进行支持。RC 通过监控运行中的 Pod 来保证集群中运行指定数目的 Pod 副本。指定的数目可以是多个也可以是1个；少于指定数目，RC 就会启动运行新的 Pod 副本；多于指定数目，RC 就会杀死多余的 Pod 副本。即使在指定数目为1的情况下，通过 RC 运行 Pod 也比直接运行 Pod 更明智，因为 RC 也可以发挥它高可用的能力，保证永远有1个 Pod 在运行。

#### 微服务在集群内部的负载均衡
在 Kubernetes 内部，一个 Pod 只是一个运行服务的实例，随时可能在一个节点上停止，在另一个节点以一个新的 IP 启动新的 Pod，因此不能以确定的 IP 和端口号提供服务。在 Kubernetes 中真正对应一个微服务的概念是服务（Service），每个服务会对应一个集群内部有效的虚拟 IP，集群内部通过虚拟 IP 访问一个微服务。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/5779cd18e6189.png" alt="图2  Kubernetes集群内的负载均衡" title="图2  Kubernetes集群内的负载均衡" />

图2  Kubernetes 集群内的负载均衡

在 Kubernetes 集群中，集群管理员管理的是一系列抽象资源，例如对应一个微服务的抽象资源就是 Service；对应一个微服务实例的抽象资源就是 Pod。从 Service 到 Pod 的数据转发是通过 Kubernetes 集群中的负载均衡器，即 kube-proxy 实现的。Kube-proxy 是一个分布式代理服务器，在 Kubernetes 的每个 Minion 节点上都有一个；这一设计体现了它的伸缩性优势，需要访问服务的节点越多，提供负载均衡能力的 kube-proxy 就越多，高可用节点也随之增多。与之相比，我们平时在服务器端做个反向代理做负载均衡，还要进一步解决反向代理的负载均衡和高可用问题。Kube-proxy 对应的是 Service 资源。每个节点的 Kube-proxy 都会监控 Master 节点针对 Service 的配置。当部署一个新的 Service 时，计入这个 Service 的 IP 是10.0.0.1，port 是1234，那每个 Kube-proxy 都会在本地的 IP Table 上加上 redirect 规则，将所有发送到10.0.0.1：1234的数据包都 REDIRECT 到本地的一个随机端口，当然 Kube-proxy 也在这个端口监听着，然后根据负载均衡规则，例如 round robin 把数据转发到不同的后端 Pod。

#### 可替换选择的组网方案
Kube-proxy 只负责把服务请求的目标替换成目标 Pod 的端口，并不指定如何实现从当前节点将数据包发送到指定 Pod；后面如何再发送到 Pod，那是 Kubernetes 自身的组网方案解决的。这里也体现了 Kubernetes 优秀的设计理念，即组网方式和负载均衡分式可以独立选择，互不依赖。Kubernetes 对组网方案的要求是能够给每个 Pod 以内网可识别的独立 IP 并且可达。如果用 flannel 组网，那就是每个节点上的 flannel 把发向容器的数据包进行封装后，再用隧道将封装后的数据包发送到运行着目标 Pod 的 Minion 节点上。目标 Minion 节点再负责去掉封装，将去除封装的数据包发送到目标 Pod 上。

### 云平台的多租户隔离

对于一个完整的云平台，最基础的需求是多租户的隔离，我们需要考虑租户 Kubernetes 集群之间的隔离问题。就是我们希望把租户自己的虚拟机和容器集群互联互通，而不同租户之间是隔离不能通信的。使用 OpenStack 的多租户私有网络可以实现网络的隔离。OpenStack 以前叫租户即 Tenant，现在叫项目即 Project；Tenant 和 Project 是一个意思，即把独立分配管理的一组资源放在一起管理，与其他组的资源互相隔离互不影响。为了提供支持多租户的容器和虚拟机组合服务，可以把容器集群和虚拟机放到一个独立租户的私有网络中去，这样就达到了隔离作用。这种隔离有两个效果，一是不同租户之间的容器和虚拟机不能通信，二是不同租户可以重用内网 IP 地址，例如10.0.0.1这个 IP，两个租户都可用。放在同一个私有网络中的容器和虚拟机，互相访问通过内网通信，相对于绕道公网，不仅节省公网带宽，而且可以大大提高访问性能。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/5779cd67f4214.png" alt="图3  集成OpenStack和Kubernetes的多租户支持 " title="图3  集成OpenStack和Kubernetes的多租户支持 " />

图3  集成 OpenStack 和 Kubernetes 的多租户支持

### 对外发布应用服务

虚拟机的内网 IP，Kubernetes 集群节点的 IP，服务的虚拟 IP 以及服务后端 Pod 的 IP，这些 IP 在外网都是不可见的，集群外的客户端无法访问。想让外网的客户端访问虚拟机，最简单是用浮动 IP；例如，一个 VM 有一个内网 IP，我们可以给它分配一个外网浮动 IP；外网连接这个浮动 IP，就会转发到相应的内网固定 IP 上。

浮动 IP 的模式只适用于虚拟机，不适用于 Kubernetes 中的服务。在 OpenStack 网络中，支持外网要利用 nodePort 和 Load Balancer 结合的方式。比如这里有个服务，它的服务的虚拟 IP（也叫 Cluster IP）是10.0.0.1，端口是1234，那么我们知道内网的客户端就是通过这个 IP 和端口访问的。为了让外网访问相应的服务，Kubernetes 集群中针对每个服务端口，会设置一个 node Port 31234，群中的每个节点都会监听这个端口，并会在通过 IP Table 的 REDIRECT 规则，将发向这个端口的数据包，REDIRECT 对应服务的虚拟 IP 和端口。所有发向 nodePort 31234这个端口的数据包，会被转发到微服务的虚拟 IP 和端口上去，并进一步 REDIRECT 到 Kube-proxy 对应的端口上去。再进一步，只要从外网进来的数据包，就发送到服务节点的 nodePort 端口上。那我们用 Load Balancer 来做，Load Balancer 的前端是它的内网 IP 和发布端口，后端是所有内网的节点 IP 和 nodePort。因此所有发到负载均衡器的包，会做一次负载均衡，均衡地转发到某个节点的 nodePort 上去，到达节点的 nodePort 的数据包又会转发给相应的服务。为了让外网访问，我们要给 Load Balancer 绑一个外网 IP，例如111.111.111.1，这样所有来自外网，发送到111.111.111.1:11234的数据包，都会转发给对应的微服务10.0.0.1:1234，并最终转发给某个随机的 Pod 副本。

实现了以上工作，我们部署在 Kubernetes 集群中的服务就可以实现在 OpenStack 的基础平台上对外发布和访问了。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/5779cdd38cb00.png" alt="图4  Kubernetes微服务的对外发布" title="图4  Kubernetes微服务的对外发布" />

图4  Kubernetes 微服务的对外发布

### 基于 Kuryr 的组网方案

Docker 从1.6版开始，将网络部分抽离出来成为 Libnetwork 项目，使得第三方可以以插件机制为 Dockers 容器开发不同网络管理方案。Libnetwork 包括4种驱动类型：

Null：顾名思义，就是没有网络支持。

Bridge：传统的 Docker0 网桥机制，只适用于单主机容器之间的通信，不能支持跨主机通信。

Overlay：在下层主机网络的上层，基于隧道封装机制，搭建层叠网络，实现跨主机的通信；Overlay 无疑是架构最简单清晰的网络实现机制，但数据通信性能则大受影响。

Remote：Remote 驱动并不真正实现驱动，而是以 REST 服务的方式定义了与第三方网络驱动交互的机制和管理接口。第三方容器网络方案主要通过这一机制与 Docker 集成。

说到这里，必须要提到 OpenStack 的容器网络管理项目 Kuryr，正是一个 Remote 驱动的实现，集成 Dockers 和 Openstack Neutron 网络。Kuryr 英文的原意是“信使”，本身也不是网络配置的一个具体实现，而是来自 Docker 用户操作意图转换为对 Openstack Neutron API 的操作意图。具体来说，容器管理平台的操作会通过以下方式与 Openstack 集成：

- Kubernetes 的网络配置管理操作（例如 Kubectl 命令）会由一个 Kubectl 操作转换成对 Dockers 引擎的操作；

- Dockers 引擎的操作转换成对 Libnetwork 的 Remote 驱动的操作；

- 对 Remote 驱动的操作，通过 Kuryr 转换成对 Neutron API 的操作；

- 对 Neutron API 的操作，通过 Neutron 插件的机制转换成对具体网络方案驱动的操作。

相对于 Docker 网络驱动，Flannel 是 CoreOS 团队针对 Kubernetes 设计的一个网络规划服务；简单来说，它的功能是让集群中的不同节点主机创建的 Docker 容器都具有全集群唯一的虚拟 IP 地址，并使 Docker 容器可以互连。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/5779ce3e8412f.png" alt="图5  OpenStack Kuryr调用关系示意图" title="图5  OpenStack Kuryr调用关系示意图" />

图5  OpenStack Kuryr 调用关系示意图

### 基于 Flannel 网络的组网方案

在集成 Kubernetes 的网络方案中，基于 Flannel 的网络方案 Kubernetes 的默认实现也是配置最简单直接的。在 Flannel 网络中，容器的 IP 地址和服务节点的 IP 地址处于不同的网段，例如将10.1.0.0/16网段都用于容器网络，将192.168.0.0/24网段用于服务节点。每个 Kubernetes 服务节点上的 Pod 会形成一个独立子网，例如节点192.168.0.100上的容器子网为10.1.15.0/24；容器子网与服务节点的对应关系保存在 Etcd 上。每个服务节点都会安装 flanneld 程序，它会截获所有发向容器集群网段10.1.0.0/24的数据包。对于发向容器集群网段的数据包，如果是指向其他节点对应的容器地址的，flannel 会通过隧道协议封装成指向目的服务节点的数据包；如果是指向本节点对应的容器地址的，flannel 将会转发给 docker0。Flannel 支持不同的隧道封装协议，常用的是 Flannel UDP 和 Flannel VxLan，根据实际测试结果，VxLan 封装在吞吐量和网络延时性能上都要好于 UDP，因此一般推荐使用 VxLan 的封装方式。	

在 Kubernetes 集群中的 Flannel 网络非常类似于 Docker 网络的 Overlay 驱动，都是基于隧道封装的层叠网络，优势和劣势都非常明显。

层叠网络的优势

- 对底层网络依赖较少，不管底层是物理网络还是虚拟网络，对层叠网络的配置管理影响较少；

- 配置简单，逻辑清晰，易于理解和学习，非常适用于开发测试等对网络性能要求不高的场景。

层叠网络的劣势

- 网络封装是一种传输开销，对网络性能会有影响，不适用于对网络性能要求高的生产场景；

- 由于对底层网络结构缺乏了解，无法做到真正有效的流量工程控制，也会对网络性能产生影响；

- 某些情况下也不能完全做到与下层网络无关，例如隧道封装会对网络的 MTU 限制产生影响。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/5779ce9835866.png" alt="图6  Flannel网络机制示意图" title="图6  Flannel网络机制示意图" />

图6  Flannel 网络机制示意图

### 基于 Calico 网络的组网方案

与其他的 SDN 网络方案相比，Calico 有其独特的之处，就是非层叠网络，不需要隧道封装机制，依靠现有的三层路由协议来实现软件定义网络（SDN）。Calico 利用 iBGP 即内部网关协议来实现对数据包转发的控制。在基于隧道封装的网络方案中，网络控制器将宿主机当作隧道封装的网关，封装来自虚拟机或容器的数据包通过第四层的传输层发送到目的宿主机做处理；与此相对，Calico 网络中的控制器将宿主机当作内部网关协议中的网关节点，将来自虚拟机或容器的数据包通过第三层的路由发送到目的宿主机做处理。内部网关协议的一个特点是要求网关节点的全连接，即要求所有网关节点之间两两连接；显然，这样链路数随节点的增长速度是平方级的，会导致链路数的暴涨。为解决这个问题，当节点较多时，要引入路由反射器（RR）。其作用是将一个全连接网络转变成一个星形网络，由路由反射器作为星形网络的中心连接所有节点，使得其他节点不需要直接连接，从而使链路数从 O(n2)变为 O(n)。因此 Calico 的解决方案是所有节点中选择一些节点做为路由反射器，路由反射器之间是全联通的，非路由反射器节点之间则需要路由反射器间接连接。

Calico 网络的优势

- 没有隧道封装的网络开销；

- 相比于通过 Overlay 构成的大二层层叠网络，用 iBGP 构成的扁平三层网络扩展模式更符合传统IP网络的分布式结构；

- 不会对物理层网络的二层参数如 MTU 引入新的要求。

Calico 网络的劣势

- 最大的问题是不容易支持多租户，由于没有封装，所有的虚拟机或者容器只能通过真实的 IP 来区分自己，这就要求所有租户的虚拟机或容器统一分配一个地址空间；而在典型的支持多租户的网络环境中，每个租户可以分配自己的私有网络地址，租户之间即使地址相同也不会有冲突；

- 不容易与其他基于主机路由的网络应用集成。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/5779cef40b22f.png" alt="图7  包含路由反射器的Calico网络结构图" title="图7  包含路由反射器的Calico网络结构图" />

图7  包含路由反射器的 Calico 网络结构图

### 不同网络方案对 OpenStack 和 Kubernetes 的影响

总结一下引入 Kuryr、Overlay 和 Calico 技术对云平台解决方案的作用和带来的影响。如果我们形象地比喻不同的组网技术，基于 Overlay 是最重的，开销最大、但也是最稳定的，就像一只乌龟；基于 Calico 较为轻量，但与复杂的网络环境集成就不太稳定，就像一只蹦蹦跳跳的兔子；基于 Kuryr 的技术，因为 Kuryr 本身并不提供网络控制功能，而只是提供下面一层网络控制功能到容器网络的管理接口封装，就像穿了一件马甲或戴了一顶帽子。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/5779cf313dde2.png" alt="表1  不同组网方案的作用与影响" title="表1  不同组网方案的作用与影响" />

表1  不同组网方案的作用与影响

在一个集成了 OpenStack 和 Kubernetes 的平台，分别在 OpenStack 层和 Kubernetes 层要引入一种网络解决方案，OpenStack 在下层，Kubernetes 在上层。考虑作为 IaaS 层服务，多租户隔离和二层网络的支持对 OpenStack 网络更为重要，某些组合的网络方案基本可以排除：例如，Kuryr 作为一个管理容器网络的框架，一定要应用在 Kubernetes 层，而不是在下面的 OpenStack 层，于是，所有将 Kuryr 作为 OpenStack 层组网的方案都可以排除。

综合 OpenStack 层和 Kubernetes 层网络，较为合理的网络方案可能为以下几种：

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/5779cf5dbb489.png" alt="表2  不同网络集成方案的比较" title="表2  不同网络集成方案的比较" />

表2  不同网络集成方案的比较

如图8所示，第一种组合，底层用 Calico 连通 OpenStack 网络，上层用 Kuryr 将容器网络与 IaaS 网络打通，像一个小兔子戴帽子；这种组网方式无疑性能开销是最少的，但因为没有方便的多租户隔离，不适合需要多租户隔离的场景。第二种组合，底层用 Overlay 进行多租户隔离，上层用 Kuryr 将容器网络与 IaaS 网络打通，像一个乌龟戴帽子；这种组网方式兼顾了性能和多租户隔离的简便性，比较适合统一管理容器和虚拟机的网络方案。第三种组合，底层用 Overlay 进行多租户隔离，上层用 Calico 实现容器集群组网，像小兔子站在乌龟上；这种组网方式下，性能较好，容器层的网络管理基本独立于 IaaS 层，有利于实验验证 Calico 跟 Kubernetes 的集成，但实践中 Calico 的组网方式还是稍显复杂。第四种组合，上下两层都是 Overlay 网络，好像两个乌龟垒在一起，是一种性能开销最大的方式，但同时也是初期最容易搭建和实现的组网方式，比较适合于快速集成和验证 Kubernetes 的整体功能。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/5779cfaad2295.png" alt="图8  使用不同网络技术集成OpenStack与Kubernetes" title="图8  使用不同网络技术集成OpenStack与Kubernetes" />

图8  使用不同网络技术集成 OpenStack 与 Kubernetes

### 总结

本文探讨了基于 OpenStack 和 Kubernetes 构建云平台过程中，网络层面的技术解决方案。特别的，我们介绍了如何在基于 OpenStack 和 Kubernetes 的云平台中，实现租户隔离和服务对外发布。同时，我们介绍了 Kubernetes 的可替换集群组网模型，探讨了几种不同技术特点的组网方式，分析了 Kuryr、Flannel 和 Calico 这些网络技术的特点和对系统的技术影响，并探讨了 OpenStack 和 Kubernetes 不同组网方案的适用场景。