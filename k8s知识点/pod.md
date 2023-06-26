## pod介绍

> Pod 是一组紧密关联的`容器集合`，它们共享 PID、IPC、Network 和 UTS namespace，是Kubernetes 调度的`基本单位`。Pod 的设计理念是支持多个容器在一个 Pod 中共享网络和文件系统，可以通过进程间通信和文件共享这种简单高效的方式组合完成服务。我们知道容器本质上就是进程，那么 Pod 实际上就是进程组了，只是这一组进程是作为一个整体来进行调度的。
>
> Pod 还可以包含在 Pod 启动期间运行的 [Init 容器](https://v1-25.docs.kubernetes.io/zh-cn/docs/concepts/workloads/pods/init-containers/)。 你也可以在集群支持[临时性容器](https://v1-25.docs.kubernetes.io/zh-cn/docs/concepts/workloads/pods/ephemeral-containers/)的情况下， 为调试的目的注入临时性容器。

![kubernetes pod](https://gitee.com/root_007/md_file_image/raw/master/202306261040129.png)

## pod创建流程

![k8s pod process](https://gitee.com/root_007/md_file_image/raw/master/202306261041196.png)

> - 用户通过 REST API 创建一个 Pod
> - apiserver 将其写入 etcd
> - scheduluer 检测到未绑定 Node 的 Pod，开始调度并更新 Pod 的 Node 绑定
> - kubelet 检测到有新的 Pod 调度过来，通过 container runtime 运行该 Pod
> - kubelet 通过 container runtime 取到 Pod 状态，并更新到 apiserver 中

## pod联网

> 每个 Pod 都在每个地址族中获得一个唯一的 IP 地址。 Pod 中的每个容器共享网络名称空间，包括 IP 地址和网络端口。 **Pod 内**的容器可以使用 `localhost` 互相通信。 当 Pod 中的容器与 **Pod 之外**的实体通信时，它们必须协调如何使用共享的网络资源（例如端口）。
>
> 在同一个 Pod 内，所有容器共享一个 IP 地址和端口空间，并且可以通过 `localhost` 发现对方。 他们也能通过如 SystemV 信号量或 POSIX 共享内存这类标准的进程间通信方式互相通信。 不同 Pod 中的容器的 IP 地址互不相同，如果没有特殊配置，就无法通过 OS 级 IPC 进行通信。 如果某容器希望与运行于其他 Pod 中的容器通信，可以通过 IP 联网的方式实现。
>
> Pod 中的容器所看到的系统主机名与为 Pod 配置的 `name` 属性值相同

## 容器特权模式

> 在 Linux 中，Pod 中的任何容器都可以使用容器规约中的 [安全上下文](https://v1-25.docs.kubernetes.io/zh-cn/docs/tasks/configure-pod-container/security-context/)中的 `privileged`（Linux）参数启用特权模式。 这对于想要使用操作系统管理权能（Capabilities，如操纵网络堆栈和访问设备）的容器很有用。

### 安全上下文

安全上下文（Security Context）定义 Pod 或 Container 的特权与访问控制设置。 安全上下文包括但不限于：

- 自主访问控制（Discretionary Access Control）： 基于[用户 ID（UID）和组 ID（GID）](https://wiki.archlinux.org/index.php/users_and_groups) 来判定对对象（例如文件）的访问权限。
- [安全性增强的 Linux（SELinux）](https://zh.wikipedia.org/wiki/安全增强式Linux)： 为对象赋予安全性标签。
- 以特权模式或者非特权模式运行。
- [Linux 权能](https://linux-audit.com/linux-capabilities-hardening-linux-binaries-by-removing-setuid/): 为进程赋予 root 用户的部分特权而非全部特权。

- [AppArmor](https://v1-25.docs.kubernetes.io/zh-cn/docs/tutorials/security/apparmor/)：使用程序配置来限制个别程序的权能。
- [Seccomp](https://v1-25.docs.kubernetes.io/zh-cn/docs/tutorials/security/seccomp/)：过滤进程的系统调用。
- `allowPrivilegeEscalation`：控制进程是否可以获得超出其父进程的特权。 此布尔值直接控制是否为容器进程设置 [`no_new_privs`](https://www.kernel.org/doc/Documentation/prctl/no_new_privs.txt)标志。 当容器满足一下条件之一时，`allowPrivilegeEscalation` 总是为 true：
  - 以特权模式运行，或者
  - 具有 `CAP_SYS_ADMIN` 权能
- readOnlyRootFilesystem：以只读方式加载容器的根文件系统。

#### 案例演示

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo
spec:
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
  volumes:
  - name: sec-ctx-vol
    emptyDir: {}
  containers:
  - name: sec-ctx-demo
    image: busybox:1.28
    command: [ "sh", "-c", "sleep 1h" ]
    volumeMounts:
    - name: sec-ctx-vol
      mountPath: /data/demo
```

> - `runAsUser` 字段指定 Pod 中的所有容器内的进程都使用用户 ID 1000 来运行。`runAsGroup` 字段指定所有容器中的进程都以主组 ID 3000 来运行。 如果忽略此字段，则容器的主组 ID 将是 root（0）
> -  当 `runAsGroup` 被设置时，所有创建的文件也会划归用户 1000 和组 3000。 由于 `fsGroup` 被设置，容器中所有进程也会是附组 ID 2000 的一部分。 卷 `/data/demo` 及在该卷中创建的任何文件的属主都会是组 ID 2000。

进入容器中执行命令

```bash
/ $ id
uid=1000 gid=3000 groups=2000,3000
/ $ ps 
PID   USER     TIME  COMMAND
    1 1000      0:00 sleep 1h
   20 1000      0:00 sh
   51 1000      0:00 ps
/ $ echo "123" > /data/demo/test  
/ $ ls -l /data/demo/test  
-rw-r--r--    1 1000     2000             4 Jun 26 06:00 /data/demo/test
```

####  Container 设置安全性上下文

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: security-context-demo-2
spec:
  securityContext:
    runAsUser: 1000
  containers:
  - name: sec-ctx-demo-2
    image: gcr.io/google-samples/node-hello:1.0
    securityContext:
      runAsUser: 2000
      allowPrivilegeEscalation: false

```

>  Container 设置的安全性配置仅适用于该容器本身，并且所指定的设置在与 Pod 层面设置的内容发生重叠时，会重写 Pod 层面的设置。Container 层面的设置不会影响到 Pod 的卷。