## Kubernetes 微服务架构应用实践



文/吴治辉

Kubernetes 是新一代基于先进容器技术的微服务架构平台，它将当前火爆的容器技术与微服务架构两大吸引眼球的技术点完美的融为一体，并且切切实实地解决了传统分布式系统开发过程中长期存在的痛点。

本文假设读者已经熟悉并掌握了 Docker 技术，因此不会再花费篇幅介绍。正是通过轻量级的容器隔离技术，Kubernetes 实现了“微服务”化的特性，同时借助于 Docker 提供的基础能力，使得平台的自动化能力得以实现。

### 概念与原理

作为架构师，我们做了这么多年的分布式系统，其实真正关心的并不是服务器、交换机、负载均衡器、监控与部署这些事物，我们真正关心的是“服务”本身，并且在内心深处，我们渴望能实现图1所示的这段“愿景”：
我的系统中有 ServiceA、ServiceB、ServiceC 三种服务，其中 ServiceA 需要部署3个实例、而 ServiceB 与 ServiceC 各自需要部署5个实例，我希望有一个平台（或工具）帮我自动完成上述13个实例的分布式部署，并且持续监控它们。当发现某个服务器宕机或者某个服务实例故障的时候，平台能够自我修复，从而确保在任何时间点，正在运行的服务实例的数量都是我所预期的。这样一来，我和团队只需关注服务开发本身，无需再为头疼的基础设施和运维监控的事情而头大了。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/57760d7ac3bd5.jpg" alt="图1  分布式系统架构愿景" title="图1  分布式系统架构愿景" />

图1  分布式系统架构愿景

直到 Kubernetes 出现之前，没有一个公开的平台声称实现了上面的“愿景”，这一次，又是 Google 的神作惊艳了我们。Kubernetes 让团队有更多的时间关注与业务需求和业务相关的代码本身，从而在很大程度上提升了整个软件团队的工作效率与投入产出比。

Kubernetes 的核心概念只有以下几个：

1. Service

2. Pod

3. Deployments(RC)

Service 表示业务系统中的一个“微服务”，每个具体的 Service 背后都有分布在多个机器上的进程实例来提供服务，这些进程实例在 Kubernetes 里被封装为一个个 Pod，Pod 基本等同于 Docker Container，稍有不同的是 Pod 其实是一组密切捆绑在一起并且“同生共死”的 Docker Container，从模型设计的角度来说，的确存在一个服务实例需要多个进程来提供服务并且它们需要“在一起” 的情况。

Kubernetes 的 Service 与我们通常所说的“Service”有一个明显的不同，前者有一个虚拟 IP 地址，称之为“ClusterIP”，服务与服务之间以“ClusterIP+服务端口”的方式进行访问，而无需复杂的服务发现 API。这样一来，只要知道某个 Service 的 ClusterIP，就能直接访问该服务，为此，Kubernetes 提供了两种方式来解决 ClusterIP 的发现问题。

第一种是通过环境变量，比如我们定义了一个名为 ORDER\_SERVICE 的 Service，分配的 ClusterIP 为10.10.0.3，则在每个服务实例的容器中，会自动增加服务名到ClusterIP 映射的环境变量：ORDER\_SERVICE\_SERVICE\_HOST=10.10.0.3，于是程序可以通过服务名简单获得对应的 ClusterIP。

第二种是通过 DNS，每个服务名与 ClusterIP 的映射关系会被自动同步到 Kubernetes 集群内置的 DNS 组件里，于是直接通过对服务名的 DNS Lookup 机制就找到对应的 ClusterIP 了，这种方式更加直观。

由于 Kubernetes 的 Service 这一独特设计实现思路，使得所有以 TCP /IP 方式进行通信的分布式系统都能很简单地迁移到 Kubernetes 平台上。如图2所示，当客户端访问某个 Service 的时候，Kubernetes 内置的组件 kube-proxy 透明实现了到后端 Pod 的流量负载均衡、会话保持、故障自动恢复等高级特性。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/57760dd051f50.jpg" alt="图2  Kubernetes负载均衡原理" title="图2  Kubernetes负载均衡原理" />

图2  Kubernetes 负载均衡原理

Kubernetes 是如何绑定 Service 与 Pod 的呢？它如何区分哪些 Pod 对应同一个 Service？答案也很简单——“贴标签”。每个 Pod 都可以贴一个或多个不同的标签（Label），而每个 Service 都有一个“标签选择器”（Label Selector）确定了要选择拥有哪些标签的对象，比如下面这段 YAML 格式的内容定义了一个称之为 ku8-redis-master 的 Service，它的标签选择器内容为“app: ku8-redis-master”，表明拥有“app= ku8-redis-master”这个标签的 Pod 都是为它服务的。

```
apiVersion: v1
kind: Service
metadata:
  name: ku8-redis-master
spec:
  ports:
    - port: 6379
  selector:
    app: ku8-redis-master
```

下面是对应的 Pod 定义，注意它的 labels 属性内容：

```
apiVersion: v1
kind: Pod
metadata:
  name: ku8-redis-master
  labels:
       app: ku8-redis-master
spec:
      containers:
        - name: server
          image: redis
          ports:
            - containerPort: 6379
      restartPolicy: Never
```

最后，我们来看看 Deployment/RC 的概念，它的作用是告诉 Kubernetes，某种类型的 Pod（拥有某个特定标签的 Pod）需要在集群中创建几个副本实例，Deployment/RC 的定义其实是 Pod 创建模板（Template）+Pod 副本数量的声明（replicas）：

```
apiVersion: v1
kind: ReplicationController
metadata:
  name: ku8-redis-slave
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app: ku8-redis-slave
    spec:
      containers:
        - name: server
          image: devopsbq/redis-slave
          env:
            - name: MASTER_ADDR
              value: ku8-redis-master
          ports:
            - containerPort: 6379
```

### Kubernetes 开发指南

本节我们以一个传统的 Java 应用为例，来说明如何将其改造迁移到 Kubernetes 的先进微服务架构平台上来。

如图3所示，示例程序是一个跑在 Tomcat 里的 Web 应用，为了简化，没有用任何框架，直接在 JSP 页面里通过 JDBC 操作数据库。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/57760efb133e6.jpg" alt="图3  待改造的Java Web应用" title="图3  待改造的Java Web应用" />

图3  待改造的 Java Web 应用

上述系统中，我们将 MySQL 服务与 Web 应用分别建模为 Kubernetes 中的一个 Service，其中 MySQL 服务的 Service 定义如下：

```
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  ports:
    - port: 3306
  selector:
app: mysql_pod
```

MySQL服务对应的 Deployment/RC 的定义如下：

```
apiVersion: v1
kind: ReplicationController 
metadata:
  name: mysql-deployment
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: mysql_pod
    spec:
      containers:
        - name: mysql
          image: mysql
          imagePullPolicy: IfNotPresent 
          ports:
            - containerPort: 3306
          env:
            - name: MYSQL_ROOT_PASSWORD
              value: "123456"
```

下一步，我们需要改造 Web 应用中获取 MySQL 地址的这段代码，从容器的环境变量中获取上述 MySQ L服务的 IP 与 Port：

```
String ip=System.getenv("MYSQL_SERVICE_HOST");
       String port=System.getenv("MYSQL_SERVICE_PORT");
       ip=(ip==null)?"localhost":ip;
       port=(port==null)?"3306":port;  
      conn = java.sql.DriverManager.getConnection("jdbc:mysql://"+ip+":"+port+"?useUnicode=true&characterEncoding=UTF-8", "root","123456");
```

接下来，将此 Web 应用打包为一个标准的 Docker 镜像，名字为 k8s\_myweb_image，这个镜像直接从官方 Tomcat 镜像上添加我们的 Web 应用目录 demo 到 webapps 目录下即可，Dockerfile 比较简单，如下所示：

```
FROM tomcat
MAINTAINER bestme <bestme@hpe.com>
ADD demo /usr/local/tomcat/webapps/demo</bestme@hpe.com>
```

类似之前的 MySQL 服务定义，下面是这个 Web 应用的 Service 定义：

```
apiVersion: v1
kind: Service
metadata:
  name: hpe-java-web
spec:
  type: NodePort
  ports:
    - port: 8080
      nodePort: 31002 
  selector:
    app: hpe_java_web_pod
```

我们看到这里用了一个特殊的语法：NodePort，并且赋值为31002，作用是让此 Web 应用容器里的8080端口被 NAT 映射到 kuberntetes 里每个 Node 上的31002端口，从而我们可以用 Node 的 IP 和端口31002来访问 Tomcat 的8080端口，比如我本机的可以通过<http://192.168.18.137:31002/demo/>来访问这个 Web 应用。

下面是 Web 应用的 Service 对应的 Deployment/RC 的定义：

```
apiVersion: v1
kind: ReplicationController
metadata:
  name: hpe-java-web-deployement
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: hpe_java_web_pod
    spec:
      containers:
        - name: myweb
          image: k8s_myweb_image
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8080
```

定义好所有 Service 与对应的 Deployment/RC 描述文件后（总共4个 YAML 文件），我们可以通过 Kubernetes 命令行工具 kubectrl –f create xxx.yaml 提交到集群里，如果一切正常，Kubernetes 会在几分钟内自动完成部署，你会看到相关的资源对象都已经创建成功：

```
-bash-4.2# kubectl get svc
NAME           CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
hpe-java-web   10.254.183.22   nodes         8080/TCP   36m
kubernetes     10.254.0.1      <none>        443/TCP    89d
mysql          10.254.170.22   <none>        3306/TCP   36m
-bash-4.2# kubectl get pods
NAME                             READY     STATUS    RESTARTS   AGE
hpe-java-web-deployement-q8t9k   1/1       Running   0          36m
mysql-deployment-5py34           1/1       Running   0          36m
-bash-4.2# kubectl get rc
NAME                       DESIRED   CURRENT   AGE
hpe-java-web-deployement   1         1         37m
mysql-deployment           1         1         37m</none></none>
```

### 结束语

从上面步骤来看，传统应用迁移改造到 Kubernetes 上还是比较容易的，而借助于 Kubernetes 的优势，即使一个小开发团队，也能在系统架构和运维能力上迅速接近一个大研发团队的水平。

<img src="http://ipad-cms.csdn.net/cms/attachment/201607/577610fe366b2.jpg" alt="图4  基于Kubernetes的PaaS平台架构" title="图4  基于Kubernetes的PaaS平台架构" />

图4  基于 Kubernetes 的 PaaS 平台架构

此外，为了降低 Kubernetes 的应用门槛，我们（惠普中国 CMS 研发团队）开源了一个 Kubernetes 的管理平台 Ku8 eye，项目地址为<https://github.com/bestcloud/ku8eye>，他很适合作为小公司的内部 PaaS 应用管理平台，其功能架构类似图4所示的 Ku8  Manager 企业版，Ku8 eye 采用 Java 开发完成，是目前唯一开源的 Kubernetes 图形化管理系统，也希望更多爱好开源和有能力的同行参与进来，将它打造成为国内最好的云计算领域开源软件。