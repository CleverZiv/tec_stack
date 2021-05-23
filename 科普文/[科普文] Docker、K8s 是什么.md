# [科普文] Docker、K8s 是什么

## 虚拟机和容器

虚拟机属于虚拟化技术，而容器技术，也是虚拟化技术，但它是**轻量级**的虚拟化

那它到底哪里轻量级了呢？

虚拟机就是在当前物理机上安装一些虚拟机管理系统（如VMWare、VirtualBox），虚拟机管理系统应用”硬件虚拟化技术“，可以模拟出操作系统需要的硬件：CPU、内存、IO设备，从而创造出更多个”子电脑“。这些子电脑的特点：

1. 子电脑之间是相互隔离的，互不影响
2. 子电脑都有各自的独立的操作系统，比如可以装 ubuntu、windows等

容器**不需要虚拟出整个操作系统，只需要虚拟出一个小规模的环境**

除此之外，容器在对系统资源的利用上、启动时间上等优于虚拟机，所以现在容器越来越流行。

最后再举个例子，解释下两者的不同：

物理机：

<img src="/Users/lengzefu/Library/Application Support/typora-user-images/image-20210510144027701.png" alt="image-20210510144027701" style="zoom:50%;" />

虚拟机：

<img src="/Users/lengzefu/Library/Application Support/typora-user-images/image-20210510144100516.png" alt="image-20210510144100516" style="zoom:50%;" />

<img src="/Users/lengzefu/Library/Application Support/typora-user-images/image-20210510144119967.png" alt="image-20210510144119967" style="zoom:50%;" />

- 地基相当于物理机的硬件资源
- 多个虚拟机共享一个物理机的硬件资源，但它也需要自己盖一层”地板“，可以认为是操作系统（guest os）
- 容器从1到多的过程，不需要盖一层地板，只需要在同一套房子里加几个”隔板“就可以，非常轻量级

## 所以 Docker 是什么？

Docker 并不是容器本身，它是创建容器的工具，是应用容器引擎。对应于虚拟机中的虚拟机管理系统（如VMWare、VirtualBox）

Docker 有两句口号：

> **“Build, Ship and Run”**（搭建、发送、运行）
>
> **“Build once，Run anywhere”（搭建一次，到处能用）**

举个例子：

我来到一片空地，想建个房子，于是我搬石头、砍木头、画图纸，一顿操作，终于把这个房子盖好了。

<img src="/Users/lengzefu/Library/Application Support/typora-user-images/image-20210510144811243.png" alt="image-20210510144811243" style="zoom:50%;" />

结果，我住了一段时间，想搬到另一片空地去。这时候，按以往的办法，我只能再次搬石头、砍木头、画图纸、盖房子。

但是，跑来一个老巫婆，教会我一种魔法。



这种魔法，可以把我盖好的房子复制一份，做成“镜像”，放在我的背包里。

<img src="/Users/lengzefu/Library/Application Support/typora-user-images/image-20210510144847863.png" alt="image-20210510144847863" style="zoom:50%;" />

等我到了另一片空地，就用这个“镜像”，复制一套房子，摆在那边，拎包入住。

<img src="/Users/lengzefu/Library/Application Support/typora-user-images/image-20210510144909803.png" alt="image-20210510144909803" style="zoom:50%;" />

这个例子中对应了 Docker 中的三大核心概念

- 镜像（Image）
- 容器（Container）
- 仓库（Repository）

那个放在包里的“镜像”，就是**Docker镜像**。而我的背包，就是**Docker仓库**。我在空地上，用魔法造好的房子，就是一个**Docker容器**。

说白了，这个Docker镜像，是一个特殊的文件系统。它除了提供容器运行时所需的程序、库、资源、配置等文件外，还包含了一些为运行时准备的一些配置参数（例如环境变量）。镜像不包含任何动态数据，其内容在构建之后也不会被改变。



也就是说，每次变出房子，房子是一样的，但生活用品之类的，都是不管的。谁住谁负责添置。



每一个镜像可以变出一种房子。那么，我可以有多个镜像呀！



也就是说，我盖了一个欧式别墅，生成了镜像。另一个哥们可能盖了一个中国四合院，也生成了镜像。还有哥们，盖了一个非洲茅草屋，也生成了镜像。。。



这么一来，我们可以交换镜像，你用我的，我用你的，岂不是很爽？

<img src="/Users/lengzefu/Library/Application Support/typora-user-images/image-20210510145127180.png" alt="image-20210510145127180" style="zoom:50%;" />

负责对Docker镜像进行管理的，是**Docker Registry服务**（类似仓库管理员）。最常使用的Registry公开服务，是官方的**Docker Hub**，这也是默认的 Registry，并拥有大量的高质量的官方镜像。

## 那 K8S 又是啥？

就在Docker容器技术被炒得热火朝天之时，大家发现，如果想要将Docker应用于具体的业务实现，是存在困难的——编排、管理和调度等各个方面，都不容易。于是，人们迫切需要一套管理系统，对Docker及容器进行更高级更灵活的管理。

就在这个时候，K8S出现了。

**K8S，就是基于容器的集群管理平台，它的全称，是kubernetes。**Kubernetes 这个单词来自于希腊语，含义是**舵手**或**领航员**。

简单看一下 K8S 的架构，里面有一些名词概念需要了解

一个K8S系统，通常称为一个**K8S集群（Cluster）**。

这个集群主要包括两个部分：

- **一个Master节点（主节点）**
- **一群Node节点（计算节点）**

<img src="/Users/lengzefu/Library/Application Support/typora-user-images/image-20210510145414576.png" alt="image-20210510145414576" style="zoom:50%;" />

一看就明白：Master节点主要还是负责管理和控制。Node节点是工作负载节点，里面是具体的容器。

深入来看这两种节点。

首先是**Master节点**。

<img src="/Users/lengzefu/Library/Application Support/typora-user-images/image-20210510145447497.png" alt="image-20210510145447497" style="zoom:50%;" />

Master节点包括API Server、Scheduler、Controller manager、etcd。

- API Server是整个系统的对外接口，供客户端和其它组件调用，相当于“营业厅”。
- Scheduler负责对集群内部的资源进行调度，相当于“调度室”。
-  Controller manager负责管理控制器，相当于“大总管”。

然后是**Node节点**。

<img src="/Users/lengzefu/Library/Application Support/typora-user-images/image-20210510145530417.png" alt="image-20210510145530417" style="zoom:50%;" />

Node节点包括Docker、kubelet、kube-proxy、Fluentd、kube-dns（可选），还有就是**Pod**。

- Pod：是Kubernetes最基本的操作单元。一个Pod代表着集群中运行的一个进程，它内部封装了一个或多个紧密相关的容器。除了Pod之外，K8S还有一个**Service**的概念，一个Service可以看作一组提供相同服务的Pod的对外访问接口。
- Docker，不用说了，创建容器的。
- Kubelet，主要负责监视指派到它所在Node上的Pod，包括创建、修改、监控、删除等。
- Kube-proxy，主要负责为Pod对象提供代理。
- Fluentd，主要负责日志收集、存储与查询。

至此从虚拟机到容器再到Docker，再到K8S的介绍都完了

## 如何工作的？

<img src="/Users/lengzefu/Library/Application Support/typora-user-images/image-20210510160337044.png" alt="image-20210510160337044" style="zoom:50%;" />

1. 容器需要互相写作，通过 Pod 完成，Pod 可以看成是一个服务实例
2. Ingress（入境权）：对外暴露服务，通过域名进行映射



请求调用流程：

<img src="/Users/lengzefu/Library/Application Support/typora-user-images/image-20210510160941897.png" alt="image-20210510160941897" style="zoom:50%;" />