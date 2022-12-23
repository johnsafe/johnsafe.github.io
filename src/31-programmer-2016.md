## Motan：支撑微博千亿调用的轻量级 RPC 框架

文 / 李庆丰

Motan（[https://github.com/weibocom/motan](https://github.com/weibocom/motan)）是微博技术团队研发的基于 Java 的轻量级 RPC 框架，已在内部大规模应用多年，每天稳定支撑微博上亿次的内部调用。

### RPC调用优势 

随着公司业务发展，微博内部调用和依赖越来越多，传统方式逐渐显现出弊端。
1. jar 包依赖调用使得服务间耦合太紧，相互影响，同时也存在跨语言调用问题；
2. HTTP 依赖调用在协议上比较重，常在性能和效率上出现瓶颈。
越是大型复杂的系统，越需要轻量的依赖调用方式，RPC 依赖调用很好地解决了上述问题。

### 典型RPC框架对比

目前，业界 RPC 框架大致分为两类，一种偏重服务治理，另一种侧重跨语言调用。服务治理型的 RPC 框架代表是 Dubbo 和 DubboX。前者是阿里开源的分布式服务框架，实现高性能的 RPC 调用同时提供了丰富的管理功能，是一款应用广泛的优秀 RPC 框架，但现在维护更新较少。后者则是当当基于 Dubbo 扩展，支持 REST 风格的远程调用、Kryo/FST 序列化，增加了一些新功能。

这类 RPC 框架的特点是功能丰富，提供高性能远程调用、服务发现及服务治理能力，适用于大型服务的解耦及治理，对于特定语言（如 Java）项目可以实现透明化接入。缺点是语言耦合度较高，跨语言支持难度较大。

跨语言调用型 RPC 框架有 Thrift、gRPC、Hessian、Hprose 等。这类框架侧重于服务的跨语言调用，能支持大部分语言，从而进行语言无关调用，非常适合多语言调用场景。但这类框架没有服务发现相关机制，实际使用时需要代理层进行请求转发和负载均衡策略控制。 

Motan 倾向于服务治理型，跨语言方面正在尝试与 PHP 调用集成。与 Dubbo 系列相比，功能或许不那么全，扩展实现也没那么多，但更注重简单、易用以及高并发高可用场景。

### 功能特点

Motan 是一套轻量级的 RPC 框架，具有服务治理能力，简单、易用、高可用。其主要特色如下： 
1. 无侵入集成、简单易用，通过 Spring 配置方式，无需额外代码即可集成分布式调用能力；
2. 集成服务发现和服务治理能力，灵活支持多种配置管理组件，如 Consul、ZooKeeper 等；
3. 支持自定义动态负载均衡、跨机房流量调整等高级服务调度能力；
4. 基于高并发、高负载场景优化，具备 Failover、Failfast 能力，保障 RPC 服务高可用。

Motan 的架构设计，分为服务提供方（RPC Server）、服务调用方（RPC Client）、注册中心（Registry）三个角色，Server 向 Registry 注册声明所提供的服务；Client 向 Registry 订阅指定服务，与 Registry 返回的服务列表的 Server 建立连接，进行 RPC 服务调用；Client 通过 Registry 感知 Server 的状态变更。三者的交互关系如图1所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/577b6d9d2ce3e.png" alt="图1  Motan架构" title="图1  Motan架构" />

图1  Motan架构

服务模块化设计方便灵活扩展，Motan 主要包括 register、transport、serialize、protocol、cluster 等，各个模块都支持通过 SPI 进行扩展，其交互如图2。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/577b6dc774eb7.jpg" alt="图2  模块交互" title="图2  模块交互" />

图1  Motan架构

register 模块：用来和注册中心进行交互，包括注册服务、订阅服务、服务变更通知、服务心跳发送等功能；Server 端会在系统初始化时通过 register 模块注册服务，Client 端在系统初始化时会通过 register 模块订阅到具体提供服务的 Server 列表，当 Server 列表发生变更时也由 register 模块通知 Client。

protocol 模块：用来进行 RPC 服务的描述和 RPC 服务的配置管理，这一层还可以添加不同功能 ﬁlter 用来完成统计、并发限制等功能。

serialize 模块：将 RPC 请求中的参数、结果等对象进行序列化与反序列化，即进行对象与字节流的互相转换；默认使用对 Java 更友好的 hessian2进行序列化。transport 模块用来进行远程通信，默认使用 Netty NIO 的 TCP 长链接方式。 

cluster 模块：Client 端使用的模块，cluster 是一组可用的 Server 在逻辑上的封装，包含若干可以提供 RPC 服务的 Server，实际请求时会根据不同的高可用与负载均衡策略选择一个可用的 Server 发起远程调用。

在进行 RPC 请求时，Client 通过代理机制调用 cluster 模块，cluster 根据配置的HA和 LoadBalance 选出一个可用的 Server，通过 serialize 模块把 RPC 请求转换为字节流，然后通过 transport 模块发送到 Server 端。

服务配置化增强了 Motan 的易用性，Motan 框架中将功能模块抽象为四个可配置的元素，分别为：
1. protocol：服务通信协议。服务提供方与消费方进行远程调用的协议，默认为 Motan 协议，使用 hessian2进行序列化，Netty 作为 Endpoint 以及使用 Motan 自定义的协议编码方式。
2. registry：注册中心。服务提供方将服务信息（包含 IP、端口、服务策略等信息）注册到注册中心，服务消费方通过注册中心发现服务。当服务发生变更，注册中心负责通知各个消费方。
3. service：服务提供方服务。使用方将核心业务抽取出来，作为独立的服务。通过暴露服务并将服务注册至注册中心，从而使调用方调用。
4. referer：服务消费方对服务的引用，即服务调用方。 

Motan 推荐使用 Spring 配置 RPC 服务，目前扩展了6个自定义 Spring XML 标签：motan:protocol、motan:registry、motan:basicService、motan:service、motan:basicReferer，以及motan:referer。

高可用是 Motan 的一大特点，支持多种服务治理和高可用机制，包括：灵活多样的集群负载均衡策略，支持 ActiveWeight/Random/RoundRobin/LocalFirst/Consistent 等6种策略，并支持自定义扩展；自动集成 Failover、Failfast 容错策略，实现故障节点自动摘除，自动探测恢复，有效进行服务故障隔离，远离服务卡死及雪崩；连接池自定义控制，根据业务场景灵活配置；支持多机房间调用流量压缩、动态流量调整，实现真正的跨 IDC 的高可用。

基于高并发、高负载场景的优化，具备在高压力场景下的高可用能力，基准测试情况如下。

Server 端，并发多个 Client，连接数50、并发数100的场景： 
1. 空包请求：单 Server TPS 18W 
2. 1K String 请求：单 Server TPS 8.4W 
3. 5K String 请求：单 Server TPS 2W 

Client 端（场景对比如图3所示）。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/577b6e4342eb0.png" alt="图3  场景对比" title="图3  场景对比" />

图3  场景对比

Motan 提供了基础性能测试框架，欢迎使用者进行性能评估，源码请参考 Benchmark 文档 [https://github.com/weibocom/motan/tree/master/motan-benchmark](https://github.com/weibocom/motan/tree/master/motan-benchmark)。使用及易用性方面，使用 Spring 进行配置，业务代码无需修改，工程依赖只涉及核心5个模块，且可以按需依赖。关于项目中的具体步骤，请参考快速入门文档[https://github.com/weibocom/motan/blob/master/docs/wiki/zh_quickstart.md](https://github.com/weibocom/motan/blob/master/docs/wiki/zh_quickstart.md)。

### 是否重复造轮子

前文提到，当前业界已有一些优秀的 RPC 框架，微博技术团队为什么要再造一个 RPC 框架呢？也正如上文所述，当前业界可供选择并持续维护的优秀 RPC 框架并不多。同时鉴于微博的内部调用量非常大，有很多定制化场景，要做到平滑迁移到这些 RPC 框架也需要做不少定制化改造，最终我们决定自主研发。主要从以下4个方面考虑：
1. 框架的性能和可用性需要定制化。微博内部调用量级非常大，业界很少有类似场景应用经验可以借鉴，需要针对高并发和复杂逻辑场景定制优化，如 Motan 的 Failover 和 Failfast 机制等。
2. 尽量做到平滑迁移。线上业务迁移需要保障业务改造尽量少，支持可快速回退，这个必须具备有效的机制保障，如 Motan 的 inJvm 机制等。
3. 未来多语言兼容接入诉求。微博整体技术体系包括 Java 和 PHP，还有部分 Erlang、C++等，未来希望能通过这套服务框架解决整体内部依赖调用问题。
4. 技术积累储备及掌控力。微博具有一批实战经验丰富的技术专家，有实力又熟悉微博场景。

### 发展及开源

Motan 当前在微博内部已广泛应用，每天支撑着上亿的内部调用，这也是个持续改进优化的过程。从服务发现、服务容错、快速失败、故障降级等多方面，针对复杂业务架构及高并发场景进行不断定制优化改进。随着虚拟化技术的兴起，弹性调度成为成熟技术框架不可或缺的能力，新的 Motan 框架技术负责人也适时对其增加了数据流量压缩、动态流量调整、多注册中心支持等功能，让它能适应时代的变化。 

为了方便其他团队的复用，针对 Motan 核心功能进行抽离和封装，去除掉微博自身依赖，形成今天的开源版本，希望能发挥开源社区的力量，进 一步发展和发挥 Motan 的价值。

### 期待

在这唯快不破的互联网时代，软件的开发速度前所未有。这得益于软件已有模块的大规模复用。在过去的几年里，开源软件无疑在这方面做出了巨大的贡献。

微博技术团队的快速成长，受益于开源社区，同时也希望能为开源社区贡献自己的力量。Motan 是经过大规模实践的轻量级RPC框架，希望未来 能有更多优秀的开源人进一步完善优化。也期待更多的公司可以享受它带来的便利。

### Q & A 

**1.与业界已有的 RPC 框架，如 Dubbo 相比，Motan 有什么优势？**

Dubbo 功能较丰富，与 Dubbo 的分层来对比，Motan 的模块层次更简单，没有exchange和 directory 等。从压测的结果来看，在微博的业务场景下 Motan 的性能比 Dubbo 要好些。 

**2. 现在的版本能用于生产环境吗？**
 
Motan 支持了微博的绝大部分底层核心业务，目前看来比较稳定。但是不排除会有一些使用环境不一致造成的问题，建议测试一下再应用到生产环境。 

**3. Motan 支持跨语言调用吗？**

目前暂时只支持 Java 应用，针对 PHPYAR 框架的支持已在开发中，未来会支持更多语言。 

**4. 开源版 Motan 与微博内部的版本功能一样吗？**
 
开源版包含了内部版本中的大部分功能，主要是去除了内部的依赖组件相关的功能。
 
**5. Motan 支持异步调用吗，如何实现？**
 
请求在传输层面为异步调用，不需要额外配置，Motan 本身还不支持异步调用。
 
**6. 在使用 Motan 时遇到了问题，应该去哪里提问？**

可以在 GitHub 提交 Issue（[https://github.com/weibocom/motan/issues](https://github.com/weibocom/motan/issues)）。