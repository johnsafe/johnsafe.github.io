## DC/OS 服务开发指南

文/陈冉

>DC/OS 是一个基于 Apache Mesos 的分布式操作系统，它会像管理一台计算机一样去管理多种异构资源，其中包括虚拟机、物理机、容器等。DC/OS 会自动调度和管理资源，自动分发任务和作业，也会简化安装和分布式服务的管理，另外，还包括用于远程管理和集群监控，服务 Web 接口和空用的 CLI。

为了更简单理解 DC/OS，我们通过重用 Linux 的内核和用户空间作为解释。核心区主要是一个不能够被用户访问并且是低层次操作保护区，主要包括资源隔离、安全、进程隔离等。用户空间是用于用户应用程序和更高级服务，比如 GUI。

在继续深潜之前，我们先介绍 DC/OS 的架构。

Master：整合所有 Agent 收集过来的资源，并且提供给各种框架。

Scheduler：服务的 Scheduler，比如 Marathon scheduler。

User：也称之为客户端，是一个可以基于内部和外部的集群并且可以产生服务的应用，比如用户提交一个 Marathon 应用。

Agent：可以代表 Framework 运行不连续的 Mesos 任务，是一个被 Mesos Master 注册的 Agent。Agent 节点的类似词包括 Worker、Slave 节点。我们可以有混合资源 Agents，包括基于公有云的公有 Agent 和基于私有云的私有节点。

![](https://i.imgur.com/GJSJZtP.jpg)

图1 DC/OS 组件

Executor：启动一个 Agent 节点为一个应用服务运行任务（Task）。

Task：能被 Mesos Framework 调度的工作单元，并且可以运行在 Mesos Agent 上。

Process：通过客户端触发的一个逻辑集合（Logical Collection），集合中包括很多任务，其中包括 Marathon 的 App 或者 Chronos 的 Job。

详细时序如图2所示：

![](https://i.imgur.com/k5YT5uG.jpg)

图2 DC/OS 工作流时序图

### 服务包

在 DC/OS 里，Universe 包是在集群里可以安装的服务。我们区别于选中的包和社区的包。选中的包是一个被通过认证过的包。

所有在包里的服务都被要求满足 DC/OS 项目组定的标准。

### Admin Router 和 Web Interface 结合

在默认情况下，一个 DC/OS 服务是要被部署到私有 Agent 节点的。为了允许用户能够对服务进行配置控制和监控，Admin Router 代理调用在私有集群上的 Master 节点服务。可以通过 HTTP 服务接口的路径去申请资源和 artifacts。服务的接口提供 Web Interface、RESTful 接入。当通过 DC/OS CLI 运行一个命令，通常会调用 RESTful 接口和相关的 scheduler 服务进行通讯。

当一个 Framework Scheduler 在注册阶段和 Mesos Master 注册为一个 webui_url 时，Admin Router 会自动整合到一起，有几个限制：

URL 必须不用“／”结尾。比如，internal.dcos.host.name:10000 是正确的，internal.dcos.host.name:10000/ 是错误的。

DC/OS 支持1 URL 和 Port。

当 webui_url 已经提供时，所有服务都会显示在 DC/OS web interface 包括 Link。Link 是 Admin Router 代理 URL 的名字，基于命名方式 /service/<service_name>。

例如：<dcos_host>/service/unicorn 是 webui_url 的代理。如果你提供一个 Web interface，将会集成到 DC/OS Web Interface 里，然后用户可以通过点击Linker进行快速访问。

服务监控检查信息是通过以下 DC/OS service tab 提供的：

1. 这里有服务监控检查被定义在 marathon.json 文件中，比如：
    <img src="http://ipad-cms.csdn.net/cms/attachment/201608/579ee981d1d1f.png" alt="代码1" title="代码1" />
2. Framework-name 属性在 marathon.json 是合法的。
    <img src="http://ipad-cms.csdn.net/cms/attachment/201608/579ee9984601d.png" alt="代码2" title="代码2" />
3. Framework 的属性在 package.json 文件中设置为 true，比如：
    <img src="http://ipad-cms.csdn.net/cms/attachment/201608/579ee9b37d2ad.png" alt="代码3" title="代码3" />

你也可以把代理部署到公共的 Agent，这样就可以通过 Admin Router 在外网访问服务。建议使用 DC/OS web interface 把 Admin Router 和 scheduler 配置和控制起来。同时也建议通过 CLI 和 RESTful 接口访问 scheduler。

### 包结构

每个包都包括 package.json、config.json 和 marathon.json；DC/OS 服务说明书中包括以上文件的描述。

**package.json**

"name": "cassandra", 参数在包库里指定会定义 DC/OS 服务名。这里也必须是在文件中的第一个参数。

关注以下服务描述。假设所有的用户都熟悉 DC/OS 和 Mesos。

这里的 Tags 参数常用于用户搜索(dcos package search <criteria>)。也可以增加 tags 从而区别的服务。避免使用 Mesos，Mesosphere，DC/OS 和 datacenter 等。比如 unicors 服务可能包括"tags": ["rainbows", "mythical"]。

参数 preInstallNotes 给用户一个选项，就是在开始安装之前可以标识相关操作。比如，可以解释对服务的资源需求："preInstallNotes":"Unicorns take 7 nodes with 1 core each and 1TB of ram."

参数 postInstallNotes 在安装过程后给用户一个信息。主要用于提供文档 URL，教材或者两者都有。比如："postInstallNotes": "Thank you for installing the Unicorn service.\n\n\tDocumentation: http://<your-url>\n\tIssues: https://github.com/"

参数 postUninstallNotes 给提供用户信息，主要用于反安装。比如，在重新卸载之前的清空。一个通常的问题就是清空ZooKeeper的属性。比如: “postUninstallNotes": "The Unicorn DC/OS Service has been uninstalled and will no longer run.\nPlease follow the instructions at http://<your-URL> to clean up any persisted state" }

**config.json**

所有没有条件区域的 marathon.json 文件需要的属性都是需求区域（这里属性都是用户提供）。

**marathon.json**

第二级别（nested）属性必须是一个framework-name附加一个服务名的值，比如：
"framework-name" : "{{unicorn-framework-name}}" 

使用id参数的同样的值，比如：

"id" : "{{unicorn-framework-name}}" 

所有服务使用的 URLs 必须传递给服务通过命令行或者环境变量访问的。

### 如何创建包

以下是如何详细创建 DC/OS 服务的工作流：

1. Fork 一个 Universe GitHub repository.
2. 在创建 DC/OS 服务库时：

    命名服务, 比如,unicorn。DC/OS 包库的目录结构为: repo/packages/<initial-letter>/<service-name>/<version>。

    在 repo/packages 为服务创建一个目录，比如：repo/packages/U/unicorn。

    创建一个版本索引目录。比如,repo/packages/U/unicorn/0。

    增加 package.json, config.json、marathon.json 这三个文件到索引目录。

    如果有 CLI 子命令集，也可以将 command.json 文件加到索引目录中。

3. 在 DC/OS 里测试服务:

    配置DC/OS并指向本地库中，比如，如果fork的库是https://github.com/example/universe 并使用version-2.x分支，再通过命令增加到DC/OS配置中

    ```
    $ dcos package repo add my-repo https://github.com/example/universe/archive/version-2.x.zip
    ```

### 命名方式和目录结构

把 JSON 文件放入索引目录后，在 universe>/scripts 目录中有一些脚本。按照数字顺序运行库中的脚本。如果脚本运行成功，就可以继续执行下一个。
1. 运行脚本 0-validate-version.sh 验证版本信息。
2. 运行脚本 1-validate-packages.sh 验证 command.json、config.json 和 package.json 文件中的语法以及 schema。
3. 运行脚本 2-build-index.sh 增加 DC/OS 服务到 index.json 文件中。
4. 运行脚本 3-validate-index.sh 验证 index.json 文件想了解更多 JSON 文件内容，可以访问 Universe 文档。