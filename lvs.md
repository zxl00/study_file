## 负载均衡

> 负载均衡技术Load Balance简称LB是构建大型网站必不可少的架构策略之一。它的目的是把用户的请求分发到多台后端的设备上，用以均衡服务器的负载。我们可以把负载均衡器划分为两大类：硬件负载均衡器和软件负载均衡器，这里重点介绍软件实现方法中的LVS。

## LVS相关术语

> **LB：负载均衡器**
> **HA：高可用**
> VS：调度器
> RS：提供业务服务的服务器的机器
> **CIP：客户端IP，用户IP**
> VIP：vs的外网IP,**虚拟IP**
> DIP：vs的内网IP
> RIP：后端服务的IP，RS的IP

## nginx负载均衡特点

> 1. **工作在网络的7层之上**，可以针对http应用做一些分流的策略，比如针对域名、目录结构；
> 2. Nginx安装和配置比较简单，测试起来比较方便；
> 3. 也可以承担高的负载压力且稳定，一般能支撑超过上万次的并发；
> 4. Nginx可以通过端口检测到服务器内部的故障，比如根据服务器处理网页返回的状态码、超时等等，并且会把返回错误的请求重新提交到另一个节点，不过其中缺点就是不支持url来检测；
> 5. Nginx对请求的**异步**处理可以帮助节点服务器减轻负载；
> 6. Nginx能支持http和Email，这样就在适用范围上面小很多；
> 7. 默认有三种调度算法: 轮询、weight以及ip_hash（可以解决会话保持的问题），还可以支持第三方的fair和url_hash等调度算法；

## LVS工作原理

> 根据**请求报文的目标IP和目标协议及端口将调度转发至某RS**，根据调度算法来挑选RS，LVS是内核级的功能，工作在INPUT链的位置，将发往INPUT的流量进行处理

### 查看内核支持LVS

```shell
[root@localhost boot]# grep -i -C 5 ipvs /boot/config-3.10.0-1062.el7.x86_64  
CONFIG_NETFILTER_XT_MATCH_ESP=m
CONFIG_NETFILTER_XT_MATCH_HASHLIMIT=m
CONFIG_NETFILTER_XT_MATCH_HELPER=m
CONFIG_NETFILTER_XT_MATCH_HL=m
CONFIG_NETFILTER_XT_MATCH_IPRANGE=m
CONFIG_NETFILTER_XT_MATCH_IPVS=m
CONFIG_NETFILTER_XT_MATCH_LENGTH=m
CONFIG_NETFILTER_XT_MATCH_LIMIT=m
CONFIG_NETFILTER_XT_MATCH_MAC=m
CONFIG_NETFILTER_XT_MATCH_MARK=m
CONFIG_NETFILTER_XT_MATCH_MULTIPORT=m
--
CONFIG_IP_VS_IPV6=y
# CONFIG_IP_VS_DEBUG is not set
CONFIG_IP_VS_TAB_BITS=12

#
# IPVS transport protocol load balancing support
#
CONFIG_IP_VS_PROTO_TCP=y
CONFIG_IP_VS_PROTO_UDP=y
CONFIG_IP_VS_PROTO_AH_ESP=y
CONFIG_IP_VS_PROTO_ESP=y
CONFIG_IP_VS_PROTO_AH=y
CONFIG_IP_VS_PROTO_SCTP=y

#
# IPVS scheduler
#
CONFIG_IP_VS_RR=m
CONFIG_IP_VS_WRR=m
CONFIG_IP_VS_LC=m
CONFIG_IP_VS_WLC=m
--
CONFIG_IP_VS_SH=m
CONFIG_IP_VS_SED=m
CONFIG_IP_VS_NQ=m

#
# IPVS SH scheduler
#
CONFIG_IP_VS_SH_TAB_BITS=8

#
# IPVS application helper
#
CONFIG_IP_VS_FTP=m
CONFIG_IP_VS_NFCT=y
CONFIG_IP_VS_PE_SIP=m
```

> 带有IPVS的就是支持lvs

## LVS工作模式简介

> nat：修改请求报文的目标ip，多目标ip的DNAT
> **dr：操作封装新的mac地址**
> tun：在原请求IP报文之外添加一个ip首部
> fullnat：修改请求报文的源和目标ip

## NAT工作模式详解

本质是多目标ip的DNAT，通过将请求报文中的目标地址和目标端口修改为某挑选出的RS的RIP和port实现转发

![](https://gitee.com/root_007/md_file_image/raw/master/202303151721670.gif)



![image-20230315172150103](https://gitee.com/root_007/md_file_image/raw/master/202303151721192.png)

> 1、RIP和DIP应在同一个IP网络，且应使用私网地址，RS的网关要指向DIP
> 2、请求报文和响应报文都必须经由vs转发，所以vs容易成为瓶颈
> 3、支持端口映射，可修改请求报文的目标port
> 4、vs必须是linux系统，rs可以是任意系统

访问流量说明：

![lvs-nat](https://gitee.com/root_007/md_file_image/raw/master/202303151726296.png)

> 当用户请求到达LVS，此时请求的数据报文会先到内核空间的PREROUTING链。 此时报文的源IP为CIP，目标IP为VIP。
> PREROUTING检查发现数据包的目标IP是本机，将数据包送至INPUT链。
> LVS比对数据包请求的服务是否为集群服务，若是，修改数据包的目标IP地址为后端服务器IP，然后将数据包发至POSTROUTING链。 此时报文的源IP为CIP，目标IP为RIP。
> POSTROUTING链通过选路，将数据包发送给RS
> RS比对发现目标为自己的IP，开始构建响应报文发回给LVS。 此时报文的源IP为RIP，目标IP为CIP。
> LVS在响应客户端前，此时会将源IP地址修改为自己的VIP地址，然后响应给客户端。 此时报文的源IP为VIP，目标IP为CIP。

### iptables简单介绍

> PREROUTING : 路由前
> INPUT : 数据包流入口
> FORWARD : 转发管卡
> OUTPUT : 数据包出口
> POSTROUTING : 路由后

## DR工作模式详解

直接路由，**lvs默认模式应用最广泛的**，通过请求报文重新封装一个mac首部进行转发，源mac是DIP所在的接口的mac地址，目标mac是某挑选出的RS的RIP所在接口的mac地址，源ip/port，以及目标ip/port保持不变

![](https://gitee.com/root_007/md_file_image/raw/master/202303151734220.gif)



> RS的RIP可以使用私网地址，也可以是公网地址，RIP与DIR在同一IP网络，RIP的网关不能指向DIP，以确保响应报文不会经由director(lvs服务器)
> RS和lvs要在同一个物理网络
> 请求报文要经由lvs，但响应报文不经由lvs，而由rs直接发往client
> **不支持端口映射(端口不能修改)**
> 无需开启ip_forward
> rs可以是任意系统	

访问流量说明：

![linux-dr](https://gitee.com/root_007/md_file_image/raw/master/202303151739387.png)

> 1、首先用户用CIP请求VIP。
> 2、根据上图可以看到,**不管是lvs还是RS上都需要配置相同的VIP**,那么当用户请求到达我们的集群网络的前端路由器的时候,请求数据包的源地址为CIP目标地址为VIP,此时路由器会发广播问谁是VIP,那么我们集群中所有的节点都配置有VIP,此时谁先响应路由器那么路由器就会将用户请求发给谁,这样一来我们的集群系统是不是没有意义了,那我们可以在网关路由器上配置静态路由指定VIP就是lvs,或者使用一种机制**不让RS 接收来自网络中的ARP地址解析请求**,这样一来用户的请求数据包都会经过lvs。
> 3、当用户请求到达lvs，此时请求的数据报文会先到内核空间的PREROUTING链。 此时报文的源IP为CIP，目标IP为VIP。
>  4、PREROUTING检查发现数据包的目标IP是本机，将数据包送至INPUT链。
>  5、IPVS比对数据包请求的服务是否为集群服务，若是，将请求报文中的源MAC地址修改为DIP的MAC地址，将目标6、MAC地址修改RIP的MAC地址，然后将数据包发至POSTROUTING链。 此时的源IP和目的IP均未修改，仅修改了源MAC地址为DIP的MAC地址，目标MAC地址为RIP的MAC地址
>  7、由于DS和RS在同一个网络中，所以是通过二层来传输。POSTROUTING链检查目标MAC地址为RIP的MAC地址，那么此时数据包将会发至RS。
>  8、RS发现请求报文的MAC地址是自己的MAC地址，就接收此报文。处理完成之后，将响应报文通过lo接口传送给eth0网卡然后向外发出。 此时的源IP地址为VIP，目标IP为CIP
> 9、响应报文最终送达至客户端。

## TUN工作模式详解

不修改报文IP首部(源IP为CIP，目标IP为VIP)，而在原IP报文之外在封装一个IP首部(源IP为DIP，目标IP为RIP)，将报文发往挑选出的目标RS，RS直接响应给客户端(源IP为VIP，目标IP为CIP)

![](https://gitee.com/root_007/md_file_image/raw/master/202303161631746.gif)

> 1. RIP和DIP可以不处在同一物理网络中，RS的网关一般不能指向DIP，且RIP可以和公网通信，也就是说集群节点可以跨互联网实现，DIP、VIP、RIP可以是公网地址
> 2. RS的tun接口上需要配置VIP，以便接收lvs转发过来的数据包，以及作为响应报文源ip
> 3. lvs转发给RS时需要借助隧道，隧道外层IP头部的源IP是DIP，目标IP是RIP，而RS响应给客户端的IP头部是根据隧道内层的IP头分析得到的，源IP是VIP，目标IP是CIP
> 4. 请求报文要经由lvs，但响应不经由lvs，响应有RS自己完成
> 5. 不支持端口映射RS系统需要支持隧道功能

访问流量说明：

![lvs-tun](https://gitee.com/root_007/md_file_image/raw/master/202303161635440.png)

> 1. 当用户请求到达Director Server，此时请求的数据报文会先到内核空间的PREROUTING链。 此时报文的源IP为CIP，目标IP为VIP 。
> 2. PREROUTING检查发现数据包的目标IP是本机，将数据包送至INPUT链。
> 3. IPVS比对数据包请求的服务是否为集群服务，若是，在请求报文的首部再次封装一层IP报文，封装源IP为为DIP，目标IP为RIP。然后发至POSTROUTING链。 此时源IP为DIP，目标IP为RIP。
> 4. POSTROUTING链根据最新封装的IP报文，将数据包发至RS（因为在外层封装多了一层IP首部，所以可以理解为此时通过隧道传输）。 此时源IP为DIP，目标IP为RIP。
> 5. RS接收到报文后发现是自己的IP地址，就将报文接收下来，拆除掉最外层的IP后，会发现里面还有一层IP首部，而且目标是自己的lo接口VIP，那么此时RS开始处理此请求，处理完成之后，通过lo接口送给eth0网卡，然后向外传递。 此时的源IP地址为VIP，目标IP为CIP

## DR模式实战案例

### 服务器及IP规划

| 主机名 | IP                                      | 用途          |
| ------ | --------------------------------------- | ------------- |
| lvs    | 192.168.59.130、**VIP：192.168.59.135** | 调度器lvs     |
| rs1    | 192.168.59.131                          | 后端服务器RS1 |
| rs2    | 192.168.59.132                          | 后端服务器RS2 |
| client | 192.168.59.133                          | client        |

### 系统信息

> 系统：centos7
>
> 内存：1G
>
> CPU：1核
>
> 磁盘：20G

[^说明]: 该服务器信息仅供测试学习使用配置

### 部署详细步骤

- **所有服务器上都要执行**

```yaml
[root@localhost ~]#  yum install  -y  vim iotop bc gcc gcc-c++ glibc glibc-devel pcre pcre-devel openssl  openssl-devel zip unzip zlib-devel   net-tools lrzsz tree ntpdate telnet lsof tcpdump wget libevent libevent-devel git

[root@localhost ~]#  systemctl stop firewalld && systemctl disable firewalld

[root@localhost ~]#  sed -i 's/enforcing/disabled/' /etc/selinux/config  && setenforce 0

[root@localhost ~]#  iptables -F && iptables -X && iptables -Z
```

- **lvs服务器上执行**

  ```yaml
  ### 设置主机名
  [root@localhost ~]#  hostnamectl set-hostname lvs
  ### 使主机名生效
  [root@localhost ~]#  exec bash
  ### 安装ipvsadm
  [root@lvs ~]#  yum install ipvsadm -y
  ### 配置vip，将其配置在ens33:0上，这个ens33:0 你可以自定义
  [root@lvs ~]#  ifconfig ens33:0 192.168.59.135 broadcast 192.168.59.135 netmask 255.255.255.255 up
  ### 设置路由
  [root@lvs ~]#  route add -host 192.168.59.135 dev ens33:0
  ### 开启转发
  ### [root@lvs ~]#  echo "1" >/proc/sys/net/ipv4/ip_forward
  ### 添加转发规则
  [root@lvs ~]#  ipvsadm -A -t 192.168.59.135:80 -s rr
  [root@lvs ~]#  ipvsadm -a -t 192.168.59.135:80 -r 192.168.59.131:80 -g -w 1
  [root@lvs ~]#  ipvsadm -a -t 192.168.59.135:80 -r 192.168.59.132:80 -g -w 1
  
  ```

  > -A:添加一个虚拟服务，使用ip地址、端口号、协议来唯一定义一个虚拟服务 
  >
  > -t:使用TCP服务，该参数后需加主机与端口信息 
  >
  > -s:指定lvs的调度算法 
  >
  > rr:轮询算法 
  >
  > -a:添加一台真实服务器 
  >
  > -r:设置真实服务器IP与端口 
  >
  > -g:设置lvs工作模式为DR直连路由 
  >
  > -w:指定真实服务器权重

  查看当前lvs服务器中ipvsadm转发规则

  ```yaml
  [root@lvs ~]# ipvsadm 
  IP Virtual Server version 1.2.1 (size=4096)
  Prot LocalAddress:Port Scheduler Flags
    -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
  TCP  lvs:http rr
    -> 192.168.59.131:http          Route   1      0          0         
    -> 192.168.59.132:http          Route   1      0          0   
  ```

- **rs1服务器上执行**

  ```yaml
  ### 设置主机名
  [root@localhost ~]#  hostnamectl set-hostname rs1
  ### 使主机名生效
  [root@localhost ~]#  exec bash
  ### 安装httpd，当然也可以按照nginx
  [root@rs1 ~]#  yum install httpd -y
  ### 填写测试数据为主机名，为了分辨转发到那台rs
  [root@rs1 ~]#  echo rs1 > /var/www/html/index.html
  ### 启动httpd
  [root@rs1 ~]#  systemctl start httpd
  ### 设置VIP
  [root@rs1 ~]#  ifconfig lo:0 192.168.59.135 broadcast 192.168.59.135 netmask 255.255.255.255 up
  ### 添加路由表
  [root@rs1 ~]#  route add -host 192.168.59.135 dev lo:0
  ### 关闭arp解析
  [root@rs1 ~]#  echo "1" >/proc/sys/net/ipv4/conf/lo/arp_ignore
  
  [root@rs1 ~]#  echo "2" >/proc/sys/net/ipv4/conf/lo/arp_announce
  
  [root@rs1 ~]#  echo "1" >/proc/sys/net/ipv4/conf/all/arp_ignore
  
  [root@rs1 ~]#  echo "2" >/proc/sys/net/ipv4/conf/all/arp_announce
  
  [root@rs1 ~]#  sysctl -p
  
  ```

- **rs2服务器上执行**

  ```yaml
  ### 设置主机名
  [root@localhost ~]#  hostnamectl set-hostname rs2
  ### 使主机名生效
  [root@localhost ~]#  exec bash
  ### 安装httpd，当然也可以按照nginx
  [root@rs1 ~]#  yum install httpd -y
  ### 填写测试数据为主机名，为了分辨转发到那台rs
  [root@rs1 ~]#  echo rs2 > /var/www/html/index.html
  ### 启动httpd
  [root@rs1 ~]#  systemctl start httpd
  ### 设置VIP
  [root@rs1 ~]#  ifconfig lo:0 192.168.59.135 broadcast 192.168.59.135 netmask 255.255.255.255 up
  ### 添加路由表
  [root@rs1 ~]#  route add -host 192.168.59.135 dev lo:0
  ### 关闭arp解析
  [root@rs1 ~]#  echo "1" >/proc/sys/net/ipv4/conf/lo/arp_ignore
  
  [root@rs1 ~]#  echo "2" >/proc/sys/net/ipv4/conf/lo/arp_announce
  
  [root@rs1 ~]#  echo "1" >/proc/sys/net/ipv4/conf/all/arp_ignore
  
  [root@rs1 ~]#  echo "2" >/proc/sys/net/ipv4/conf/all/arp_announce
  
  [root@rs1 ~]#  sysctl -p
  ```

### client访问测试

- 在client上执行

  ```http
  [root@localhost ~]# while true;do curl http://192.168.59.135;sleep 1;done
  ```

- 测试结果截图

  ![image-20230316170910955](https://gitee.com/root_007/md_file_image/raw/master/202303161709010.png)

## ipvsadm简单使用

- 语法格式

  > ipvsadm  [参数]

- 常用参数

  | 参数 | 说明                                   |
  | ---- | -------------------------------------- |
  | -A   | 添加一条新的虚拟服务                   |
  | -E   | 编辑虚拟服务                           |
  | -D   | 删除虚拟服务                           |
  | -C   | 清除所有虚拟服务                       |
  | -R   | 恢复虚拟服务规则                       |
  | -S   | 保存虚拟服务规则                       |
  | -a   | 在一个虚拟服务中添加一个新的真实服务   |
  | -e   | 编辑某个真实服务                       |
  | -d   | 删除某个真实服务                       |
  | -L   | 显示内核中的虚拟服务规则               |
  | -Z   | 将准发消息的统计清零                   |
  | -h   | 显示帮助信息                           |
  | -t   | TCP协议的虚拟服务                      |
  | -u   | UDP协议的虚拟服务                      |
  | -f   | 说明是经过iptables标记过的服务类型     |
  | -s   | 使用的调度算法                         |
  | -p   | 持久稳固的服务                         |
  | -M   | 指定客户地址的子网掩码                 |
  | -r   | 真是的服务器                           |
  | -g   | 指定LVS的工作模式为DR                  |
  | -i   | 指定LVS的工作模式为TUN                 |
  | -m   | 指定LVS的工作模式为NAT                 |
  | -w   | 真实服务器的权值                       |
  | -c   | 显示ipvs中目前存在的连接               |
  | -6:  | 如果fwmark用的是ipv6地址需要指定此选项 |

- 常用案例

  添加一个虚拟服务，使用轮询算法：

  ```
  [root@lvs ~]#  ipvsadm -A  -t 192.168.59.135:80 -s rr
  ```

  修改虚拟服务的算法为加权轮询：

  ```
  [root@lvs ~]#  ipvsadm -E -t 192.168.59.135:80 -s wrr
  ```

  删除一个虚拟服务：

  ```
  [root@lvs ~]#  ipvsadm -D -t 192.168.59.135:80 
  ```

  添加一个真实服务器，使用DR模式，设置权重为2：

  ```
  [root@lvs ~]#  ipvsadm -a -t 192.168.59.135:80 -r 192.168.59.131:80 -g -w 2
  ```

  修改真实服务器的权重为5：

  ```
  [root@lvs ~]#  ipvsadm -e  -t 192.168.59.135:80 -r 192.168.59.131:80 -g -w 5
  ```

  删除一个真实服务器：

  ```
  [root@lvs ~]#  ipvsadm -d -t 192.168.59.135:80 -r 192.168.59.131:80 
  ```

  查看当前虚拟服务和各RS的权重信息：

  ```
  [root@lvs ~]#  ipvsadm  -Ln
  ```

  查看当前ipvs模块中记录的链路信息：

  ```
  [root@lvs ~]#  ipvsadm  -lnc
  ```

  查看当前ipvs模块中的转发情况：

  ```
  [root@lvs ~]#  ipvsadm  -Ln --stats
  ```

## keepalived+lvs实战案例

### 服务器及IP规划

| 主机名     | IP             | 用途            |
| ---------- | -------------- | --------------- |
| lvs-master | 192.168.59.130 | lvs、keepalived |
| lvs-backup | 192.168.59.134 | lvs、keepalived |
| rs1        | 192.168.59.131 | 后端服务器RS1   |
| rs2        | 192.168.59.132 | 后端服务器RS2   |
| client     | 192.168.59.133 | 客户端          |

> **VIP：192.168.59.135**

### 部署详细步骤

- **所有服务器上都要执行**

  ```bash
  [root@localhost ~]#  yum install  -y  vim iotop bc gcc gcc-c++ glibc glibc-devel pcre pcre-devel openssl  openssl-devel zip unzip zlib-devel   net-tools lrzsz tree ntpdate telnet lsof tcpdump wget libevent libevent-devel git
  
  [root@localhost ~]#  systemctl stop firewalld && systemctl disable firewalld
  
  [root@localhost ~]#  sed -i 's/enforcing/disabled/' /etc/selinux/config  && setenforce 0
  
  [root@localhost ~]#  iptables -F && iptables -X && iptables -Z
  ```

- **lvs-master服务器上执行**

  ```
  ### 设置主机名
  [root@localhost ~]#  hostnamectl set-hostname lvs-master
  ### 使主机名生效
  [root@localhost ~]#  exec bash
  ### 安装ipvsadm、keepalived
  [root@lvs ~]#  yum install ipvsadm  keepalived -y
  ### 备份keepalived默认文件
  [root@lvs-master ~]# cp /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.bak
  ### 修改keepalived.conf文件
  
  cat > /etc/keepalived/keepalived.conf <<EOF
! Configuration File for keepalived
  
  global_defs {
     router_id LVS_DEVEL
  }
  
  vrrp_instance VI_1 {
      state MASTER            # 两个 DS，一个为 MASTER 一个为 BACKUP
      interface ens33        # 当前 IP 对应的网络接口，通过 ifconfig 查询
      virtual_router_id 62    # 虚拟路由 ID(0-255)，在一个 VRRP 实例中主备服务器 ID 必须一样
      priority 200            # 优先级值设定：MASTER 要比 BACKUP 的值大
      advert_int 1            # 通告时间间隔：单位秒，主备要一致
      nopreempt
      authentication {        # 认证机制，主从节点保持一致即可
          auth_type PASS
          auth_pass 1111
      }
      virtual_ipaddress {
          192.168.59.135      # VIP，可配置多个
      }
  }
  
  # LB 配置
  virtual_server 192.168.59.135 80  {
      delay_loop 3                    # 设置健康状态检查时间
      lb_algo wrr                      # 调度算法，这里用了 wrr 轮询算法
      lb_kind DR                      # 这里测试用了 Direct Route 模式
      protocol TCP
      persistence_timeout 50  # 会话保持时间，这段时间内，同一ip发起的请求将被转发到同一个realserver
      real_server 192.168.59.131 80 {
          weight 1
          TCP_CHECK {
              connect_timeout 3
              retry 3　　　　　　      # 旧版本为 nb_get_retry 
              delay_before_retry 3　　　
              connect_port 80
          }
      }
      real_server 192.168.59.132 80 {
          weight 1
          TCP_CHECK {
              connect_timeout 3
              retry 3
              delay_before_retry 3
              connect_port 80
          }
      }
  }
  
  EOF
  
  ### 清空自己的ipvsadm规则，防止前期使用过该主机
  [root@lvs-master ~]# ipvsadm -C
  ### 查看路由，如果有vip的路由，需要删除
  [root@lvs-master ~]# route -n
  ### 启动keepalived
  [root@lvs-master ~]# systemctl start keepalived
  ### 查看keepalived启动状态
  [root@lvs-master ~]# systemctl stauts  keepalived
  ### 查看ipvsadm生成的规则
  [root@lvs-master ~]# ipvsadm -Ln
  IP Virtual Server version 1.2.1 (size=4096)
  Prot LocalAddress:Port Scheduler Flags
    -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
  TCP  192.168.59.135:80 wrr
    -> 192.168.59.131:80            Route   1      0          0         
    -> 192.168.59.132:80            Route   1      0          0     
   
   ### 查看VIP
   [root@lvs-master ~]# ip a 
  1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
      link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
      inet 127.0.0.1/8 scope host lo
         valid_lft forever preferred_lft forever
      inet6 ::1/128 scope host 
         valid_lft forever preferred_lft forever
  2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
      link/ether 00:0c:29:e6:67:b6 brd ff:ff:ff:ff:ff:ff
      inet 192.168.59.130/24 brd 192.168.59.255 scope global noprefixroute dynamic ens33
         valid_lft 1131sec preferred_lft 1131sec
      inet 192.168.59.135/32 scope global ens33
         valid_lft forever preferred_lft forever
      inet6 fe80::8025:1574:74a4:6f5b/64 scope link noprefixroute 
         valid_lft forever preferred_lft forever
  3: ens34: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
      link/ether 00:0c:29:e6:67:c0 brd ff:ff:ff:ff:ff:ff
      inet 172.28.96.48/20 brd 172.28.111.255 scope global noprefixroute dynamic ens34
         valid_lft 63865sec preferred_lft 63865sec
      inet6 fe80::611c:9efb:e632:6746/64 scope link noprefixroute 
         valid_lft forever preferred_lft forever
  
  ```
  
  > 主要：修改keepalived.conf文件时，要根据自己的实际情况进行修改哦
  >
  > 特别需要注意的是：
  >
  > interface ens33  根据自己的网卡进行修改，可以是自己规划IP时的那个网卡，或者自己新建一个虚拟网卡
  >
  > state MASTER 这个需要根据该服务器是keepalived的主还是备来决定
  >
  > priority 200 这个值也是的，master的值 > backup的值
  >
  > 当我们在lvs-master中查看vip时，发现有vip的存在  **inet 192.168.59.135/32 scope global ens33**
  
- **lvs-backup服务器上执行**

  ```
  ### 设置主机名
  [root@localhost ~]#  hostnamectl set-hostname lvs-backup
  ### 使主机名生效
  [root@localhost ~]#  exec bash
  ### 安装ipvsadm、keepalived
  [root@lvs-backup ~]# yum install ipvsadm  keepalived -y
  ### 备份keepalived默认文件
  [root@lvs-backup ~]# cp /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.bak
  ### 修改keepalived.conf文件
  
  cat > /etc/keepalived/keepalived.conf <<EOF
  ! Configuration File for keepalived
  
  global_defs {
     router_id LVS_DEVEL
  }
  
  vrrp_instance VI_1 {
      state BACKUP            # 两个 DS，一个为 MASTER 一个为 BACKUP
      interface ens33        # 当前 IP 对应的网络接口，通过 ifconfig 查询
      virtual_router_id 62    # 虚拟路由 ID(0-255)，在一个 VRRP 实例中主备服务器 ID 必须一样
      priority 100            # 优先级值设定：MASTER 要比 BACKUP 的值大
      advert_int 1            # 通告时间间隔：单位秒，主备要一致
      nopreempt
      authentication {        # 认证机制，主从节点保持一致即可
          auth_type PASS
          auth_pass 1111
      }
      virtual_ipaddress {
          192.168.59.135      # VIP，可配置多个
      }
  }
  
  # LB 配置
  virtual_server 192.168.59.135 80  {
      delay_loop 3                    # 设置健康状态检查时间
      lb_algo wrr                      # 调度算法，这里用了 rr 轮询算法
      lb_kind DR                      # 这里测试用了 Direct Route 模式
      protocol TCP
      persistence_timeout 50  # 会话保持时间，这段时间内，同一ip发起的请求将被转发到同一个realserver
      real_server 192.168.59.131 80 {
          weight 1
          TCP_CHECK {
              connect_timeout 3
              retry 3　　　　　　      # 旧版本为 nb_get_retry 
              delay_before_retry 3　　　
              connect_port 80
          }
      }
      real_server 192.168.59.132 80 {
          weight 1
          TCP_CHECK {
              connect_timeout 3
              retry 3
              delay_before_retry 3
              connect_port 80
          }
      }
  }
  
  EOF
  
  ### 清空自己的ipvsadm规则，防止前期使用过该主机
  [root@lvs-backup ~]# ipvsadm -C
  ### 查看路由，如果有vip的路由，需要删除
  [root@lvs-backup ~]# route -n
  ### 启动keepalived
  [root@lvs-backup ~]# systemctl start keepalived
  ### 查看keepalived启动状态
  [root@lvs-backup ~]# systemctl stauts  keepalived
  ### 查看ipvsadm生成的规则
  [root@lvs-backup ~]# ipvsadm -Ln
  IP Virtual Server version 1.2.1 (size=4096)
  Prot LocalAddress:Port Scheduler Flags
    -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
  TCP  192.168.59.135:80 wrr
    -> 192.168.59.131:80            Route   1      0          0         
    -> 192.168.59.132:80            Route   1      0          0  
    
  ### 查看VIP
  [root@lvs-backup ~]# ip a 
  1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
      link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
      inet 127.0.0.1/8 scope host lo
         valid_lft forever preferred_lft forever
      inet6 ::1/128 scope host 
         valid_lft forever preferred_lft forever
  2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
      link/ether 00:0c:29:b0:03:cd brd ff:ff:ff:ff:ff:ff
      inet 192.168.59.134/24 brd 192.168.59.255 scope global noprefixroute dynamic ens33
         valid_lft 1188sec preferred_lft 1188sec
      inet6 fe80::3ab8:22:8b13:b1be/64 scope link noprefixroute 
         valid_lft forever preferred_lft forever
  3: ens34: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
      link/ether 00:0c:29:b0:03:d7 brd ff:ff:ff:ff:ff:ff
      inet 172.28.96.214/20 brd 172.28.111.255 scope global noprefixroute dynamic ens34
         valid_lft 77200sec preferred_lft 77200sec
      inet6 fe80::afa9:88ee:5731:167d/64 scope link noprefixroute 
         valid_lft forever preferred_lft forever
  
  ```

  > 当我们在lvs-backup中查看vip时，并没有发现有vip的存在

- 手动模拟lvs故障

  ```
  #### 停止lvs-master上的keepalived，此时vip就不存在lvs-master上了，也就代表该服务器提供的功能down了
  [root@lvs-master ~]# systemctl stop keepalived
  ### 再次查看vip是否存在
  [root@lvs-master ~]# ip a | grep 192.168.59.135
  
  ```

  > 此时vip已经不存在lvs-master上了

  在lvs-backup上查看是否有vip

  ```
  [root@lvs-backup ~]# ip a | grep 192.168.59.135
      inet 192.168.59.135/32 scope global ens33
  
  ```

  > 此时vip已经漂移到了lvs-backup上了
  >
  > **别忘记了再次启动lvs-master上的keepalived**
  >
  > systemctl start  keepalived

- **rs1服务器上执行**

  ```
  ### 设置主机名
  [root@localhost ~]#  hostnamectl set-hostname rs1
  ### 使主机名生效
  [root@localhost ~]#  exec bash
  ### 安装httpd，当然也可以按照nginx
  [root@rs1 ~]#  yum install httpd -y
  ### 填写测试数据为主机名，为了分辨转发到那台rs
  [root@rs1 ~]#  echo rs1 > /var/www/html/index.html
  ### 启动httpd
  [root@rs1 ~]#  systemctl start httpd
  ### 设置VIP
  [root@rs1 ~]#  ifconfig lo:0 192.168.59.135 broadcast 192.168.59.135 netmask 255.255.255.255 up
  ### 添加路由表
  [root@rs1 ~]#  route add -host 192.168.59.135 dev lo:0
  ### 关闭arp解析
  [root@rs1 ~]#  echo "1" >/proc/sys/net/ipv4/conf/lo/arp_ignore
  
  [root@rs1 ~]#  echo "2" >/proc/sys/net/ipv4/conf/lo/arp_announce
  
  [root@rs1 ~]#  echo "1" >/proc/sys/net/ipv4/conf/all/arp_ignore
  
  [root@rs1 ~]#  echo "2" >/proc/sys/net/ipv4/conf/all/arp_announce
  
  [root@rs1 ~]#  sysctl -p
  
  ```

- **rs2服务器上执行**

  ```
  ### 设置主机名
  [root@localhost ~]#  hostnamectl set-hostname rs2
  ### 使主机名生效
  [root@localhost ~]#  exec bash
  ### 安装httpd，当然也可以按照nginx
  [root@rs1 ~]#  yum install httpd -y
  ### 填写测试数据为主机名，为了分辨转发到那台rs
  [root@rs1 ~]#  echo rs2 > /var/www/html/index.html
  ### 启动httpd
  [root@rs1 ~]#  systemctl start httpd
  ### 设置VIP
  [root@rs1 ~]#  ifconfig lo:0 192.168.59.135 broadcast 192.168.59.135 netmask 255.255.255.255 up
  ### 添加路由表
  [root@rs1 ~]#  route add -host 192.168.59.135 dev lo:0
  ### 关闭arp解析
  [root@rs1 ~]#  echo "1" >/proc/sys/net/ipv4/conf/lo/arp_ignore
  
  [root@rs1 ~]#  echo "2" >/proc/sys/net/ipv4/conf/lo/arp_announce
  
  [root@rs1 ~]#  echo "1" >/proc/sys/net/ipv4/conf/all/arp_ignore
  
  [root@rs1 ~]#  echo "2" >/proc/sys/net/ipv4/conf/all/arp_announce
  
  [root@rs1 ~]#  sysctl -p
  ```

- client服务器上测试

  ```
  [root@localhost ~]# while true;do curl http://192.168.59.135;sleep 1;done
  rs2
  rs1
  rs2
  rs1
  rs2
  
  ```

  此时也可以再次模拟故障，停止lvs-master上的keepalived，或者直接关机，再次在client上进行访问

官网介绍参数：
https://www.keepalived.org/manpage.html