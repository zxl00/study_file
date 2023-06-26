## k8s组件

![image-20230625153153681](https://gitee.com/root_007/md_file_image/raw/master/202306251531750.png)

### control plane组件

- kube-apiserver

  >  该组件负责公开了 Kubernetes API，负责处理接受请求的工作
  >
  > `kube-apiserver` 设计上考虑了水平扩缩，也就是说，它可通过部署多个实例来进行扩缩。 你可以运行 `kube-apiserver` 的多个实例，并在这些实例之间平衡流量
  >
  > API Server 提供了资源对象的唯一操作入口，其它所有组件都必须通过它提供的 API 来操作资源数据。**只有 API Server 会与 etcd 进行通信，其它模块都必须通过 API Server 访问集群状态**。API Server 作为 Kubernetes 系统的入口，封装了核心对象的增删改查操作。API Server 以 RESTFul 接口方式提供给外部客户端和内部组件调用，API Server 再对相关的资源数据（`全量查询 + 变化监听`）进行操作，以达到实时完成相关的业务功能

- etcd

  > 一致且高度可用的键值存储，用作 Kubernetes 的所有集群数据的后台数据库。
  >
  > 如果你的 Kubernetes 集群使用 etcd 作为其后台数据库， 请确保你针对这些数据有一份 [备份](https://v1-25.docs.kubernetes.io/zh-cn/docs/tasks/administer-cluster/configure-upgrade-etcd/#backing-up-an-etcd-cluster)计划。
  >
  > 你可以在官方[文档](https://etcd.io/docs/)中找到有关 etcd 的深入知识。

- kube-scheduler

  > -  负责监视新创建的、未指定运行[节点（node）](https://v1-25.docs.kubernetes.io/zh-cn/docs/concepts/architecture/nodes/)的 [Pods](https://v1-25.docs.kubernetes.io/zh-cn/docs/concepts/workloads/pods/)， 并选择节点来让 Pod 在上面运行。
  >
  > - 调度决策考虑的因素包括单个 Pod 及 Pods 集合的资源需求、软硬件及策略约束、 亲和性及反亲和性规范、数据位置、工作负载间的干扰及最后时限。
  >
  > - 分析当前 Kubernetes 集群中所有 Node 节点的资源 (包括内存、CPU 等) 负载情况，然后依据资源占用情况分发新建的 Pod 到 Kubernetes 集群中可用的节点
  >
  > - 监测 Kubernetes 集群中未分发和已分发的所有运行的 Pod
  >
  > - 监测 Node 节点信息，由于会频繁查找 Node 节点，所以 Scheduler 同时会缓存一份最新的信息在本地
  >
  >   分发 Pod 到指定的 Node 节点后，会把 Pod 相关的 Binding 信息写回 API Server，以方便其它组件使用
  >
  > kube-scheduler 获取节点信息的主要来源之一就是 kubelet 反馈给 Kubernetes API Server 的节点状态信息。具体来说，kubelet 是运行在每个节点上的代理进程，负责管理该节点上的容器和 Pod，并将节点状态报告给 Kubernetes API Server。
  >
  > kubelet 会定期向 Kubernetes API Server 发送 Node 资源对象，其中包括了有关节点的详细信息，例如节点名称、IP 地址、标签、资源使用情况等

- kube-controller-manager

  > 负责运行[控制器](https://v1-25.docs.kubernetes.io/zh-cn/docs/concepts/architecture/controller/)进程
  >
  > 从逻辑上讲， 每个[控制器](https://v1-25.docs.kubernetes.io/zh-cn/docs/concepts/architecture/controller/)都是一个单独的进程， 但是为了降低复杂性，它们都被编译到同一个可执行文件，并在同一个进程中运行。
  >
  > 这些控制器包括(但不仅限于)：
  >
  > - 节点控制器（Node Controller）：负责在节点出现故障时进行通知和响应，Kubelet 在启动时会通过 API Server 注册自身的节点信息，并定时向 API Server 汇报状态信息。API Server 在接收到信息后将信息更新到 Etcd 中。Node Controller 通过 API Server 实时获取 Node 的相关信息，实现管理和监控集群中的各个 Node 节点的相关控制功能。
  > - 任务控制器（Job Controller）：监测代表一次性任务的 Job 对象，然后创建 Pods 来运行这些任务直至完成
  > - 端点分片控制器（EndpointSlice controller）：填充端点分片（EndpointSlice）对象（以提供 Service 和 Pod 之间的链接）。
  > - 服务账号控制器（ServiceAccount controller）：为新的命名空间创建默认的服务账号（ServiceAccount）。以保证名为 default 的 ServiceAccount 在每个命名空间中存在。
  > - 副本控制器(Replication Controller) ：主要是定期关联 Replication Controller (RC) 和 Pod，以保证集群中一个 RC (一种资源对象) 所关联的 Pod 副本数始终保持为与预设值一致。
  > - 资源配额控制器(ResourceQuota Controller) : 资源配额管理控制器用于确保指定的资源对象在任何时候都不会超量占用系统上物理资源。
  > - 名称空间控制器(Namespace Controller) : 用户通过 API Server 可以创建新的 Namespace 并保存在 Etcd 中，Namespace Controller 定时通过 API Server 读取这些 Namespace 信息来操作 Namespace。比如：Namespace 被 API 标记为优雅删除，则将该 Namespace 状态设置为 Terminating 并保存到 Etcd 中。同时 Namespace Controller 删除该 Namespace 下的 ServiceAccount、Deployment、Pod 等资源对象。
  > - 令牌控制器(Token Controller) : 令牌控制器作为 Controller Manager 的一部分，主要用作：监听 serviceAccount 的创建和删除动作以及监听 secret 的添加、删除动作。
  > - 服务控制器(Service Controller) : 服务控制器主要用作监听 Service 的变化。比如：创建的是一个 LoadBalancer 类型的 Service，Service Controller 则要确保外部的云平台上对该 Service 对应的 LoadBalancer 实例被创建、删除以及相应的路由转发表被更新。
  > - 端点控制器(Endpoint Controller) : Endpoints 表示了一个 Service 对应的所有 Pod 副本的访问地址，而 Endpoints Controller 是负责生成和维护所有 Endpoints 对象的控制器。Endpoint Controller 负责监听 Service 和对应的 Pod 副本的变化。定期关联 Service 和 Pod (关联信息由 Endpoint 对象维护)，以保证 Service 到 Pod 的映射总是最新的。
  >
  > Endpoint Controller 和 EndpointSlice Controller 之间的关系可以简单地概括为：
  >
  > 1. Endpoint Controller 监视 Service 和 Pod 并创建 Endpoints。
  > 2. EndpointSlice Controller 监视 Service 和 Pod 并创建 EndpointSlices。
  >
  > 需要注意的是，EndpointSlices 资源类型的引入使得 Endpoint 的管理更加高效和可扩展。由于 Endpoints 资源类型只能容纳少量的网络端点，因此在大规模集群中管理 Endpoint 可能会面临一些挑战。而通过使用 EndpointSlice，Kubernetes 可以更好地处理大规模集群中的网络端点，并提供更快速和可靠的服务发现。

- cloud-controller-manager

  > `cloud-controller-manager` 仅运行特定于云平台的控制器。 因此如果你在自己的环境中运行 Kubernetes，或者在本地计算机中运行学习环境， 所部署的集群不需要有云控制器管理器。

### node组件

- kubelet

  > - `kubelet` 会在集群中每个[节点（node）](https://v1-25.docs.kubernetes.io/zh-cn/docs/concepts/architecture/nodes/)上运行。 它保证[容器（containers）](https://v1-25.docs.kubernetes.io/zh-cn/docs/concepts/overview/what-is-kubernetes/#why-containers)都运行在 [Pod](https://v1-25.docs.kubernetes.io/zh-cn/docs/concepts/workloads/pods/) 中。
  >
  > - kubelet 接收一组通过各类机制提供给它的 PodSpecs， 确保这些 PodSpecs 中描述的容器处于运行状态且健康。 kubelet 不会管理不是由 Kubernetes 创建的容器。
  >
  > - kubelet 的另外一个重要功能就是调用网络插件（`CNI`）和存储插件（`CSI`）为容器配置网络和存储功能，同样的 kubelet 也是把这两个重要功能通过接口暴露给外部了，所以如果我们想要实现自己的网络插件，只需要使用 CNI 就可以很方便的对接到 Kubernetes 集群当中去。
  > - 定时上报本地 Node 的状态信息给 API Server
  > - kubelet 是 Master 和 Node 之间的桥梁，接收 API Server 分配给它的任务并执行
  > - kubelet 通过 API Server 间接与 Etcd 集群交互来读取集群配置信息
  > - kubelet 在 Node 上做的主要工作具体如下：
  >   1. 设置容器的环境变量、给容器绑定 Volume、给容器绑定 Port、根据指定的 Pod 运行一个单一容器、给指定的 Pod 创建 Network 容器
  >   2. 同步 Pod 的状态
  >   3. 在容器中运行命令、杀死容器、删除 Pod 的所有容器

- kube-proxy

  > - [kube-proxy](https://v1-25.docs.kubernetes.io/zh-cn/docs/reference/command-line-tools-reference/kube-proxy/) 是集群中每个[节点（node）](https://v1-25.docs.kubernetes.io/zh-cn/docs/concepts/architecture/nodes/)上所运行的网络代理， 实现 Kubernetes [服务（Service）](https://v1-25.docs.kubernetes.io/zh-cn/docs/concepts/services-networking/service/) 概念的一部分。
  >
  > - kube-proxy 维护节点上的一些网络规则， 这些网络规则会允许从集群内部或外部的网络会话与 Pod 进行网络通信。
  >
  > - 如果操作系统提供了可用的数据包过滤层，则 kube-proxy 会通过它来实现网络规则。 否则，kube-proxy 仅做流量转发。
  > - 每创建一个 Service，kube-proxy 就会从 API Server 获取 Services 和 Endpoints 的配置信息，然后根据其配置信息在 Node 上启动一个 Proxy 的进程并监听相应的服务端口。
  > - 当接收到外部请求时，kube-proxy 会根据 Load Balancer 将请求分发到后端正确的容器处理
  > - kube-proxy 不但解决了同一宿主机相同服务端口冲突的问题，还提供了 Service 转发服务端口对外提供服务的能力。
  > - kube-proxy 后端使用`随机、轮循`等负载均衡算法进行调度。

- 容器运行时(Container Runtime)

  > 容器运行环境是负责运行容器的软件。
  >
  > Kubernetes 支持许多容器运行环境，例如 [containerd](https://containerd.io/docs/)、 [CRI-O](https://cri-o.io/#what-is-cri-o) 以及 [Kubernetes CRI (容器运行环境接口)](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-node/container-runtime-interface.md) 的其他任何实现

### 插件

插件使用 Kubernetes 资源（[DaemonSet](https://v1-25.docs.kubernetes.io/zh-cn/docs/concepts/workloads/controllers/daemonset/)、 [Deployment](https://v1-25.docs.kubernetes.io/zh-cn/docs/concepts/workloads/controllers/deployment/) 等）实现集群功能。 因为这些插件提供集群级别的功能，插件中命名空间域的资源属于 `kube-system` 命名空间。

- 服务发现

  > [CoreDNS](https://coredns.io/) 是一种灵活的，可扩展的 DNS 服务器，可以 [安装](https://github.com/coredns/deployment/tree/master/kubernetes)为集群内的 Pod 提供 DNS 服务

- 联网和网络策略

  > - [Calico](https://docs.projectcalico.org/latest/introduction/) 是一个联网和网络策略供应商。 Calico 支持一套灵活的网络选项，因此你可以根据自己的情况选择最有效的选项，包括非覆盖和覆盖网络，带或不带 BGP。 Calico 使用相同的引擎为主机、Pod 和（如果使用 Istio 和 Envoy）应用程序在服务网格层执行网络策略。
  > - [Flannel](https://github.com/flannel-io/flannel#deploying-flannel-manually) 是一个可以用于 Kubernetes 的 overlay 网络提供者。
  >
  > 更多插件可参考： https://v1-25.docs.kubernetes.io/zh-cn/docs/concepts/cluster-administration/addons/

## k8s架构图

![kubernetes arch](https://gitee.com/root_007/md_file_image/raw/master/202306251547121.png)

![kubernetes high level component archtecture](https://gitee.com/root_007/md_file_image/raw/master/202306251549627.png)



## 集群注意事项

> - 每个节点(node)的pod数量不超过110
> - 节点数不超过5000
> - pod总数不超过150000
> - 容器总数不超过300000

