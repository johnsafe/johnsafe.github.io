## 新型资源管理工具 Myriad 使用初探



文/陈冉

Myriad 是由 MapR 主导并且由 eBay、Mesosphere 参与的项目，目前还没有分享 Myriad 相关实际研究成果的文章。本文将会展示 Linker Networks 最近对于 Myriad 项目的研究，并结合实际研究阐述 Myriad 项目的具体工作原理。首先本文简单介绍该项目的研究背景与意义；之后介绍其构建、启动和使用过程；最后将以源码为基础解释 Myriad 的启动和使用原理。

### 研究背景与意义

#### YARN
YARN（Yet Another Resource Negotiator）是一种通用的资源管理系统，旨在克服 MRv1 的缺陷，为了更大规模的数据处理而诞生。它包含四个主要的组件：

- ResourceManager，管理全局资源；

- NodeManager，作为 ResourceManager 的代理管理节点资源；

- ApplicationMaster 代替 MRv1 里的 JobTracker；

- Container 作为资源载体在 NodeManager 的监控之中执行 ApplicationMaster 任务。

#### Mesos
Mesos 是 Apache 下的开源分布式资源管理框架，它被称为分布式系统的内核。它包含四个主要的组件：

- Mesos-master：管理 Framework 与 Slave，分配和管理 Slave 的资源给 Framework；

- Mesos-slave：管理 Task，给 Executor 分配资源；

- Framework：利用 MesosSchedulerDiver 接入到 Mesos 的框架；

- Executor：执行器，启动计算框架中的 task。

#### 两者的不同
Mesos 与 Yarn 的不同主要表现在：

- scheduler：Mesos 让 framework 决定 Mesos 提供的这个资源是否适合该 job，从而接受或者拒绝这个资源。而对于 YARN 来说，决定权在于其自身，所以从 scaling 的角度来说，Mesos 更 scalable。

- 其次 YARN 只为 Hadoop jobs 提供了一个 static partitioning。而 Mesos 的设计目标是为各个框架（Hadoop、Spark、Web Services 等）提供 dynamical partitioning，让各个集群框架共用数据中心机器。虽然 YARN 有发展，但是改变不了其为了 Hadoop 而设计的初衷（MR2）。

- Mesos 的资源分配粒度更细，比 YARN 管理资源更加精细。

#### Myriad 的作用
YARN 在处理 Mapreduce job 时有着天然优势，而 Mesos 却不具备，Myriad 项目的出现正好解决了这一难题。这里，Myriad 向 Mesos 申请资源并交给 Yarn 来使用。Myriad 实现了 Mesos 的接口，这样它可以和 Mesos 通信并且申请资源，在 Mesos 当中执行 Yarn 的任务。

从 Mesos 的角度来看，Myriad 具有 Myriad Scheduler 和 Executor 两个组件。Myriad Scheduler 是两个框架合并的关键，具体来说，Myriad Scheduler 实现了 YARN 中的 Fair Scheduler 资源调度器并以此为基础实现了 Mesos 接口，于是 Myriad Scheduler 就同时具备了向 Mesos 集群申请资源并在 Yarn 集群内部分配资源的功能。

### Myriad 构建

#### Myriad 必要条件
按照官方说明，Myriad 的运行需要以下条件：

-  JDK 1.7+ （Java 编译与运行环境）

- Gradle （编译工具）

- Hadoop 2.7.0 （实测 CDH 的版本也可以运行，具体会在下文中解释）

- Hadoop HDFS （用于 share runtime data 和运行文件）

- ZooKeeper （管理 Mesos 集群）

- Mesos with Marathon (也可以没有 Marathon)

- Mesos-DNS (需配合 Marathon 把 Myriad 注册为 Mesos 的 Framework 才可用)

Marathon 和 MesosDNS 并不是必须的，如果没有它们则需要在配置文件中指定 ResourceManager 的 url，并且在 hosts 下添加所有节点的访问信息。此外，如果使用了 MesosDNS 并且在 yarn-site.xml 中没有指定 yarn.resourcemanager.hostname ，还需要添加一个环境变量：YARN\_RESOURCEMANAGER_OPTS=-Dyarn.resourcemanager.hostname=rm.marathon.mesos。

此外值得注意的是，Myriad 目前只处在 Alpha 阶段，Bug 依然很多功能也不完善，比如 Cgroups 功能还有 Bug，flux up/down 失败等。

下面就如何构建 Myriad 分享我们的经验。

#### 编译 Myriad

```
./gradlew build
```

scheduler jars

```
$PROJECT_HOME/myriad-scheduler/build/libs/
```

executor jars

```
$PROJECT_HOME/myriad-executor/build/libs/
```

#### 拷贝 jar 文件

```
cp myriad-scheduler/build/libs/*.jar /opt/hadoop-2.7.0/share/hadoop/yarn/lib/
cp myriad-executor/build/libs/myriad-executor-0.1.0.jar /opt/hadoop-2.7.0/share/hadoop/yarn/lib/
cp myriad-scheduler/build/resources/main/myriad-config-default.yml /opt/hadoop-2.7.0/etc/hadoop/
```

对于 CDH 版本，需要把 libmesos.so 等 so 文件复制到 share/lib/native 目录下，否则启动报错。

#### 配置文件

1.mapred-site.xml

示例：

```
<configuration>
    <property>
        <name>mapreduce.shuffle.port</name>
        <value>${myriad.mapreduce.shuffle.port}</value>
    </property>
</configuration>
```

2.yarn-site.xml

示例：
<https://cwiki.apache.org/confluence/display/MYRIAD/Sample%3A+yarn-site.xml>

这里可能需要添加 yarn.resourcemanager.hostname 来指定 ResourceManager 的位置，如果使用 Marathon 和 Mesos-DNS 这一变量就可以设置成 rm.marathon.mesos（也可以和本节最开始提到的一样，通过环境变量的形式来进行设定）。

3.myriad-config-default.yml

示例：
<https://cwiki.apache.org/confluence/display/MYRIAD/Sample%3A+myriad-config-default.yml>

配置文件内部的 jvmMaxMemoryMB 千万不要调小，否则可能出现莫名的启动错误；executor 中的 path 需要指定 NFS 或者 HDFS 当中的路径，务必要保证集群中所有节点都可以 access 到；nodeManagerUri 可以是 HTTP 地址也可以是 HDFS 的地址，按照官方要求，在打包时最好把 yarn-site.xml 文件删除，因为这个文件会在运行时自动生成；YARN\_HOME 必须是一个相对路径，因为 Mesos 在执行任务时执行路径是不一定的。

#### 打包并上传执行文件

首先删除 yarn-site.xml 然后执行：

```
tar -zxvf hadoop-myriad.tgz hadoop-myriad 
hadoop fs -put hadoop-myriad.tgz /dist/
```

#### 启动和 log 目录

```
./sbin/yarn-daemon.sh start resourcemanager
log文件：${HADOOP_HOME}/logs
```

### 运行实例与解释

我们利用 CDH 的 hadoop-2.6.0-cdh5.7.0 进行了测试，发现它也可以很好地运行 Myriad。

#### 简单使用方法
我们在 CDH 的版本之上测试了两种简单的 Myriad 使用方法。

- flux up/down

方式1：web 页面上直接添加或者删除 Instance

方式2：使用 RestAPI

HTTP 请求方式：uri

PUT：/api/cluster/flexup

PUT：/api/cluster/flexdown

- run mapreduce job

hadoop jar HADOOP_HOME/share/hadoop/mapreduce/hadoop-mapreduce-examples-.jar teragen 10000 /outDir

hadoop jar HADOOP_HOME/share/hadoop/mapreduce/hadoop-mapreduce-examples-.jar terasort /outDir /terasortOutDir

#### 解释
这里首先介绍两个概念。

- AuxiliaryService，一个 nodemanager 的定制化组件，有点像 mapreduce 中的 shuffle，可以自己根据需求定制。
- FairScheduler，是 Yarn 中资源调度器的一种（还有 FifoScheduler 和 CapacityScheduler）由 Facebook 开发，它能使得 hadoop 应用能够被多用户公平地共享整个集群资源的调度器。

想必有些读者注意到 yarn-site.xml 当中的一个配置项：

```
<property>
    <name>yarn.resourcemanager.scheduler.class</name>
    <value>org.apache.myriad.scheduler.yarn.MyriadFairScheduler</value>
</property>
```

如果阅读源代码可以发现，Myriad 重写了 Yarn 的 FairScheduler 资源调度器，并且在 initialize Yarn 的过程之中启动了 MesosDriver 并把 Myriad 作为了 Mesos 的 Module 来进行注册。这个重写过程并没有对 Yarn FairScheduler 本身的一些内容进行改变，只是增加了与 Mesos 交互的过程，在 Myriad Scheduler（Yarn RM）启动完成之后，程序还会自动找到合适的 Mesos Slave 根据配置数量相应地把 Myriad Executor（Yarn NM）实例运行起来。

关于 NM，yarn-site.xml 之中也有一些相关的配置项：

```
<property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle,myriad_executor</value>
        <!-- If using MapR distro, please use the following value:<value>mapreduce_shuffle,mapr_direct_shuffle,myriad_executor</value> -->
    </property>
    <property>
        <name>yarn.nodemanager.aux-services.myriad_executor.class</name>
       <value>org.apache.myriad.executor.MyriadExecutorAuxService</value>
    </property>
```

在这里，Myriad 利用 MyriadExecutorAuxService 实现了与 Yarn 程序的交互并且在 MyriadExecutor 中实现了 Mesos Executor 的接口从而可以把 Yarn 的 NM 运行在 Myriad Framework 中的 Task 里。