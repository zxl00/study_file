## 消息队列

> 消息队列（Message Queue），是分布式系统中重要的组件，其通用的使用场景可以简单地描述为：
>
> 当不需要立即获得结果，但是并发量又需要进行控制的时候，差不多就是需要使用消息队列的时候。
>
> 解决问题：
>
> 1. 应用解耦：多应用间通过消息队列对同一消息进行处理，避免调用接口失败导致整个过程失败
> 2. 异步处理：多应用对消息队列中同一消息进行处理，应用间并发处理消息，相比串行处理，减少处理时间
> 3. 流量削峰：广泛应用于秒杀或抢购活动中，避免流量过大导致应用系统挂掉的情况

## 常见消息队列

> - RabbitMQ
> - RocketMQ
> - ActiveMQ
> - Kafka

## kafka介绍

https://kafka.apache.org/documentation/#gettingStarted

## kafka简单入门

#### 生产者

> kafka是一个消息队列，把消息放到这个队列里的叫做**生产者**

#### 消费者

> 从队列里边消费的叫**消费者**

![image-20230327195357342](https://gitee.com/root_007/md_file_image/raw/master/202303271953444.png)

#### topic(主题)

> 一个消息中间件，队列不单单只有一个，我们往往有多个多列，而我们生产者和消费者就需要知道：数据应该丢给那个队列，我们需要给队列取名字，topic这就是队列的名字，类似与mysql中表的概念

![image-20230327195700253](https://gitee.com/root_007/md_file_image/raw/master/202303271957310.png)

​	当我们给队列取名字之后，生产者就知道向那个队列中丢消息，消费之也知道从那个队列中取消息。**我们可以有多个生产者向同一队列中丢消息，多个消费者从一个队列中取消息**

![image-20230327195923509](https://gitee.com/root_007/md_file_image/raw/master/202303271959565.png)

#### partition(分区)

> 为了提高一个队列(主题、topic)的吞吐量，kafka会把topic进行分区

![image-20230327200224089](https://gitee.com/root_007/md_file_image/raw/master/202303272002127.png)

> 在实际生产过程中，往往是一个生产者向一个topic的partition(分区)丢数据，消费者向一个topic的partition(分区)中取数据

![image-20230327200527515](https://gitee.com/root_007/md_file_image/raw/master/202303272005570.png)

#### Broker(了解)

> 一台kafka服务器叫做**Broker**，kafka集群就是多台kafka机器

![image-20230327201010882](https://gitee.com/root_007/md_file_image/raw/master/202303272010947.png)

一个topic会分为多个partition，实际上partition会分布在不同的Broker中

![image-20230327201217824](https://gitee.com/root_007/md_file_image/raw/master/202303272012884.png)

#### 备份

> 现在我们已经知道了往topic里边丢数据，实际上这些数据会分到不同的partition上，这些partition存在不同的broker上。分布式肯定会带来问题：“万一其中一台broker(Kafka服务器)出现网络抖动或者挂了，怎么办？”
>
> Kafka是这样做的：我们数据存在不同的partition上，那kafka就把这些partition做**备份**。比如，现在我们有三个partition，分别存在三台broker上。每个partition都会备份，这些备份散落在**不同**的broker上

![image-20230327201738531](https://gitee.com/root_007/md_file_image/raw/master/202303272017590.png)

***备份分区仅仅用作于备份，不做读写。如果某个Broker挂了，那就会选举出其他Broker的partition来作为主分区，这就实现了高可用**

另外值得一提的是：当生产者把数据丢进topic时，我们知道是写在partition上的，那partition是怎么将其持久化的呢？（不持久化如果Broker中途挂了，那肯定会丢数据嘛)。

Kafka是将partition的数据写在**磁盘**的(消息日志)，不过Kafka只允许**追加写入**(顺序访问)，避免缓慢的随机 I/O 操作。

> - Kafka也不是partition一有数据就立马将数据写到磁盘上，它会先**缓存**一部分，等到足够多数据量或等待一定的时间再批量写入(flush)。



#### offset

> Kafka就是用`offset`来表示消费者的消费进度到哪了，每个消费者会都有自己的`offset`。说白了`offset`就是表示消费者的**消费进度**

#### 消费者流程(了解即可)

![image-20230327202417383](https://gitee.com/root_007/md_file_image/raw/master/202303272024448.png)

> 一个消费者消费三个分区，当然也支持多个消费者一起消费，此时我们称**消费组**

![image-20230327202713928](https://gitee.com/root_007/md_file_image/raw/master/202303272027993.png)

> 本来是一个消费者消费三个分区的，现在我们有消费者组，就可以**每个消费者去消费一个分区**（也是为了提高吞吐量）
>
> - 如果消费者组中的某个消费者挂了，那么其中一个消费者可能就要消费两个partition了
>
> - 如果只有三个partition，而消费者组有4个消费者，那么一个消费者会空闲
>
> - 如果多加入一个**消费者组**，无论是新增的消费者组还是原本的消费者组，都能消费topic的全部数据。（消费者组之间从逻辑上它们是**独立**的）
>
> 如果一个消费者组中的某个消费者挂了，那挂掉的消费者所消费的分区可能就由存活的消费者消费。那**存活的消费者是需要知道挂掉的消费者消费到哪了**，此时就需要offset了，offset就不做过多的解释了，可以将其理解为消费消息的经纬度，也就是消费位置

## 实战案例

说明：服务器配置仅供学习使用，生产环境请按需求增加资源

#### 服务器配置

系统：centos7

内存：1G

cpu：1V

磁盘：20G

台数：3台

#### 服务器规划

| 主机名 | IP             |
| ------ | -------------- |
| node1  | 192.168.59.131 |
| node2  | 192.168.59.132 |
| node4  | 192.168.59.134 |

#### 将主机名添加到hosts解析

3台服务器都要执行

```
[root@lvs-backup ~]#  cat >>/etc/hosts<<EOF
192.168.59.131 node1
192.168.59.132 node2
192.168.59.134 node4
EOF
```

#### 下载kafka资源

node1服务器执行

```
[root@lvs-backup ~]# wget https://archive.apache.org/dist/kafka/3.0.0/kafka_2.12-3.0.0.tgz
```

#### 解压kafka资源

node1服务器执行

```
[root@lvs-backup ~]# tar -xvf kafka_2.12-3.0.0.tgz  -C /usr/local/
```

#### 修改配置文件

node1服务器执行

```
[root@lvs-backup config]# cd /usr/local/kafka_2.12-3.0.0/config


[root@lvs-backup config]# grep  -v -e '^#' -e '^$' server.properties  
broker.id=0
num.network.threads=3
num.io.threads=8
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600
log.dirs=/data/kafka
num.partitions=1
num.recovery.threads.per.data.dir=1
offsets.topic.replication.factor=1
transaction.state.log.replication.factor=1
transaction.state.log.min.isr=1
log.retention.hours=168
log.segment.bytes=1073741824
log.retention.check.interval.ms=300000
zookeeper.connect=node1:2181,node2:2181,node4:2181/kafka
zookeeper.connection.timeout.ms=18000
group.initial.rebalance.delay.ms=0


### 创建资源目录

[root@lvs-backup config]# mkdir -p /data/kafka

```

#### node2和node4

node2和node4服务器执行node1相同的操作

注意修改配置文件时，需要修改

```
broker.id=0
```

> 此时的broker.id=0 不能重复，node2可以设置为broker.id=2，node4可以设置为broker.id=3

#### 启动zookeeper

node1、node2、node4都要执行

```
[root@lvs-backup ~]#  cd /usr/local/kafka_2.12-3.0.0
[root@lvs-backup ~]#  ./bin/zookeeper-server-start.sh -daemon  ./config/zookeeper.properties
```

#### 启动kafka

```
[root@lvs-backup ~]#  cd /usr/local/kafka_2.12-3.0.0

[root@lvs-backup ~]#   ./bin/kafka-server-start.sh -daemon ./config/server.properties
```

#### kakfa启动顺序以及关闭顺序

> 启动顺序：
> zookeep -- >  kafka
>
> 关闭顺序  
>
> kafka  -- > zookeep

#### kafka-topics.sh使用

node1或者node2或者node3上执行都可以

参数

> --bootstrap-server  连接zookee
>
> --create  创建主题
>
> --delete  删除主题
>
> --describe  查看主题详情 
>
>
> --list    列出所有主题
>
>
> --alter   修改主题
>
> --topic 操作的topic名称
>
> --partitions  设置分区数
>
> --replication-factor  设置副本数

查看当前服务器中所有topic

```
[root@rs1 kafka_2.12-3.0.0]# ./bin/kafka-topics.sh --bootstrap-server node1:9092 --list 
```

创建topic，名称为first，分区为1 副本为2

```
[root@rs1 kafka_2.12-3.0.0]# ./bin/kafka-topics.sh --bootstrap-server node1:9092 --create --topic first --partitions 1 --replication-factor 2
```

查看刚创建的topic为first的详情

```
[root@rs1 kafka_2.12-3.0.0]# ./bin/kafka-topics.sh --bootstrap-server node1:9092 --topic first --describe 
Topic: first	TopicId: aHSt0K9VTaqlSC4xoeE7Cg	PartitionCount: 1	ReplicationFactor: 2	Configs: segment.bytes=1073741824
	Topic: first	Partition: 0	Leader: 0	Replicas: 0,2	Isr: 0,2
```

修改topic，将分区增加为3，注意分区只能增加不能减少

```
[root@rs1 kafka_2.12-3.0.0]# ./bin/kafka-topics.sh --bootstrap-server node1:9092   --topic first --alter --partitions 3 

```

#### kafka-console-producer.sh使用

生产者

```
[root@rs1 kafka_2.12-3.0.0]# ./bin/kafka-console-producer.sh --bootstrap-server node1:9092 --topic first 
```

消费者

```
[root@rs2 kafka_2.12-3.0.0]# ./bin/kafka-console-consumer.sh  --bootstrap-server node1:9092 --topic first
```

![image-20230327210045566](https://gitee.com/root_007/md_file_image/raw/master/202303272100614.png)

![image-20230327210052848](https://gitee.com/root_007/md_file_image/raw/master/202303272100888.png)

> 在消息发送的过程中，涉及到了两个线程  main线程和sender线程，在main线程中创建了一个双端队列RecordAccumulator。main线程将消息发送给RecordAccumulator，sender线程不断从RecordAccumulator中拉取消息发送到kafka Broker

## kafka配置文件解析

```shell
#broker的全局唯一编号，不能重复
broker.id=0

#用来监听链接的端口，producer或consumer将在此端口建立连接
port=9092

#处理网络请求的线程数量，也就是接收消息的线程数。
#接收线程会将接收到的消息放到内存中，然后再从内存中写入磁盘。
num.network.threads=3

#消息从内存中写入磁盘是时候使用的线程数量。
#用来处理磁盘IO的线程数量
num.io.threads=8

#发送套接字的缓冲区大小
socket.send.buffer.bytes=102400

#接受套接字的缓冲区大小
socket.receive.buffer.bytes=102400

#请求套接字的缓冲区大小
socket.request.max.bytes=104857600

#kafka运行日志存放的路径
log.dirs=/export/servers/logs/kafka

#topic在当前broker上的分片个数
num.partitions=2

#我们知道segment文件默认会被保留7天的时间，超时的话就
#会被清理，那么清理这件事情就需要有一些线程来做。这里就是
#用来设置恢复和清理data下数据的线程数量
num.recovery.threads.per.data.dir=1

#segment文件保留的最长时间，默认保留7天（168小时），
#超时将被删除，也就是说7天之前的数据将被清理掉。
log.retention.hours=168

#滚动生成新的segment文件的最大时间
log.roll.hours=168

#日志文件中每个segment的大小，默认为1G
log.segment.bytes=1073741824

#上面的参数设置了每一个segment文件的大小是1G，那么
#就需要有一个东西去定期检查segment文件有没有达到1G，
#多长时间去检查一次，就需要设置一个周期性检查文件大小
#的时间（单位是毫秒）。
log.retention.check.interval.ms=300000

#日志清理是否打开
log.cleaner.enable=true

#broker需要使用zookeeper保存meta数据
zookeeper.connect=zk01:2181,zk02:2181,zk03:2181

#zookeeper链接超时时间
zookeeper.connection.timeout.ms=6000

#上面我们说过接收线程会将接收到的消息放到内存中，然后再从内存
#写到磁盘上，那么什么时候将消息从内存中写入磁盘，就有一个
#时间限制（时间阈值）和一个数量限制（数量阈值），这里设置的是
#数量阈值，下一个参数设置的则是时间阈值。
#partion buffer中，消息的条数达到阈值，将触发flush到磁盘。
log.flush.interval.messages=10000

#消息buffer的时间，达到阈值，将触发将消息从内存flush到磁盘，
#单位是毫秒。
log.flush.interval.ms=3000

#删除topic需要server.properties中设置delete.topic.enable=true否则只是标记删除
delete.topic.enable=true

#此处的host.name为本机IP(重要),如果不改,则客户端会抛出:
#Producer connection to localhost:9092 unsuccessful 错误!
host.name=kafka01
advertised.host.name=192.168.239.128
```

> 参考文章： https://bbs.huaweicloud.com/blogs/252230
>

## kafka优化

- 硬件配置

  > ssd磁盘
  >
  > 加大带宽
  >
  > 堆内存调整(kafka启动脚本中根据需求更改)

- 其他优化可参考

  https://note.youdao.com/s/X2YaIKMc