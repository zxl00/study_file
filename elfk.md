## 项目背景

> 运维人云需要对系统和业务日志进行精准把控，便于分析系统和业务状态。日志往往分布在不同的服务器上，传统的方式使用跳板机登录服务器一台一台的查看日志，这样既繁琐，效率又低，所以我们需要**集中化日志管理工具**，将不同服务器上的日志统一收集，统一展示，当然最好可以进行绘图分析

## ELK介绍

> elk是一套开源的日志分析系统，由`elasticsearch`+`logstash`+`kibana`组成，它可以实时的收集、处理、展示、分析数据

### 常用架构

> - `datasource`->`logstash`->`elasticsearch`->`kibana`
>
> - `datasource`->`filebeat`->`logstash`-> `elasticsearch`->`kibana`
>
> - `datasource`->`filebeat`->`logstash`->`redis/kafka`->`logstash`-> `elasticsearch`->`kibana`
>
> - `datasource`->`filebeat`->`kafka`->`logstash`->`elasticsearch`->`kibana`(最常用)
>
>   备注： `datasource`为日志数据

### 组件介绍

以最常用为例

> - `filebeat`：轻量级数据收集引擎
> - `kafka`：消息队列
> - `logstash`：数据收集处理引擎。支持动态的从各种数据源搜集数据，并对数据进行过滤、分析、丰富、统一格式等操作，然后存储以供后续使用。
> - `elasticsearch`：分布式搜索引擎。具有高可伸缩、高可靠、易管理等特点。可以用于全文检索、结构化检索和分析，并能将这三者结合起来。`Elasticsearch` 是用Java 基于 `Lucene` 开发
> - `kibana`：可视化化平台。它能够搜索、展示存储在 `Elasticsearch` 中索引数据。使用它可以很方便的用图表、表格、地图展示和分析数据

### 重要组件详细介绍

#### filebeat

架构图

![filebeat](https://gitee.com/root_007/md_file_image/raw/master/202304191353790.png)

工作原理

> 当您启动`Filebeat`时，它会启动一个或 更多的输入，在您指定的日志数据的位置中查找。为了 `Filebeat`定位的每个日志，`Filebeat`启动一个收集器。每个 收集器读取新内容的单个日志，并将新日志数据发送到 `libbeat`，它聚合事件并将聚合的数据发送到配置的`Filebeat`输出

harvseter

> 收割器负责阅读单个文件的内容。采集器逐行读取每个文件，并将内容发送到输出。为每个文件启动一个收集器。收集器负责打开和关闭文件，这意味着在收集器运行时文件描述符保持打开状态。如果文件在获取时被删除或重命名，`Filebeat`会继续读取该文件。这有一个副作用，即磁盘上的空间将被保留，直到收割机关闭。默认情况下，`Filebeat`保持文件打开，直到达到`close_inactive`

input

> 输入负责管理`harvseter`并查找要读取的所有源。
>
> 如果输入类型为`log`，则输入查找驱动器上与定义的glob路径匹配的所有文件，并为每个文件启动收集器。
>
> `Filebeat`目前支持[多种`input`类型](https://www.elastic.co/guide/en/beats/filebeat/7.17/configuration-filebeat-options.html#filebeat-input-types)。每个输入类型可以定义多次。`log`输入检查每个文件，以查看是否需要启动收集器，是否已经在运行，或者是否可以忽略该文件（参见[`ignore_older`](https://www.elastic.co/guide/en/beats/filebeat/7.17/filebeat-input-log.html#filebeat-input-log-ignore-older)）。只有当文件大小在收获器关闭后发生了变化时，才会拾取新行。
>
> 参考文档： https://www.elastic.co/guide/en/beats/filebeat/7.17/filebeat-input-log.html

常用配置文件详解

```yaml
filebeat:
    spool_size: 1024                                    # 最大可以攒够 1024 条数据一起发送出去
    idle_timeout: "5s"                                  # 否则每 5 秒钟也得发送一次
    registry_file: ".filebeat"                          # 文件读取位置记录文件，会放在当前工作目录下。所以如果你换一个工作目录执行 filebeat 会导致重复传输！
    config_dir: "path/to/configs/contains/many/yaml"    # 如果配置过长，可以通过目录加载方式拆分配置
    prospectors:                                        # 有相同配置参数的可以归类为一个 prospector
            fields:
                ownfield: "mac"                         # 类似 logstash 的 add_fields
            paths:
                - /var/log/system.log                   # 指明读取文件的位置
                - /var/log/wifi.log
            include_lines: ["^ERR", "^WARN"]            # 只发送包含这些字样的日志
            exclude_lines: ["^OK"]                      # 不发送包含这些字样的日志
            document_type: "apache"                     # 定义写入 ES 时的 _type 值
            ignore_older: "24h"                         # 超过 24 小时没更新内容的文件不再监听。在 windows 上另外有一个配置叫 force_close_files，只要文件名一变化立刻关闭文件句柄，保证文件可以被删除，缺陷是可能会有日志还没读完
            scan_frequency: "10s"                       # 每 10 秒钟扫描一次目录，更新通配符匹配上的文件列表
            tail_files: false                           # 是否从文件末尾开始读取
            harvester_buffer_size: 16384                # 实际读取文件时，每次读取 16384 字节
            backoff: "1s"                               # 每 1 秒检测一次文件是否有新的一行内容需要读取
            paths:
                - "/var/log/apache/*"                   # 可以使用通配符
            exclude_files: ["/var/log/apache/error.log"]
            input_type: "stdin"                         # 除了 "log"，还有 "stdin"
            multiline:                                  # 多行合并
                pattern: '^[[:space:]]'
                negate: false
                match: after
output:
    ...
```

filebeat多行日志收集

```yaml
parsers:
- multiline:
    type: pattern
    pattern: '^\['
    negate: true
    match: after
```

> 在上述示例中，我们使用 `multiline` 配置选项来指定多行日志的处理方式。具体来说，我们使用 `pattern` 参数指定一个正则表达式，该表达式用于匹配日志记录的第一行。在这里，我们使用 `'^\['` 模式来匹配以方括号开头的行，例如 `'[INFO]'` 或 `'[ERROR]'` 等。
>
> 然后，我们使用 `negate` 参数将模式取反，这意味着任何不匹配模式的行都被视为单独的事件。最后，我们使用 `match` 参数指定将匹配行（即以方括号开头的行）添加到前一个行（即上一个单独的事件）的后面，从而形成一个完整的多行日志事件。
>
> 参考文档： https://www.elastic.co/guide/en/beats/filebeat/7.17/multiline-examples.html

filebeat添加字段

```yaml
filebeat.inputs:
- type: log
  fields_under_root: true
  processors:
  - add_fields:
      fields:
        project: 'project_test_demo'   # 添加的字段
  enabled: true
  paths:
    - /data/logs/*.log

output.kafka:
...
```

> 可选字段，可以指定这些字段将其信息添加到输入，字段可以是标量值、数组、字典或任何嵌套的组合，默认情况下，在此出指定的字段为分组在输出文档的fields字典下，**要存储自定义字段作为顶级字段**，请将`fields_under_root`选择设置为true

filebeat使用target添加字段

```yaml
processors:
  - add_fields:
      target: project
      fields:
        name: myproject
        id: '574734885120952459'
```

效果为：

```JSON
{
  "project": {
    "name": "myproject",
    "id": "574734885120952459"
  }
}
```

> `add_fields`处理器向事件添加附加字段。 字段可以是 标量值、数组、字典或它们的任何嵌套组合。 **如果目标字段已经存在，add_fields处理器将覆盖该字段**。 默认情况下，您指定的字段将分组在fields 事件中的子字典。将字段分组到不同的 子字典，使用target设置。将字段存储为 顶级字段

filebeat 输出到kafka

```yaml
output.kafka:
  # initial brokers for reading cluster metadata
  hosts: ["kafka1:9092", "kafka2:9092", "kafka3:9092"]

  # message topic selection + partitioning
  topic: '%{[fields.log_topic]}'
  partition.round_robin:
    reachable_only: false

  required_acks: 1
  compression: gzip
  max_message_bytes: 1000000
```

> 事件大于 max_message_bytes将被撤销，要避免此问题请确保`filebeat`不会生成大于 `max_message_bytes`

filebeat完整配置文件实例

```yaml
filebeat.inputs:
- type: log
  fields_under_root: true
  processors:
  - add_fields:
      fields:
        project: 'project_test_demo'
  enabled: true
  paths:
    - /data/logs/*.log

output.kafka:
    enabled: true
    hosts: ["192.168.50.38:9092","192.168.50.39:9092","192.168.50.40:9092"]
    topic: "log-demo"

```

filebeat使用script

```yaml
processors:
  - script:
      lang: javascript
      source: >
        var console = require('console');
        function process(event) {
           var logPath =  event.Get("log.file.path");
           console.log("xxxxxxxxxxxxxxxxxxxxxxxxxxxxx",logPath);
        }
```

> 获取文件路径，更多案例请参考**项目实战**章节

#### logstash

工作原理

> `Logstash` 是一个开源的数据收集引擎，可以从各种来源收集数据，并将其转换和输出到不同的目的地。以下是 `Logstash` 的工作原理：
>
> 1. 输入：`Logstash` 从多个输入中收集数据，例如文件、网络套接字、消息队列等。
> 2. 过滤：`Logstash` 将从输入中收集到的数据传递给过滤器，通过过滤器对数据进行处理、转换和丰富等操作，例如解析 `JSON` 数据、增加字段、删除字段、重命名字段等。
> 3. 输出：经过过滤后的数据被发送到指定的输出位置，如 Elasticsearch、Redis、Amazon S3 等。
> 4. 插件：`Logstash` 的强大之处在于其插件架构，用户可以根据需要选择或编写插件来扩展 `Logstash` 的功能。插件可以用于输入、过滤、输出、编码、解码等方面。
> 5. 可靠性：为了保证数据的完整性和可靠性，`Logstash` 提供了一些机制来缓存、重新尝试和重放失败的事件，以便在出现故障时能够恢复数据流。

input

```json
input{
      kafka{
        bootstrap_servers => ["192.168.50.38:9092,192.168.30.39:9092,192.168.50.40:9092"]
        client_id => "log-demo"
        group_id => "logstash-es"
        auto_offset_reset => "latest"
        consumer_threads => 5
        decorate_events => "true"
        topics => ["log-demo"]
        type => "kafka-to-elas"
        codec => json
      }
}
```

> `bootstrap_servers`: 指定 Kafka 集群中某些 Kafka 代理的地址和端口号列表
>
> `client_id`: 指定与 Kafka 代理通信时要使用的客户端 ID。这个 ID 可以用来追踪在 Kafka 代理上运行的 Logstash 实例，**多个客户端(Logstash)同时消费需要设置不同的client_id，注意同一分组的客户端数量≤kafka分区数量**
>
> `group_id`: 指定消费者组的名称，多个消费者可以在同一个组内进行协调以实现负载均衡
>
> `auto_offset_reset`: 当没有可用的偏移量时，设置从哪里开始读取数据。这里我们将其设置为 `latest`，表示从最新的消息开始读取
>
> `consumer_threads`: 指定 Logstash 使用的消费者线程数。如果没有指定，则默认使用 1
>
> `decorate_events`: 将 Kafka 消息转换成 Logstash 的事件对象，并添加一些元数据信息（例如主题、分区和偏移量等）。这里我们将其设置为true
>
> `topics`: 指定要消费的 Kafka 主题列表
>
> `type`: 为 Logstash 事件指定一个类型。这里我们将其设置为 `kafka-to-elas`，以便稍后用于区分其他输入来源
>
> `codec`: 指定消息的编解码器，以及如何将其转换为 Logstash 事件对象。这里我们使用 `json` 编解码器，以便将 JSON 格式的消息转换为Logstash事件对象

logstash filter mutate添加字段

```yaml
filter {
  mutate {
    add_field => {
      "env" => "%{[tags][0]}"
    }
  }
}
```

> 关于取值，使用%{}，其中{}里面使用json取值方法

logstash filter mutate删除字段

```
filter {
  mutate {
    remove_field => ["@version"]
  }
```

> 当我们删除的字段为顶级字段，只有一个值时，可以省略%{}

logstash使用ruby添加字段

```yaml
filter {
  mutate {
    add_field => {
      "env" => "%{[tags][0]}"
    }
  }
  ruby {
    code => "
      hostNameValues = event.get('[host][name]')
      event.set('hostvalue',hostNameValues)
     "
  }
}
```

> 其中`hostvalue`为字段名称，`hostNameValues`为字段的值

![](https://gitee.com/root_007/md_file_image/raw/master/202304191552609.png)

logstash使用if判断

```yaml
input{
      kafka{
        bootstrap_servers => ["192.168.50.38:9092,192.168.30.39:9092,192.168.50.40:9092"]
        client_id => "elk-log"
        group_id => "elk-log"
        auto_offset_reset => "latest"
        consumer_threads => 5
        decorate_events => true
        topics_pattern  => "log.*"
        type => "kafka-to-elas"
        codec => json
      }
}

filter {
  mutate {
    add_field => {
      "env" => "%{[tags][0]}"
    }
    remove_field => ["@version"]
  }
  ruby {
    code => "
      hostNameValues = event.get('[host][name]')
      event.set('hostvalue',hostNameValues)
     "
  }
  if [serviceName] == 'go-log' {
    ruby {
      code => "
        envNew = event.get('[env]')
        event.set('envNew',envNew)
      "
    }

  }
}

output {
  stdout {}
}

```

![image-20230413135653934](https://gitee.com/root_007/md_file_image/raw/master/202304191554953.png)

logstash auto_offset_reset 配置

```yaml
input{
      kafka{
        bootstrap_servers => ["192.168.50.38:9092,192.168.30.39:9092,192.168.50.40:9092"]
        client_id => "elk-log"
        group_id => "elk-log"
        auto_offset_reset => "earliest"
        consumer_threads => 3
        decorate_events => true
        topics_pattern  => "log.*"
        type => "kafka-to-elas"
        codec => json
      }
}

```

> `auto_offset_reset` 是 Kafka Consumer Group 的一个参数，用于指定 Consumer Group 在找不到 Offset 或者 Offset 失效时应该从何处开始读取消息。它的取值范围包括 `earliest`（从最早的消息开始）和 `latest`（从最新的消息开始）

## 项目实战

说明：服务器配置仅供学习使用，生产环境请按需求增加资源

#### 服务器配置

系统：centos7

内存：8G

cpu：4V

磁盘：40G

台数：4台

#### 服务器规划

| IP            | 服务                |
| ------------- | ------------------- |
| 192.168.50.38 | es、kafka、filebeat |
| 192.168.50.39 | es、kafka、filebeat |
| 192.168.50.40 | es、kafka、filebeat |
| 192.168.50.47 | kibana、logstash    |

> es、filebeat、logstash、kibana版本保持一致

#### 组件版本规划

| 组件名称      | 组件版本 |
| ------------- | -------- |
| filebeat      | 7.17.9   |
| kafka         | 3.0.0    |
| logstash      | 7.17.9   |
| elasticsearch | 7.17.9   |
| kibana        | 7.17.9   |

### es集群部署

192.168.50.38、192.168.50.39、192.168.50.40服务器上执行

> 下载地址：https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.17.9-linux-x86_64.tar.gz

192.168.50.38服务器上执行

```bash
### 下载es文件
# wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.17.9-linux-x86_64.tar.gz 

### 创建存放elk的目录
#  mkdir /data/soft -p

### 解压es到指定目录
#  tar -xvf elasticsearch-7.17.9-linux-x86_64.tar.gz  -C /data/soft/

### 修改es配置文件
# cd /data/soft/elasticsearch-7.17.9/
# cat > config/elasticsearch.yml <<EOF

cluster.name: my-application  #集群名称，所有节点必须一致
node.name: node-1  # 节点名称，每个节点必须不一致
path.data: /data/elk/data  # 索引数据存放路径，所有节点建议一致
path.logs: /data/elk/logs  # 日志路径，所有节点建议一致
bootstrap.memory_lock: true  # es启动时，是否锁内存，建议为true
network.host: 192.168.50.38  # 当前es节点的ip地址
http.port: 9200  # 端口号，所有节点建议一致
discovery.seed_hosts: ["192.168.50.38", "192.168.50.39","192.168.50.40"]  # es集群节点直接互相发现，填写所有节点ip:port
cluster.initial_master_nodes: ["node-1"]  # 集群初始化，那个节点为master，这里配置一个节点名称即可，所有节点一致
action.destructive_requires_name: true

EOF

### 创建es所需目录
#  mkdir /data/elk/data -p && mkdir /data/elk/logs -p

### 创建es启动用户为es
# useradd es

### 修改文件所属人
# chown es /data/ -R 

### 修改es所需系统配置
# cat >> /etc/security/limits.conf <<EOF

es             hard           nofile        65535
es             soft           nofile        65535
es                            nproc         4096
es             hard           memlock       unlimited
es             soft           memlock       unlimited

EOF

# cat >>  /etc/sysctl.conf <<EOF

vm.swappiness=1
vm.max_map_count=262144
EOF
# 此时最好重启一下服务器

### 切换为es用户
# su es

### 启动es，可以现在前台启动，看是否报错，没有报错再切换为后台启动
# /data/soft/elasticsearch-7.17.9/bin
#  ./elasticsearch  -d

```

192.168.50.39服务器上执行

```bash
### 下载es文件
# wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.17.9-linux-x86_64.tar.gz 

### 创建存放elk的目录
#  mkdir /data/soft -p

### 解压es到指定目录
#  tar -xvf elasticsearch-7.17.9-linux-x86_64.tar.gz  -C /data/soft/

### 修改es配置文件
# cd /data/soft/elasticsearch-7.17.9/
# cat > config/elasticsearch.yml <<EOF

cluster.name: my-application  #集群名称，所有节点必须一致
node.name: node-2  # 节点名称，每个节点必须不一致
path.data: /data/elk/data  # 索引数据存放路径，所有节点建议一致
path.logs: /data/elk/logs  # 日志路径，所有节点建议一致
bootstrap.memory_lock: true  # es启动时，是否锁内存，建议为true
network.host: 192.168.50.39  # 当前es节点的ip地址
http.port: 9200  # 端口号，所有节点建议一致
discovery.seed_hosts: ["192.168.50.38", "192.168.50.39","192.168.50.40"]  # es集群节点直接互相发现，填写所有节点ip:port
cluster.initial_master_nodes: ["node-1"]  # 集群初始化，那个节点为master，这里配置一个节点名称即可，所有节点一致
action.destructive_requires_name: true

EOF

### 创建es所需目录
#  mkdir /data/elk/data -p && mkdir /data/elk/logs -p

### 创建es启动用户为es
# useradd es

### 修改文件所属人
# chown es /data/ -R 

### 修改es所需系统配置
# cat >> /etc/security/limits.conf <<EOF

es             hard           nofile        65535
es             soft           nofile        65535
es                            nproc         4096
es             hard           memlock       unlimited
es             soft           memlock       unlimited

EOF

# cat >>  /etc/sysctl.conf <<EOF

vm.swappiness=1
vm.max_map_count=262144
EOF
# 此时最好重启一下服务器

### 切换为es用户
# su es

### 启动es，可以现在前台启动，看是否报错，没有报错再切换为后台启动
# /data/soft/elasticsearch-7.17.9/bin
#  ./elasticsearch  -d

```

192.168.50.40服务器上执行

```bash
### 下载es文件
# wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.17.9-linux-x86_64.tar.gz 

### 创建存放elk的目录
#  mkdir /data/soft -p

### 解压es到指定目录
#  tar -xvf elasticsearch-7.17.9-linux-x86_64.tar.gz  -C /data/soft/

### 修改es配置文件
# cd /data/soft/elasticsearch-7.17.9/
# cat > config/elasticsearch.yml <<EOF

cluster.name: my-application  #集群名称，所有节点必须一致
node.name: node-3  # 节点名称，每个节点必须不一致
path.data: /data/elk/data  # 索引数据存放路径，所有节点建议一致
path.logs: /data/elk/logs  # 日志路径，所有节点建议一致
bootstrap.memory_lock: true  # es启动时，是否锁内存，建议为true
network.host: 192.168.50.40   # 当前es节点的ip地址
http.port: 9200  # 端口号，所有节点建议一致
discovery.seed_hosts: ["192.168.50.38", "192.168.50.39","192.168.50.40"]  # es集群节点直接互相发现，填写所有节点ip:port
cluster.initial_master_nodes: ["node-1"]  # 集群初始化，那个节点为master，这里配置一个节点名称即可，所有节点一致
action.destructive_requires_name: true

EOF

### 创建es所需目录
#  mkdir /data/elk/data -p && mkdir /data/elk/logs -p

### 创建es启动用户为es
# useradd es

### 修改文件所属人
# chown es /data/ -R 

### 修改es所需系统配置
# cat >> /etc/security/limits.conf <<EOF

es             hard           nofile        65535
es             soft           nofile        65535
es                            nproc         4096
es             hard           memlock       unlimited
es             soft           memlock       unlimited

EOF

# cat >>  /etc/sysctl.conf <<EOF

vm.swappiness=1
vm.max_map_count=262144
EOF
# 此时最好重启一下服务器

### 切换为es用户
# su es

### 启动es，可以现在前台启动，看是否报错，没有报错再切换为后台启动
# /data/soft/elasticsearch-7.17.9/bin
#  ./elasticsearch  -d

```

> elasticsearch.yml用于配置Elasticsearch
> jvm.options用于配置Elasticsearch JVM设置
> log4j2.properties用于配置Elasticsearch日志

#### 验证es集群

3台es中随便访问一台即可

![image-20230410154648991](https://gitee.com/root_007/md_file_image/raw/master/202304101546070.png)



### kibana部署

> 下载地址：https://artifacts.elastic.co/downloads/kibana/kibana-7.17.9-linux-x86_64.tar.gz

192.168.50.47服务器上执行

```bash
#  wget  https://artifacts.elastic.co/downloads/kibana/kibana-7.17.9-linux-x86_64.tar.gz
#  mkdir /data/soft -p
# tar -xvf kibana-7.17.9-linux-x86_64.tar.gz  -C /data/soft/ 
#  cd  /data/soft/kibana-7.17.9-linux-x86_64
#  cat > config/kibana.yml <<EOF

server.port: 5601
server.host: "192.168.50.47"
server.name: "TPLN_ELK"
elasticsearch.hosts: ["http://192.168.50.38:9200","http://192.168.50.39:9200","http://192.168.50.40:9200",]
kibana.index: ".kibana"
kibana.defaultAppId: "home"
elasticsearch.pingTimeout: 1500
elasticsearch.requestTimeout: 30000
ops.interval: 5000
i18n.locale: "zh-CN"

EOF

#  useradd es

# chown es /data/ -R 

#  cat > /usr/lib/systemd/system/kibana.service <<EOF

[Unit]
Description=kibana
After=network.target
 
[Service]
User=es
ExecStart=/data/soft/kibana-7.17.9-linux-x86_64/bin/kibana
ExecStop=/usr/bin/kill -15 $MAINPID
ExecReload=/usr/bin/kill -HUP $MAINPID
Type=simple
RemainAfterExit=yes
PrivateTmp=true
# file size
LimitFSIZE=infinity
# cpu time
LimitCPU=infinity
# virtual memory size
LimitAS=infinity
# open files
LimitNOFILE=65535
# processes/threads
LimitNPROC=64000
# locked memory
LimitMEMLOCK=infinity
# total threads (user+kernel)
TasksMax=infinity
TasksAccounting=false
 
[Install]
WantedBy=multi-user.target


EOF

#   systemctl start kibana.service










```

#### 访问kibana验证

![image-20230410161056656](https://gitee.com/root_007/md_file_image/raw/master/202304101610736.png)

### kafka集群部署

> 如果需要使用zookeeper部署，请参考另外一篇文章，本次实验，不再使用zookeeper
>
> 下载地址： https://archive.apache.org/dist/kafka/3.0.0/kafka_2.12-3.0.0.tgz

192.168.50.38服务器上执行

```bash
#  wget https://archive.apache.org/dist/kafka/3.0.0/kafka_2.12-3.0.0.tgz
#  tar -xvf kafka_2.12-3.0.0.tgz  -C /data/soft/
#  cd  /data/soft/kafka_2.12-3.0.0
#  cat > config/kraft/server.properties <<EOF

process.roles=broker,controller
node.id=1
controller.quorum.voters=1@192.168.50.38:9093,2@192.168.50.39:9093,3@192.168.50.40:9093
listeners=PLAINTEXT://192.168.50.38:9092,CONTROLLER://192.168.50.38:9093
inter.broker.listener.name=PLAINTEXT
advertised.listeners=PLAINTEXT://192.168.50.38:9092
controller.listener.names=CONTROLLER
listener.security.protocol.map=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT,SSL:SSL,SASL_PLAINTEXT:SASL_PLAINTEXT,SASL_SSL:SASL_SSL
num.network.threads=3
num.io.threads=8
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600
log.dirs=/tmp/kraft-combined-logs
num.partitions=1
num.recovery.threads.per.data.dir=1
offsets.topic.replication.factor=1
transaction.state.log.replication.factor=1
transaction.state.log.min.isr=1
log.retention.hours=168
log.segment.bytes=1073741824
log.retention.check.interval.ms=300000

EOF

#  yum install java-1.8.0-openjdk java-1.8.0-openjdk-devel -y

### 设置集群的uuid    生成了随机字符串：fRSifcmESiqs4nkhNWtoYw
#  ./bin/kafka-storage.sh random-uuid


```

192.168.50.39服务器上执行

```bash
#  wget https://archive.apache.org/dist/kafka/3.0.0/kafka_2.12-3.0.0.tgz
#  tar -xvf kafka_2.12-3.0.0.tgz  -C /data/soft/
#  cd  /data/soft/kafka_2.12-3.0.0
#  cat > config/kraft/server.properties <<EOF

process.roles=broker,controller
node.id=2
controller.quorum.voters=1@192.168.50.38:9093,2@192.168.50.39:9093,3@192.168.50.40:9093
listeners=PLAINTEXT://192.168.50.39:9092,CONTROLLER://192.168.50.39:9093
inter.broker.listener.name=PLAINTEXT
advertised.listeners=PLAINTEXT://192.168.50.39:9092
controller.listener.names=CONTROLLER
listener.security.protocol.map=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT,SSL:SSL,SASL_PLAINTEXT:SASL_PLAINTEXT,SASL_SSL:SASL_SSL
num.network.threads=3
num.io.threads=8
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600
log.dirs=/tmp/kraft-combined-logs
num.partitions=1
num.recovery.threads.per.data.dir=1
offsets.topic.replication.factor=1
transaction.state.log.replication.factor=1
transaction.state.log.min.isr=1
log.retention.hours=168
log.segment.bytes=1073741824
log.retention.check.interval.ms=300000

EOF

#  yum install java-1.8.0-openjdk java-1.8.0-openjdk-devel -y


```

192.168.50.40服务器上执行

```bash
#  wget https://archive.apache.org/dist/kafka/3.0.0/kafka_2.12-3.0.0.tgz
#  tar -xvf kafka_2.12-3.0.0.tgz  -C /data/soft/
#  cd  /data/soft/kafka_2.12-3.0.0
#  cat > config/kraft/server.properties <<EOF

process.roles=broker,controller
node.id=3
controller.quorum.voters=1@192.168.50.38:9093,2@192.168.50.39:9093,3@192.168.50.40:9093
listeners=PLAINTEXT://192.168.50.40:9092,CONTROLLER://192.168.50.40:9093
inter.broker.listener.name=PLAINTEXT
advertised.listeners=PLAINTEXT://192.168.50.40:9092
controller.listener.names=CONTROLLER
listener.security.protocol.map=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT,SSL:SSL,SASL_PLAINTEXT:SASL_PLAINTEXT,SASL_SSL:SASL_SSL
num.network.threads=3
num.io.threads=8
socket.send.buffer.bytes=102400
socket.receive.buffer.bytes=102400
socket.request.max.bytes=104857600
log.dirs=/tmp/kraft-combined-logs
num.partitions=1
num.recovery.threads.per.data.dir=1
offsets.topic.replication.factor=1
transaction.state.log.replication.factor=1
transaction.state.log.min.isr=1
log.retention.hours=168
log.segment.bytes=1073741824
log.retention.check.interval.ms=300000

EOF

#  yum install java-1.8.0-openjdk java-1.8.0-openjdk-devel -y


```

192.168.50.38、192.168.50.39、192.168.50.40服务器上执行

```bash
#  cd  /data/soft/kafka_2.12-3.0.0
#  ./bin/kafka-storage.sh format -t   fRSifcmESiqs4nkhNWtoYw    -c   ./config/kraft/server.properties
#  ./bin/kafka-server-start.sh   -daemon     ./config/kraft/server.properties
```

> 其中`RSifcmESiqs4nkhNWtoYw` 这个字符串，是在192.168.50.38服务器上生成的uuid

### filebeat部署



> 下载地址： https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-7.17.9-linux-x86_64.tar.gz

192.168.50.38、192.168.50.39、192.168.50.40服务器上执行

```bash
### 下载filebeat
#  wget    https://artifacts.elastic.co/downloads/beats/filebeat/filebeat-7.17.9-linux-x86_64.tar.gz

### 解压filebeat
#  tar -xvf filebeat-7.17.9-linux-x86_64.tar.gz  -C /data/soft/

### 进入解压后的文件
#  cd   /data/soft/filebeat-7.17.9-linux-x86_64

### 修改配文件
#   cat > filebeat.yml <<EOF

filebeat.inputs:
- type: log
  tags: ["dev"]
  fields_under_root: true
  processors:
    - add_fields:
        fields:
          serviceName: ""

  processors:
    - script:
        lang: javascript
        source: >
          var console = require('console');
          function process(event) {
             var logPath =  event.Get("log.file.path");
             var fileName = getFileName(logPath);
             var fileServiceName = getServiceName(fileName);
             event.Put('serviceName',fileServiceName);
          }
          function getFileName(o) {
             var pos = o.lastIndexOf('/');
             return o.substring(pos + 1);
          }
          function getServiceName(o) {
             return o.replace(/\.|_/g,'-')
          }
  enabled: true
  paths:
    - /data/logs/*.log
output.kafka:
    enabled: true
    hosts: ["192.168.50.38:9092","192.168.50.39:9092","192.168.50.40:9092"]
    topic: 'log-%{[serviceName]}'
    max_message_bytes: 5242880
    partition.round_robin:
      reachable_only: true
    keep-alive: 120
    required_acks: 1


EOF

### 创建/data/logs目录，便我们实验收集日志
#  mkdir  -p  /data/logs

### 创建日志文件
#   touch /data/logs/{chiness.log,english.log,go.log,math_achievement_detailed.log}

### 在创建的文件随机写入一下字符串，这里实验没有使用多行收集，需要的话按照日志格式自行配置
#   cd    /data/logs
#      echo  `ifconfig | grep inet` >> go.log && echo `date` >> chiness.log 

### 启动filebeat，实验暂时使用前台方式启动，验证没有问题再使用后台启动
#   ./filebeat -e -c filebeat.yml

```

### logstash部署

> 下载地址： https://artifacts.elastic.co/downloads/logstash/logstash-7.17.9-linux-x86_64.tar.gz

192.168.50.47服务器上执行

```bash
###  下载logstash
#  wget    https://artifacts.elastic.co/downloads/logstash/logstash-7.17.9-linux-x86_64.tar.gz

###  解压logstash
#  tar -xvf logstash-7.17.9-linux-x86_64.tar.gz  -C /data/soft/

### 进入解压后的文件
#  cd   /data/soft/logstash-7.17.9

### 修改配置文件
#    cat > config/logstash.conf <<EOF

input{
      kafka{
        bootstrap_servers => ["192.168.50.38:9092,192.168.30.39:9092,192.168.50.40:9092"]
        client_id => "elk-log"
        group_id => "elk-log"
        auto_offset_reset => "latest"
        consumer_threads => 5
        decorate_events => true
        topics_pattern  => "log.*"
        type => "kafka-to-elas"
        codec => json
      }
}

filter {
  mutate {
    add_field => {
      "env" => "%{[tags][0]}"
    }
    remove_field => ["@version"]
  }
  ruby {
    code => "
      hostNameValues = event.get('[host][name]')
      event.set('hostvalue',hostNameValues)
     "
  }
  if [serviceName] == 'go-log' {
    ruby {
      code => "
        envNew = event.get('[env]')
        event.set('envNew',envNew)
      "
    }

  }
}

output{
        elasticsearch{
               hosts => ["192.168.50.38:9200","192.168.50.39:9200","192.168.50.40:9200"]
               index => "log-%{[serviceName]}"
               timeout => 300
          }

}


EOF

### 启动logstash，实验暂时使用前台方式启动  验证没有问题再使用后台启动
#   ./bin/logstash   -f config/logstash.conf
```

### 验证日志系统可用性

访问kibana

![image-20230419164722001](https://gitee.com/root_007/md_file_image/raw/master/202304191647064.png)

再次向日志文件中写入数据(随机一台服务器)

```
# cd /data/logs/
# echo  `ifconfig | grep inet` >> go.log && echo `date` >> chiness.log 
```

> 192.168.50.38 中写入的

在kibana中查看es中索引

![image-20230419164938248](https://gitee.com/root_007/md_file_image/raw/master/202304191649358.png)

![image-20230419165049575](https://gitee.com/root_007/md_file_image/raw/master/202304191650641.png)

> 发现，我们刚在日志文件中添加数据，此时，已经创建的该文件的索引

创建索引模式

![image-20230419165300881](https://gitee.com/root_007/md_file_image/raw/master/202304191653993.png)

![image-20230419165318117](https://gitee.com/root_007/md_file_image/raw/master/202304191653218.png)

![image-20230419165458214](https://gitee.com/root_007/md_file_image/raw/master/202304191654294.png)

![image-20230419165605727](https://gitee.com/root_007/md_file_image/raw/master/202304191656789.png)

> 只有当输入的索引匹配后，才能创建

![image-20230419165642932](https://gitee.com/root_007/md_file_image/raw/master/202304191656034.png)

查看日志

![image-20230419165716112](https://gitee.com/root_007/md_file_image/raw/master/202304191657235.png)

![image-20230419165826179](https://gitee.com/root_007/md_file_image/raw/master/202304191658288.png)

查看自定义的字段

![image-20230419170100552](https://gitee.com/root_007/md_file_image/raw/master/202304191701659.png)

## 完善当前配置

#### 配置es索引生命周期

![image-20230419175600609](https://gitee.com/root_007/md_file_image/raw/master/202304191756723.png)

![image-20230419175618122](https://gitee.com/root_007/md_file_image/raw/master/202304191756232.png)

![image-20230419175930175](https://gitee.com/root_007/md_file_image/raw/master/202304191759286.png)

![image-20230419175956245](https://gitee.com/root_007/md_file_image/raw/master/202304191759378.png)

![image-20230419180120780](C:/Users/TPLN-2169/AppData/Roaming/Typora/typora-user-images/image-20230419180120780.png)

> 满足热阶段后，选择delete阶段，这个需要根据需求来哦，这个可是删除哦

#### 索引引用生命周期

修改logstash配置文件

```yaml
output{
        elasticsearch{
               hosts => ["192.168.50.38:9200","192.168.50.39:9200","192.168.50.40:9200"]
               index => "log-%{[serviceName]}"
               ilm_enabled => true
               ilm_policy => "log-dev-policy"
               timeout => 300
          }

}

```

> 重启一下logstash

#### 查看索引中的策略是否生效

![image-20230419180450219](https://gitee.com/root_007/md_file_image/raw/master/202304191804350.png)

#### es跨域设置

在elasticsearch.yml文件中添加以下配置

```yaml
http.cors.enabled: true
http.cors.allow-origin: "*"
```



#### filebeat设置systemctl启动

```bash
#   cat  > /etc/systemd/system/filebeat.service <<EOF 
[Unit]
Description=Filebeat sends log files to Logstash or directly to Elasticsearch.
Documentation=https://www.elastic.co/guide/en/beats/filebeat/current/index.html
Wants=network-online.target
After=network-online.target

[Service]
User=root
Group=root
Environment="BEAT_LOG_OPTS=-e"
ExecStart=/data/soft/filebeat-7.17.9-linux-x86_64/filebeat -c /data/soft/filebeat-7.17.9-linux-x86_64/filebeat.yml
Restart=always

[Install]
WantedBy=multi-user.target

EOF

#    systemctl daemon-reload
#   systemctl start  filebeat
 
```

> 根据自己的实际情况进行修改`ExecStart`里的内容

#### logstash设置systemctl启动

```bash
#    cat  > /etc/systemd/system/logstash.service <<EOF 
[Unit]
Description=Filebeat sends log files to Logstash or directly to Elasticsearch.
Documentation=https://www.elastic.co/guide/en/beats/filebeat/current/index.html
Wants=network-online.target
After=network-online.target

[Service]
User=root
Group=root
Environment="BEAT_LOG_OPTS=-e"
ExecStart=/data/soft/logstash-7.17.9/bin/logstash -f /data/soft/logstash-7.17.9/config/logstash.conf
Restart=always

[Install]
WantedBy=multi-user.target


EOF

#    systemctl daemon-reload
#   systemctl start  logstash
```

#### elasticsearch优化

> 1. 堆内存调整  `config/jvm.options`中的`Xms`和`Xmx`，根据服务器的内存进行调整，一般为整体可用内存的70%左右

#### elasticsearch根据时间自动创建索引

```yaml
input{
      kafka{
        bootstrap_servers => ["192.168.50.38:9092,192.168.30.39:9092,192.168.50.40:9092"]
        client_id => "elk-log"
        group_id => "elk-log"
        auto_offset_reset => "earliest" 
        consumer_threads => 3
        decorate_events => true
        topics_pattern  => "log.*"
        type => "kafka-to-elas"
        codec => json
      }
}

filter {
  date {
    match => ["timestamp", "yyyy-MM-dd HH:mm:ss"]
  }
  mutate {
    add_field => {
      "env" => "%{[tags][0]}"
    }
    remove_field => ["@version"]
  }
  ruby {
    code => "
      hostNameValues = event.get('[host][name]')
      event.set('hostvalue',hostNameValues)
     "
  }
  if [serviceName] == 'go-log' {
    ruby {
      code => "
        envNew = event.get('[env]')
        event.set('envNew',envNew)
      "
    }

  }
}

output{
        elasticsearch{
               hosts => ["192.168.50.38:9200","192.168.50.39:9200","192.168.50.40:9200"]
               index => "log-%{[serviceName]}-%{+YYYY.MM.dd}"
               timeout => 300
          }

}
```

> logstash完整配置文件，其中在filter过滤器中使用date，将时间信息提取出来，并赋值给timestamp，然后使用 `%{+YYYY.MM.dd}` 格式化字符串作为索引名称，将当前日期作为索引的一部分

效果如下

![image-20230421095630736](https://gitee.com/root_007/md_file_image/raw/master/202304210956785.png)

#### filebeat多行收集

对于`java`项目的日志，很多堆栈信息，如果不使用多行收集，日志查看起来就很困难，其中日志消息内容大致为：

```java
2023-04-14 16:32:42.062 ERROR [english-book-workflow,,,] 6 --- [d2-857851adf5dc] c.a.n.client.config.impl.ClientWorker    : longPolling error :
java.net.UnknownHostException: nacos-headless
	at java.net.AbstractPlainSocketImpl.connect(AbstractPlainSocketImpl.java:184) ~[na:1.8.0_161]
	at java.net.SocksSocketImpl.connect(SocksSocketImpl.java:392) ~[na:1.8.0_161
2023-04-14 16:32:44.063 ERROR [english-book-workflow,,,] 6 --- [d2-857851adf5dc] c.a.n.c.config.http.ServerHttpAgent      : [NACOS IOException httpPost] currentServerAddr: http
java.net.UnknownHostException: nacos-headless
	at java.net.AbstractPlainSocketImpl.connect(AbstractPlainSocketImpl.java:184) ~[na:1.8.0_161]
	at java.net.SocksSocketImpl.connect(SocksSocketImpl.java:392) ~[na:1.8.0_161]
	at java.net.Socket.connect(Socket.java:589) ~[na:1.8.0_161]
[2023-04-21 10:20:54.884 | TID:a21d429243ec4e098423dce61678f1d9.64.16820436548834911 |  |  |  |  |  |  |  |  | BASIC | XNIO-1 task-4 INFO  c.ienglish.server.book.controller.HealthController - Method: [healthTest] execute times: [0], Response: [{"code":0,"message":"success","success":true}] ]
[2023-04-21 10:20:58.068 | TID:a21d429243ec4e098423dce61678f1d9.64.16820436580674913 |  |  |  |  |  |  |  |  | BASIC | XNIO-1 task-7 INFO  c.ienglish.server.book.controller.HealthController - RequestMapping: [/health], Method: [healthTest], Param: [[]] ]
[2023-04-21 10:20:58.068 | TID:a21d429243ec4e098423dce61678f1d9.64.16820436580674913 |  |  |  |  |  |  |  |  | BASIC | XNIO-1 task-7 INFO  c.ienglish.server.book.controller.HealthController - Method: [healthTest] execute times: [0], Response: [{"code":0,"message":"success","success":true}] ]
```

> 其中包含了错误日志和正常日志，还有一些**格式不统一的问题**

filebeat配置文件为：

```yaml
filebeat.inputs:
- type: log
  tags: ["dev"]
  fields_under_root: true
  processors:
    - add_fields:
        fields:
          serviceName: ""

  processors:
    - script:
        lang: javascript
        source: >
          var console = require('console');
          function process(event) {
             var logPath =  event.Get("log.file.path");
             var fileName = getFileName(logPath);
             var fileServiceName = getServiceName(fileName);
             event.Put('serviceName',fileServiceName);
          }
          function getFileName(o) {
             var pos = o.lastIndexOf('/');
             return o.substring(pos + 1);
          }
          function getServiceName(o) {
             return o.replace(/\.|_/g,'-') 
          }
  enabled: true
  paths:
    - /data/logs/*.log
  multiline.pattern: '^(\[[0-9]{4}-[0-9]{2}-[0-9]{2}\ [0-9]{2}:[0-9]{2}:[0-9]{2}|[0-9]{4}-[0-9]{2}-[0-9]{2}\ [0-9]{2}:[0-9]{2}:[0-9]{2})'
  multiline.negate: true
  multiline.match: after
  multiline.max_lines: 100
output.kafka:
    enabled: true
    hosts: ["192.168.50.38:9092","192.168.50.39:9092","192.168.50.40:9092"]
    topic: 'log-%{[serviceName]}'
    max_message_bytes: 5242880
    partition.round_robin:
      reachable_only: true
    keep-alive: 120
    required_acks: 1

```

> `multiline.pattern`: 多行的正则表达式
>
> `multiline.negate`: 匹配到指定模式之前合并多行
>
> `multiline.match`: 匹配到新行之前将先合并旧行
>
> `multiline.max_lines`: 了最大行数，超过这个数目的行将被忽略

## 日志报警

es对接grafana

![image-20230421165010141](https://gitee.com/root_007/md_file_image/raw/master/202304211650227.png)

关于日志报警构想

> 1、获取es中数据(json格式),晒选出所需字段，其中必要字段为message、表示时间字段等
>
> 2、通过alertmanager接口将报警信息发送至alertmanager
>
> 3、alertmanager通过路由组等报警规则发送至其他接收报警渠道

日志告警实现代码 `zxl/prometheus_script/elkAlert_alertmanager`

实现效果展示

![image-20230424165901782](https://gitee.com/root_007/md_file_image/raw/master/202304241659902.png)

## kibana仪表盘制作

![image-20230423133623163](https://gitee.com/root_007/md_file_image/raw/master/202304231336229.png)

![image-20230423133747306](https://gitee.com/root_007/md_file_image/raw/master/202304231337355.png)

![image-20230423133814344](https://gitee.com/root_007/md_file_image/raw/master/202304231338425.png)

根据自己需求添加即可

![image-20230423134026324](https://gitee.com/root_007/md_file_image/raw/master/202304231340437.png)

## 基于k8s的filebeat部署

参考filebeat.yaml

```yaml
filebeat.inputs:
- type: container
  paths:
    - /var/log/containers/*.log
    #- /root/tmp/*.log
  document_type: "english-server"
  multiline.pattern: '^\[|[0-9]{4}-[0-9]{2}-[0-9]{2}\ [0-9]{2}:[0-9]{2}:[0-9]{2}|(25[0-5]|2[0-4]\d|[0-1]\d{2}|[1-9]?\d)\.(25[0-5]|2[0-4]\d|[0-1]\d{2}|[1-9]?\d)\.(25[0-5]|2[0-4]\d|[0-1]\d{2}|[1-9]?\d)\.(25[0-5]|2[0-4]\d|[0-1]\d{2}|[1-9]?\d)'
  multiline.negate: true
  multiline.match: after
  multiline.max_lines: 100
  processors:
    - add_kubernetes_metadata:
        host: ${NODE_NAME}
        matchers:
        - logs_path:
            logs_path: "/var/log/containers/"
            #logs_path: "/root/tmp/"
  processors:
    - add_cloud_metadata:
    - add_host_metadata:
  processors:
    - script:
       lang: js
       id: dispose
       #file: /usr/share/filebeat/dispose.js
       source: >
         var console = require('console');
         var envs = new Array("general-components","english-qa","english-prod","chinese-old-qa","chinese-old","chinese-new-qa","chinese-new","bigdata-dev","bigdata-qa","bigdata-prod","kong-prod","french-prod","math-prod","preschool-prod","quality-prod");
         function process(event) {
             var logPath = event.Get("log.file.path");
             var fileName = getFileName(logPath);
             var curEnv = getEnv(fileName);
             console.log(curEnv);
             if("" != curEnv){
               event.Put('nameSpace',curEnv);
               var idx = fileName.indexOf(curEnv);
               var appStr = fileName.substring(0,idx-1);
               var appName = getAppName(appStr);
               event.Put('appName',appName);
             }else{
               event.Cancel();
               return;
             }
         }
         function getFileName(o) {
             var pos = o.lastIndexOf('/');
             return o.substring(pos + 1);
         }
         function getEnv(o){
             var rs = "";
             for (var i=0; i<envs.length; i++) {
               var value = envs[i];
               if(o.indexOf(value) != -1){
                 rs = value;
                 break;
               }
             }
             return rs;
         }
         function getAppName(o){
             return o.replace(/(.*)-.*?-.*?$/,'$1');
         } 
- type: container
  paths:
     - /var/log/containers/nginx-ingress-controller*kube-system*.log
  document_type: "english-server"
  multiline.pattern: '^(25[0-5]|2[0-4]\d|[0-1]\d{2}|[1-9]?\d)\.(25[0-5]|2[0-4]\d|[0-1]\d{2}|[1-9]?\d)\.(25[0-5]|2[0-4]\d|[0-1]\d{2}|[1-9]?\d)\.(25[0-5]|2[0-4]\d|[0-1]\d{2}|[1-9]?\d)'
  multiline.negate: true
  multiline.match: after
  multiline.max_lines: 100
  processors:
    - add_kubernetes_metadata:
        host: ${NODE_NAME}
        matchers:
        - logs_path:
            logs_path: "/var/log/containers/"
  processors:
    - add_cloud_metadata:
    - add_host_metadata:
  processors:
    - script:
       lang: js
       id: system_filter
       source: >
         function process(event) {
             event.Put("appName","nginx-ingress-controller")
             event.Put("nameSpace","kube-system")
         } 
processors:
  - add_kubernetes_metadata:
     host: ${NODE_NAME}
     matchers:
     - logs_path:
         logs_path: "/var/log/containers/"
cloud.id: ${ELASTIC_CLOUD_ID}
cloud.auth: ${ELASTIC_CLOUD_AUTH}
output.elasticsearch:
  enabled: false
  hosts: ['${ELASTICSEARCH_HOST:elasticsearch}:${ELASTICSEARCH_PORT:9200}']
  username: ${ELASTICSEARCH_USERNAME}
  password: ${ELASTICSEARCH_PASSWORD}
output.kafka:
  enabled: true # 增加kafka的输出
  hosts: ["10.16.202.197:9092", "10.16.201.158:9092", "10.16.200.67:9092", "10.16.202.211:9092", "10.16.202.210:9092"]
  topic: 'log-%{[appName]}'
  max_message_bytes: 5242880
  partition.round_robin:
    reachable_only: true
  keep-alive: 120
  required_acks: 1
```

> 请根据实际情况自行修改
