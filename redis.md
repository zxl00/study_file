## redis主从复制

### 概念

> 主从复制模型中，有多个redis节点，其中有且仅有一个为主节点master，从节点可以有多个

模型图

![image-20230207150423997](https://gitee.com/root_007/md_file_image/raw/master/202302071504077.png)

### 主从复制过程

![image-20230207153031173](https://gitee.com/root_007/md_file_image/raw/master/202302071530222.png)

​	流程分析

> 1、从节点需要配置配置文件，说明该节点为slave
>
> 2、主节点启动、从节点启动
>
> 3、从节点发起不同请求
>
> 4、主节点执行BGSLAVE
>
> 5、主节点将内存中的数据保存到磁盘
>
> 6、将保存的数据rdb同步到slave的磁盘
>
> 7、从节点加载数据到内存
>
> 8、从节点返回给主节点同步成功的信号

## 主从复制详解

### 全量同步

> **介绍**：
>
> ​	master节点创建全量的RDB快照文件，通过网络连接发送给slave节点，slave节点加载快照文件恢复数据(前面已经介绍过了)，然后再继续发送复制过程中`复制积压缓冲区`新增命令，使之达到数据一致状态
>
> **使用情景**：
>
> ​	1、从节点与主节点第一次建立连接，从节点使用PSYNC ？ -1来进行全量同步
>
> ​	2、从节点的偏移量(复制积压缓冲区里的位置信息)不在`复制积压缓冲区`中
>
> ​	3、replid不一致的情况
>
> 备注：从节点同步命令： PSYNC <replid> <offset>，其中replid为复制ID(可以理解为master节点的标识) offset为偏移量

### 部分同步

> **介绍**：
>
> ​	故名思意，就是同步缺少的部分
>
> **使用情景**：
>
> ​	1、从节点的偏移量在`复制积压缓冲区`中

### 命令传播

> **介绍**：
>
> ​	如果master-slave节点保持连接，master节点将持续向slave节点发送命令流，以保证master节点数据集发生的改变同样作用在slave节点数据集上，这些命令包含：客户端请求、key过期、数据淘汰以及其他所有引起数据集变更的操作
>
> **使用场景**：  
>
> ​	1、同步完成后
>
> **备注**：当完成同步操作之后，主从将会进入命令传播阶段，此时master、slave数据是一致的，当master执行完新的写命令后，会通过传播程序把改命令追加至`复制积压缓冲区`,然后异步的发送给slave。slave接收命令并执行，同时更新slave维护的偏移量(offset)

### 复制积压缓冲区

![image.png](https://gitee.com/root_007/md_file_image/raw/master/202302091435771.png)

> - 图示Redis角色为Master，其复制ID（replid）为xxxx，当前的复制偏移量（offset）为1010；
> - 它有一个复制积压缓冲区（backlog），容量（backlog_size）为100，backlog起点相对于offset的偏移量（backlog_off）为1000，当前backlog存储的命令字节数（backlog_hislen）为11个，对应了backlog中[1000,1010]偏移量范围内的字节；
> - offset始终与backlog中最后一个字节的偏移量相同。

![image.png](https://gitee.com/root_007/md_file_image/raw/master/202302091436290.png)

> - Redis-1：replid和offset为默认值，说明它从未与主节点进行过同步操作，所以是进行全量同步；
> - Redis-2：replid主从节点一致，slave_offset>=backlog_off并且slave_offset<offset，说明该从节点丢失的数据可以通过复制积压缓冲区找回，所以可以进行部分同步；
> - Redis-3：replid主从节点一致，slave_offset<backlog_off，说明该节点丢失的数据过多，通过复制积压缓冲区无法找回，所以是进行全量同步；
> - Redis-4：replid主从节点一致，之前不是与当前节点进行主从复制，所以是进行全量同步；

## 集群介绍

### 主从复制模式

### 哨兵模式

#### 概念

> 哨兵模式是一种特殊的模式，首先redis提供了哨兵的命令，哨兵是一个独立的进程，作为进程，它会独立运行，其原理是哨兵通过发送命令，等待redis服务器相应，从而监控运行的多个redis实例

#### 单哨兵架构图

![image-20230210143200400](https://gitee.com/root_007/md_file_image/raw/master/202302101432526.png)

> - 哨兵(sentinel)通过发送命令，让redis实例返回监控运行状态，包括主从实例
>
> - 当哨兵检测到master宕机后，会自动将slave切换为master，然后通过`发布订阅模式`通知其他的从实例，修改配置文件，让他们切换master
> - 当旧master上线后，并不会master的身份，而是最为一个新的slave

#### 优缺点

> 优点：
>
> - 存活检测，自动切换主从
> - 主从备份，数据安全
>
> 缺点：
>
> - 哨兵为单机，冗余性较低
> - 主从模式，数据冗余较多
> - 单master，存在写数据性能问题

#### 多哨兵架构图

![image-20230210144403168](https://gitee.com/root_007/md_file_image/raw/master/202302101444239.png)

> 主要解决哨兵单点问题

故障切换过程

> 假设master节点宕机了，哨兵1(假设为哨兵1)首先检测到这个结果，系统并不会马上进行下线，这仅仅是哨兵1认为的master节点不可用，这个现象为`主观下线`，当其他也检测到master不可用时，并且数量达到了一定阈值，那么哨兵就会对该master节点进行一次投票，投票的结果(选出领头哨兵)，然后有该领头哨兵对master进行`客观下线`，然后从所有slave中选出优先级最高的slave，如果优先级相同，选择数据最新的，如果数据都一样，选择run ID最小的slave作为master

### Cluster模式

#### 概念

> 集群完全去中心化，采用多主多从，所有reids节点彼此互联(ping-pong机制)，每部使用二进制协议优化传输速度和宽带。
>
> 客户端与redis节点直连，不需要中间代理层，客户端不需要连接集群所有节点，连接任一一个节点即可
>
> 每个分区都是由一个master和slave组成，分片之间时互相平行的

#### 架构图

![C4E10224-E663-4D5D-BD74-9F4AB9F0C4A1.png](https://gitee.com/root_007/md_file_image/raw/master/202302101714747.png)

> 该架构很容易添加或者删除节点，当需要添加节点时，只需要从其他的节点移除槽，分到新节点即可。

#### 优缺点

> 优点：
>
> - 无中心架构，按照卡槽分布在多个节点，节点间数据共享，可动态调整数据分布
> - 可扩展性，动态添加、删除节点
> - 高可用性，部分节点不可用时，集群任可用，通过slave做备用数据副本，能够实现自动下线
>
> 缺点：
>
> - client实现复杂，提高了开发难度
> - 节点因为某些原因发生阻塞，被判断下线，这种下线是不需要的
> - 数据异步复制，不能保证数据一致性

### 数据丢失问题

### 主从切换过程

> 异步复制导致数据丢失：
>
> - 因为master-->slave的复制时异步的，所以可能部分数据还没复制到slave，master就宕机了，此时这部分数据就丢失了
>
> 解决办法：
>
> - 前提：至少一个slave，数据延迟同步不能超过10s
> - 当数据复制同步延迟超过10s时，master就不会在接收任何请求了。可以尽量避免数据丢失问题

### 脑裂

> 集群脑裂导致数据丢失：
>
> - 脑裂也就是说，某个master所在机器突然脱离的正常的网络，跟其他slave不能连接，但是实际上master还存活着，此时哨兵就会认为master宕机了，然后选举slave切换成master，这个时候集群中就有两个master。此时若某个slave被却换为master，但是可能client还没来得及切换新的master，还继续向旧master写数据就会丢失
>
> 解决办法：
>
> - 前提：至少一个slave，数据延迟同步不能超过10s
> - 当数据复制同步延迟超过10s时，master就不会在接收任何请求了。可以尽量避免数据丢失问题