## 云计算 ZStack 分布式集群部署

文/郭华星

>本文介绍 ZStack 近期主要新增功能和修复内容，以及通过近乎生产环境的部署实践向读者展现 ZStack 分布式架构优势。

2015年8月份，ZStack 发布0.9版本，带来重量级的特性支持——管理上实现统一的分布式存储 Ceph。Ceph 是加州大学 Santa Cruz 分校的 Sage Weil（DreamHost 的联合创始人）专为博士论文设计的新一代自由软件分布式文件系统。Ceph 以其分布式架构著称，其具备存放海量数据、高性能和高可靠特性。当初开发 Ceph 的公司 Inktank 在2014年5月被 RedHat 收购。

ZStack 在此版本支持 Ceph 块存储 RBD 作为主存储（Primary Storage）和备份存储（Backup Storage），前者是存放云主机运行访问的磁盘数据，而后者是模板、快照和光盘镜像存放的位置。ZStack 首度支持 Ceph 即实现了云盘和镜像的统一。对比 OpenStack Cinder 和 Glance 在 Ceph/RBD 支持方案上缓慢的演进过程，ZStack 一步到位实现，并超越 CloudStack+KVM+Ceph 方案繁复歧义的实现（CloudStack 在 KVM 虚拟化场景下，二级存储不能使用 Ceph/RBD，需要另外考虑）。

在软件工程里面，软件项目发布1.0版本意味着本项目已经达到成熟、稳定和可靠的水平。2016年1月份，ZStack 发布1.0版本，是技术发展的历史性里程碑，代表 ZStack 有勇气面对实际生产环境的应用部署，并能带来数据中心自动化运维价值。

此版本支持一个显著特性是，通过 Linux 命名空间（Name Space）在 KVM 虚拟化物理节点实现分布式 DHCP 功能。对比过去版本，DHCP 必须在虚拟路由上提供服务。如果在网络不稳定的环境下，虚拟路由未能完全启动成功，或者某些原因导致虚拟路由无法正常工作，此时云主机是无法优先启动的。这将会给用户带来非常糟糕的体验，延迟了云主机启动时间，影响业务系统的恢复。因此，通过在 Linux 命名空间实现分布式 DHCP，ZStack kvmagent 守护进程直接在物理服务器网桥（Bridge）上操作。在千万级云主机并发启动环境下，分布式 DNS 实现了服务的分散化，是真正意义上的分布式网络服务。

当前 ZStack 最新版本为1.0.1，已经涵盖了云计算 IaaS 的主流功能，并在某些方面超越以往的 IaaS。ZStack 新功能的集成和细微的完善，得益于借鉴以往的云计算架构和开放的技术讨论社区，当然，离不开 ZStack 开发团队高水平的代码实现能力。

虽然 ZStack 在其发展之初定位致力于从架构上解决易用性、稳定性、高性能和扩展性问题。但是从 ZStack 技术爱好者反馈的情况，有想把 ZStack 从单节点部署推广到集群部署，有想测试 Ceph 分布式存储系统，但是困惑于资料的分散和非权威性。因此，笔者在实验环境搭建 ZStack 集群，采用“KVM 虚拟化+Ceph/RBD 存储+分布式 DHCP”技术架构，构建了一个近乎生产环境的云计算 IaaS 解决方案。

本案例的实施架构如图1所示：

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d55479ba398.png" alt="图1 ZStack集群案例实施架构" title="图1 ZStack集群案例实施架构" />

图1 ZStack 集群案例实施架构

其中节点的信息为（如表1）

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d554894b1a3.png" alt="如表1 节点的信息" title="如表1 节点的信息" />

目前 ZStack 1.0.1官方以 CentOS 7.2 x86\_64 发行版本作为开发和测试环境，所以实际生产环境的 ZStack 集群部署在 CentOS 7.2 x86\_64 较好，很多隐藏问题会在开发过程中首先解决。本案例使用 CentOS 7.2 x86\_64 部署所有节点，并配置 CentOS-Base.repo 和 epel.repo 指向阿里云开源镜像 mirror.aliyun.com。另外，全集群节点关闭 SELinux。

此外，由于默认 CentOS 7.2 x86\_64 系统选择 mini 安装后的缺乏常规工具，同时使用 Firewalld 作为防火墙服务，需要切换 Iptables，具体在各节点执行如下更改命令：

```
yum -y install vim wget bash-completion net-tools pciutils sysstat iptables iptables-services 
service firewalld stop && service iptables restart &&  chkconfig firewalld off && chkconfig iptables on
```

接下来展开部署过程：(a)部署 Ceph，(b)部署 Galera MariaDB 集群，(c)部署 Rabbitmq 集群，(d)ZStack 管理节点集群。最后在 ZStack 中完成添加 KVM 虚拟化节点和 Ceph 存储节点，初始化分布式 DHCP 服务，创建第一个可用的云主机。

### 部署 Ceph 存储

笔者工作中涉及 Ceph 的配置、测试与运维，使用不同的 Ceph 版本会对具体测试和运维有不同影响，读者可以通过 Ceph 社区官方手册部署官方发布的二进制版本。当前的长期支持版（LTS，Long Term Support）是 Ceph 0.94.6，开发代号为 Hammer。读者也可以下载对应版本的 Ceph 源码（http://download.ceph.com/tarballs/）在本地进行编译后安装。同时，如果读者想跟随 RedHat Ceph Storage 的版本，可以下载 RedHat 公开的源码编译后部署。具体托管位置是：ftp://ftp.redhat.com/redhat/linux/enterprise/7Server/en/RHCEPH/SRPMS/ 。使用 RedHat Ceph 版本的好处在于：获得类似商业版本的良好稳定性（包含积累性的补丁文件），以及集成的 LTTng（ http://lttng.org/ ）调试工具。

编译成功后在 RPM 包在 root/rpmbuild/RPMS/获得。读者可以根据 Ceph 社区的文档进行安装与初始化，其涉及的 Linux 内核参数、XFS 格式化/挂载参数、Ceph 参数、NUMA 绑定和网卡 IRQ 中断优化等手段，相关内容请查看 Ceph 社区资料和厂商公开的优化文档。

#### 部署 Galera MariaDB 集群

MariaDB 数据库是 MySQL 数据库的一个分支，主要由开源社区在维护，采用 GPL 授权许可。MariaDB 的目的是完全兼容 MySQL，包括 API 和命令行，使之能轻松成为 MySQL 的代替品。Galera Cluster 是 MariaDB 的多活多主集群，为 MariaDB 提供了同步复制，因此其可以保证多个 MariaDB 高可用。读者可以查看有关 Galera MariaDB 文档完成整个部署过程。另外还需要在 MariaDB 创建 ZStack 访问账户 zstack：

```
[root@zstack-1 ~]# mysql -uroot -p
mysql> grant ALL PRIVILEGES on *.* to zstack@"%" Identified by "zstack123";
mysql> grant ALL PRIVILEGES on *.* to zstack@"localhost" Identified by "zstack123";
mysql> grant ALL PRIVILEGES on *.* to zstack@"zstack-1" Identified by "zstack123";
mysql> grant ALL PRIVILEGES on *.* to root@"%" Identified by "zstack123";
mysql> grant ALL PRIVILEGES on *.* to root@"localhost" Identified by "zstack123";
mysql> grant ALL PRIVILEGES ON *.* TO root@"%" IDENTIFIED BY "zstack123" WITH GRANT OPTION;
mysql> flush privileges;
```

### RabbitMQ 集群

消息队列（MQ，Message Queue），一般用于应用系统解耦、消息异步分发，能够提高系统吞吐量。消息队列的产品有很多，常见的开源消息队列组件有：ZeroMQ、RabbitMQ、ActiveMQ和Kafka/Jafka 等。ZStack 在架构上设计了消息队列服务，并默认支持 RabbitMQ（如图2所示）。

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d554608054d.png" alt="图2 ZStack消息队列模型" title="图2 ZStack消息队列模型" />

图2 ZStack 消息队列模型

本部署案例里，在 zstack-1、zstack-2 和 zstack-3 节点安装 RabbitMQ 集群，并配置镜像模式，使得消息内容在不同 RabbitMQ 节点内相互同步复制（Mirror），以满足高可用场景。读者可以查看有关 RabbitMQ 集群文档完成整个部署过程。

创建 ZStack 访问 RabbitMQ 服务用户权限：

```
[root@zstack-1 ~]# rabbitmqctl add_user zstack zstack123
[root@zstack-1 ~]# rabbitmqctl set_user_tags zstack administrator
[root@zstack-1 ~]# rabbitmqctl change_password zstack zstack123
[root@zstack-1 ~]# rabbitmqctl set_permissions -p / zstack ".*" ".*" ".*"
```

### 部署 ZStack-Server 集群

获取 ZStack 1.0.1正式版：

```
wget -c http://download.zstack.org/releases/1.0/1.0.1/zstack-installer-1.0.1.bin
```

在 zstack-1 执行安装：

```
[root@zstack-1 software]# ./zstack-installer-1.0.1.bin -i -I eth0
```

安装完成后，ZStack 会部署在/usr/local/zstack/目录下，并且会安装 zstack-ctl 和 zstack-cli 组件。通过 zstack-ctl 在 zstack-1 执行数据建表以及初始化：

```
[root@zstack-1 ~]# zstack-ctl deploydb --host=11.0.5.150 --port=3306 --zstack_password=zstack123 --root-password=zstack123
```

修改 zstack-1 的配置：

```
zstack-ctl configure DB.password=zstack123
zstack-ctl configure CloudBus.serverIp.0=11.0.5.150
zstack-ctl configure CloudBus.serverIp.0=11.0.5.151
zstack-ctl configure CloudBus.serverIp.0=11.0.5.152
zstack-ctl configure CloudBus.rabbitmqUsername=zstack
zstack-ctl configure CloudBus.rabbitmqPassword=zstack123
zstack-ctl configure management.server.ip=11.0.5.150
```

启动 zstack-1 管理服务【zstack-ctl start\_node】。其他节点通过执行【zstack-ctl install\_management\_node --host=IP】命令完成安装过程。在各管理节点上安装 ZStack 的 Web 图形界面 Dashboard 服务【zstack-ctl install\_ui】启动完成后，在浏览器中访问其中一个管理节点的地址http://ip:5000/。默认登录用户 admin，密码 password。

以上过程部署了 Galera MariaDB 集群、RabbitMQ 集群和 ZStack 管理节点。ZStack 管理节点通过用户 zstack 访问 MariaDB，并且在配置上，目前 zstack-1、zstack-2 和 zstack-3 这3个节点只访问其中 zstack-1 的 MariaDB 数据库。对于实现故障转移，读者可以考虑部署负载均衡器，使用 haproxy+keepalived 或者 lvs+keepalived 的解决方案，或者考虑商业负载均衡设备实现，并建议使用 master-slave-slave 的策略，优先转发到 zstack-1 MariaDB，故障转移后切换到 zstack-2，再次到 zstack-3。本案例不做具体实现，相信读者可以从其他途径获取有效信息。同样道理，访问 zstack-1、zstack-2 和 zstack-3 能够以打开 Web 界面，可以在用户浏览器和 ZStack 管理节点之间架设负载均衡器，是用基于源 IP 哈希的策略，把不同客户端的访问转发到既定的 ZStack 管理节点上。而 ZStack 管理服务访问 RabbitMQ 节点，是优先访问第1个，若不成功则依次访问其他节点，故不需要实现负载均衡策略。

### ZStack 资源初始化

上文已经部署好 Ceph 存储和 ZStack 管理节点，相信读者迫不及待想了解如何在 ZStack 添加计算资源、存储资源和网络资源。ZStack 的管理逻辑上分为三大类，即计算资源、存储资源和网络资源。计算资源从逻辑上划分区域（Zone）、集群（Cluster）、主机（Host）和云主机（Instance）；存储资源从逻辑上划分主存储（Primary Storage）和备份存储（Backup Storage），其中主存储是云盘的运行所在空间，而备份存储存放光盘镜像（ISO）、模板（Template）和快照（SnapShot）；网络资源从逻辑上划分第二层网络（L2 Network）、第三层网络（L3 Network）和网络服务（Network Service）。其结构相当清晰，如果读者有 CloudStack 或者 OpenStack 的基础认识，将会对这些概念容易理解。

本案例使用 Web 图形界面的方式创建“ZStack+ KVM +Ceph +分布式 DNS”的场景，如果读者对其他应用场景有兴趣，可浏览 ZStack 官方网站。以下主要记录 Ceph 存储资源和分布式 DHCP 的初始化（如图3、图4所示）。

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d554513fa32.png" alt="图3 添加Ceph作为主存储" title="图3 添加Ceph作为主存储" />

图3 添加 Ceph 作为主存储

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d5543e7de40.png" alt="图4 添加Ceph作为备份存储" title="图4 添加Ceph作为备份存储" />

图4 添加 Ceph 作为备份存储

重复提及一次，KVM 物理机节点具有3个网卡，eth0 连接管理网络，eth2 连接存储网络，而 eth1 连接数据网络，也就是说承载云主机的网络流量。在本次部署案例里，eth0 和 eth2 是 novlan，eth1 是 vlan 模式，尽量模仿生产环境，提供二层隔离网络。在数据网络，笔者实验环境分配网络 vlan id 52，网段是11.0.52.0/24，其中可用于云主机的 IP 地址范围为11.0.52.200~11.0.52.250。如图5、图6所示，创建 L2、L3 网络。

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d554305baa8.png" alt="图5 创建L2网络" title="图5 创建L2网络" />

图5 创建 L2 网络

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d554235535b.png" alt="图6 创建L3网络" title="图6 创建L3网络" />

图6 创建 L3 网络

至此，Flat-Network 的分布式 DHCP 已经配置完成。最后，激动人心的时刻到来了，成功创建第一台云主机（如图7、图8所示）

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d55411cb65b.png" alt="图7 创建第一台云主机" title="图7 创建第一台云主机" />

图7 创建第一台云主机

<img src="http://ipad-cms.csdn.net/cms/attachment/201603/56d553faa54e0.png" alt="图8 成功创建" title="图8 成功创建" />

图8 成功创建

### 回顾与总结

首先我们再查看一下 Ceph 的 RBD Pool 的信息，执行【ceph osd dump】显示其状态。ZStack 在 Ceph 上自动建立了4个存储池，pri-c 开头的池存放云盘模板缓存，pri-v-r 开头的池存放云主机的系统磁盘，pri-v-d 开头的池存放云主机的数据磁盘，bak-t 开头的池存放镜像和模板。当前在 Web 界面，添加主存储和备份存储都未能指定 RBD Pool，通过 zstack-cli 则可以指定已经创建好的 RBD Pool。默认 ZStack 创建 RBD Pool 的 pg\_num 和 pgp\_num 都是100，读者可以根据实际环境调整参数；默认 ZStack 创建的 RBD Pool 在 CRUSH Default Rule，若读者想调整不同 RBD Pool 在不同的 CRUSH 上以实现 SSD、SAS 和 SATA 等不同存储介质的区分使用，可以通过执行【ceph osd pool set poolname crush_ruleset id】的方式在线调整。

行文至此，笔者介绍了 ZStack 功能发展历程和通过“KVM 虚拟化+Ceph/RBD 存储+分布式 DHCP”的近乎生产环境部署案例，可以认为 ZStack 是成熟的云计算 IaaS 解决方案。同时，在实际生产环境部署中，企业用户需要驾驭不同开源组件的功能和特性，特别是 KVM 虚拟化和 Ceph 分布式存储的调优，也可以通过第三方获得技术支持。俗语说，好马也要配好鞍。ZStack 有很强的管理大规模虚拟化和海量存储能力，我们期待 ZStack 技术社区持续发展，在云端技术领域多一种技术选择，也许这种选择是合适的，也是对的。