## 认识Docker

- docker是基于linux kernel和namespace、cgroups技术实现的

## docker架构图

![docker 架构](https://gitee.com/root_007/md_file_image/raw/master/202306140948395.png)

> - 当我们要创建一个容器的时候，现在`Docker Daemon`并不能直接帮我们创建，而是请求`Containerd`来创建一个容器，`Containerd`收到请求后，也不会直接去操作容器，而是创建一个叫做`Containerd-shim`的进程，这个进程去操作容器，我们指定容器进程是需要一个父进程来做状态收集，维持stdin等fd打开等工作，这时又引入了一个`containerd-shim`的垫片，主要是为了避免因`Containerd`挂掉使得整个宿主机上的所有容器都退出
> - 创建容器需要做一些`namespace`和`cgroups`的配置，以及挂载`root`文件系统等操作，这些操作其实已经有了标准的规范，那就是`OCI`，`runc`就是它的一个参考实现，这个标准其实就是一个文档，主要规定了容器镜像的结构、容器需要接收那些操作指令，比如：`create`、`start`、`stop`、`delete`等这些命令，`runc`就可以按照`OCI`文档来创建一个符合规范的容器
> - 所以真正启动容器是通过`containerd-shim`去调用runc来启动容器的，`runc`启动完容器后本身会直接退出，`containerd-shim`则会成为容器进程的父进程，负责收集容器进程的状态上报给`containerd`，并在容器中pid为1的进程退出后接管容器中的子进程进行清理，确保不会出现僵尸进程

## CRI

> - CRI(Container Runtime Interface容器运行时接口)，本质上就是k8s定义的一组与容器运行时进行交互的接口，所以只要是实现了这套接口的容器运行时都可以对接到k8s平台上，其中dockershim就是k8s对接docker到CRI接口上的一个垫片实现

![cri shim](https://gitee.com/root_007/md_file_image/raw/master/202306141031582.png)

![dockershim](https://gitee.com/root_007/md_file_image/raw/master/202306141039023.png)

![kubelet cri](https://gitee.com/root_007/md_file_image/raw/master/202306141117788.png)

> - kubelet通过gRPC框架与容器运行时或shim进行通信，其中kubelet作为客户端，CRI shim(也可能是容器运行时本身)作为服务端，当我们使用docker时，在k8s集群中创建一个pod，首先kubelet通过CRI接口调用dockershim，请求创建一个容器，kubelet可以看作一个简单的CRI的client，而dockershim就是接收请求的server，都是内置在kubelet中的，然后dockershim接收到请求后，转化成docker daemon能识别的请求，发送到docker daemon上请求创建一个容器，然后去调用containerd，然后创建containerd-shim进程，最后通过进程去调用runc去真正创建容器
> - CRI定义的API主要包括两个gRPC服务，ImageService和RuntimeService，其中ImageService服务主要负责镜像拉取、查看和删除等操作，RuntimeService则是用来管理pod和容器的生命周期以及与容器交互的调用(exec/attach/port-forward)等操作

## Docker技术原理

- chroot

  > 在unix和linux系统的一个操作，针对正在运行的软件的进程和它的子进程，改变它外显的根目录，一个运行在这个环境下，经由chroot设置根目录的程序，它不能对这个指定根目录外的文件进行访问动作，不能读取，也不能更改它的内容

- namespace

  > 对内核资源进行隔离，使得容器中的进程都可以在单独的命名空间中运行，并且只可以访问当前容器命名空间的资源，可以隔离进程ID、主机名、用户ID、文件名、网络访问和进程通信等资源
  >
  > 使用的namespace列举：
  >
  > ```go
  > 	namespace名称：mount(mnt)、作用：隔离挂载点
  > 	namespace名称：process ID(pid)、作用：隔离进程ID
  > 	namespace名称：network(net)、作用：隔离网络设备，端口号等
  > 	namespace名称：interprocess communication(ipc)、作用：隔离system v ipc和posix message queues
  > 	namespace名称：UTS namespace(uts)、作用：隔离主机名和域名
  > 	namespace名称：User namespace(user)、作用：隔离用户和用户组
  > 	namespace名称：control group(cgropu)Namespace、作用：隔离cgroup根目录
  > 	namespace名称：Time Namespace、作用：隔离系统时间
  > ```

- cgroup

  > 对进程、进程组做资源限制，注意虽然可以实现资源的限制，但是并不能保证资源的使用
  >
  > cgroup工作目录为/sys/fs/cgroup
  >
  > ```bash
  > [root@localhost cgroup]# pwd
  > /sys/fs/cgroup
  > [root@localhost cgroup]# ll
  > 总用量 0
  > drwxr-xr-x 5 root root  0 2月  20 11:04 blkio
  > lrwxrwxrwx 1 root root 11 2月  20 11:04 cpu -> cpu,cpuacct
  > lrwxrwxrwx 1 root root 11 2月  20 11:04 cpuacct -> cpu,cpuacct
  > drwxr-xr-x 5 root root  0 2月  20 11:04 cpu,cpuacct
  > drwxr-xr-x 3 root root  0 2月  20 11:04 cpuset
  > drwxr-xr-x 5 root root  0 2月  20 11:04 devices
  > drwxr-xr-x 3 root root  0 2月  20 11:04 freezer
  > drwxr-xr-x 3 root root  0 2月  20 11:04 hugetlb
  > drwxr-xr-x 5 root root  0 2月  20 11:04 memory
  > lrwxrwxrwx 1 root root 16 2月  20 11:04 net_cls -> net_cls,net_prio
  > drwxr-xr-x 3 root root  0 2月  20 11:04 net_cls,net_prio
  > lrwxrwxrwx 1 root root 16 2月  20 11:04 net_prio -> net_cls,net_prio
  > drwxr-xr-x 3 root root  0 2月  20 11:04 perf_event
  > drwxr-xr-x 5 root root  0 2月  20 11:04 pids
  > drwxr-xr-x 5 root root  0 2月  20 11:04 systemd
  > 
  > ```
  >
  > 

- 联合文件系统

  > 镜像构建、容器运行环境
  >
  > 一种分层的轻量级文件系统，它可以把多个目录内容联合挂载到同一个目录下，从而形成一个单一的文件系统，联合文件系统时docker镜像和容器的基础， 因为它可以使docker把镜像做成分层的结构，从而使得镜像的每一层都可以被共享
  >
  > 例如：两个镜像都是由base为centos构建的，那么这个base的镜像在物理机上只需要存储一份即可
  >
  > 常见的联合文件系统：
  >
  > 1. AUFS
  > 2. Devicemapper
  > 3. OverlayFS
  >
  > centos7中默认支持的文件系统
  >
  > ```bash
  > [root@localhost cgroup]# cat  /proc/filesystems
  > nodev	sysfs
  > nodev	rootfs
  > nodev	ramfs
  > nodev	bdev
  > nodev	proc
  > nodev	cgroup
  > nodev	cpuset
  > nodev	tmpfs
  > nodev	devtmpfs
  > nodev	debugfs
  > nodev	securityfs
  > nodev	sockfs
  > nodev	dax
  > nodev	bpf
  > nodev	pipefs
  > nodev	configfs
  > nodev	devpts
  > nodev	hugetlbfs
  > nodev	autofs
  > nodev	pstore
  > nodev	mqueue
  > 	xfs
  > nodev	overlay
  > 
  > ```
  >
  > 介绍一下Overlay2：centos推荐使用，将所有的目录称为层，把这些层统一展现到一个目录的过程称为联合挂载，把目录的下一层称为lowerdir，上一层称为upperdir，联合挂载后的结果称为merged

  ![image-20230222142442359](https://gitee.com/root_007/md_file_image/raw/master/202302221424440.png)

## docker三个核心概念

- **镜像**：是一个只读的文件和文件夹组合，是docker容器启动的先决条件
- **容器**：是镜像的运行实体。容器运行着真正的应用进程，容器有初建、运行、停止、暂停、删除5种状态。在容器内部无法看到主机上的进程、环境变量、网络信息等。
- **仓库**：存储和分发镜像的仓库

## docker创建流程

​	示例说明：docker run -it --name=busubox busubox

> 1. docker会检查本地是否存在busybox镜像
> 2. 使用busybox镜像创建并启动一个容器
> 3. 分配文件系统，并在镜像只读层外创建一个读写层
> 4. 从docker ip池中分配一个ip给容器
> 5. 执行用户启动命令运行镜像

## 使用Dockerfile构建镜像

> **优点**：
> 	易于版本化管理
> 	过程可追溯
> 	屏蔽构建环境异构
> **书写原则**：
> 	单一职责
> 		不同功能的应用应该尽量拆分为不同的容器，每个容器只负责单一业务进程
> 	提供注释信息
> 		晦涩难懂的代码尽量添加注释
> 	保持容器最小化
> 		应该尽量避免安装无用的软件包
> 	合理选择基础镜像
> 		容器的核心是应用，只要基础镜像能够满足应用的运行环境即可
> 	使用.dockerignore文件
> 		忽略代码中不需要的文件
> 	尽量使用构建缓存
> 	正确设置时区
> 	使用国内加速软件源加速镜像构建
> 	最小化镜像层数

- 常用指令

  ### FROM

  指明构建的新镜像是来自于哪个基础镜像，如果没有选择 tag，那么默认值为 latest

  ```bash
  FROM centos:7
  ```

  ### LABEL

  功能是为镜像指定标签。也可以使用 LABEL 来指定镜像作者。

  ```
  LABEL maintainer="xxx.qq.com"
  ```

  ### RUN

  构建镜像时运行的 Shell 命令，比如构建的新镜像中我们想在 /usr/local 目录下创建一个 java 目录。

  ```
  RUN mkdir -p /usr/local/java
  ```

  ### ADD

  拷贝文件或目录到镜像中。src 可以是一个本地文件或者是一个本地压缩文件，压缩文件会自动解压。还可以是一个 url，如果把 src 写成一个 url，那么 ADD 就类似于 wget 命令，然后自动下载和解压。

  ```
  ADD jdk-11.0.6_linux-x64_bin.tar.gz /usr/local/java
  ```

  ### COPY

  　拷贝文件或目录到镜像中。用法同 ADD，只是不支持自动下载和解压。

  ```
  COPY jdk-11.0.6_linux-x64_bin.tar.gz /usr/local/java
  ```

  ### EXPOSE

  暴露容器运行时的监听端口给外部，可以指定端口是监听 TCP 还是 UDP，如果未指定协议，则默认为 TCP。

  ```
  EXPOSE 80 443 8080/tcp
  ```

  ### ENV

  设置容器内环境变量。

  ```
  ENV JAVA_HOME /usr/local/java/jdk-11.0.6/
  ```

  ### CMD

  启动容器时执行的 Shell 命令。在 Dockerfile 中只能有一条 CMD 指令。如果设置了多条 CMD，只有最后一条 CMD 会生效。

  ```
  CMD ehco $JAVA_HOME
  ```

  ### ENTRYPOINT

  启动容器时执行的 Shell 命令，同 CMD 类似，不会被 docker run 命令行指定的参数所覆盖。在 Dockerfile 中只能有一条 ENTRYPOINT 指令。如果设置了多条 ENTRYPOINT，只有最后一条 ENTRYPOINT 会生效。

  ```
  ENTRYPOINT ehco $JAVA_HOME
  ```

  > 如果在 Dockerfile 中同时写了 ENTRYPOINT 和 CMD，并且 CMD 指令不是一个完整的可执行命令，那么 CMD 指定的内容将会作为 ENTRYPOINT 的参数；
  >
  > 如果在 Dockerfile 中同时写了 ENTRYPOINT 和 CMD，并且 CMD 是一个完整的指令，那么它们两个会互相覆盖，谁在最后谁生效

  ### WORKDIR

  为 RUN、CMD、ENTRYPOINT 以及 COPY 和 AND 设置工作目录。

  ```
  WORKDIR /usr/local
  ```

  ### VOLUME

  指定容器挂载点到宿主机自动生成的目录或其他容器。一般的使用场景为需要持久化存储数据时。

  ```
  # 容器的 /var/lib/mysql 目录会在运行时自动挂载为匿名卷，匿名卷在宿主机的 /var/lib/docker/volumes 目录下
  VOLUME ["/var/lib/mysql"]
  ```

- CMD和NETRYPOINT

  > **相同点：**
  >
  > CMD/NETRYPOINT ["command","param"]  # exec模式 推荐使用
  >
  > CMD/NETRYPOINT command param        # shell模式
  >
  > **不同点：**
  >
  > dockerfile中如果使用了NETRYPOINT指令，启动docker容器时需要使用--entrypoint参数才能覆盖dockerfile中的NETRYPOINT指令，而使用CMD设置的指令则可以被docker run后面的参数直接覆盖NETRYPOINT指令可以结合CMD指令使用，也可以单独使用，而CMD指令只能单独使用
  >
  > **NETRYPOINT和CMD使用场景：**
  >
  > CMD：希望镜像足够灵活
  >
  > NETRYPOINT：镜像只执行单一的具体程序，并且不希望用户在执行docker run时覆盖默认程序

- ADD和COPY

  > **COPY：**
  >
  > 只支持基本的文件和文件夹拷贝功能  # 推荐使用
  >
  > **ADD：**
  > 支持更多文件来源类型，比如自动提取tar包，并且可以支持源文件为URL格式

  参考文档： https://docs.docker.com/engine/reference/builder/

## Docker安全

> 基于内核的若隔离系统如何保障安全性呢？
>
> docker是基于linux内核的namespace技术实现的资源隔离的，所有的容器都共享主机的内核
>
> docker自身安全性改进
>
> docker自身是基于linux的多种namespace实现的
>
> User Namespace： 主要用来做容器内用户和主机的用户的隔离，使用容器中的root用户映射到主机上的非root用户
>
> 在私有镜像仓库安装安全扫描组件，对上传镜像进行检查
>
> 在拉取镜像时，要确保从受信任的镜像仓库拉取，并且与镜像仓库通信使用https协议
>
> 加强内核安全和管理
>
> 宿主机及时升级内核漏洞

## Docker组件

```bash
[root@localhost bin]# ls  | grep -e docker.* -e container.* -e ctr -e runc
containerd
containerd-shim
containerd-shim-runc-v1
containerd-shim-runc-v2
ctr
docker
docker-compose
dockerd
dockerd-rootless-setuptool.sh
dockerd-rootless.sh
docker-init
docker-proxy
genl-ctrl-list
nl-tctree-list
rootlesskit-docker-proxy
runc
runcon
truncate
```

- **docker**

  > 是一个二进制文件，对用户可见的操作形式为docker命令，通过docker命令可以完成所有的docker客户端与服务端的通信
  >
  > **docker客户端与服务端的交互过程：**
  >
  > docker组件向服务端发送请求后，服务端根据请求执行具体的动作并将结果返回给docker，docker解析服务端返回的结果，并将结果通过命令行标准输出展示给用户

- **dockerd**

  > 用来接收客户端发送的请求，执行具体的处理任务，处理完成后将结果返回给客户端
  >
  > docker客户端和服务端的通信形式必须保持一致(默认套接字)，否则将无法通信，如果想要通过远程方式dockerd可以早dockerd启动的时候添加-H参数指定监听的host和prot

- **docker-init**

  > 在容器内部，当业务进程没有回收子进程的能力时，在执行docekr run启动容器时可以添加--init参数
  >
  > ![image-20230220152850134](https://gitee.com/root_007/md_file_image/raw/master/202302221512521.png)

- **docker-proxy**

  > 主要是用来做端口映射的，当使用docker run命令启动容器时，如果使用了-p参数docker-proxy组件会把容器内相应的端口映射到主机上，底层依赖iptables实现

- **containerd**

  > 是从docker 1.11版本正式从dockerd中剥离的，完全遵循了OCI标准，并且是完全社区化运营的
  >
  > **作用**：镜像的管理、容器网络相关资源管理、存储相关资源管理、接收dockerd的请求，通过适当的参数调用runc启动容器
  >
  > 包含一个后台常驻进程，默认的socket路径为/run/containerd/containerd.sock，dockerd通过UNIX套接字向containerd发送请求，containerd接收到请求后负责执行相关动作并把执行结果返回给dockerd

- **containerd-shim**

  > 主要作用是将containerd和真正的容器进程进行解耦，使用containerd-shim作为容器进程的父进程，从而实现重启containerd不影响已经启动的容器进程

- **ctr**

  > containerd-ctr,是containerd的客户端，主要用来开发和调制在没有dockerd的环境中，ctr可以直接向containerd守护进程发送造作容器的请求

- **runc**

  > 容器运行时

  ![image-20230220155546483](https://gitee.com/root_007/md_file_image/raw/master/202302221523228.png)

## Docker网络

​	**CNM**： docker发布的容器网络标准，只要满足CNM接口的网络方案都可以接入到docker容器网络中

​	**沙箱(Sandbox)**：代表了一系列网络堆栈的配置，其中包含路由信息、网络接口等网络资源的管理

​	**网络(Network)**：一组可以互相通信的接入点，它将多接入点组成一个子网，并且多个接入点之间可以互相通信

​	**Libnetwork**：通过插件的形式为docker提供网络功能，libnetwork是开源的，使用goalng编写，完全遵循CNM网络规范，是CNM的官方实现

- Libnetwork常见的网络模式：

> **null空网络模式**：
>
> 使用docker创建null网络模式的容器时，容器拥有独立的net namespace，在这种模式下，docker除了为容器创建了net namespace外，没有创建任何网卡接口、IP地址、路由等网络配置
>
> **birdge桥接模式**：
>
> 使用birdge网络可以实现容器与容器的互通，可以从一个容器直接通过容器IP访问到另外一个容器，使用bridge网络可以实现主机与容器的互通，在容器内启动的业务，可以从主机直接请求
>
> ```
> linux veth：veth是linux中的虚拟设备接口，都是成对出现的，可以用来连接虚拟网络设备，例如：veth可以用来连通两个net namespace，从而使得两个net namespace之间可以互相访问
> linux bridge：linux bridge是一个虚拟设备，是用来连接网路的设备，可以用来转发两个net namespace内的流量
> ```
>
> **host主机网络模式**：
>
> libnetwork不会为容器创建新的网络配置和net namespace，docker容器中的进程直接共享主机的网络配置，可以直接使用主机的网络信息，除了网络共享主机的网络外，其他的包括进程、文件系统、主机名都是与主机隔离的
>
> **container网络模式**：
>
> 允许一个容器共享另一个容器的网络命名空间，当两个容器需要共享网络，但其他资源仍然需要隔离时就可以使用了

![image-20230221143034435](https://gitee.com/root_007/md_file_image/raw/master/202302221524970.png)

## 镜像多级构建

> 前提：docker大于1.17
> 	在一个Dockerfile中声明多个FROM，选择性的将一个阶段生产的文件拷贝到另外一个阶段中，从而实现最终的镜像只保留需要的环境和文件
> 	停止在特定的构建阶段：在编译阶段调试Dockerfile文件    docker  build --target builder -t  http-server:latest   其中--target为停止阶段
> 	COPY --from: 从远程拷贝文件到自己的镜像中，示例： COPY --form=nginx:latest /etc/nginx/nginx.conf /etc/local/nginx.conf

示例：

```yaml
FROM golang:1.17.10 AS builder
ENV GO111MODULE on
ENV GOPROXY https://goproxy.io
RUN mkdir /app
ADD . /app/
WORKDIR /app
RUN go build -o main .
FROM centos:centos7
RUN mkdir /app
WORKDIR /app
COPY --from=builder /app/ .
RUN chmod +x  main
ENTRYPOINT [ "/bin/sh", "/app/docker-entrypoint.sh" ]

```

## docker-compose

> 在安装docker compose之前，确保机器上已经运行了docker
>
> 在使用docker compose启动容器时，会默认使用docker-compose.yml文件，docker compose模板有三个版本 v1、v2、v3
>
> **docker-compse.yml文件分为三个部分**：
>
> 1. service：服务，服务定义了容器启动的各项配置，像执行docker run命令时传递的容器启动的参数
> 2. network：网络，网络定义了容器的网络配置，像执行docker network create命令创建网络配置
> 3. volumes：数据卷，定义了容器的配置，像执行docker volume create命令创建数据卷

