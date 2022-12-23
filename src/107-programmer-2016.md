## Kubernetes 从部署到运维详解

文/杨乐

>Kubernetes 作为容器集群管理工具，其活跃的社区吸引了广大开发者热情关注，刺激了容器周边生态的快速发展，为众多互联网企业采用容器集群架构升级内部 IT 平台设施，以及构建高效大规模计算体系提供了技术基础，本文详解介绍了 Kubernetes 从部署到运维的全程详解。

Kubernetes 对计算资源进行了更高层次的抽象，通过将容器进行细致的组合，将最终的应用服务交给用户。Kubernetes 在模型建立之初就考虑了容器跨机连接的要求，支持多种网络解决方案，同时在 Service 层次构建集群范围的 SDN 网络。其目的是将服务发现和负载均衡放置到容器可达的范围，这种透明的方式便利了各个服务间的通信，并为微服务架构的实践提供了平台基础。而在 Pod 层次上，作为 Kubernetes 可操作的最小对象，其特征更是对微服务架构的原生支持。

### 架构及部署

Kubernetes 是2014年6月开源的，采用 Go 语言开发，每个组件互相之间使用的是 Master API 的方式，Kubernetes 的架构模式是用 Master-slave 模式，并且支持多种联机网络，支持多种分布式存储架构。

<img src="http://ipad-cms.csdn.net/cms/attachment/201611/581830052ca68.png" alt="图1 Kubernetes架构" title="图1 Kubernetes架构" />

图1 Kubernetes架构

Master 的核心组件是 API server，对外提供 REST API 服务接口。Kubernetes 所有的信息都存储在分布式系统 ETCD 中。Scheduler 是 Kubernetes 的调度器，用于调度集群的主机资源。Controller 用于管理节点注册以及容器的副本个数等控制功能。

在 Node 上的核心组件是 kubelet，它是任务执行者，会跟 apiserver 进行交互，获取资源调度信息。 kubelet 会根据资源和任务的信息和调度状态与 Docker 交互，调用 Docker 的 API，创建、删除与管理容器，而 kube-Proxy 可以根据从 API 获取的信息以及整体的 Pod 架构状态组成虚拟 NAT 网络。

### 快速部署过程

在最新更新的 Kubernetes 1.4版本中，社区开发了专用的部署组件 kubeadm, 用来完成 Kubernetes 集群整体的部署过程。kubeadm 是对以往手动或脚本部署的简化，集成了 manifest 配置、参数设置、认证设置、集群网络部署以及安全证书的获取。需要注意的是 kubeadm 目前默认从 gcr.io 镜像中心获取，若需使用其他镜像源，需要更改源码编译出定制版本。

Ubuntu 16.04环境的使用过程如表1：

<img src="http://ipad-cms.csdn.net/cms/attachment/201611/581830119d48d.jpg" alt="表1 设定部署主机资源" title="表1 设定部署主机资源" />

表1 设定部署主机资源

在所有节点上（包括 master 和 node）部署基础运行环境部署内容包括 docker, kubelet, kubeadm, kubectl Kubernetes-cni

运行如下命令：

```
# curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
# cat <<eof> /etc/apt/sources.list.d/Kubernetes.list
deb http://apt.Kubernetes.io/ Kubernetes-xenial main
EOF
# apt-get update
# apt-get install -y docker.io kubelet kubeadm kubectl Kubernetes-cni</eof>
```

部署Master

在 Master 上运行 Kubeadm init，运行结束后，可获得集群的 token，以及提示在 node 上运行的命令：

```
Kubernetes master initialised successfully!
You can connect any number of nodes by running:
```

添加节点

在 master 上的 kubeadm init 的输出中获取相应的参数并在 Node 上运行：

```
kubeadm join --token <token> <master-ip></master-ip></token>
```

部署网络

Kubernetes 集群需要可以跨主机的网络解决方案，使得位于不同主机的 pod 可以相互通信。目前有多种类似的解决方案，例如 weave network, calico 或者 Canal。当前使用的是 weave net

```
# kubectl apply -f https://git.io/weave-kube
daemonset "weave-net" created
```

关于网络，对于隔离要求较高的场景需求，采用 calico 是比较合适的选择。calico 会帮助容器在主机间搭建纯二层的网络，在每个主机上维护一个路由表，用来获取目标容器所在主机的可达路径，以及本机容器的路有项。利用 iptables 的防火墙机制去做隔离。容器之间跨主机进行交互时，IP 包从容器出发，经过本地路有表选路，通过目标网段所在主机的路有项，到达目标主机，然后在目标主机内，进入路有选路前，先经由 iptables 隔离规则（可设置）进行判断决定是否丢弃或返回，然后再经路有选路到目标容器，最终到达目标容器。整个过程没有任何封包解包的过程，传输效率较高。

隔离规则可以设定在同一用户名下的哪些容器可以被隔离成一组，被隔离的容器间可通信，而与其它容器不可通信。或者设置规则来组成更加丰富的隔离效果。

### 部署组件详解

在 Kubernetes 1.4版本利用 kubeadm 部署过程中，安装的插件包括 kube-discovery、kube-proxy 和 kube-dns，分别负责部署阶段配置的分发、NAT 网络和集群系统的 DNS 服务。值得注意的是 kube-proxy 不再通过 manifest 来运行，是通过插件用 demonset 的方式来部署。

#### kubeadm

在 kubeadm init 运行阶段，首先创建验证 token 以及静态 pod 文件（manifest 的文件），以及运行所需的证书和 kubeconfig。完成这些工作后，kubeadm 会等待 apiserve r的启动并正常运行，以及等待首个节点的接入。

由于 master 节点启动 kubelet 来运行 manifest 文件，并且在 kubelet.conf 中设置了需要连接当前主机为 master，因此此 master 也会作为 node 节点接入，并且 master 作为节点运行也为 kube-discovery 提供运行的必须环境。

```
/usr/bin/kubelet --kubeconfig=/etc/
Kubernetes/kubelet.conf --requirekubeconfig=
true --pod-manifestpath=/
etc/Kubernetes/manifests
--allow-privileged=true --networkplugin=
cni --cni-conf-dir=/etc/cni/
net.d --cni-bin-dir=/opt/cni/bin
--cluster-dns=100.64.0.10 --clusterdomain=
cluster.local --v=4
```

当节点加入并且 apiserver 运行正常后，将标示 master 的角色，kubeadm.alpha.Kubernetes.io/role=master。

#### Kube-discovery

随后在部署运行 kube-discovery 时，在 kube-discovery 所运行的 pod 上设置 node 的亲和性，并将它限制成为必须在 master 上运行的 pod。由于 kube-discovery 的主要功能是证书及 token 等配置的管理与分发，并且后续的节点加入时只需要一个简单的 master ip 信息，因此将 kube-discovery 限制到了 master 节点运行，以此统一服务的入口。

这些通过在 pod 的 annotations 来实现。

```
annotations:
scheduler.alpha.Kubernetes.io/affinity: '{"nodeAffinity":{"requiredDuringSchedulingIgnoredDuringExecution":{"nodeSelectorTerms":[{"matchExpressions":[{"key":"kubeadm.alpha.Kubernetes.io/role","operator":"In","values":["master"]}]}]}}}'
 
```

requiredDuringSchedulingIgnoredDuringExecution 表示在调度过程中需要满足的条件，和后面的 nodeSelectorTerms 在一起类似于 nodeselector。这样设置即可将 pod 限制到 master 主机中。

#### Kube-dns

从 Kubernetes 1.3开始，kube-dns 已经不再使用 etcd+skydns+kube2sky 的方式。而是使用了 DNS 缓存及转发工具 dnsmasq，以及利用 skydns 库和本地内存缓存组合而成的 kube-dns。

Kube-dns 将从Kubernetes master 中监听变动的 service 和 endpoint 信息，并建立从 service ip 到 service name 的域名映射（对于无 service 的，将会建立 pod ip 和相应域名的映射）。Kube-dns 将这些信息存放在本地的内存缓冲中，并监听127.0.0.1:10053提供服务。

Dnsmasq 是简单的域名服务、缓存和转发工具，这里利用它的转发功能将 kube-dns 的 dns 服务转接到外部，参数--server=127.0.0.1:10053。

通过如下命令即可测试，例如 clustcluster-dns 的 ip 为100.64.0.10，在任意节点上运行:

```
ubuntu@i-0mwnrf67:~$ dig @100.64.0.10 Kubernetes.default.svc.cluster.local
; <<>> DiG 9.10.3-P4-Ubuntu <<>> @100.64.0.10 Kubernetes.default.svc.cluster.local
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 59251
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;Kubernetes.default.svc.cluster.local. IN A
;; ANSWER SECTION:
Kubernetes.default.svc.cluster.local. 5 IN A    100.64.0.1
;; Query time: 13 msec
;; SERVER: 100.64.0.10#53(100.64.0.10)
;; WHEN: Tue Oct 25 18:12:04 CST 2016
```

### 运维管理

#### 租户资源管理 Namespace（命名空间）

namespace 是 Kubernetes 用来做租户管理的对象。每个租户独享自己的逻辑空间，包括 replication controller、 pod、 service、 deployment、configmap 等。

常用的查看方式：

```
kubectl get <resource type=""> <resource name=""> --namespace=<namespace></namespace></resource></resource>
```

或查看所有 namespace 的某类资源

```
kubectl get <resource type=""> <resource name=""> --all-namespaces</resource></resource>
```

例如查看所有 pod，并希望看到所部署节点位置

```
kubectl get pod –all-namespaces –o wide
```

查看 namespace 为 test 的 replication controller，以及 labetkubectl get rc –namespace=test –show-labels 配置管理 ConfigMap。

当服务运行在容器中，需要访问外部的变量，或者需要根据环境的不同更改配置文件。比如，DB 以传统方式运行在容器云之外，当服务启动时，需要初始化包含 DB 信息的配置文件。当需要切换 DB 时，就需要更改配置文件，当容器中有服务在运行时，并不推荐登录到容器内进行文件配置更改。

合理的方式是利用 Kubernetes 的配置管理，将配置信息写入到 ConfigMap，并挂载到对应的 pod 中。
 
例如 golang-sample 需要访问配置文件 db.json，内容如下：

```
{
        "dbType": "mysql",
        "host": "192.168.1.22",
        "user": "tenxcloud",
        "password": "password",
        "port": "3306",
        "connectionLimit": 200,
        "connectTimeout": 20000,
        "database": "sample"
    }
```

将 db.json 写入 config.yaml 中：

```
apiVersion: v1
data:
  db.json: |-
    {
        "dbType": "mysql",
        "host": "192.168.1.22",
        "user": "tenxcloud",
        "password": "password",
        "port": "3306",
        "connectionLimit": 200,
        "connectTimeout": 20000,
        "database": "sample"
    }
kind: ConfigMap
metadata:
  name: config-sample
  namespace: sample
```

创建 configmap 对象：

```
kubectl create –f config.yaml
```

在 pod 中添加对应的 volume

```
apiVersion: v1
kind: Pod
metadata:
  labels:
    name: golang-sample
  name: golang-sample
  namespace: sample
spec:
  containers:
  - image: inde.tenxcloud.com/sample/golang-sample:latest
    volumeMounts:
    - mountPath: /usr/src/app/config/db/
      name: configmap-1
  volumes:
  - configMap:
      items:
      - key: db.json
        path: db.json
      name: config-sample
    name: configmap-1
```

pod 创建运行后，服务在 pod 容器内只需要读取固定位置的配置文件，当配置需要改变时，更新 ConfigMap 并重新分发到 pod 内，这样重启容器后，容器内所挂载的配置也会相应更新。当需要 pod 容器同时使用一个 ConfigMap 时，更新 ConfigMap 内容的同时，可以批量更新容器的配置。

#### 主机运维管理

对于运维操作来说，kubectl 是一个很便利的命令行工具，首先可以对各种资源进行操作，比如添加、获取、删除，通过更多命令参数得到指定的信息。

获取资源列表及详细信息的方式可通过 kubectl get 来进行，具体的操作运行 kubectl get --help 即可。

实践使用过程中，对节点的运维操作会影响到应用的使用和资源的调度，比如由于配置升级需要对节点主机进行重启，需要考虑已经运行在其上的容器的状况，用户的希望是对资源池的操作尽量少的影响容器应用，同时资源池的变动和上层的容器服务解藕。

当需要对主机进行维护升级时，首先将节点主机设置成不可调度模式：

```
kubectl cordon［nodeid］
```

然后需要将主机上正在运行的容器驱赶到其它可用节点：

```
kubectl drain ［nodeid］
```

当容器迁移完毕后，运维人员可以对该主机进行操作，配置升级性能参数调优等。当对主机的维护操作完毕后， 再将主机设置成可调度模式：

```
kubectl uncordon [nodeid]
```

这样新创建的容器即可以分配到该主机，可以通过 kubectl patch 对资源对象进行实时修改，比如为 service 增加端口，为 pod 修改容器镜像版本。Annotation 可以帮助用户更好的设置 Kubernetes 自定义插件。用户可以在自建组件中获取资源中对应的 annotation 以此进行操作。通过 kubectl label 可以方便的对资源打标签，比如对 node 打标签，然后容器调度时可指定分配到对应标签的主机。