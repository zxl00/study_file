## 防火墙介绍

> linux防火墙是由Netfilter组件提供的，Netfilter是工作在内核空间的，集成在linux内核中
>
> Netfilter是linux2.4.x之后的新一代linux防火墙机制，是linux内核的一个子系统，Netfilter采用模块化涉及，具有良好的扩充性，提供扩展各种网络服务的结构化底层框架，Netfilter与IP协议栈无缝契合，并允许对数据进行过滤、地址转换、处理等操作

## 防火墙分类

> 按照保护范围：
>
> 1. 主机防火墙
> 2. 网络防火墙
>
> 按照实现方式：
>
> 1. 硬件防火墙
> 2. 软件防火墙
>
> 按照网络协议：
>
> 1. 网络层防火墙
> 2. 应用层防火墙

## NETFILTER内核模块查看

```bash
[root@rs2 ~]# cat /boot/config-3.10.0-1160.el7.x86_64 | grep NETFILTER
CONFIG_NETFILTER=y
# CONFIG_NETFILTER_DEBUG is not set
CONFIG_NETFILTER_ADVANCED=y
CONFIG_BRIDGE_NETFILTER=m
CONFIG_NETFILTER_NETLINK=m
CONFIG_NETFILTER_NETLINK_ACCT=m
```

## iptables简单介绍

> 由软件包iptables提供命令行工具，工作在用户空间，用来编写规则，并将写好的规则发送到Netfilter，告诉内核如何处理信息报
>
> 当然还有一些其他的工具用来编写规则并将规则发往Netfilter，如：firewalld，nft
>
> 一般来说。我们使用iptables就可以了

#### 查看iptables版本

```bash
[root@rs2 ~]# iptables --version
iptables v1.4.21

```

#### 查看iptables相关包

```bash
[root@rs2 ~]# rpm -ql iptables 
/etc/sysconfig/ip6tables-config
/etc/sysconfig/iptables-config
/usr/bin/iptables-xml
/usr/lib64/libip4tc.so.0
/usr/lib64/libip4tc.so.0.1.0
/usr/lib64/libip6tc.so.0
/usr/lib64/libip6tc.so.0.1.0
/usr/lib64/libiptc.so.0
/usr/lib64/libiptc.so.0.0.0
/usr/lib64/libxtables.so.10
/usr/lib64/libxtables.so.10.0.0
/usr/lib64/xtables
/usr/lib64/xtables/libip6t_DNAT.so
/usr/lib64/xtables/libip6t_DNPT.so
/usr/lib64/xtables/libip6t_HL.so
/usr/lib64/xtables/libip6t_LOG.so

```

> 其中so结尾的为模块文件

#### iptables组成

- 五表(table)

  表的优先级：raw-->mangle-->nat-->filter

  > filter：过滤规则表，根据预定义的规则过滤符合条件的数据包
  >
  > nat：地址转换规则表
  >
  > mangle：修改数据标记位规则表
  >
  > raw：关闭启用的连接跟踪机制，加快封包穿越防火墙速度
  >
  > security：用于强制访问网络规则，由linux安全模块实现

- 五链(chain)

  > INPUT,OUTPUT,FORWARD,PREROUTING,POSTROUTING

##### 表链对应关系

![](https://gitee.com/root_007/md_file_image/raw/master/202303231446550.png)

##### 数据包过滤匹配流程

![](https://gitee.com/root_007/md_file_image/raw/master/202303231447958.png)

##### 三种报文流向

> 流入本机：PREROUTING-->INPUT-->用户空间进程
> 流出本机：用户空间进程-->OUTPUT-->POSTROUTING
> 转发：PREROUTING-->FORWARD-->POSTROUTING

## 查看表支持的链

- raw表

  ```bash
  [root@rs2 ~]# iptables -t raw -L| grep -i chain
  Chain PREROUTING (policy ACCEPT)
  Chain OUTPUT (policy ACCEPT)
  
  ```

- mangle表

  ```bash
  [root@rs2 ~]# iptables -t mangle -L| grep -i chain
  Chain PREROUTING (policy ACCEPT)
  Chain INPUT (policy ACCEPT)
  Chain FORWARD (policy ACCEPT)
  Chain OUTPUT (policy ACCEPT)
  Chain POSTROUTING (policy ACCEPT)
  
  ```

- nat表

  ```bash
  [root@rs2 ~]# iptables -t nat -L| grep -i chain
  Chain PREROUTING (policy ACCEPT)
  Chain INPUT (policy ACCEPT)
  Chain OUTPUT (policy ACCEPT)
  Chain POSTROUTING (policy ACCEPT)
  
  ```

- filter表

  ```
  [root@rs2 ~]# iptables -t filter -L| grep -i chain
  Chain INPUT (policy ACCEPT)
  Chain FORWARD (policy ACCEPT)
  Chain OUTPUT (policy ACCEPT)
  
  ```

  > filter表为默认表，我们可以 -t filter省略

## iptables命令

> 基本语法帮助： iptables   --help 或者 man  iptables
>
> 模块扩展帮助： man iptables-extensions

#### 语法格式

```powershell
iptables  （选项）  （参数）
iptables -t 表名 <-A/I/D/R> 规则链名 [规则号] <-i/o 网卡名> -p 协议名 <-s 源IP/源子网> --sport 源端口 <-d 目标IP/目标子网> --dport 目标端口 -j 动作

-t 表
-A 链
-s 源地址
-j 动作
```

> -F：从链中删除所有规则
>
> -vnl --line-number：在数字模式下,完整地列出所有规则
>
> -A：将规则添加到链的末尾
>
> -I：插入链;如果没有 ,则作为第一个规则
>
> -D：将规则 从链中删除
>
> -P：设置默认策略:iptables -P INPUT (DROP|ACCEPT)，简单说就是iptables为黑名单还是白名单，默认为黑名单

#### 动作

> ACCEPT ：接收数据包。
> DROP ：丢弃数据包。
> REJECT ： 封包被拒,并且防火墙发送错误消息(默认情况下是 ICMP 端口不可到达息)
> LOG ：(记录) - 关于封包的信息记录到系统日志;继续检查链中的下一项规则

## iptables实战

#### 基本使用

- 拒绝192.168.59.0/24网段的机器访问

```bash
[root@localhost ~]# iptables -AINPUT -s 192.168.59.0/24 -j REJECT  
# 其他服务器(192.168.59.131)访问本机  
[root@rs1 ~]# ping 192.168.59.133 
PING 192.168.59.133 (192.168.59.133) 56(84) bytes of data.
From 192.168.59.133 icmp_seq=1 Destination Port Unreachable
```

> 本机如果是在这个192.168.59.0/24范围内，也将会被拒绝，终端连接也将会断开
>
> 可以使用REJECT，也可以使用DROP

- 拒绝指定IP的机器访问

  ```bash
  [root@localhost ~]# iptables -AINPUT -s 192.168.59.131 -j DROP 
  ### 查看设置的规则
  [root@localhost ~]# iptables -nvL --line-numbers
  Chain INPUT (policy ACCEPT 84 packets, 6288 bytes)
  num   pkts bytes target     prot opt in     out     source               destination         
  1        0     0 DROP       all  --  *      *       192.168.59.131       0.0.0.0/0           
  
  Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
  num   pkts bytes target     prot opt in     out     source               destination         
  
  Chain OUTPUT (policy ACCEPT 58 packets, 5872 bytes)
  num   pkts bytes target     prot opt in     out     source               destination   
  ```

  > 使用192.168.59.131 机器进行ping本机，验证DROP动作
  >
  > [root@rs1 ~]# ping 192.168.59.133 
  > PING 192.168.59.133 (192.168.59.133) 56(84) bytes of data.
  >
  > 可以使用REJECT，自行添加测试，看看有什么不同

- 删除规则

  ```bash
  [root@localhost ~]# iptables -D INPUT 1
  ```

  > 前边看到了 **拒绝指定IP的机器访问**的时候，生成的规则num为1，所以我们删除的时候需要指定该num值
  >
  > 思考：-D INPUT 中这个INPUT是什么？？？

- 插入规则

  前提：先创建最少2个规则

  ```bash
  root@localhost ~]# iptables -nvL --line-numbers
  Chain INPUT (policy ACCEPT 6 packets, 428 bytes)
  num   pkts bytes target     prot opt in     out     source               destination         
  1      337 28308 DROP       all  --  *      *       192.168.59.131       0.0.0.0/0           
  2        0     0 DROP       all  --  *      *       192.168.59.132       0.0.0.0/0           
  
  Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
  num   pkts bytes target     prot opt in     out     source               destination         
  
  Chain OUTPUT (policy ACCEPT 4 packets, 576 bytes)
  num   pkts bytes target     prot opt in     out     source               destination   
  ```

  再次创建一条，将其插入到2号规则前边

  ```bash
  [root@localhost ~]# iptables -IINPUT 2  -s 192.168.59.130 -j DROP 
  [root@localhost ~]# iptables -nvL --line-numbers
  Chain INPUT (policy ACCEPT 6 packets, 428 bytes)
  num   pkts bytes target     prot opt in     out     source               destination         
  1      398 33432 DROP       all  --  *      *       192.168.59.131       0.0.0.0/0           
  2        0     0 DROP       all  --  *      *       192.168.59.130       0.0.0.0/0           
  3        0     0 DROP       all  --  *      *       192.168.59.132       0.0.0.0/0           
  
  Chain FORWARD (policy ACCEPT 0 packets, 0 bytes)
  num   pkts bytes target     prot opt in     out     source               destination         
  
  Chain OUTPUT (policy ACCEPT 4 packets, 576 bytes)
  num   pkts bytes target     prot opt in     out     source               destination    
  ```

- 指定网卡设置规则

  ```bash
  [root@localhost ~]# iptables -A INPUT -i lo -j ACCEPT
  ```

  > 其中 -i 为指定网卡

- 指定协议添加规则

  ```bash
  [root@localhost ~]# iptables -I INPUT 3 -s 192.168.59.130   -p icmp -j DROP
  ```

  > 禁止icmp(ping) 访问，这里其实是使用了隐式模块，因为协议的名字就是模块的名字，所以模块省略了

- 指定IP和端口添加规则

  ```bash
  [root@localhost ~]# iptables -I INPUT -s 192.168.59.130 -p tcp --dport 22   -j DROP 
  ```

  > 源192.168.59.130 禁止访问本机的22端口
  > --dport 目标端口 后边的端口可以写一个或者是连续的 例如 [80:88]
  > --sport 源端口

#### 显示模块使用

简单介绍常用模块

> multiport：以离散的方式定义多端口匹配最多指定15个端口
>
> - -sports port 其中port port1,prot2,port3...  指定多个源端口
>
> - -dports port 其中port port1,prot2,port3...  指定多个目标端口
>
> - --ports prot 其中port port1,prot2,port3...  指定多个源和目标端口
>
> iprange: 指定连续的ip地址范围
>
> - --src-range ip1-ip2 
>
> mac： 指定源mac地址
>
> - --mac-source

案例演示

- multiport

  前提：先启动2个python程序，一个端口为8000 一个为8002

  ```bash
  [root@rs2 ~]# iptables -A INPUT -s 192.168.59.131 -m multiport -p tcp --dports 8000,8002 -j DROP
  ```

- iprange

  ```bash
  
  ```

- mac

  ```bash
  [root@rs2 ~]# iptables -A INPUT  -m mac --mac-source 00:0c:29:a5:b0:7d  -j ACCEPT
  ```

  