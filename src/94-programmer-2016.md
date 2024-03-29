## 容器集群管理技术对比

文/吴恒

企业 IT 架构是指机房硬件管理和应用开发交付模式，前者涉及物理资源的高效便捷管理，衍生出虚拟化技术；后者关系到开发运维协同工作，加速容器技术的发展。特别是在互联网化已成为促进信息技术产业发展和应用创新主要推手的当代，迫切需要应对“交付方式移动化、用户峰值极限化、版本更新高频化、失效恢复实时化”的应用新特点。因此，企业对应用管控的性能、敏捷性和可靠性提出了更高的要求， IT 架构转型升级迫在眉睫。

容器是一种新型物理资源抽象方法，其核心思想是操作系统内核的复用，通过提供应用运行环境描述规范，自动构建应用（进程）的沙箱运行环境，达到应用相互隔离的目的。如图1所示，与虚拟机技术相比，容器回避了冗余 OS（Duplicated OS）问题，理论具有与物理机接近的性能；容器引入应用环境描述规范，机器识别替代基于脚本和文档的应用环境构建，容易维护，不易出错。因此，容器具有更好的性能，更优的敏捷性，正逐渐成为构建应用系统运行环境的主流基础设施。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/5779d2a131f19.jpg" alt="图1   虚拟机和容器区别" title="图1   虚拟机和容器区别" />

图1   虚拟机和容器区别

容器集群是一种将多台物理机抽象为逻辑上单一容器的技术，在有效应对规模化管理需求的同时，通常会引入基于记录和回放的失效恢复机制，大大提高容器的可靠性。当前，容器集群已成为容器产业化的事实管理方式，呈现出“百花争鸣”的竞争态势，本文将考虑资源管理和应用管控两个方面，并从生态角度进行简单讨论，对已有容器集群管理技术进行初步分析，可为企业容器集群技术选择提供依据。

### 现状：百花争鸣

容器集群已成为实现应用敏捷管理的主流手段，公有云 Amazon、Microsoft、Google、Aliyun 纷纷推出新产品抢占容器市场，开源界 OpenShift、Cloudfoundry、Rancher、Docker Cloud、OpenStack 也都将容器作为战略方向构建生态，百花争鸣。各家采用的容器集群管理技术也不尽相同，主要包括 Kubernetes、Mesos、Swarm/Swarmkit、Nova 四大类，呈现出群雄争霸的局面，如表1所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/5779d3258c7d4.jpg" alt="表1  主流容器集群管理技术" title="表1  主流容器集群管理技术" />

表1  主流容器集群管理技术

初步观察，各种容器集群管理技术都具有良好的产业支持，也有（较）重量的开源跟进，但仔细分析，能发现其中的异同：

1. Kubernetes：大多为互联网企业使用，主要用于用于简化互联网应用的管理，特别是容错方面；

2. Mesos：大多面向企业内部，强调资源的统一管理；

3. Swarm/Swarmkit：大多为公有云厂商，具有规模化资源管理和个性化应用管控需求，比如应用的灰度发布；

4. Nova：大多为面向具有 OpenStack 基础的企业内部，存在虚拟机和容器两种解决方案，需要统一管理。

由此可见，资源管理和应用管控是企业 IT 架构两个关键要素。其中，资源管理是指具备物理资源统一抽象、按需供给、负载均衡等能力，实现容器与物理资源的合理关联放置；应用管控是以应用逻辑为核心，统一管理具有关联关系的多个容器，比如应用的典型架构涉及三层，包括 Web 服务器、应用服务器和数据库服务器，会部署在不同容器中。Docker 官方调研报告将应用管理作为容器核心能力之一，认为 Kubernetes 和 Swarm/Swarmkit 优势明显，如表2所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/5779d32c900b5.jpg" alt="表2  主流容器集群管理对资源管理和应用管控的支持" title="表2  主流容器集群管理对资源管理和应用管控的支持" />

表2  主流容器集群管理对资源管理和应用管控的支持

从生态环境来看，Kubernetes 吸取了 Google Borg 的精髓，后者被认为是 Google 近十年最“神秘的系统”，具有线上成熟的大规模类容器管理经验，因此是当前互联网厂商模仿和使用的对象；Mesos 受 Apache 支持，最初面向短任务场景（如 Hadoop，在有限时间内完成并释放资源），专注资源按需隔离、快速决策和回收管理，但扩展到 Facebook、Twitter 等7*24类服务型应用，需定制应用管理能力；Swarm/Swarmkit 由 Docker 社区亲自推动，通过 Docker 本身能力的增强，以更加简洁有效的方式实现类似 Kubernetes（swarmkit 已经发布）的应用管控能力，社区关注度和贡献者发展很快；Nova 得易于 OpenStack 社区推动，但落地应用一方面强依赖于 OpenStack 实施的广度和深度，另一方面需要扩展 OpenStack 支持应用管控。具体生态对比如表3所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/5779d3343bf2a.jpg" alt="表3  主流容器集群管理生态基础" title="表3  主流容器集群管理生态基础" />

表3  主流容器集群管理生态基础

综上所述，结合 Docker 官网关于应用管控的发展趋势，成熟稳定的 Kubernetes 和原生支持的 Swarm/Swarmkit 具有更广阔的发展前景。OpenStack 也具有一定的竞争优势，得益于 OpenShift、Cloudfoundry 等关注应用管控平台的天然支持，但复杂性是其短板。相对而言，就支持应用管控能力而言，Mesos 的生态环境相对较差，如表4所示。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/5779d33ea4b2b.jpg" alt="表4  主流容器集群管理如何支持应用管控" title="表4  主流容器集群管理如何支持应用管控" />

表4  主流容器集群管理如何支持应用管控

### 未来：双雄争霸

从 Docker 官方的发展规划、开源应用管理平台的转型力度（OpenShift3基于 Docker 技术重新实现应用管理逻辑、Cloudfoundry 大力投入支持基于 Dokcer 进行应用管理），基于 Docker 的公有云的定位和具体实现，可见资源管理和应用管控是容器集群的两大关键要素。通过前述已有技术的分析和生态预期，Kubernetes 和 Swarmkit 应具有更广阔的发展前景。尽管当前，基于 Kubernetes 的容器集群管理方案一直是容器大会的热点。但值得预期，随着 swarmkit 的不断成熟和社区推动，会有越来越多的相关解决方案。未来，双雄争霸将会是容器集群管理的焦点。