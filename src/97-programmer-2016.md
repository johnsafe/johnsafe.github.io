## 关于 Docker 你不知道的事——Docker Machine

文/晏东

>除了众所周知的 Docker Engine，Docker 还有一个非常重要的开源项目 Docker Machine。本文将介绍 Docker Machine 的相关信息。

如果你不是 Docker 的狂热爱好者，你眼中的 Docker 很可能指的是 Docker Engine，即 Docker 的 daemon 和 client。Docker Engine 是 C/S 架构，都拥有相同的可执行程序 docker。通过不同方式会以不同的角色运行：

```
Usage: docker [OPTIONS] COMMAND [arg...]
      docker daemon [ --help | ... ]
```

docker daemon 就是服务器方式运行，docker COMMAND 就是我们通常用的 docker client。但是 Docker Engine 只是 Docker 众多开源项目的一种，今天我们要介绍另一个非常重要的开源项目 Docker Machine。

### 什么是 Docker Machine

如果你想在 Windows 或者 Mac 机器上运行 Docker 怎么办？由于 Docker 需要借助 Linux 内核的 CGroup 和 Namespace，要在这两个系统上运行，就需要借助虚拟机。Docker Machine 就是帮助你构建拥有 Docker 运行环境的虚拟机，并能够远程的管理虚拟机及其中的容器。

很多人也许会觉得奇怪，容器不是更轻量级吗，为什么不直接使用容器，还要使用虚拟机。主要有下面几个考虑：

跨平台的支持，在非 Linux 下环境，仍然需要虚拟机来运行，如图1。

<img src="http://ipad-cms.csdn.net/cms/attachment/201604/56fb8d06c23ab.jpg" alt="图1  跨平台支持" title="图1  跨平台支持" />

图1  跨平台支持

希望管理远端的系统，并且希望在一个地方总控。比如我希望快速部署 Docker 在云主机上，快速扩容虚拟结点，这些都可以借助 Docker Machine，如图2。

<img src="http://ipad-cms.csdn.net/cms/attachment/201604/56fb8d1063291.jpg" alt="图2  管理远端系统" title="图2  管理远端系统" />

图2  管理远端系统

那么 Docker Machine 和 Docker Engine 有什么区别呢？后者是一个运行 Docker 的程序，它可以运行在虚拟机也可以运行在物理机上。前者是将 Docker Engine 安装在虚拟机上，并进行了打包，使其可以快速生成和方便管理。如图3。

<img src="http://ipad-cms.csdn.net/cms/attachment/201604/56fb8d1ce3449.jpg" alt="图3  Docker Machine和 Docker Engine 示意图" title="图3  Docker Machine和 Docker Engine 示意图" />

图3  Docker Machine和 Docker Engine 示意图

Docker Machine 的执行步骤是：

Docker Machine 命令发起调用

内部转化成 Docker 命令行调用；

Docker Client 通过 REST API 向远端发起调用；

远端的 Daemon 收到请求，并返回相应数据；

### Docker Machine 支持哪些环境

Docker Machine 是一个开源项目，地址是：https://github.com/docker/machine

它主要是通过 Python 来开发的。在它的目录里面有一个 driver 目录，如图4。

<img src="http://ipad-cms.csdn.net/cms/attachment/201604/56fb8d257ce6f.jpg" alt="图4  Docker Machine支持的虚拟主机类型" title="图4  Docker Machine支持的虚拟主机类型" />

图4  Docker Machine 支持的虚拟主机类型

看见了吗，这就是 Docker Machine 支持的所有虚拟主机类型，我们不难看出从基本的 Virtualbox 到 Azure，再到 AWS，都在其支持范围内。换句话说，只要你根据 driver 的要求填入参数，就可以通过一个命令创建云主机，一个命令动态扩展主机。Docker Machine 会给 VM 安装一个操作系统，默认的本地系统格式 Boot2Docker，远端的系统是 Ubuntu 12.04。

### 国内的 Driver

通过 Docker Machine 可以很好地将虚拟机和容器结合起来，因此如果你已经拥有内部虚拟机环境，或者拥有云主机可以快速使用 Docker。但是国内的公有云厂商反应速度都比较滞后，目前还不是 docker-machine 天然支持的驱动。不过，阿里云和 UCloud 已经加入了对 docker-machine 的支持，我们可以在第三方的插件里面找到。