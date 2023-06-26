#### 了解k8s

1. k8s是一个全新的基于容器技术的分布式架构领先方案，是容器云的优秀平台选型方案，已成为新一代的基于容器技术的Paas平台的重要底层框架，也是云原生技术生态圈的核心， Service Mesh、ServerLess等新一代分部署架构及技术纷纷基于k8s实现。

2. 利用k8s提供的解决方案，我们不仅节省开发成本，还可以将更多的精力集中与业务本身，而且由于k8s提供了强大的自动化机制，所以系统后期的运维难度和运维成本也大幅度下降

3. k8s是一个完备的分布式系统支撑平台。k8s具有完备的集群管理能力，包括多层次的安全防护和准入机制、多租户应用支撑能力、透明的服务注册和服务发现机制、内建的智能负载均衡器、强大的故障发现和自我修复能力、服务滚动升级和在线扩容能力、可扩展的资源自动调度机制、以及多粒度的资源配额管理能力。

4. 在k8s中，service是分布式集群架构的核心。一个service对象拥有如下关键特征

   - 拥有唯一指定名称

   - 拥有一个虚拟IP地址和端口号

   - 能够提供某种远程服务能力

   - 能够将客户端对服务的访问请求转发到一组容器应用上

     - service的服务进程通常基于Socket通信方式对外提供服务，如redis、mysql或者是实现了某个具体业务特定TCP Server进程，虽然一个Service通常由多个相关服务进程提供服务，每个服务进程都有一个独立的Endpoint(IP+Port)访问点，但k8s能够让我们通过Service(clusterIp+Service Port)连接指定的服务。这个Service一旦被创建就不再变化。这意味着我们再也不用为k8s集群中应用服务进程IP地址变化而头疼了。
     - 容器提供了强大的隔离能力，所以我们有必要把为service提供服务的这组进程放入容器中进行隔离，为此k8s设计了pod对象，将每个服务进程都包装到相应的pod中，使其成为pod中运行的一个容器，为了建立service和pod间的关联关系，k8s首先给每个pod贴上一个标签(label)，然后给相应的service定义标签选择器，这样一来就解决了service和pod的关联问题

     

5. 简单介绍一下pod

   - pod运行在一个被成为节点(node)的环境中，这个节点可以是物理机，也可以是私有云或者公有云中的一个虚拟机，在一个节点上能够运行多个pod
   - 每个pod都运行这一个特殊的被称为Pause的容器(根容器、祖容器等多种称呼)，其他容器为业务容器，这些业务容器共享Pause容器的网络栈和volume挂载卷，因此他们之间的通信和数据交换更为高效。我们可以使用这一特性，将一组密切相关的服务进程放入到同一个pod中。**需要注意的是**，并不是每个pod和它里面运行的容器都被映射到service上，只有提供服务的那组pod才会被映射为一个服务。

6. 集群方面

   - k8s将集群中的机器划分为Master和Node。在master上运行着集群管理相关的一些进程：`kube-apiserver  kube-controller-manager kube-scheduler`  这些进程实现了整个集群的资源管理、pod调度、弹性伸缩、安全控制、系统监控和纠错等管理功能，并且都是自动完成的
   - Node作为集群中的工作节点，其上运行着真正的应用程序。在node上，k8s管理的最小运行单元是pod，在node上运行着k8s的`kbuelet kube-proxy`服务进程，这些服务进程负责pod的创建、启动、监控、重启、销毁以及实现软件模式负载均衡。



#### 简单入门

1. 从一个简单的例子说明一下

   ```yaml
   apiVersion: apps/v1 # API版本
   kind: Deployment # 副本控制器
   metadata:
     labels: # 标签
       app: mysql
     name: mysql # 对象名称，全局唯一
   spec:
     replicas: 1 # 预期副本数
     selector:
       matchLabels:
         app: mysql
     template: # pod模板
       metadata:
         labels:
           app: mysql
       spec:
         containers: # 定义容器
         - image: mysql:5.7
           name: mysql
           ports:
           - containerPort: 3306 # 容器应用监听端口号
           env: # 注入容器内的环境变量
           - name: MYSQL_ROOT_PASSWORD
           value: "123456"
   ```

   - 以上`yaml`文件中的kind属性，用来表明此资源对象的类型，spec部分是Deployment的相关属性定义，`spec.selector`是`Dp`的pod选择器，符合条件的pod实例受到该`dp`的管理，确保集群中有且仅有一个`replics`个pod运行，当集群中运行的pod数量少于`rc`时，`dp`控制器会根据在`spec.teplate`部分定义的pod模板生成新的pod实例。`spec.template.metadata.labels`指定了该pod的标签，labels必须匹配之前的`spec.selector`.

   - 当我们开始创建该资源时，k8s根据mysql这个dp的定义自动创建pod，由于pod的调度和创建需要花费一些时间，比如需要调度到那个节点，而且下载pod所需镜像也需要一定的时间，所以一开始pod的状态为Pending。在pod成功创建后，其状态为Running

   - 当我们去其调度到的node执行`docker ps | grep mysql`时，可以看到有一个pause容器的产生。

   - 然后我们在为`dp`资源的pod创建一个service

     ```yaml
     apiVersion: apps/v1
     kind: Service # 表明时service
     metadata:
       name: mysql # service的全局唯一名称
     spec:
       ports:
         - port: 3306 # service提供服务的端口号
       selector: # service对应的pod拥有这里定义的标签
         app: mysql
     ```



#### k8s的基本概念和术语

1. 资源对象概述

   - k8s中的基本概念和术语大多是围绕资源对象来说的，资源对象总体上分为两大类
     - 某种资源的对象：例如节点（node）、pod、服务（service）存储卷（volume）
     - 与资源对象相关的事物与动作：例如标签（label）、注解（annotation）、名称空间（namespace）、deployment、hpa、pvc
   - 资源对象一般包括几个通用属性：版本、类别（kind）、名称、标签、注解
   - 资源对象的名称要唯一
   - 资源对象的标签是很重要的数据，也是k8s的一大设计特性
   - 注解可以理解为一种特殊的标签，不过更多的是与程序挂钩、通常用于实现资源对象属性的自定义扩展。

2. 集群类

   - 集群表示一个由master和node组成的k8s集群

   - master：集群的控制节点。运行着关键进程 

     - kube-apiserver，提供HTTP RESTful API接口的主要服务，是k8s里对所有资源进行crud等操作的唯一入口，也是集群控制的入口进程
     - kube-controller-manager，k8s里所有资源对象的控制中心，，可以理解为资源对象的 “大总管”
     - kube-scheduler，负载资源调度(pod调度)的进程

   - node：工作负载节点，每个node都会被master分配一些工作负载(docker容器)，当某个node宕机时，其上的工作负载会被master自动转移到其他node节点上，在node上运行着

     - kubelet：负责pod对应容器的创建、启停等任务，同时与master密切协作，实现集群的管理的基本功能
     - kube-proxy：实现k8s service的通信与负载均衡机制的服务
     - 容器运行时(如docker)：负责本机的容器创建和管理

   - 在默认的情况下，kubelet会向master注册自己，这也是k8s推荐的node管理方式，一旦node被纳入了集群管理范畴，kubelet进程就会定时向master汇报自身的情报，例如：操作系统，主机cpu内存，以及该node运行那些pod，这样master就可以获知每个node的资源使用情况，并实现高效均衡的资源调度策略，而某个node在超过指定时间不上报信息时，会被master判定为"失联"，该node的状态被标记为不可用(Not Ready)，master随后会触发"工作负载大转移"的自动流程

   - 我们查看一下ndoe的信息 `kubectl describe node k8s-node201`

     ```shell
     [root@k8s-master ~]# kubectl describe node k8s-node201 
     Name:               k8s-node201
     Roles:              <none>
     Labels:             beta.kubernetes.io/arch=amd64
                         beta.kubernetes.io/os=linux
                         kubernetes.io/arch=amd64
                         kubernetes.io/hostname=k8s-node201
                         kubernetes.io/os=linux
     Annotations:        kubeadm.alpha.kubernetes.io/cri-socket: /var/run/dockershim.sock
                         node.alpha.kubernetes.io/ttl: 0
                         projectcalico.org/IPv4Address: 192.168.70.201/24
                         projectcalico.org/IPv4IPIPTunnelAddr: 10.244.146.128
                         volumes.kubernetes.io/controller-managed-attach-detach: true
     CreationTimestamp:  Wed, 21 Sep 2022 11:08:15 +0800
     Taints:             <none>
     Unschedulable:      false
     Lease:
       HolderIdentity:  k8s-node201
       AcquireTime:     <unset>
       RenewTime:       Fri, 25 Nov 2022 20:20:02 +0800
     Conditions:
       Type                 Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
       ----                 ------  -----------------                 ------------------                ------                       -------
       NetworkUnavailable   False   Wed, 21 Sep 2022 11:10:48 +0800   Wed, 21 Sep 2022 11:10:48 +0800   CalicoIsUp                   Calico is running on this node
       MemoryPressure       False   Fri, 25 Nov 2022 20:18:44 +0800   Wed, 21 Sep 2022 11:08:15 +0800   KubeletHasSufficientMemory   kubelet has sufficient memory available
       DiskPressure         False   Fri, 25 Nov 2022 20:18:44 +0800   Wed, 21 Sep 2022 11:08:15 +0800   KubeletHasNoDiskPressure     kubelet has no disk pressure
       PIDPressure          False   Fri, 25 Nov 2022 20:18:44 +0800   Wed, 21 Sep 2022 11:08:15 +0800   KubeletHasSufficientPID      kubelet has sufficient PID available
       Ready                True    Fri, 25 Nov 2022 20:18:44 +0800   Wed, 21 Sep 2022 11:09:05 +0800   KubeletReady                 kubelet is posting ready status
     Addresses:
       InternalIP:  192.168.70.201
       Hostname:    k8s-node201
     Capacity:
       cpu:                4
       ephemeral-storage:  51175Mi
       hugepages-1Gi:      0
       hugepages-2Mi:      0
       memory:             8008984Ki
       pods:               110
     Allocatable:
       cpu:                4
       ephemeral-storage:  48294789041
       hugepages-1Gi:      0
       hugepages-2Mi:      0
       memory:             7906584Ki
       pods:               110
     System Info:
       Machine ID:                 3f0a62ea080741aaaede1dad8e89618c
       System UUID:                EA784D56-17AC-6DAF-DA22-FACC86275D2E
       Boot ID:                    aa4436f7-d6ed-4cc2-b1e1-9125ff1d5b6c
       Kernel Version:             3.10.0-1160.el7.x86_64
       OS Image:                   CentOS Linux 7 (Core)
       Operating System:           linux
       Architecture:               amd64
       Container Runtime Version:  docker://20.10.7
       Kubelet Version:            v1.20.15
       Kube-Proxy Version:         v1.20.15
     PodCIDR:                      10.244.3.0/24
     PodCIDRs:                     10.244.3.0/24
     Non-terminated Pods:          (5 in total)
       Namespace                   Name                                   CPU Requests  CPU Limits  Memory Requests  Memory Limits  AGE
       ---------                   ----                                   ------------  ----------  ---------------  -------------  ---
       kube-system                 calico-node-dm5rf                      250m (6%)     0 (0%)      0 (0%)           0 (0%)         65d
       kube-system                 kube-proxy-9g49p                       0 (0%)        0 (0%)      0 (0%)           0 (0%)         65d
       monitoring                  kube-state-metrics-7749b7b647-cjptm    40m (1%)      160m (4%)   230Mi (2%)       330Mi (4%)     19d
       monitoring                  node-exporter-9tj5z                    112m (2%)     270m (6%)   200Mi (2%)       220Mi (2%)     65d
       nginx-sample-env-dev        nginx-6f8bbbd5ff-xfr5s                 50m (1%)      50m (1%)    50Mi (0%)        50Mi (0%)      21d
     Allocated resources:
       (Total limits may be over 100 percent, i.e., overcommitted.)
       Resource           Requests    Limits
       --------           --------    ------
       cpu                452m (11%)  480m (12%)
       memory             480Mi (6%)  600Mi (7%)
       ephemeral-storage  0 (0%)      0 (0%)
       hugepages-1Gi      0 (0%)      0 (0%)
       hugepages-2Mi      0 (0%)      0 (0%)
     Events:              <none>
     
     ```

     - node的基本信息：名称、标签、创建时间等
     - node当前的运行状态：node启动后会做一系列的自检工作，比如磁盘空间是否不足(DiskPressure)、内存是否不足(MemoryPressure)、网络是否正常(Networkunavailable)、PID资源是否充足(PIDPressure)，在一切正常时才设置node为Ready状态
     - node的主机地址与主机名
     - mode上的资源数量：描述node可用的系统资源 cpu、内存、最大可调度pod数量等
     - node可分配的资源量：描述node当前可用于分配的资源量
     - 主机信息：主机ID、系统UUID、Kernel版本等
     - 当前运行的pod列表概要信息
     - 已分配的资源使用概要信息
     - node相关的event信息

   - 如果一个node存在问题，比如安全隐患等。我们可以给这个node打上一个特殊的标签---污点(Taint)，避免新的容器被调度进来，如果某些pod可以短期容忍这个标签(可以调度上去)，可以使用`Toleration`来对某种污点进行容忍。

   - 一个集群可以创建很多个名称空间(namespace),属于不同的ns的资源对象逻辑上相互隔离，用户创建资源不指定ns，默认为default。

3. 应用类

   - service讲解

     - 一般来说Service指的是无状态服务，通常由多个程序副本提供服务，特殊情况下也可以是有状态的单实例服务
     - k8s里的service拥有一个全局唯一的虚拟IP地址ClusterIP，service一旦被创建，k8s就会自动为它分配一个可用的ClusterIP地址，而且在service的整个生命周期中，它的clusterip地址都不会改变，客户端可以通过虚拟ip地址+服务的端口直接访问该服务，在通过部署k8s集群的DNS服务，就可以实现service Name到clusterip地址的DNS映射功能。

   - pod讲解

     - pod示意图

       ![image-20221125204457053](C:\Users\THINK\AppData\Roaming\Typora\typora-user-images\image-20221125204457053.png)

       - 为多进程之间的协作提供一个抽象模型，使用pod作为基本的调度、复制等管理工作的最小单位，让多个应用进程呢能一起有效的调度和伸缩

       - pod里的多个业务容器共享pause容器的ip、共享pause容器挂接的volume，这样既简化了密切关联的业务容器之间的通信问题，也很好的解决了他们之间的文件共享问题

       - k8s为每个pod都分配了唯一的ip地址，成为pod ip，一个pod里有多个容器共享pod ip，k8s要求底层网络支持集群内任意两个pod之间的tcp/ip直接通信，这通常采用二层网络技术实现(flannel、openvswitch...)，因此我们需要牢记，**在k8s里，一个pod里的容器与另外主机上的pod容器能够直接通信**

       - pod分类

         - 普通pod：我们常见的pod，一旦被创建，就会放入etcd中存储，随后被k8s调度到某个具体的node上绑定，该pod就被对应的node上的kubelt进程实例化为一组相关的Docker容器并启动，当然我们pod中的某个容器down了，k8s会自动检测到这个问题并且重启，如果node节点down了，就会迁移pod
         - 静态pod：不被k8s中的etcd存储，而是存放在node节点上的一个文件中，并且只能在该node上启动。

       - pod的yaml简要说明

         ```yaml
         apiVersion: v1
         kind: Pod
         metadata:
           name: myweb
           labels:
             name: myweb
         spec:
           containers:
           - name: myweb
             image: xxxx
             ports:
             - containerPort: 8080
         ```

         - 创建后，会生成一个ip，pod ip加端口，就形成了一个新的概念--Endpoint，代表此pod里的一个服务进程的对外通信地址，一个pod也可以具有多个ep(Endpoint)

       - 标签与选择器

         - 一个资源对象可以定义任意数量的Label，通一个label也可以被添加到任意数量的资源对象上，label通常在资源定义时确定，**建议不要资源对象创建后，在手动添加标签**
         - 常用的标签如下
           - 版本标签：release
           - 环境标签：environment
           - 架构标签：tier
           - 分区标签：partition
           - 质量管控标签：track
         - 给某个资源对象定义一个label，就相当于给它打了一个标签，随后我们可以通过label selector(标签选择器)查询和筛选拥有某些标签的资源对象。

       - pod与deployment

         - 前面经过service是无状态的服务，可以由多个pod副本实例提供服务，如果我们一个个的去创建pod这显然就太麻烦了，也不符合k8s设计的理念

         - pod模板之deployment(该yaml来自上面的简单入门章节)

           ```yaml
           apiVersion: apps/v1 # API版本
           kind: Deployment # 副本控制器
           metadata:
             labels: # 标签
               app: mysql
             name: mysql # 对象名称，全局唯一
           spec:
             replicas: 1 # 预期副本数
             selector:
               matchLabels:
                 app: mysql
             template: # pod模板
               metadata:
                 labels:
                   app: mysql
               spec:
                 containers: # 定义容器
                 - image: mysql:5.7
                   name: mysql
                   ports:
                   - containerPort: 3306 # 容器应用监听端口号
                   env: # 注入容器内的环境变量
                   - name: MYSQL_ROOT_PASSWORD
                   value: "123456"
           ```

           1. 当我们创建该资源时，不仅仅会创建pod，还会创建一个ReplicaSet(rs)，rs负责replicas里的副本数量的控制

       - service的clusterip

         - k8s内部在每个Node上都运行了一套全局的虚拟负载均衡器，自动注入并自动实时更新集群中所有service的路由表，通过iptables、ipvs机制，把对应的service的请求转发到其后端对应的pod实例上，并在内部实现服务的负载均衡与会话保持机制。当然了k8s还为service实现了clusterip的机制，该service一旦被创建，clusterip地址就不会发生改变了，这样一来，每个服务就变成了具备唯一ip地址通信节点，远程服务之间通信就变成了普通的tcp网络通信

         - 为什么说clusterip为虚拟ip呢?

           - clusterip地址仅仅作用于k8s service这个对象，并由k8s管理和分配ip地址(来源于clusterip地址池)，与node和maste所在的网络完全无关
           - 因为没有一个"实体网络对象"来响应，所以clusterip地址无法被ping通，clusterip地址只能与service port组成一个具体的访问端点单独的cluster ip不具备tcp/ip通信基础
           - cluster ip属于k8s集群这个封闭的空间，集群外的节点要访问这个通信端口需要额外的工作。

         - **特殊的service**

           - Headless Service(无头服务)，它与普通是service的关键区别在于它没有cluster ip地址，如果解析headless service的DNS域名映射，则返回的是该service对应pod的全部ep列表，这意味着客户端是直接与后端的pod建立tcp/ip连接通信的，**没有通过虚拟ip地址进行转发，因此通信更为高效**

         - 多个Endpoint的情况下如何定义svc呢？

           ```yaml
           apiVersion: apps/v1
           kind: Service
           metadata:
             name: tomcat-service
           spec:
             ports:
             - port: 8080
               name: service-port
             - port: 8005
               name: shutdown-port
             selector:
               tier: fronted
           ```

           - 如果存在多个ep的情况下，我们需要定义一个名称来进行区分 `spec.ports.name`

       - serice的外网访问问题

         - k8s的三种ip
           - Node IP： node节点的ip地址
           - Pod IP： pod的ip地址
           - Service IP： service的ip地址
           - 说明：
             - node ip是k8s集群中每个节点的物理网卡的ip地址，是一个真实存在的物理网络，所有属于这个网络的服务器都能直接通信，不管是否在k8s集群的内外，**这也就表明了，k8s集群之外的节点访问k8s内部某个节点或者tcp/ip时，都必须通过node ip通信**
             - pod ip是每个pod的ip地址，在使用docker作为容器支撑引擎的情况下，它是docker engine根据docker0网桥的ip地址段进行分配的，通常是一个虚拟二层网络，前面讲过，k8s要求不同node上的pod都能互相通信，所以k8s中一个pod里的容器方位另外一个pod里容器时，就是通过pod ip所在的虚拟二层网络进行通信的，而真实的tcp/ip流量是通过node ip所在的物理网卡流出的
             - cluster ip属于k8s内部的地址，外部如果想要访问，这时就引出了一个NodePort的概念，nodeport也是解决集群外的应用访问集群内部服务的直接、有效的常见做法。
               - 简单说一下nodeport，这里先不做过多的解释，nodeport的实现方式：在k8s集群的每个node上都为需要外部访问的service开启一个对应的tcp端口监听，外部系统只要用任意的node的ip地址+nodePort的端口号即可访问该服务，当然这里的限制也有，比如负载均衡，浪费端口号等，解决方案也有例如：ingress，这里先不做过多的解释了，后面单独讲解

       - 有状态的应用集群

         - 有状态集群中的共性
           - 每个节点都有固定的身份ID，通过这个ID集群中的成员可以互相发现并通信
           - 集群的规模比较固定，集群规模不能随意变动
           - 集群中的每个节点都是有状态的，通常会持久化数据到存储中，每个节点在重启后都需要使用原有的持久化数据
           - 集群中成员节点的启动顺序(以及关闭顺序)通常是确定的
           - 如果磁盘损坏，则集群里的某个节点无法正常运行，集群功能受损
         - 资源对象StatefulSet(sts)，适用有状态集群
           - sts里的每个pod都有稳定、唯一的网络标识，可以用来发现集群内的其他成员，假设sts名称为kafka，那么第一个pod为kafka-0，第二个为kafka-1 一次类推
           - sts控制的pod部分启动顺序是受控的，操作第n个pod时，前n-1个pod已经是运行且running状态了
           - sts里的pod采用稳定的持久化存储卷，通过pv或者pvc来实现，删除pod时，默认不会删除sts相关的存储卷(为了数据安全)
           - sts一般与headless service配合使用，sts在headless service的基础上又为sts控制的每个pod实例都创建了一个DNS域名 `(podname).(headless service name)`

       - 批处理(简单介绍一下)

         ```yaml
         apiVersion: batch/v1
         kind: Job
         metadata:
           name: pi
         spec:
           template:
             spec:
               containers:
               - name: pi
                 image: perl
                 command: ["perl","-Mbignum=bpi","-wle","print bpi(100)"]
               restartPolicy: Never
           parallelism: 1 # 并发
           completions: 5 # 运行任务总数
         ```

       - 应用配置

         - ConfigMap(cm)，保存配置项的(k=v)的一个map，cm是分布式系统中的”配置中心“的独特实现方式之一。
           - 用户将配置文件的内容保存到cm中，文件名可作为key，value就是整个文件的内容，多个配置文件可以被放入到同一个cm中
           - **在pod里将cm定义为特殊的volume进行挂载，在pod被调度到某个node上时，cm里的配置文件会自动还原到本地目录下，然后映射到pod里指定的配置目录下，这样用户的程序就可以无感知的读取配置了**
           - 在cm的内容发生修改后，k8s会自动重新获取cm的内容 ，并在目标节点上更新对应的文件。
         - secret，主要是为了敏感数据设置的应用配置，比如数据库的用户名密码，应用的数字证书，token等等，secret可以加密基于base64，在1.7版本后可以对secret中的数据进行实质性的加密(未测试)

     - HPA(简单介绍一下)

       - Horizontal Pod Autoscaler(HPA)，我们可以将hpa理解为pod横向自动扩容，即自动控制pod数量的增加减少，通过追踪分析指定dp控制的所有目标pod的负载情况，来确定是否需要有针对性调整目标pod的副本数量，这是hpa的原理。k8s内置了基于pod cpu 内存进行扩容的机制。

         ```yaml
         apiVersion: autoscaling/v1
         kind: HorizontalPodAutoscaler
         metadata:
           name: php-apache
           namespace: default
         spec:
           maxReplicas: 10
           minReplicas: 1
           scaleTargetRef:
             kind: Deployment
             name: php-apache
           targetCPUUtilizationPercentage: 90
         ```

         - 该yaml定义了了一个hpa，其控制对象为一个name为hph-apaceh的dp的pod副本，这些pod副本的cpu利用率超过90%时，会执行动态扩容机制，限定副本数为1-10。

     - VPA(说明一下，基本不用)

       - Vertical Pod Autoscaler(VPA)，即垂直pod扩容，它根据容器资源使用率自动推测并设置pod合理的cpu和内存的需求指标，从而更加精确的调度pod，实现整体上节省资源的目标，vpa不能与hpa共同操作同一组目标pod。

     - 存储类

       - 存储类资源对象主要包括Volume、Persistent Volume(pv)、pvc、Stotage Class(sc)

       - Volume

         - Volume是pod中能够被多个容器访问的共享目录，k8s中的volume用途和目的与docker中的volume比较类似，但是不能等价，首先k8s中的volume被定义在pod上，被一个pod里的多个容器挂载到具体的文件目录下，其次k8s中的volume与pod生命周期相同，但与容器的生命周期不相关，当容器终止或者重启时，volume中的数据不不会丢失

           ```yaml
             template:
               metadata:
                 labels:
                   app: app-demo
                   tier: frontend
                 spec:
                   volumes:
                     - name: datavo1
                       emptyDir: {}
                   containers:
                   - name: tomecat-demo
                   image: tomcat
                   volumeMounts:
                     - mountPath: /mydata-data
                       name: datavo1
           ```

           - 上边是一个有volume的yaml，其他内容没有补充，该yaml定义了一个名为datavo1的volume，并将其挂载到容器的/mydata-data 目录下

           - emptyDir

             - 一个emptyDir是在pod分配到node时创建的，初始内容为空，并且无须指定宿主机上对应的目录文件，因为这是k8s自动分配的一个目录，当pod从node上移除时，emptyDir中的数据也被永久移除了。
             - 用途：
               - 临时空间，例如用于某些应用程序运行时所需的临时目录，且无须永久保存
               - 长时间任务执行过程中使用的临时目录
               - 一个容器需要从另外一个容器中获取数据的目录(多容器共享目录)
             - **meptyDir使用的时节点的存储介质，例如磁盘、网络存储等，还可以使用emptyDir.medium属性，把这个属性设置为Memory，就可以使用内存的后端存储了。需要注意的时，这种情况下，使用的内存会被计入到容器的内存消耗中，将受资源限制机制的管理**

           - hostPath

             - 为在pod上挂载宿主机上的文件或目录

             - 用途：

               - 在容器应用程序生成的日志文件需要永久保存时
               - 需要访问宿主机上文件时

             - 注意：

               - 在不同的node配置pod时，可能会因为宿主机上的目录和文件不同导致volume上目录和文件访问结果不同

               - 如果使用了资源配置管理，则k8s无法将hostPath在宿主机上使用资源纳入管理

                 ```yaml
                         volumes:
                         - name: "persintent-storage"
                           hostPaht:
                             path: "/data"
                 ```

                 使用是宿主机的/data目录

           - 其他类型volume

             - iscsi：将iSCSi存储设备上的目录挂载到pod中
             - nfs：将NFS Server上的目录挂载到pod中
             - glusterfs：将开源的GlusterFS网络系统挂载到pod中
             - rdb：将Ceph块设备共享存储挂载到pod中
             - gitRepo：通过挂载一个空目录，并从git库克隆一个git repository以供pod使用
             - configmap：将配置数据挂载为容器的文件
             - secret：将secret数据挂载为容器文件

         - 动态存储

           - 使用volume时，发现不是很方便，存在一下弊端

             - 配置参数繁琐，存在大量手工操作
             - 预定义的静态volume，可能不符合目标的应用需求，比如容量问题，性能问题等

           - 动态存储主要是pv、sc、pvc。pv表示由系统创建的一个存储卷，可以被理解成k8s集群中某个网络存储对应的一块存储，它与volumes类似，但pv并不是被定义在pod上的，而是独立于pod之外定义的。

           - 存储类有很多种，创建什么类型的pv，就需要用到sc了

             ```yaml
             apiVersion: storage.k8s.io/v1
             kind: StorageClass
             metadata:
               name: standard
             provisioner: kubernetes.io/aws-ebs # 第三方存储插件
             parameters:
               type: gp2
             reclaimPolicy: Retain # 回收策略
             allowVolumeExpansiion: true
             mountOptions:
               - debug
             volumeBindingMode: Immediate
             ```

             - 上边是一个sc的yaml问价，其中几个关键的属性：provisioner、parameters和reclaim Policy。
             - 关于nfs的完整实例可以查看 https://note.youdao.com/s/IGwRd3gk

4. 安全类

   - 从本质上说k8s可以被看做一个多用户共享资源的资源管理系统，这里的用户主要分为两大类，我们开发的运行在pod里的应用，普通用户，如典型的kubectl命令，基本上由指定的运维人员使用，在更多的情况下我们开发的pod应用需要通过api server查询，创建，以及管理其他相关资源对象，所以这类用户才是关键用户。为此k8s设计了`Service Account` 并进一步完善了RBAC
   - 默认情况下，k8s在每个ns中都会创建一个默认的名称概为default的service account(sa),因此sa不能全局使用，只能在被它所在的ns中的pod使用。
   - sa是通过secret来保存对应的用户身份凭证的
   - RBAC使用参考： https://note.youdao.com/s/CALBZjr4

#### k8s安装

1. 使用kubeadm安装 https://note.youdao.com/s/Gu2rKGX1

2. 二进制安装  https://note.youdao.com/s/Vth6tDQZ

3. 说明点：

   - 容器运行时(CRI)

     - 目前支持的运行时，Docker、Containerd、CRI-O和frakti等

   - 端口号

     - api server： 8080 http  6443 https
     - controller manager： 10252
     - scheduler：10250  10255(只读端口)
     - etcd：2379(共客户端访问) 2380(共etcd集群内部节点之间访问)
     - DNS：53(UDP、TCP)

   - 修改kubeadm默认配置

     - kubeadm config print init-defaults： 输出kubeadm init命令默认参数的内容
     - kubeadm config print join-defaults：输出kubeadm jon命令默认参数
     - kubeadm config migrate：在新旧版本之间进行配置转换
     - kubeadm config images list：列出所需镜像
     - kubeadm config images pull：拉取所需镜像到本地

   - 修改docker默认驱动

     - k8s默认设置cgroup驱动为systemd，而docker服务的cgroup默认驱动为cgroupfs，建议保持一致修改为systemd

       - docker的配置文件  /etc/docker/daemon.json，修改之后reload一下 重启

       ```
       {
           "exec-opts":["native,cgroupdrive=systemd"]
       }
       ```

4. pod和容器的生命周期管理

   - pod由一组应用容器组成，其中包含共有的环境资源和资源约束，在CRI里，这个环境被成为PodSandbox

   - 在启动pod之前，kubelet调用RuntimeService.RunPodSanbox来创建环境，这一过程包括为pod设置网络资源。PodSanbox被激活后，就可以独立创建、启动、停止、删除不同的容器了，kubelet会在停止和删除PodSandbox之前先停止和删除其中的容器。

   - kubelet的职责在于通过RPC管理容器的生命周期，实现容器生命周期的钩子、存活和检测检测以及pod的重启策略等。

   - 补充一下CRI的主要组件

     ![image-20221128143532571](C:\Users\THINK\AppData\Roaming\Typora\typora-user-images\image-20221128143532571.png)

     - Protocol buffers api包含两个grpc服务，ImageService和RuntimeService
       - imageservice提供了从仓库中拉取镜像、查看和移除镜像的功能
       - runtimeservice负责pod和容器的生命周期管理，以及与容器的交互exec等			



#### pod讲解

1. pod的基本用法

   - 在k8s中对长时间运行容器的要求其主程序需要一直在前台运行，否则设置了副本之后，就会陷入创建销毁的循环中，对于无法执行前台的程序，可以借助Supervisor。
   - 静态pod，前面已经说过了，在说一下，静态pod是由kubelet进行管理的仅存在于特定的node上的pod，他们不能通过apiserver进行管理，无法与replicationcontroller、dp、ds进行关联，别切kubelet无法对他们进行健康检测。由于静态pod无法通过api server直接管理，所以在master上尝试删除这个pod时，会使其变成Pending状态，且不会被删除。

2. pod的配置管理

   - 应用部署的一个最佳实践是将应用所需的配置信息与程序分离，这样可以使应用程序更好的复用，通过不同的配置也能实现更灵活的功能。

   - comfigmap概述

     - 生成容器的环境变量
     - 设置容器启动命令的启动参数(需设置为环境变量)
     - 以volume的形式挂载为容器内部的文件或目录

   - 在pod中使用cm

     - 通过环境变量的方式

       ```yaml
       apiVersion: v1
       kind: ConfigMap
       metadata:
         name: cm-appvars
       data:
         apploglevel: info
         appdtadir: /var/data
         ---
       apiVersion: v1
       kind: Pod
       metadata:
         name: cm-test-pod
       spec:
         containers:
         - name: cm-test
           image: busybox
           command: ["/bin/sh","-c","env|grep APP"]
           env:
           - name: APPLOGLEVEL # 定义环境变量的名称
             valueFrom: # key apploglevel对应值
               configMapKeyRef:
                 name: cm-appvars # 环境变量曲子cm-appvars
                 key: apploglevel 
           - name:
             valueFrom:
               configMapKeyRef:
                 name: cm-appvars
                 key: appdtadir
       ```

       - 我们也可以使用`envFrom`来实现，其可以将cm、secret中所有的k=v自动生成环境变量

         ```yaml
         apiVersion: v1
         kind: ConfigMap
         metadata:
           name: cm-appvars
         data:
           apploglevel: info
           appdtadir: /var/data
         ---
         apiVersion: v1
         kind: Pod
         metadata:
           name: cm-test-pod
         spec:
           containers:
           - name: cm-test
             image: busybox
             command: ["/bin/sh","-c","env|grep APP"]
             envFrom:
             - configMapKeyRef:
               name: cm-appvars
         ```

     - 使用挂载卷的方式

       ```yaml
       apiVersion: v1
       kind: ConfigMap
       metadata:
         name: cm-appconfigfiles
       data:
         key-serverxml: |
           123
         key-loggingproperties: "456"
       ---
       apiVersion: v1
       kind: Pod
       metadata:
         name: cm-test-app
       spec:
         containers:
         - name: cm-test-app
           image: tomcat
           ports:
           - containerPort: 8080
           volumeMounts:
           - name: serverxml # 引用volume名称
             mountPath: /configfiles # 挂载到容器内的目录
         volumes: # 定义volume
         - name: serverxml # volume名称
           ConfigMap: # 使用cm 名称为cm-appconfigfiles
             name: cm-appconfigfiles
             items:
             - key: key-serverxml
               path: server.xml
             - key: key-loggingproperties
               path: logging.properties
       ```

       - items为列表，主要是可以自定义挂载的文件名字，如果不适用items，则volumemount方式在容器内的目录下为每个item都生成一个文件名为key的文件。

     - 使用cm的限制

       - cm必须在pod之前创建
       - 如果pod使用envFrom基于cm定义环境变量，则无效的环境变量名称(不符合定义的，例如以数字，特殊字符开头)将被忽略，并在事件中被记录为InvalidVariableNames
       - cm受名称空间的限制，只有处于同一个ns中的pod才可以使用
       - cm无法用于静态pod

   - pod生命周期和重启策略

     - pod的状态

       | 状态值    | 描述                                                         |
       | --------- | ------------------------------------------------------------ |
       | Pending   | Api server已经创建该pod，丹pod内还有一个或多个容器的镜像没有创建 |
       | Running   | pod内所有容器已经创建，且至少有一个处于运行状态、正在启动状态或正在重启状态 |
       | Succeeded | pod内所有容器成功执行后退出，且不会重启                      |
       | Failed    | pod内所有容器已退出，但至少有一个容器为退出失败状态          |
       | Unknown   | 由于某种原因无法获取该pod的状态，例如网络通信导致            |

     - pod重启策略

       - Always：当容器失效时，由kubelet自动重启该容器
       - OnFailure：当容器终止运行且退出码不为0时，由kubelet自动重启该容器
       - Never：不论容器运行状态如何，kubelet不会重启该容器
         - kubelet重启失效事件，最多为5分钟，并且在成功重启后的10分钟重置该时间

   - pod健康检查和服务可用性检查

     - 探针分类
       - LivenessProbe：用于判断容器是否存活状态(running)，如果该探针探测到容器不健康，则kubelet将杀掉该容器，并根据容器的重启策略做相应的处理。
       - ReadinessProbe：用于判断容器是否可用状态(ready)，达到ready状态的pod才可以接受请求，对于被service管理的pod，service与pod endpoint的关联关系也将基于pod是否ready进行设置，如果在运行过程中ready状态变为false，则系统自动将其从service的后端ep列表中隔离出去，后续再把恢复到ready状态的pod加回后端ep列表，这样就保证客户端在访问service时不会被转发到不可用的pod实例上。
       - StartupProbe：某些应用会遇到启动比较慢的情况，例如应用程序启动时需要与远程服务器建立连接，生命周期中这种有且仅有一次的超时延迟，可以使用该探针
     - 探针实现方式
       - ExecAction：在容器内部运行一个命令，该命令的返回码为0，在表示容器健康
       - TCPSocketAction：通过容器的IP地址和端口号执行tcp检查，如果能够建立tcp连接，则表明容器健康
       - HTTPGetAction：通过容器ip地址、端口号以及路径调用http get方法，如果相应的状态码大于等于200小于400，则认为健康。
     - 探针配置
       - 对于每种探针，都需要设置initialDelaySeconds和timeoutSeconds两个参数
       - initialDelaySeconds：启动容器后进行首次健康检查的等待时间，单位为s
       - timeoutSeconds：检查检查发送请求后等待响应的超时时间，单位为s，当超时发生时，kubelet会认为该容器已经无法提供服务，将会重启。
     - 自定义探针
       - 可用性探针可能不能满足我们的需求，所以引入了pod ready++特性对readiness探针机制进行扩展，通过pod readiness gates机制，具体使用方式是用户提供一个外部的控制器来设置响应pod的可用性状态。

   - pod调度

     - 先说一下rc，其实rc的继承者并不是deployment，而是replicaset(rs)，因为rs进一步增强了rc标签选择器的灵活性，之前rc的标签选择器只能选择一个标签，而rs拥有集合式的标签选择器，可以选择多个pod标签。

     - k8s的滚动升级，就是利用了rs的特性设计实现的，同时deployment也是通过rs来实现pod副本自动控制功能的。我们不应该直接使用底层的rs来控制pod副本，而应该用过rs的deployment对象来控制副本。

     - NodeSelector(定向调度)

       - 简单说就是通过node上的标签和pod的nodeselector属性匹配进行定向调度。如果我们给多个node都定义了相同的标签，则scheduler会根据调度算法从这组node中挑选一个可用的node进行调度。

     - NodeAffinity(node亲和性调度)

       - 目前有两种节点亲和性调度

         - RequireDuringSchedulingIgnoredDuringExecution：必须满足指定的规则才可以调度到pod到node上(与nodeselector很像，但是使用的是不同的语法)，相当于**硬限制**

         - PreferredDuringSchedulingIgnoredDuringExecution：强调优先满足指定规则，调度器会尝试调度pod到node上，但并不强求，相当于**软限制**，多个优先级规则还可以设置权重，以定义执行的先后顺序。

           ```yaml
           apiVersion: v1 
           kind: Pod
           metadata:
             name: with-node-affinity
           spec:
             affinity:
               nodeAffinity:
                 requireDuringSchedulingIgnoredDuringExecution:
                   nodeSelectorTerms:
                   - matchExpressions:
                     - key: beta.kubernetes.io/arch
                     operator: In
                     values:
                     - amd64
                 preferredDuringSchedulingIgnoredDuringExecution:
                 - weight: 1
                   preferredDuringSchedulingIgnoredDuringExecution:
                     preference:
                       matchExpressions:
                       - key: disk-type
                       operator: In
                       values:
                       - ssd
             containers:
             - name: with-node-affinity
               image: xxxx
           ```

           - `requireDuringSchedulingIgnoredDuringExecution`：要求只能运行在amd64的节点上(beta.kubernetes.io/arch  in  amd64)
           - `preferredDuringSchedulingIgnoredDuringExecution`：要求尽量运行在磁盘类型为ssd的节点上(disk-type in ssd)
           - NodeAffinity语法支持的操作符包括In、NotIn、Exists、NodesNotExist、Gt、Lt。虽然没有排斥的功能，但是可以借助NotIn、DoesNotExist实现其功能
           - **NodeAffinity规则设置**：
             - 如果同时定义了nodeSelector和nodeAffinity，那么必须两个条件同时满足，pod才能最终运行到指定的node上
             - 如果nodeAffinity指定了多个nodeSelectorTerms，那么其中一个能匹配到即可
             - 如果在nodeSelectorTerms中有多个matchExperssions，则一个节点必须满足所有matchExpressions才能运行该pod。

     - PodAffinity(pod亲和性)与互斥调度策略

       - 概念讲解：简单的说，就是相关联的两种或者多种pod是否可以在同一个拓扑域中存在或者互斥，前者为PodAffinity，后者为PodAntiAffinity。

       - pod亲和与互斥的调度具体做法：通过在pod的定义上增加topologyKey属性，来声明对应的目标拓扑区域内几种相关联的pod要在一起或者不在一起，这与节点亲和性相同，pod亲和性与互斥的条件设置也是 `requiredDuringSchedulingIgnoredDuringExecution` 和 `preferredDuringSchedulingIgnoredDuringExecution` ，pod亲和性被定义于podspec的affinity字段的podaffinity字段中，pod间的互斥性则被定义于同一层的podantiaffinity字段中。

         ```yaml
         apiVersion: v1
         kind: Pod
         metadata:
           name: pod-flag
           labels:
             security: "S1"
             app: "nginx"
         spec:
           containers:
           - name: nginx
             image: nginx
         
         ---
         apiVersion: v1
         kind: Pod
         metadata:
           name: pod-affintiy
         spec:
           affinity:
             podAffinity:
               requireDuringSchedulingIgnoredDuringExecution:
               - labelSelector:
                 matchExpressions:
                 - key: security
                   operator: In
                   values:
                   - S1
                 topologyKey: kubernetes.io/hostname
           containers:
           - name: with-pod-affinity
             image: xxxx
         ```

         - 首先创建了一个带有标签 `security: "S1"  app: "nginx" `的pod，然后创建另外一个pod带有节点亲和性，其亲和性的对象为带有标签  `security: "S1"`的pod，topologyKey这个设置是node维度的，一般使用k8s自带的标签。
         - 我们apply后可以发现这个2个pod在同一个node上运行，那么当我们删除这个节点的 `kubernetes.io/hostname` 标签，重新apply后，发现pod一直处于pending状态，这是因为找不到满足条件的node了。

       - 接下来我们再次创建一个pod，满足互斥性设计

         ```yaml
         apiVersion: v1
         kind: Pod
         metadata:
           name: anti-affintiy
         spec:
           affinity:
             podAffinity:
               requireDuringSchedulingIgnoredDuringExecution:
               - labelSelector:
                 matchExpressions:
                 - key: security
                   operator: In
                   values:
                   - S1
                 topologyKey: topology.kubernetes.io/zone
             podAntiAffinity:
               requireDuringSchedulingIgnoredDuringExecution:
               - labelSelector:
                 matchExpressions:
                 - key: app
                   operator: In
                   values:
                   - nginx
                 topologyKey: kubernetes.io/hostname
           containers:
           - name: anti-affinity
             image: xxxx
         ```

         - 上边的yaml包含了亲和性和互斥性调度，首先是新pod与 security=S1的pod为同一个zone，其次是不与app=nginx的pod为同一个node。

       - 与节点亲和性类似，pod亲和性也包含的操作符 In、NotIn、Exists、DoesNotExist、Gt、Lt

       - topologyKey限制：

         - 在pod亲和性和RequiredDuringScheduling的pod互斥性的定义中，不允许使用空的topologykey
         - 如果Admission controller包含了LimitPodHardAntiAffinityTopology，那么针对requiredduringscheduling的pod互斥性定义就被限制为kubernetes.io/hostname，要么使用自定义的topologykey，或者禁用该控制器
         - 在perferredduringscheduling类型的pod互斥性定义中，空的topologykey会被解释为kubernetes.io/hostname.failure-domain.kubernetes.io/zone以及filure-domain.beta.kubernetes.io/region的组合

       - podAffintiy规则设置注意事项：

         - 除了设置label selector和topologkey，永和还可以指定naemspace列表进行限制，同样，使用label selector对namespace进行选择，namespace的定义和label selector及topologykey同级，省略namespace的设置，表示使用定义了affinity/anti-affinity的pod的所在的命名空间，如果namespace被设置为空置("")，则表示所有名称空间。
         - 在所有关联requireDuringSchedulingIgnoredDuringExecution的matchExpressions全部满足之后，系统才能将pod调度到某个node上。

     - Taints(污点)和Tolerations(容忍)

       - 概念讲解：Taints污点，简单说就是使node拒绝pod调度进来，注意标记为Taints的node不是故障节点哦。 Tolerations可以忽略污点进行pod调度。

       - 语法格式： `$ kubectl taint node node_name key=value:NoSchedule`

         ```yaml
         ---
           tolerations:
           - key: "key"
             operator: "Equal"
             value: "value"
             effect: "NoSchedule"
         ```

         或者

         ```yaml
           tolerations:
           - key: "key"
             operator: "Exits"
             effect: "NoSchedule"
         ```

         - 注意： pod的tolerations声明中的key和effect需要与Taint的设置保持一致
         - operation的值是Exits时，无须指定value
         - operation的值是Equal，value需要相等
         - 另外，空的key配合Exits操作能够匹配所有键和值，空的effect匹配所有的effect

       - 刚刚我们说了effect和污点设置相关的，那么其他污点我们也说一下：

         - NoSchedule：不会被调度到存在该污点的node上，已经存在的pod在声明周期内不受影响
         - PreferNoSchedule：尽量不被调度到该node上，但是不是强制的
         - NoExecute：不被调度并且已经存在的pod实行驱逐行为。

       - 系统允许在同一个node上设置多个taints，也可以在pod上设置多个tolerations，其pod中的容忍匹配node上的污点，剩下的污点就是对该pod的效果。

       - 单独讲解一下Noexecute：

         ```yaml
           tolerations:
           - key: "key"
             operator: "Equal"
             value: "value"
             effect: "NoExecute"
             tolerationSeconds: 3600 # 可选
         ```

         - 如果我们不设置 `tolerationSeconds`，那么该ndoe上运行的没有该污点容忍的pod会立即被驱逐，如果设置了该字段，则会运行没有该污点容忍的pod继续运行设置的时间，当然。我们在运行运行的时间内，将NoExecute污点删除，那么就不会发生驱逐事件。
         - k8s也内置了一些对该NoExecute污点的容忍
           - key为node.kubernetes.io/not-reay，  tolerationSeconds=300
           - key为node.kubernetes.io/unreachable，  tolerationSeconds=300

     - pod优先级调度

       - 优先级抢占调度策略的核心行为分别是驱逐(Eviction)和抢占(Preemption)，这两种行为的使用场景，效果相同。Eviction是kubelet进程行为，即当一个node资源不足时，该节点上的kubelet进程会执行驱逐动作，此时kubelet会综合考虑pod的优先级、资源申请量与实际使用等信息来计算那些pod需要被驱逐；当同样优先级的pod需要驱逐时，实际使用的资源量超过申请量最大倍数的高耗能pod会被驱逐。Preemption则是Scheduler执行的行为，当一个新的pod因为资源无法满足而不能被调度时，scheduler可能选择驱逐部分低优先级的pod实例来满足此pod的调度目标。需要注意的是：scheduler可能会驱逐nodeA上的一个pod以满足nodeB上的一个新pod的调度任务。

         ```yaml
         apiVersion: scheduling.k8s.io/v1beta1
         kind: PriorityClass
         metadata:
           name: high-priority
         value: 1000000
         globalDefault: false
         description: "this priority class should be used for xyz service pods only"
         ```

         ​	上边的yaml定义了一个优先级为1000000的优先级类别，其名字为high-priority，数字越大优先级越高。

         ```
         apiVersion: v1
         kind: Pod
         metadata:
           name: nginx
           labels:
             env: test
         spec:
           containers:
           - name: nginx
             image: nginx
             imagePullPolicy: IfNotPresent
           priorityClassName: high-priority
         ```

         - 优先级调度暂时来说不推荐使用，如果需要使用需要好好规划一下才可以。如果发生了需要抢占的调度，高优先级的pod就可能抢占到节点N，并将低优先级的pod驱逐节点N，高优先级的pod中的status信息中nominatedNodeName字段会记录目标节点N的名称，需要注意的是，高优先级的poe仍然无法保证最终被调度到节点N上，在节点N上低优先级的pod被驱逐的过程中，如果有新的节点满足高优先级pod需求，就会把它调度到新的node上。如果在等待低优先级的pod退出的过程中，又出现了优先级更高的pod调度器，就会调度这个更高优先级的pod到节点N上，并重新调度之前等待的高优先级的pod。

     - DaemonSet

       - 在每一个node上都调度一个pod
       - DaemonSet(ds)的调度策略与rc类似，使用了系统内置的算法在每个node上进行调度，也可以在pod的定义中使用NodeSelector或NodeAffinity来指定满足条件的node范围进行调度。

     - pod容灾调度

       - 简答说一下，不做过多的讲解，举例说明就是需要将pod按照zone进行调度，当然了你也可以使用node标签选择，但是当某个zone失效后，那么这个zone的pod就无法迁移到其他zone了。
       - 在k8s 1.16之后引入了Even Pod Spreading特性，用于通过topologKey属性识别zone，并通过设置新的参数topologySpreadConstraints将pod均匀调度到不同的zone上。

   - Init Container(初始化容器)

     - init container的运行方式与应用容器不同，初始化容器必须先于应用容器执行完成，当设置了多个init container时，将按照顺序逐个执行，并且只有前一个init container运行成功后才能运行后一个init container，在所有的init container都成功后，才会初始化pod的各种信息，并开始创建和运行应用容器
     - 在init container的定义中也可以设置资源资源限制、volume的使用和安全策略等，但是资源限制的设置与应用容器不同：
       - 如果多个init container都定义了资源请求/资源限制，则取最大值为所有的init container的资源请求/资源限制
       - pod的有效资源请求值/资源限制取一下二者中较大值：所有应用容器的资源请求值/资源限制值之和；init container的有效资源请求值/资源限制值。
       - 调度算法基于pod的有效资源请求值/资源限制值进行计算，也就是说init container可以为初始化操作系统预留资源，即使后续应用容器无须使用这些资源
       - pod的有效Qos等级适用于init container和应用容器
       - 资源配额和限制将根据pod的有效请求值/资源限制值计算生效
       - pod级别的cgroup将基于pod的有效请求/限制，与调度机制一致。
       - init container不能设置readinessProbe探针，因为必须在他们成功运行后才能继续运行pod中国定义的普通容器。

   - pod的升级和回滚

     - deployment的升级和回滚

       - 直接更新镜像，使用命令的方式 `kubectl set image`

       - 修改yaml文件方式  通过 `kubectl edit`

       - deployment升级过程是如何的呢？

         - 更新dp时(例如：3个pod情况)，系统创建一个新的rs(ReplicaSet)，并将其副本数量扩展到1，然后将旧的rs缩减一个，之后，系统继续按照相同的更新策略对新旧两个rs进行逐个调整，最后知道新的rs运行与旧的rs相容的pod数量。(默认情况下)
         - deployment更新策略：
           - Recreate：重建
           - RollingUpdate：滚动更新，也是默认的，可以通过设置spec.strategy.rollingUpdate下的maxUnavailable和maxSurge来控制更新过程。
             - spec.strategy.rollingUpdate.maxUnavailable：更新过程中不可用pod数量上线，可以设置绝对值，也可以设置百分比。系统会向下取整。当然了maxUnavailable必须大于0
             - spec.strategy.rollingUpdate.maxSurge：用于指定更新过程中，pod总数量超过pod期望副本数部分的最大值。可以设置绝对值，也可以设置百分比。系统会向下取整

       - deployment回滚：

         - 回滚到上一个版本： `kubectl rollout undo deployment/xxxxx`

         - 查看可用于回滚的历史版本：kubectl rollout history  deployment/xxxx

           ```yaml
           [root@k8s-master ~]# kubectl rollout history  deployment/demo1-deployment 
           deployment.apps/demo1-deployment 
           REVISION  CHANGE-CAUSE
           2         <none>
           3         <none>
           4         <none>
           
           ```

           - 此时可以通过--to-revision参数回滚到指定版本 `kubectl rollout undo deployment/xxxx --to-revision=序列号`

     - DaemonSet的更新策略

       - 目前daemonset的升级策略有：OnDelete和RollingUpdate，默认为RollingUpdate
         - OnDelete：人为手动干预，手动删除一个pod才会创建一个新的，不推荐使用
         - RollingUpdate：旧版本的pod将被杀掉，然后自动创建新版本的pod。目前k8s不支持查看和管理daemonset的更新历史记录，daemonset回滚需要更改配置yaml文件来。

     - StatefulSet的更新策略

       - sts的更新策略：RollingUpdate、OnDelete、Partitioned
         - RollingUpdate：statefulset controller会删除并创建statefulset相关的每个pod对象，其处理顺序与statefulset终止pod的顺序一致，即从序号最大的pod开始重建，每次更新一个pod。
         - OnDelete：手动删除pod需要
         - Partitioned：用户指定一个序号，sts中序号大于等于此序号的pod实例会全部被升级，小于此序号的pod实例则保留旧版本的不变。即使这些pod被删除、重建也仍然保持原来的旧版本。

   - pod扩缩容

     - k8s对pod扩缩容提供了手动和自动两种方式。

       - 手动扩缩容： `kubectl scale`
       - 自动模式：需要用户根据某个性能指标或者自定义业务指标，并指定副本数量范围，也就是HPA。

     - 目前k8s使用的时 Metrics Service来完成数据采集，将采集到的pod性能指标数据通过聚合API提供给HPA控制器进行查询，聚合API这里不做过多的解释，后面会讲解。

     - HPA工作原理：

       - k8s中的某个Metrics Server持续采集所有pod副本的指标数据，HPA控制器通过Metrics Server中的API获取这些数据，基于用户定义的扩缩容规则进行计算，得到目标pod的副本数量，当目标pod副本数量与当前副本数量不同时，hpa控制器就向pod的副本控制器发起scale，调整pod的副本数量，完成扩缩容操作。

       - 指标类型

         - Master的kube-controller-manager服务持续监测目标pod的某种特性指标，以计算是否需要调整副本数量，目前k8s支持的指标类型为：

           - pod资源使用率：pod级别的性能指标，通常是一个比率值，例如cpu使用率
           - pod自定义指标：pod级别的性能指标，通常是一个数值，例如接受的请求数量
           - Object自定义指标或外部自定义指标：通常是一个数值，需要容器应用以某种方式提供，例如通过HTTP URL /metrics提供，或者使用外部服务提供的指标采集URL。

         - k8s HPA当前有一下两个版本：

           - autoscaling/v1版本仅支持基于cpu使用率指标的自动扩缩容
           - autoscaling/v2版本支持基于内存使用率指标、自定义指标以及外部指标的自动扩缩容，并且进一步扩展以支持多指标缩放能力，当定义了多个指标时，HAP会根据每个指标进行计算，其中缩放幅度最大的指标会被采纳。

         - 扩容算法解释

           - 扩容算法暂时就不写了，想了解的可以自行查找资料

           - 举例说明一下扩容算法：以cpu请求数量为例，如果用户设置的期望指标为100m，当期实际使用的指标为200m，那么计算的pod副本数量为2个，如果当前实际使用的指标值为50m，计算结果为0.5，向上取整则为1。当然了我们也可以设置容忍度让系统不做扩缩容操作，通过kube-controller-manager服务的启动参数进行设置。默认值为0.1，简单说就是可以超频低频0.9-1.1。

           - 此外存在集中pod异常的 情况：

             - pod正在被删除将不会计入目标pod副本数量
             - pod的当前指标值无法获得，本次探测不会将pod纳入目标pod副本数。后续的探测会被重新纳入计算范围
             - 如果指标类型时cpu使用率，则对于正在启动但是还未达到ready状态的pod也暂时不会纳入目标副本数量范围。

           - 其他情况：

             - 使用hpa特性时，可能因为指标的变化造成pod副本数量频繁变动，我们可以通过设置参数horizontal-pod-autoscaler-downscale-stabilization(kube-controller-manager参数)来设置冷却时间，意思就是上次扩缩容后下次的再次生效的时间，默认为5分钟。

           - hpa配置详解 v1版本

             ```yaml
             apiVersion: autoscaling/v1
             kind: HorizontalPodAutoscaler
             metadata:
               name: php-apache
             spec:
               scaleTargetRef:
               apiVersion: apps/v1
               kind: Deployment
               name: php-apache
             minReplicas: 1
             maxReplicas: 10
             targetCPUUtilizationPercentage: 50
             ```

             - scaleTargetRef：目标作用对象，可以时Deployment、ReplicationController或ReplicaSet
             - targetCPUUtilizationPercentage：期望每个pod的cpu使用率都为50%，该使用率基于pod设置的cpu request值进行计算。
             - minReplicas和maxReplicas：pod副本数量的最小值和最大值，系统将在这个范围内进行自动扩缩容操作。

           - hap配置详解 v2版本

             ```yaml
             apiVersion: autoscaling/v2beta2
             kind: HorizontalPodAutoscaler
             metadata:
               name: php-apache
             spec:
               scaleTargetRef:
                 apiVersion: apps/v1
                 kind: Deployment
                 name: php-apache
             minReplicas: 1
             maxReplicas: 10
             metrics:
             - type: Resource
               resource:
                 name: cpu
                 target:
                   type: Utilization
                   averageUtization: 50
             ```

             - scaleTargetRef、minReplicas、maxReplicas：前面已经说过了

             - metrics：目标指标值，在metrics中通过参数type定义指标的类型，通过参数target定义相应的指标目标值，系统将在指标数据达到目标值时，触发扩缩容操作

             - metrics中type设置为以下四种：

               - Resource：指的是当前伸缩对象下的pod的cpu和memory指标。只支持Utilization和AverageValue类型的目标值，对于cpu使用率，在target参数中设置averageUtilization定义目标平均cpu使用率，对于内存资源，在target参数中设置averageValue定义目标平均内存使用值

               - pods：指的是伸缩对象pod的指标，数据需要有第三方Adapter提供，只允许AverageValue类型的目标值

               - Object：k8s内部对象的指标，数据需要第三方Adapter提供，只支持Value和AverageValue类型的目标值

               - External：指的是k8s外部的指标，数据同样需要第三方Adapter提供，只支持value和averagevalue类型的目标值。

                 其中averagevalue是根据pod副本数量计算的平均值指标。resource类型的指标来子metrics server自身。pod类型和object类型都属于自定义指标类型。

     - 基于自定义指标的HPA

       - 基于prometheus的hpa架构图：

         ![image-20221130122628721](C:\Users\THINK\AppData\Roaming\Typora\typora-user-images\image-20221130122628721.png)

         - 组件说明：
           - prometheus：采集pod指标
           - custom metrics server：自定义metrics server，本例使用prometheus adapter进行具体实现，它从prometheus服务采集指标数据，通过k8s的metrics aggregation层将自定义指标api注册到master的api server中，以 /api/custom.metrics.k8s.io路径提供指标数据。
           - hpa controller：k8s的hpa控制器，用于用户定义的hpa进行自动扩缩容操作。

       - promoetheus部署可以参考： https://github.com/prometheus-operator/kube-prometheus



#### service讲解

1. service定义

   - service用于为一组提供服务的pod抽象一个稳定的网络访问地址，是k8s实现微服务的核心概念，通过service的定义设置的访问地址是DNS域名格式的服务名称，对于客户端应用来说，网络访问方式并没有改变，当然了svc(service)还提供了负载均衡器的功能。

   - service的yaml解释

     ```yaml
     apiVersion: v1 # 必选 版本
     kind: Service # 必选  类型
     metadata: # 必选 元数据
       name: string # 必选 service名称
       namespace: string # 必选 名称空间，不指定为default
       labels: # 标签列表
         - name: string # 标签名称
       annotations: # 注解
         - name: string 
     spec:
       selector: [] # 必选 标签选择器，将选择具有指定label标签的pod作为管理范围
       type: string # 必选 service类型 默认为clusterip，还有NodePort、LoadBalancer
       clusterIP: string #
       sessionAffinity: string # 是否支持session，可选值为clusterip，默认为none
       ports:
       - name: string
         protocol: string
         port: int  # 服务监听的端口号
         targetPort: int # 需要转发到后端pod的端口哈
         nodePort: int
       status:
         loadBalancer: # 外部负载均衡器
           ingress:
             ip: string
             hostname: string
     ```




#### k8s网络

- 节点网络
  - 物理上是可达的
  - 承载跨节点网络流量
  - 对CNI插件选型有影响
- 容器网络
  - pod在相同node上可达
  - pod跨node之间可达
  - CNI插件负责创建维护
- 集群网络
  - 提供服务的抽象
  - 服务到pod的动态映射
  - 提供负载均衡
  - 提供服务发现
- network namespace
  - 主机ip栈的逻辑拷贝，隔离的网络协议栈，有独立网络接口、ip、路由表和iptables规则
  - 进程可以设置不同network namespace，多个进程可以属于同一个ns
  - 每个pod是独立的network namespace
- birdge device
  - 纯软件实现的虚拟交换机，有着和物理交换机相同的功能，例如二层交换，MAC地址学习
  - 可以把tun/tap，veth pair等设备绑定到网桥上
- veth pair device 虚拟以太网卡对
  - 直连设备，A端数据会投射到B端
  - 可以跨namespace
- pod网络创建过程
  - ![image-20221229205653009](https://gitee.com/root_007/md_file_image/raw/master/202302160955655.png)
  - 创建pod时，apiserver会收到请求，根据调度算法选定一个节点创建pod，选定节点的kubelet会在本节点完成pod创建
    - 首先kubelet会调用CRI插件，cri插件会创建pod的namesapce，然后创建沙箱容器(pause sandbox)
    - 然后CRI会调用CNI插件，cni插件会创建网络接口eth0(pod内部)，然后对网络接口赋予ip地址，然后设置pod的路由表，然后设置主机的路由表，虚拟以太网卡对
- CNI简介
  - CNCF项目，提供实现无关的规范来创建配置容器网络(不仅仅用于k8s，包括Mesos，CloudFoundry,podman,cri-o)，定义执行流程和配置格式
  - 主要功能
    - 创建网络接口，连接pod之间的网络，通过overlay，route，underlay etc(不通过NAT)
    - 进行ip地址管理，分配pod ip地址
    - 提供网络安全策略
  - 工作机制
    - 通过json配置文件定义网络配置  /etc/cni/net.d/xxnet.conf
    - 通过调用可执行程序(CNI插件)来对容器网络执行配置  /opt/cni/bin/xxnet
    - 通过链式调用的方式来支持多插件的组合使用
    - 调用过程 kubelet-->CRI-->CNI-->读取配置文件-->执行二进制插件
- CNI插件 Flannel
  - ![image-20221229210408356](https://gitee.com/root_007/md_file_image/raw/master/202302160955447.png)
  - 同一个node节点上不通pod之间的通信
    - 主要是通过虚拟以太网卡对来进行通信的，一端在pod上，一端在bridge cni0上
  - 不同node上不同pod之间的通信
    - 会创建一个flannel.1的udp隧道，如何查找那个node上的pod，主要是通过路由表来进行的，实际上是通过宿主机的网卡进行通信的，主要是通过封包解包进行通信的
- Calico
  - ![image-20221229210534233](https://gitee.com/root_007/md_file_image/raw/master/202302160955344.png)
  - Felix：运行在每个节点的agent进程，主要负责网络接口管理和监听、路由、ARP管理、ACL管理和同步、状态上报等
  - confd：监听etcd的变化，通知BIRD更新配置
  - etcd：主要负责网络元数据的存储和一致性
  - BIRD：主要负责把Felix写入kernel的路由信息分发到当前calico网络，通过BGP Peer告诉另外节点的BIRD来同步信息
  - IP地址管理：可以使用host-local或calico-ipam做地址管理
  - 大规模部署时使用，摒弃所有节点互联的mesh模式，通过一个或者多个BGP Route Reflecotr来完成集中式，由于Calico BGP是一种纯三层的实现，因此可以避免与二层方案相关的数据包封装的操作，中间没有NAT没有overlay，所以它的转发效率高
- Pod的DNS策略
  - 可以单独对每个pod进行DNS策略的定义，通过dnsPolicy字段设置
    - Default：集成节点的dns配置
    - ClusterFirst：先从集群coreDNS进行域名解析，解析不到的DNS通过coreDNS的内部的自定义配置进行dns解析
    - None：根据pod的dnsConfig规范来执行
