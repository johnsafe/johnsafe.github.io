## OpenStack 数据库服务 Trove 解析与实践

文/黄明生

>在大数据分析越来越盛行的背景下，对数据库的可靠便捷管理也变得愈为重要。本文作者对 OpenStack Trove 的原理、架构与功能进行了深入介绍，并通过实践来展示 Trove 的应用。

### 云数据库服务与 OpenStack

对于公有云计算平台来说，只有计算、网络与存储这三大服务往往是不太够的。在目前互联网应用百花齐放的背景下，几乎所有应用都使用到数据库，而数据库承载的往往是应用最核心的数据。此外，在大数据分析越来越盛行的背景下，对数据库的可靠便捷管理也变得更为重要。因此，DBase as a Service（DBaaS，数据库服务）也就顺理成章地成为了云计算平台为用户创造价值的一个重要服务。

对比 Amazon AWS 中各种关于数据的服务，其中最著名的是 RDS（SQL-Base）和 DynamoDB（NoSQL），除了实现了基本的数据管理能力，还具备良好的伸缩能力、容灾能力和不同规格的性能表现。因此，对于最炙手可热的开源云计算平台 OpenStack 来说，也从 Icehouse 版加入了 DBaaS 服务，代号 Trove。直到去年底发布的 OpenStack Liberty 版本，Trove 经过了4个版本的迭代发布，目前已经成为OpenStack官方可选的核心服务之一。本文将深入介绍 Trove 的原理、架构与功能，并通过实践来展示 Trove 的应用。

### Trove 的设计目标

“Trove is Database as a Service for OpenStack. It's designed to run entirely on OpenStack, with the goal of allowing users to quickly and easily utilize the features of a relational or non-relational database without the burden of handling complex administrative tasks. ”这是 Trove 在官方首页上对这个项目的说明，有两个关键点。一个是从产品设计上说，它的定位不仅仅是关系型数据库，还涵盖了非关系型数据库的服务。另一个是从产品实现上说，它是完全基于 OpenStack 的。

![enter image description here](http://images.gitbook.cn/554d0f40-362a-11e8-87cb-a7548ca0965d)

图1  Trove 在 openStack 中的定位

从第一点可以看出 Trove 解决问题的高度已经超越了同类产品。因为我们从其他云计算平台对比去看，关系型和非关系型数据库都是由不同的服务去提供（比如 AWS 的 RDS 和 DynamoDB），而且实现上也往往互相独立的系统，不仅 UI 不同，API 也不一样。而Trove的目标是抽象尽可能多的东西，对外提供统一的 UI 和 API，尽量减少冗余实现，提升平台内聚。只要具备了实例、数据库、用户、配置、备份、集群、主从复制这些概念，不管是关系型还是非关系型数据库，都能统一管理起来。从最新的 Liberty 版本发布的情况下，目前开源的主流关系型和非关系型数据库也得到了支持，比如 MySQL（包括 Percona 和 MariaDB 分支）、Postgresql、Redis、MongoDB、CouchDB、Cassandra等等。不过根据官方的介绍，目前只有 MySQL 是得到了充分的生产性测试，其他的还处于实验性阶段。

而第二点完全基于 OpenStack 的，可以说是一个较大的创新。试想，假设你是一个云计算服务商，如果现在要提供数据库服务，只需要在原有平台软件上升级与配置一下就行，其他什么都不需要，不需要采购数据库服务器硬件，不需要规划网络，不需要规划 IDC，这是一种什么样的感觉？Trove 完全构建于 OpenStack 原有的几大基础服务之上。打个比喻，类似于 Google 著名的 BigTable 服务是构建于 GFS、Borg、Chubby 等几个基础服务之上。所以，Trove 实际上拥有了云平台的一些基础特性，比如容灾隔离、动态调度、快速响应等能力，而且从研发的角度看，也大大减少了重复造轮子的现象。

### Trove 的架构介绍

实际上，Trove 的架构（最新版本）与 OpenStack Nova 项目的架构如出一辙，可以说是 Nova 的一个简化版，也是典型的 OpenStack 项目架构风格。Trove 所管理的各个数据库引擎的差异性主要体现在 trove-guestagent 的具体 manager 和 strategies 代码实现上。架构如图2所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56a98f342582d.png" alt="图2 Trove的系统架构" title="图2 Trove的系统架构" />

图2 Trove 的系统架构

trove-api 是接入层，轻量级请求通过在接入层直接处理或者直接访问 guestagent 处理，比如获取实例列表、获取实例规格列表等；而比较重的请求则通过 message bus（OpenStack 默认实现是 Rabbitmq）中转给 trove-taskmanager 进行调度处理。trove-taskmanager 是调度处理层，主要处理较重的请求，比如创建实例、实例 resize 等。taskmanager 会通过 Nova、Swift 的 API 访问 OpenStack 基础的服务，而且是有状态的，是整个系统的核心。trove-conductor 是 guestagent 访问数据库的代理层，主要是为了屏蔽掉 guestagent 直接对数据库的访问。

在 Trove 目前的实现中，一个数据库实例一一对应到一个 VM，而 guestagent 也是运行在 VM 里面。vm 镜像包含了经过裁剪的操作系统、数据库引擎和 guestagent （镜像具体实现没有标准，数据库引擎和 guestagent 也都可以在 VM 启动时通过网络动态装载）。而实例所在分区的硬盘是通过 Cinder 提供的云硬盘。每个 VM 都会关联一个安全组防火墙，只允许数据库服务的端口通过（比如 MySQL，默认是 TCP 3306端口）。从这里可以看出，Trove 创建数据库实例是非常灵活的，后期的调度也非常方便，这些都得益于 Nova 和 Cinder。

### Trove 功能介绍

正如前面说的，实际上 Trove 是在主流的关系和非关系型数据库的一些核心概念基础上抽象出的一个系统框架，所以其实现的功能也是围绕着这些核心概念的。Trove 的概念关系图见图3。

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56a98f3e63a3d.png" alt="图3 Trove的核心概念关系" title="图3 Trove的核心概念关系" />

图3 Trove 的核心概念关系

因此 Trove 的主要功能也是围绕这几个概念实现的：datastore 管理、Instance 管理、configuration 管理、database 管理、user 管理、replication 管理、backup 管理、cluster 管理等等。

从最新的 Liberty 版本看，目前 Trove 的功能还是比较多的，而且扩展性很强。可惜的是实例统计监控功能却没有看到，而统计监控功能的缺失，应该也是导致了实例容灾的自动切换还没有实现，相信这些都会在不久的新版本中逐渐完善。不过从另外一个角度看，由于数据库对用户来说是非常关键的服务，涉及到核心数据的数据一致性问题，目前交由用户上层去确认和切换实例也不失一个明智的选择。下面以 MySQL 数据库为例，对 Trove 的一些重要功能进行分析。

#### 动态 resize 能力

分为 instance-resize 和 volume-resize，前者主要是实例运行的内存大小和 CPU 核数，后者主要是指数据库分区对应的硬盘卷的大小。由于实例是跑在 VM 上的，而 VM 的 CPU 和 Memory 的规格可以通过 Nova 来进行动态调整，所以调整是非常方便快捷的。另外硬盘卷也是由 Cinder 提供的动态扩展功能来实现 resize。resize 过程中服务会有短暂的中断，是由于 MySQLd 重启导致的。

#### 全量与增量备份

目前 MySQL 的实现中，备份是由实例 VM 上的 guestagent 运行 xtrabackup 工具进行备份，且备份后的文件会存储在 Swift 对象存储中，从备份创建实例的过程则相反。由于 xtrabackup 强大的备份功能，所以 Trove 要做的只是做一些粘胶水的工作。
 
#### 动态配置更新

目前支持实例的自定义配置，可以创建配置组应该到一组实例上，且动态 attach 到运行中的实例中生效。

#### 一主多从的一键创建

在创建数据库实例的 API 中，支持批量创建多个从实例，并以指定的实例做主进行同步复制。这样就方便了从一个已有实例创建多个从实例的操作。而且 MySQL5.6 版本之后的同步复制支持 GTID 二进制日志，使得主从实例之间关系的建立更加可靠和灵活，在 failover 处理上也更加快速。

#### 集群创建与管理（percona/mariadb 支持）

Cluster 功能目前在 MySQL 原生版本暂时不支持，但是其两个分支版本 percona 和 mariadb 基于Galera库实现的集群复制技术是支持的。另外 Liberty 版本的 Trove 也提供了对 MongoBD 的集群支持。

Trove 还提供了以其他一些许多功能，使用 troveclient 可以有详细的帮助信息输出。对于一个要求不是很高的云数据库管理平台，Trove 目前这些功能已经够用，而且能够搭配好 OpenStack 其他服务无缝运行起来。

### Trove 实践案例

在 OpenStack 上启用 Trove 服务有两个要点。一个是要制作 Trove 的数据库实例运行的 VM 镜像，因为要把 troveguestagent 和相关的数据库引擎（比如 MySQL）打包进镜像，而且需要精简 OS。另外，就是要有一个包含 Trove 服务的 OpenStack 环境。幸好 Trove 项目有个子项目 trove-integration，把这些工作都打包了，方便了开发者使用。本文实践的例子是在一个虚拟机上利用 trove-integration 建立带 Trove 的 OpenStack 的运行环境，并且制作 Trove 实例运行的 VM 镜像（MySQL 镜像），并在上面搭建 MySQL一主二从实例集群运行起来。

准备一个运行 Ubuntu 的虚拟机（trove-integration 建议用 Ubuntu），内存至少8G以上，CPU 最少2核以上，否则跑起整个环境来会比较吃力，用 libvirt 快速创建一个。

用Git 下载 trove-integration（https://github.com/OpenStack/trove-integration.git）。仔细查看目录可以看到其实是利用了 devstack 去安装整个 OpenStack 环境，而且里面还带了 diskimage-builder 工具，专门制作 OpenStack 的各种VM镜像（利用 diskimage-builder 创建 Trove 镜像在 OpenStack 官方也给了专门的讲解《Building Guest Images for OpenStack Trove》（http://docs.OpenStack.org/developer/trove/dev/building_guest_images.html）。

安装带 Trove 的 OpenStack 环境。进入 trove-integration 目录 trove-integration/scripts 下，运行./redstack install，这个过程会比较长，视乎网络速度和运行的机器的速度，在我的虚拟机上运行大概1个小时多一点。redstack 是一个入口程序，里面有很多功能，可以单独运行有详细的帮助信息。

制作 MySQL 的实例运行镜像，并把镜像导入 Glance，在 Trove 中建立 datastore（比如 MySQL）和 version（比如5.6版本）信息，运行./redstack kick-start MySQL。（其中 version 信息是包含存放在 Glance 中的 vm 镜像信息）这个过程也有点长，因为是使用 diskimage-builder 根据定义好的 element 去动态下载打包并配置镜像的。

实际上利用 trove-instegration 项目建立的实例运行的 VM 镜像，并没有把 guestagent 打包进去，而是通过 VM 文件系统中 /etc/init/troveguestagent 脚本，在 vm 启动的时候从宿主机上把 guestagent 拷贝过去，这个过程其实是很快的，因为程序比较小。

以上过程可以参考 https://wiki.OpenStack.org/wiki/Trove/trove-integration，由于实际运行可能会由于环境问题导致出错，需要重试或者具体问题具体分析。

利用 OpenStack 提供的 Trove 客户端工具 troveclient 创建 MySQL 实例并搭建集群。运行 trove -h 会有详细的帮助信息。

**创建主实例**

trove create my_inst_master 8 --size 10 --database my_inst_db --users admin:admin123 --datastore MySQL --datastore_version 5.6 

实例名字为 my_inst_master，实例规格 ID 是8（512 MB内存），硬盘卷大小是10 GB，并且创建数据库 my_inst_db 和用户 admin （密码是 admin123），数据库引擎类型是 MySQL，版本是5.6版本。

**创建主实例的备份（先随便在创建的数据库实例里创建表和插入一些数据）**

trove backup-create my\_inst\_master my\_bak.0001

创建数据库实例 my\_inst\_master 的当前的备份，备份名字为 my\_bak.0001。

**从主实例的备份创建两个从实例，并且建立主从关系**

trove create my_inst_slave 8 --size 10 --backup my_bak.0001 --replica_of my_inst_master --replica_count 2

创建两个数据库实例，名字以 MySQL\_inst\_slave 开头，实例规格 ID 为8，硬盘卷大小是10 GB，并且用备份名为 my\_bak.0001 的备份导入数据，且建立到实例 my\_inst\_master 的主从复制关系。

**动态 Resize 主实例规格**

trove resize-instance my_inst_master 2

动态调整实例 my_inst_master 的 instance 规格为2（内存2 GB 大小）。

trove resize-volume my_inst_master 20

动态调整实例 my\_inst\_master 的硬盘卷大小为20 GB。

### 对比与总结

从以上的介绍可以知道，OpenStack 的 DBaaS 服务 Trove 确实有所创新，不同与业界的类似产品实现。在这样的设计目标与实现机制下，可以说有优点也有缺点，通过与典型的云数据库平台进行对别，稍微总结如表1、表2所示。

表1 OpenStack Trove 与典型平台对比

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56a98f4cd686e.jpg" alt="表1 OpenStack Trove与典型平台对比" title="表1 OpenStack Trove与典型平台对比" />

表2

<img src="http://ipad-cms.csdn.net/cms/attachment/201602/56a98f5f7a829.jpg" alt="表2" title="表2" />

目前大规模使用 Trove 做云数据库平台有 ebay、Rackspace、HP 等公司。近来也得益于 OpenStack 的版本迭代速度较快，从 Juno 版本开始每个版本 Trove 都会带来不少新特性，项目本身也越来越受社区重视，相信在后续版本中，Trove 会补足已有的一些缺点，充分利用OpenStack平台的优势，在稳定性和可运营性方面做得更好，特别是在实例创建与调度上提升容灾级别，增加 taskmanager 这个中心点的容灾能力，增加数据库实例的监控信息、自动透明切换实例的能力，所以 Trove 的下两个版本应该是非常值得期待。