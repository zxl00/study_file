## Prometheus介绍

> prometheus是CNCF的一个开源项目，Google BorgMon监控系统的开源版本，是一个系统和服务的监控系统。周期性采集metrics指标，匹配规则和展示结果，以及触发某些条件的告警发送。
>
> 官网地址： https://prometheus.io/docs/introduction/overview/

## Prometheus特点

> - 多维度数据模型（时序数据是由指标名字和kv结构的维度定义）
> - 灵活的查询语言（PromQL）
> - 不依赖分布式存储。每个server是一个自治的节点。
> - 通过HTTP拉取收集时序数据，同时提供push gateway供用户主动推送数据，主要用于短生命周期的job。
> - 通过静态配置或服务发现来发现目标对象
> - 支持多种多样的出图和展示方式，例如自带的Web UI和[Grafana](https://cloud.tencent.com/product/tcmg?from=20065&from_column=20065)等。
> - 支持水平扩容

## Prometheus架构图

![image-20230404150939335](https://gitee.com/root_007/md_file_image/raw/master/202304041509392.png)

> 从架构图中可以看出，Prometheus Server 周期性的拉取从配置文件或者服务发现获取到的目标数据，每个目标需要通过HTTP接口暴露数据。Prometheus Server通过一定的规则汇总和记录时序数据到本地数据库。将符合检测条件的告警数据推送给Altermanager，Altermanager通过配置的通知方式发送告警。Web UI 或者Grafana通过PromQL查询Prometheus Server中的数据绘图展示。

## Prometheus下载

> 下载地址：https://github.com/prometheus/prometheus/releases/download/v2.37.6/prometheus-2.37.6.linux-amd64.tar.gz
>
> 备注：本次讲解以2.37.6版本为准

#### 文件展示

```bash
[root@rs1 prometheus-2.37.6.linux-amd64]# ll
total 208916
drwxr-xr-x  2 1001  123        38 Feb 20 05:03 console_libraries
drwxr-xr-x  2 1001  123       173 Feb 20 05:03 consoles
drwxr-xr-x 10 root root       274 Apr  4 03:00 data
-rw-r--r--  1 1001  123     11357 Feb 20 05:03 LICENSE
-rw-r--r--  1 1001  123      3773 Feb 20 05:03 NOTICE
-rwxr-xr-x  1 1001  123 111052375 Feb 20 04:40 prometheus
-rw-r--r--  1 1001  123      1779 Apr  4 01:43 prometheus.yml
-rwxr-xr-x  1 1001  123 102850693 Feb 20 04:42 promtool

```

#### 配置文件详解

非默认配置文件

```yaml
# my global config
global:
  scrape_interval: 15s  # 全局每次收集数据的间隔时常
  evaluation_interval: 30s # 全局规则扫描时间间隔
  scrape_timeout: 10s  # 全局超时时间
  # scrape_timeout is set to the global default (10s).

  external_labels:   # 用于外部系统标签，不是用于metrics哦
    monitor: codelab
    foo: bar

rule_files:   # 告警规则 ，这里设定好后，根据evaluation_interval进行加载配置，支持通配符表示文件名
  - "first.rules"
  - "my/*.rules"

remote_write:
  - url: http://remote1/push
    name: drop_expensive
    write_relabel_configs:
      - source_labels: [__name__]
        regex: expensive.*
        action: drop
    oauth2:
      client_id: "123"
      client_secret: "456"
      token_url: "http://remote1/auth"
      tls_config:
        cert_file: valid_cert_file
        key_file: valid_key_file

  - url: http://remote2/push
    name: rw_tls
    tls_config:
      cert_file: valid_cert_file
      key_file: valid_key_file
    headers:
      name: value

remote_read:
  - url: http://remote1/read
    read_recent: true
    name: default
    enable_http2: false
  - url: http://remote3/read
    read_recent: false
    name: read_special
    required_matchers:
      job: special
    tls_config:
      cert_file: valid_cert_file
      key_file: valid_key_file

scrape_configs:  # 需要采集的目标
  - job_name: prometheus  # 目标名称

    honor_labels: true
    # scrape_interval is defined by the configured global (15s).  # 全局定义的时候定义过了，也可以单独定义在该job里
    # scrape_timeout is defined by the global default (10s).

    # metrics_path defaults to '/metrics'   # 路径，获取监控数据的路径
    # scheme defaults to 'http'.  

    file_sd_configs:   # 基于文件的自动发现目标服务器
      - files:
          - foo/*.slow.json
          - foo/*.slow.yml
          - single/file.yml
        refresh_interval: 10m
      - files:
          - bar/*.yaml

    static_configs:  # 静态配置目标服务器
      - targets: ["localhost:9090", "localhost:9191"]  # 目标服务器
        labels:  # 标签
          my: label
          your: label

    relabel_configs:  #  可以在目标被抓取之前动态地重写目标的标签集来实现自定义标签及分类
      - source_labels: [job, __meta_dns_name]
        regex: (.*)some-[regex]
        target_label: job
        replacement: foo-${1}
        # action defaults to 'replace'
      - source_labels: [abc]
        target_label: cde
      - replacement: static
        target_label: abc
      - regex:
        replacement: static
        target_label: abc
      - source_labels: [foo]
        target_label: abc
        action: keepequal
      - source_labels: [foo]
        target_label: abc
        action: dropequal

    authorization:
      credentials_file: valid_token_file

    tls_config:
      min_version: TLS10

  - job_name: service-x

    basic_auth:
      username: admin_name
      password: "multiline\nmysecret\ntest"

    scrape_interval: 50s
    scrape_timeout: 5s

    body_size_limit: 10MB
    sample_limit: 1000

    metrics_path: /my_path
    scheme: https

    dns_sd_configs:
      - refresh_interval: 15s
        names:
          - first.dns.address.domain.com
          - second.dns.address.domain.com
      - names:
          - first.dns.address.domain.com

    relabel_configs:
      - source_labels: [job]
        regex: (.*)some-[regex]
        action: drop
      - source_labels: [__address__]
        modulus: 8
        target_label: __tmp_hash
        action: hashmod
      - source_labels: [__tmp_hash]
        regex: 1
        action: keep
      - action: labelmap
        regex: 1
      - action: labeldrop
        regex: d
      - action: labelkeep
        regex: k

    metric_relabel_configs:
      - source_labels: [__name__]
        regex: expensive_metric.*
        action: drop

  - job_name: service-y

    consul_sd_configs:
      - server: "localhost:1234"
        token: mysecret
        services: ["nginx", "cache", "mysql"]
        tags: ["canary", "v1"]
        node_meta:
          rack: "123"
        allow_stale: true
        scheme: https
        tls_config:
          ca_file: valid_ca_file
          cert_file: valid_cert_file
          key_file: valid_key_file
          insecure_skip_verify: false

    relabel_configs:
      - source_labels: [__meta_sd_consul_tags]
        separator: ","
        regex: label:([^=]+)=([^,]+)
        target_label: ${1}
        replacement: ${2}

  - job_name: service-z

    tls_config:
      cert_file: valid_cert_file
      key_file: valid_key_file

    authorization:
      credentials: mysecret

  - job_name: service-kubernetes

    kubernetes_sd_configs:
      - role: endpoints
        api_server: "https://localhost:1234"
        tls_config:
          cert_file: valid_cert_file
          key_file: valid_key_file

        basic_auth:
          username: "myusername"
          password: "mysecret"

  - job_name: service-kubernetes-namespaces

    kubernetes_sd_configs:
      - role: endpoints
        api_server: "https://localhost:1234"
        namespaces:
          names:
            - default

    basic_auth:
      username: "myusername"
      password_file: valid_password_file

  - job_name: service-kuma

    kuma_sd_configs:
      - server: http://kuma-control-plane.kuma-system.svc:5676

  - job_name: service-marathon
    marathon_sd_configs:
      - servers:
          - "https://marathon.example.com:443"

        auth_token: "mysecret"
        tls_config:
          cert_file: valid_cert_file
          key_file: valid_key_file

  - job_name: service-nomad
    nomad_sd_configs:
      - server: 'http://localhost:4646'

  - job_name: service-ec2
    ec2_sd_configs:
      - region: us-east-1
        access_key: access
        secret_key: mysecret
        profile: profile
        filters:
          - name: tag:environment
            values:
              - prod

          - name: tag:service
            values:
              - web
              - db

  - job_name: service-lightsail
    lightsail_sd_configs:
      - region: us-east-1
        access_key: access
        secret_key: mysecret
        profile: profile

  - job_name: service-azure
    azure_sd_configs:
      - environment: AzurePublicCloud
        authentication_method: OAuth
        subscription_id: 11AAAA11-A11A-111A-A111-1111A1111A11
        resource_group: my-resource-group
        tenant_id: BBBB222B-B2B2-2B22-B222-2BB2222BB2B2
        client_id: 333333CC-3C33-3333-CCC3-33C3CCCCC33C
        client_secret: mysecret
        port: 9100

  - job_name: service-nerve
    nerve_sd_configs:
      - servers:
          - localhost
        paths:
          - /monitoring

  - job_name: 0123service-xxx
    metrics_path: /metrics
    static_configs:
      - targets:
          - localhost:9090

  - job_name: badfederation
    honor_timestamps: false
    metrics_path: /federate
    static_configs:
      - targets:
          - localhost:9090

  - job_name: 测试
    metrics_path: /metrics
    static_configs:
      - targets:
          - localhost:9090

  - job_name: httpsd
    http_sd_configs:
      - url: "http://example.com/prometheus"

  - job_name: service-triton
    triton_sd_configs:
      - account: "testAccount"
        dns_suffix: "triton.example.com"
        endpoint: "triton.example.com"
        port: 9163
        refresh_interval: 1m
        version: 1
        tls_config:
          cert_file: valid_cert_file
          key_file: valid_key_file

  - job_name: digitalocean-droplets
    digitalocean_sd_configs:
      - authorization:
          credentials: abcdef

  - job_name: docker
    docker_sd_configs:
      - host: unix:///var/run/docker.sock

  - job_name: dockerswarm
    dockerswarm_sd_configs:
      - host: http://127.0.0.1:2375
        role: nodes

  - job_name: service-openstack
    openstack_sd_configs:
      - role: instance
        region: RegionOne
        port: 80
        refresh_interval: 1m
        tls_config:
          ca_file: valid_ca_file
          cert_file: valid_cert_file
          key_file: valid_key_file

  - job_name: service-puppetdb
    puppetdb_sd_configs:
      - url: https://puppetserver/
        query: 'resources { type = "Package" and title = "httpd" }'
        include_parameters: true
        port: 80
        refresh_interval: 1m
        tls_config:
          ca_file: valid_ca_file
          cert_file: valid_cert_file
          key_file: valid_key_file

  - job_name: hetzner
    relabel_configs:
      - action: uppercase
        source_labels: [instance]
        target_label: instance
    hetzner_sd_configs:
      - role: hcloud
        authorization:
          credentials: abcdef
      - role: robot
        basic_auth:
          username: abcdef
          password: abcdef

  - job_name: service-eureka
    eureka_sd_configs:
      - server: "http://eureka.example.com:8761/eureka"

  - job_name: ovhcloud
    ovhcloud_sd_configs:
      - service: vps
        endpoint: ovh-eu
        application_key: testAppKey
        application_secret: testAppSecret
        consumer_key: testConsumerKey
        refresh_interval: 1m
      - service: dedicated_server
        endpoint: ovh-eu
        application_key: testAppKey
        application_secret: testAppSecret
        consumer_key: testConsumerKey
        refresh_interval: 1m

  - job_name: scaleway
    scaleway_sd_configs:
      - role: instance
        project_id: 11111111-1111-1111-1111-111111111112
        access_key: SCWXXXXXXXXXXXXXXXXX
        secret_key: 11111111-1111-1111-1111-111111111111
      - role: baremetal
        project_id: 11111111-1111-1111-1111-111111111112
        access_key: SCWXXXXXXXXXXXXXXXXX
        secret_key: 11111111-1111-1111-1111-111111111111

  - job_name: linode-instances
    linode_sd_configs:
      - authorization:
          credentials: abcdef

  - job_name: uyuni
    uyuni_sd_configs:
      - server: https://localhost:1234
        username: gopher
        password: hole

  - job_name: ionos
    ionos_sd_configs:
      - datacenter_id: 8feda53f-15f0-447f-badf-ebe32dad2fc0
        authorization:
          credentials: abcdef

  - job_name: vultr
    vultr_sd_configs:
      - authorization:
          credentials: abcdef

alerting:
  alertmanagers:
    - scheme: https
      static_configs:
        - targets:
            - "1.2.3.4:9093"
            - "1.2.3.5:9093"
            - "1.2.3.6:9093"

storage:
  tsdb:
    out_of_order_time_window: 30m

tracing:
  endpoint: "localhost:4317"
  client_type: "grpc"
  headers:
    foo: "bar"
  timeout: 5s
  compression: "gzip"
  tls_config:
    cert_file: valid_cert_file
    key_file: valid_key_file
    insecure_skip_verify: true
	
```

#### 基于文件的自动发现

其文件格式为

```yaml
- targets:
    - "10.88.88.11:9100"
    - "10.88.88.12:9100"
    - "localhost:9100"
  labels: 
    role: test
```

#### 基于http的的自动发现

> 要求：接口要求返回状态码是200，HTTP请求头需要包含 Content-Type:application/json
>
> 数据格式：数据格式：列表里面包含多个对象，对象里面有一个targets字段，是一个字符串列表指定监控目标；labels字段是一个map[string]string，设置标签

```
[
  {
    "targets": [ "<host>", ... ],
    "labels": {
      "<labelname>": "<labelvalue>", ...
    }
  },
  ...
]
```

go案例

```go
// main.go
package main

import (
	"log"
	"net/http"

	"github.com/gin-gonic/gin"
)

// 配置对象
type Object struct {
	Targets []string          `json:"targets"`
	Labels  map[string]string `json:"labels"`
}

func main() {
	log.Println("start...")
	r := gin.Default()
	r.GET("/prom/node_exporter", func(c *gin.Context) {
		objs := []Object{
			{
				Targets: []string{"localhost:9100", "192.168.122.1:9100"},
				Labels:  map[string]string{"env": "testing"},
			},
		}
		c.JSON(http.StatusOK, objs)
	})
	r.Run(":8088")
}
```

#### relabel_configs详解

> 可以在目标被抓取之前动态地重写目标的标签集来实现自定义标签及分类

#### 标签详解

> Prometheus 加载 Targets 后，这些 Targets 会自动包含一些默认的标签，Target 以 __ 作为前置的标签是在系统内部使用的，这些标签不会被写入到样本数据中。
>
> 默认每次增加 Target 时会自动增加一个 instance 标签，而 instance 标签的内容刚好对应 Target 实例的 __address__ 值，这是因为实际上 Prometheus 内部做了一次标签重写处理，默认 __address__ 标签设置为 : 地址，经过标签重写后，默认会自动将该值设置为 instance 标签，所以我们能够在页面看到该标签。

#### 几个前缀标签含义

```bash
__meta_：在重新标记阶段可以使用以 _meta_ 为前缀的附加标签。它们由提供目标的服务发现机制设置的，并因机制而异。
__：目标重新标记完成后，以 __ 开头的标签将从标签集中删除。
__tmp：如果重新标记步骤仅需要临时存储标签值（作为后续重新标记步骤的输入），请使用这个标签名称前缀。这个前缀保证永远不会被 Prometheus 本身使用。
```

#### 操作标签的动作

> - replace：根据 regex 的配置匹配 source_labels 标签的值（注意：多个 source_label 的值会按照 separator 进行拼接），并且将匹配到的值写入到 target_label 当中如果有多个匹配组，则可以使用 ${1}, ${2} 确定写入的内容。如果没匹配到任何内容则不对 target_label 进行替换， 默认为 replace
> - keep：丢弃 source_labels 的值中没有匹配到 regex 正则表达式内容的 Target 实例
> -  drop：丢弃 source_labels 的值中匹配到 regex 正则表达式内容的 Target 实例
> -  hashmod：将 target_label 设置为关联的 source_label 的哈希模块
> - labelmap：根据 regex 去匹配 Target 实例所有标签的名称（注意是名称），并且将捕获到的内容作为为新的标签名称，regex 匹配到标签的的值作为新标签的值。
> - labeldrop：对 Target 标签进行过滤，会移除匹配过滤条件的所有标签
> - labelkeep：对 Target 标签进行过滤，会移除不匹配过滤条件的所有标签

#### rule配置

```yaml
groups:
- name: operations
  rules:
  - alert: node-down  # 告警的名称
    expr: up != 1
    for: 3s
    labels:
      severity: High  # 增加一个标签，常用为告警级别
    annotations:
      description: "Environment: {{ $labels.env }} Instance: {{ $labels.instance }} is Down ! ! !"
      value: '{{ $value }}'
      summary:  "The host node was down"

```

> 更多规则配置请参考：https://awesome-prometheus-alerts.grep.to/

#### 告警时间自定义

```yaml
groups:
- name: 指定特定时间范围
  rules:
  - alert: 凌晨0点到6点不触发告警
    # prometheus默认是utc时间，请注意
    expr: promQL表达式 and ON() (hour() < 16  > 22)
```

> 更多告警时间自定义请参考：https://awesome-prometheus-alerts.grep.to/sleep-peacefully

## Alertmanager下载

> 下载地址：https://github.com/prometheus/alertmanager/releases/download/v0.23.0/alertmanager-0.23.0.linux-amd64.tar.gz
>
> 备注：本次讲解以0.23.0版本为准

#### 文件展示

```bash
[root@lvs-backup alertmanager-0.23.0.linux-amd64]# ll
total 53596
-rwxr-xr-x. 1 3434 3434 30942810 Aug 25  2021 alertmanager
-rw-r--r--. 1 3434 3434     1976 Apr  4 02:58 alertmanager.yml
-rwxr-xr-x. 1 3434 3434 23913946 Aug 25  2021 amtool
drwxr-xr-x. 2 root root       35 Apr  4 01:58 data
-rw-r--r--. 1 3434 3434    11357 Aug 25  2021 LICENSE
-rw-r--r--. 1 3434 3434      457 Aug 25  2021 NOTICE

```

#### 配置文件详解

非默认配置文件

```yaml
## Alertmanager 配置文件
global:
  resolve_timeout: 5m  # 该参数定义了当Alertmanager持续多长时间未接收到告警后标记告警状态为resolved（已解决）
  # smtp配置  邮箱配置，不解释
  smtp_from: "123456789@qq.com"
  smtp_smarthost: 'smtp.qq.com:465'
  smtp_auth_username: "123456789@qq.com"
  smtp_auth_password: "auth_pass"
  smtp_require_tls: true
# email、企业微信的模板配置存放位置，钉钉的模板会单独讲如果配置。
templates:
  - '/data/alertmanager/templates/*.tmpl'  # 发送告警模板，不解释
# 路由分组
route:
  receiver: ops  # 默认的接收器名称
  group_wait: 30s # 在组内等待所配置的时间，如果同组内，30秒内出现相同报警，在一个组内发送报警。
  group_interval: 5m # 如果组内内容不变化，合并为一条警报信息，5m后发送。
  repeat_interval: 24h # 发送报警间隔，如果指定时间内没有修复，则重新发送报警。
  group_by: [alertname]  # 报警分组
  routes:  # 子路由的匹配设置
      - match:  # 匹配
          team: operations  # 标签: 标签值  如果team: operations，则使用receiver: 'ops'也就是使用ops的报警
        receiver: 'ops'  # 发送警报的接收器名称
      - match_re: # 正则匹配
          service: nginx|apache
        receiver: 'web'
      - match_re:
          service: hbase|spark
        receiver: 'hadoop'
      - match_re:
          service: mysql|mongodb
        receiver: 'db'
# 接收器指定发送人以及发送渠道
receivers:
# ops分组的定义
- name: ops
  email_configs:
  - to: '9935226@qq.com,10000@qq.com'
    send_resolved: true
    headers:
      subject: "[operations] 报警邮件"
      from: "警报中心"
      to: "小煜狼皇"
  # 钉钉配置
  webhook_configs:
  - url: http://localhost:8070/dingtalk/ops/send
    # 企业微信配置
  wechat_configs:
  - corp_id: 'ww5421dksajhdasjkhj'
    api_url: 'https://qyapi.weixin.qq.com/cgi-bin/'
    send_resolved: true
    to_party: '2'
    agent_id: '1000002'
    api_secret: 'Tm1kkEE3RGqVhv5hO-khdakjsdkjsahjkdksahjkdsahkj'

# web
- name: web
  email_configs:
  - to: '9935226@qq.com'
    send_resolved: true
    headers: { Subject: "[web] 报警邮件"} # 接收邮件的标题
  webhook_configs:
  - url: http://localhost:8070/dingtalk/web/send
  - url: http://localhost:8070/dingtalk/ops/send
# db
- name: db
  email_configs:
  - to: '9935226@qq.com'
    send_resolved: true
    headers: { Subject: "[db] 报警邮件"} # 接收邮件的标题
  webhook_configs:
  - url: http://localhost:8070/dingtalk/db/send
  - url: http://localhost:8070/dingtalk/ops/send
# hadoop
- name: hadoop
  email_configs:
  - to: '9935226@qq.com'
    send_resolved: true
    headers: { Subject: "[hadoop] 报警邮件"} # 接收邮件的标题
  webhook_configs:
  - url: http://localhost:8070/dingtalk/hadoop/send
  - url: http://localhost:8070/dingtalk/ops/send

# 抑制器配置
inhibit_rules: # 抑制规则
  - source_match: # 源标签警报触发时抑制含有目标标签的警报，在当前警报匹配 status: 'High'
      status: 'High'  # 此处的抑制匹配一定在最上面的route中配置不然，会提示找不key。
    target_match:
      status: 'Warning' # 目标标签值正则匹配，可以是正则表达式如: ".*MySQL.*"
    equal: ['alertname','operations', 'instance'] # 确保这个配置下的标签内容相同才会抑制，也就是说警报中必须有这三个标签值才会被抑制。

```

#### 可视化路由树

https://prometheus.io/webtools/alerting/routing-tree-editor/

![image-20230404154533586](https://gitee.com/root_007/md_file_image/raw/master/202304041545710.png)

![image-20230404154650278](https://gitee.com/root_007/md_file_image/raw/master/202304041546360.png)

## node_exporter下载

> 下载地址：https://github.com/prometheus/node_exporter/releases/download/v1.4.0/node_exporter-1.4.0.linux-amd64.tar.gz
>
> 备注：本次讲解以1.4.0版本为准

#### 文件展示

```
[root@rs2 node_exporter-1.4.0.linux-amd64]# ll
total 19200
-rw-r--r-- 1 3434 3434    11357 Sep 26  2022 LICENSE
-rwxr-xr-x 1 3434 3434 19640886 Sep 26  2022 node_exporter
-rw-r--r-- 1 3434 3434      463 Sep 26  2022 NOTICE
```

> 更多exporter可以在该仓库查找：https://github.com/prometheus
>
> 备注：也可以自定义exporter，或者在github中自行查找

![image-20230404162008554](https://gitee.com/root_007/md_file_image/raw/master/202304041620669.png)

## Prometheusalert下载

​		告警媒介对接工具

> 下载地址：https://github.com/feiyu563/PrometheusAlert/releases/download/v4.8.2/linux.zip
>
> 备注：本次讲解以4.8.2版本为准

#### 文件展示

```bash
[root@lvs-backup linux]# ll
total 34664
drwxr-xr-x. 2 root root       22 Apr  4 01:50 conf
drwxr-xr-x. 2 root root       61 Apr  4 02:04 db
drwxr-xr-x. 2 root root       39 Mar 31  2022 logs
-rwxr-xr-x. 1 root root 29008272 Jun 24  2022 PrometheusAlert
drwxr-xr-x. 2 root root       55 Jun  9  2022 PrometheusAlertVoicePlugin
drwxr-xr-x. 4 root root       33 Apr 11  2022 static
-rw-r--r--. 1 root root      416 Nov  8  2019 user.csv
drwxr-xr-x. 2 root root     4096 Apr 11  2022 views
-rw-r--r--. 1 root root  6472521 Jun 22  2022 zabbixclient

```

> 更多使用方法请参考：https://github.com/feiyu563/PrometheusAlert

## 实战案例

说明：服务器配置仅供学习使用，生产环境请按需求增加资源

#### 服务器配置

系统：centos7

内存：1G

cpu：1V

磁盘：20G

台数：3台

#### 服务器规划

| IP             | 服务                          |
| -------------- | ----------------------------- |
| 192.168.59.131 | prometheus、node_exporter     |
| 192.168.59.132 | node_exporter、grafana        |
| 192.168.59.134 | alertmanager、prometheusalert |

#### prometheus部署

192.168.59.131服务器上操作

```bash
### 下载prometheus
[root@rs1 ~]# wget https://github.com/prometheus/prometheus/releases/download/v2.37.6/prometheus-2.37.6.linux-amd64.tar.gz
### 创建目录/data/soft
[root@rs1 ~]#  mkdir -p   /data/soft
### 将prometheus解压到/data/soft
[root@rs1 ~]#   tar -xvf  prometheus-2.37.6.linux-amd64.tar.gz -C /data/soft
```

##### 查看配置文件

```yaml
[root@rs1 prometheus-2.37.6.linux-amd64]# vim prometheus.yml  

# my global config
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: "prometheus"

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
      - targets: ["localhost:9090"]

```

##### 启动常用参数

```
### 指定使用配置那个文件
--config.file="prometheus.yml"
### 指定监听端口号
--web.listen-address="0.0.0.0:9090"
```

> 更多参数请自行查看   prometheus -h

![image-20230404172415003](https://gitee.com/root_007/md_file_image/raw/master/202304041724119.png)

##### 启动prometheus

```bash
[root@rs1 prometheus-2.37.6.linux-amd64]# ./prometheus --config.file="prometheus.yml"
```

![image-20230404174357879](https://gitee.com/root_007/md_file_image/raw/master/202304041743974.png)

![image-20230404174425867](https://gitee.com/root_007/md_file_image/raw/master/202304041744950.png)

#### node_exporter部署

192.168.59.131、192.168.59.132服务器上操作

node_exporter具体作用，自行百度

```bash
### 下载node_exporter
[root@rs1 ~]# wget https://github.com/prometheus/node_exporter/releases/download/v1.4.0/node_exporter-1.4.0.linux-amd64.tar.gz
### 创建目录/data/soft
[root@rs1 ~]#  mkdir -p   /data/soft
### 将node_exporter解压到/data/soft
[root@rs1 ~]#   tar -xvf  node_exporter-1.4.0.linux-amd64.tar.gz   -C /data/soft
```

##### 启动常用参数

```
### 监听地址
--web.listen-address=":9100"
```

##### 启动node_exporter

```bash
[root@rs1 node_exporter]# nohup  ./node_exporter  >/dev/null 2>&1 &
```

> 通过后台启动的方式，若需要使用systemctl启动，请自行百度解决

#### Prometheus收集node_exporter信息

编辑prometheus.yaml文件

```yaml
# my global config
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "node"
    static_configs:
      - targets: ["192.168.59.131:9100"]
        labels:   # 自定义标签
          env: prod
          auth: bj
      - targets: ["192.168.59.132:9100"]
        labels:
          env: test
          auth: jaic

```

重启prometheus

通过web-ui查看

![image-20230404192621430](https://gitee.com/root_007/md_file_image/raw/master/202304041926541.png)

![image-20230404192821732](https://gitee.com/root_007/md_file_image/raw/master/202304041928810.png)



> prometheus关于标签章节讲过
>
> 默认每次增加 Target 时会自动增加一个 instance 标签，而 instance 标签的内容刚好对应 Target 实例的 __address__ 值，这是因为实际上 Prometheus 内部做了一次标签重写处理，默认 __address__ 标签设置为 : 地址，经过标签重写后，默认会自动将该值设置为 instance 标签，所以我们能够在页面看到该标签。

##### relabel_configs应用

###### 标签动作replace

修改prometheus.yaml文件

```yaml
# my global config
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "node"
    static_configs:
      - targets: ["192.168.59.131:9100"]
        labels:
          env: prod
          auth: bj
      - targets: ["192.168.59.132:9100"]
        labels:
          env: test
          auth: jaic
    relabel_configs:
      - source_labels: ['env']   # 源标签
        regex: pr.*   # 源标签的值 这里regex是使用正则
        target_label: role  # 目标标签
        replacement: admin # 目标标签的值
        action: replace  # 操作标签的动作

```

重启prometheus,当然你也可以reload，具体使用可以百度一下

通过web-ui查看

![image-20230404193852862](https://gitee.com/root_007/md_file_image/raw/master/202304041938956.png)

> 本案例使用的标签动作为**replace**，更多动作，可以参考本文的**操作标签的动作**

###### 标签动作drop

修改prometheus.yaml文件

```yaml
# my global config
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "node"
    static_configs:
      - targets: ["192.168.59.131:9100"]
        labels:
          env: prod
          auth: bj
      - targets: ["192.168.59.132:9100"]
        labels:
          env: test
          auth: jaic
    relabel_configs:
      - source_labels: ['env']
        regex: pr.*
        action: drop

```

重启prometheus,当然你也可以reload，具体使用可以百度一下

通过web-ui查看

![image-20230404194610033](https://gitee.com/root_007/md_file_image/raw/master/202304041946145.png)

> 本案例使用的标签动作为**drop**，更多动作，可以参考本文的**操作标签的动作**

#### Prometheus告警规则rule配置

##### 创建告警规则目录

```bash
[root@rs1 prometheus-2.37.6.linux-amd64]# pwd
/data/soft/prometheus-2.37.6.linux-amd64
### 创建存放告警规则的目录
[root@rs1 prometheus-2.37.6.linux-amd64]# mkdir rules
```

##### 编写告警规则

rules文件夹中，node.yaml(文件名称可以自定义)

```yaml
groups:
- name: operations
  rules:
  - alert: node-down  # 告警名称
    expr: up != 1  # 告警规则
    for: 3s  # 持续时间
    labels:  # 告警标签，一般为告警级别
      severity: High
    annotations:  # 描述信息
      description: "Environment: {{ $labels.env }} Instance: {{ $labels.instance }} is Down ! ! !"
      value: '{{ $value }}'
      summary:  "The host node was down"

```

> 该规则仅做演示使用，更多规则需要自行定义
>
> 规则参考文档：https://awesome-prometheus-alerts.grep.to/rules

修改prometheus.yaml文件，使之加载该规则

```yaml
# my global config
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
   - "rules/node.yaml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "node"
    static_configs:
      - targets: ["192.168.59.131:9100"]
        labels:
          env: prod
          auth: bj
      - targets: ["192.168.59.132:9100"]
        labels:
          env: test
          auth: jaic
    relabel_configs:
      - source_labels: ['env']
        regex: pr.*
        action: drop


```

> 主要是修改了**rule_files**字段

重启prometheus,当然你也可以reload，具体使用可以百度一下

通过web-ui查看

![image-20230404195929847](https://gitee.com/root_007/md_file_image/raw/master/202304041959960.png)

![image-20230404200706097](https://gitee.com/root_007/md_file_image/raw/master/202304042007201.png)

##### 关于告警状态解释

> Inactive：正常状态，未激活警报
>
> Pending：已满足触发条件，但没有满足发送时间条件，一般为告警规则中的for，也就是所谓的告警持续时间
>
> Firing：已满满足条件，且超过了 for 子句中的的指定持续时间
>
> 带有for子句的警报触发以后首先会先转换成 `Pending` 状态，然后在转换为 `Firing` 状态。这里需要俩个周期才能触发警报条件，如果没有设置 `for` 子句，会直接 `Inactive` 状态转换成 `Firing状态`，然后触发警报

##### 指定告警时间(了解即可)

![590270104743970490ffa67d874b2a53](https://gitee.com/root_007/md_file_image/raw/master/202304042123113.png)

#### Prometheusalert部署

192.168.59.134服务器上操作

```bash
### 下载Prometheusalert
[root@lvs-backup ~]# weget  https://github.com/feiyu563/PrometheusAlert/releases/download/v4.8.2/linux.zip
### 将linuxlinux.zip解压到指定目录
[root@lvs-backup ~]# unzip  linux.zip -d   /data/soft
```

##### 启动prometheusalert

```bash
### 添加执行权限
[root@lvs-backup linux]# chmod  +x PrometheusAlert 
### 后台启动
[root@lvs-backup linux]#  nohup  ./PrometheusAlert >/dev/null 2>&1 &
```

##### 访问prometheusalert

![image-20230404204639549](https://gitee.com/root_007/md_file_image/raw/master/202304042046643.png)

> 账号密码一样，需要修改的可以该配置文件，具体的可以参考https://github.com/feiyu563/PrometheusAlert

![image-20230404204842113](https://gitee.com/root_007/md_file_image/raw/master/202304042048209.png)

##### 创建钉钉告警模板

创建钉钉群，自行解决

添加钉钉机器人，自行解决

![image-20230404205856719](https://gitee.com/root_007/md_file_image/raw/master/202304042058809.png)

模板内容

```yaml
{{ $var := .externalURL}}{{ $status := .status}}{{ range $k,$v:=.alerts }} {{if eq $status "resolved"}}
## [告警恢复-通知]({{$var}})
#### 监控指标: {{$v.labels.alertname}}
{{ if eq $v.labels.severity "warning" }}
#### 告警级别: **<font color="#E6A23C">{{$v.labels.severity}}</font>**
{{ else if eq $v.labels.severity "critical"  }}
#### 告警级别: **<font color="#F56C6C">{{$v.labels.severity}}</font>**
{{ end }}
#### 当前状态: **<font color="#67C23A" size=4>已恢复</font>**
#### 故障主机: {{$v.labels.instance}}
* ###### 告警阈值: {{$v.labels.threshold}}
* ###### 开始时间: {{GetCSTtime $v.startsAt}}
* ###### 恢复时间: {{GetCSTtime $v.endsAt}}

#### 告警恢复: <font color="#67C23A">已恢复,{{$v.annotations.description}}</font>
{{ else }}
## [监控告警-通知]({{$var}})
#### 监控指标: {{$v.labels.alertname}}
{{ if eq $v.labels.severity "warning" }}
#### 告警级别: **<font color="#E6A23C" size=4>{{$v.labels.severity}}</font>**
#### 当前状态: **<font color="#E6A23C">需要处理</font>**
{{ else if eq $v.labels.severity "critical"  }}
#### 告警级别: **<font color="#F56C6C" size=4>{{$v.labels.severity}}</font>**
#### 当前状态: **<font color="#F56C6C">需要处理</font>**
{{ end }}
#### 故障主机: {{$v.labels.instance}}
* ###### 告警阈值: {{$v.labels.threshold}}
* ###### 持续时间: {{$v.labels.for_time}}
* ###### 触发时间: {{GetCSTtime $v.startsAt}}
{{ if eq $v.labels.severity "warning" }}
#### 告警触发: <font color="#E6A23C">{{$v.annotations.description}}</font>
{{ else if eq $v.labels.severity "critical" }}
#### 告警触发: <font color="#F56C6C">{{$v.annotations.description}}</font>
{{ end }}
{{ end }}
{{ end }}
```

机器人地址

![image-20230404210001311](https://gitee.com/root_007/md_file_image/raw/master/202304042100409.png)

> 创建机器人**Webhook**

![image-20230404210058219](https://gitee.com/root_007/md_file_image/raw/master/202304042100287.png)

消息json内容

https://github.com/feiyu563/PrometheusAlert/blob/master/doc/readme/system-customtpl.md

![image-20230404210220801](https://gitee.com/root_007/md_file_image/raw/master/202304042102918.png)

保存之后，点击测试

![image-20230404210245572](https://gitee.com/root_007/md_file_image/raw/master/202304042102628.png)

#### Alertmanager部署

192.168.59.134服务器上操作

```bash
### 下载alertmanager
[root@lvs-backup ~]# weget https://github.com/prometheus/alertmanager/releases/download/v0.23.0/alertmanager-0.23.0.linux-amd64.tar.gz
### 创建目录/data/soft
[root@rs1 ~]#  mkdir -p   /data/soft
### 将alertmanager解压到/data/soft
[root@rs1 ~]#   tar -xvf  alertmanager-0.23.0.linux-amd64   -C   /data/soft
```

##### 查看配置文件

alertmanager.yml

```yaml
route:
  group_by: ['alertname']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 1h
  receiver: 'web.hook'
receivers:
- name: 'web.hook'
  webhook_configs:
  - url: 'http://127.0.0.1:5001/'
inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'dev', 'instance']
```

##### 启动常用参数

```
--config.file="alertmanager.yml"
--web.listen-address=":9093"
```

##### 修改配置文件

alertmanager.yml

```yaml
## Alertmanager 配置文件
global:
  resolve_timeout: 1m  # 该参数定义了当Alertmanager持续多长时间未接收到告警后标记告警状态为resolved（已解决）
# 路由分组
route:
  receiver: devops  # 默认的接收器名称
  group_wait: 30s # 在组内等待所配置的时间，如果同组内，30秒内出现相同报警，在一个组内发送报警。
  group_interval: 1m # 如果组内内容不变化，合并为一条警报信息，5m后发送。
  repeat_interval: 1m # 发送报警间隔，如果指定时间内没有修复，则重新发送报警。
  group_by: [alertname,instance]  # 报警分组
  routes:  # 子路由的匹配设置
      - match:  # 匹配
          env: test  # 标签: 标签值  如果team: operations，则使用receiver: 'ops'也就是使用ops的报警
        receiver: 'devops'  # 发送警报的接收器名称
#        continue: true
      - match_re: # 正则匹配
          auth: bj|jaic
        receiver: 'web'
#        continue: true
      - match_re: # 正则匹配
          auth: bj
        receiver: 'db'
#        continue: true


# 接收器指定发送人以及发送渠道
receivers:
# ops分组的定义
- name: devops
  # 钉钉配置
  webhook_configs:
  - url: http://192.168.59.134:8080/prometheusalert?type=dd&tpl=prometheus-dd&ddurl=https://oapi.dingtalk.com/robot/send?access_token=bf13fe056eaa015c12d71555374991ef8287738ec077fca5c6c41df024270360&at=18438613802
    send_resolved: true
# web
- name: web
  webhook_configs:
  - url: http://192.168.59.134:8080/prometheusalert?type=dd&tpl=prometheus-dd&ddurl=https://oapi.dingtalk.com/robot/send?access_token=bf13fe056eaa015c12d71555374991ef8287738ec077fca5c6c41df024270360&at=18538702120
# db
- name: db
  webhook_configs:
  - url: http://192.168.59.134:8080/prometheusalert?type=dd&tpl=prometheus-dd&ddurl=https://oapi.dingtalk.com/robot/send?access_token=bf13fe056eaa015c12d71555374991ef8287738ec077fca5c6c41df024270360&at=18411111111
~                                       
```

> 其中**webhook_configs:**中的**url**，为prometheusalet模板的路径地址
>
> ![image-20230404210440504](https://gitee.com/root_007/md_file_image/raw/master/202304042104583.png)
>
> 将钉钉机器人地址换成webhook，at后的手机号码填写需要@人的手机号码

##### 启动alertmanager

```bash
[root@lvs-backup alertmanager-0.23.0.linux-amd64]# nohup ./alertmanager >/dev/null 2>&1 &
```

> 没有使用**--config.file="alertmanager.yml"**，因为默认的参数可以不写，如果你的配置文件和端口号需要改变，就需要指令来指定了

##### 修改prometheus.yaml

```yaml
# my global config
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
           - 192.168.59.134:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
   - "rules/node.yaml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "node"
    static_configs:
      - targets: ["192.168.59.131:9100"]
        labels:
          env: prod
          auth: bj
      - targets: ["192.168.59.132:9100"]
        labels:
          env: test
          auth: jaic
#    relabel_configs:
#      - source_labels: ['env']
#        regex: pr.*
#        action: drop

```

> 别忘记重启，或者reload一下哦

##### 故障模拟

停止192.168.59.132服务器上的node_exporter

通过prometheus web-ui查看告警

![image-20230404211519195](https://gitee.com/root_007/md_file_image/raw/master/202304042115284.png)

通过alertmanager web-ui查看告警

![image-20230404212106965](https://gitee.com/root_007/md_file_image/raw/master/202304042121053.png)

查看钉钉群是否收到告警消息

![image-20230404212137400](https://gitee.com/root_007/md_file_image/raw/master/202304042121448.png)

> 通过@人我们可以发现，本次告警走的是**devops**接收器，其实这是因为我们192.168.59.132的label **env=test**

##### continue的使用

> 介绍：当匹配的一个路由的时候，还会继续向下一个路由继续匹配，如果规则匹配，那么也会将告警发送到下一个告警接收器
>
> 举例：192.168.59.132的**label  env=test, auth=jaic**,如果我们使用了continue，那个当匹配到**env=test**时，使用devops接收器进行告警，还会继续向下匹配 **auth=jaic**刚好也符合下一个规则，同时也会使用**web**接收器进行告警

实际场景演示

修改alertmanager.yaml

```yaml
## Alertmanager 配置文件
global:
  resolve_timeout: 1m  # 该参数定义了当Alertmanager持续多长时间未接收到告警后标记告警状态为resolved（已解决）
# 路由分组
route:
  receiver: devops  # 默认的接收器名称
  group_wait: 30s # 在组内等待所配置的时间，如果同组内，30秒内出现相同报警，在一个组内发送报警。
  group_interval: 1m # 如果组内内容不变化，合并为一条警报信息，5m后发送。
  repeat_interval: 1m # 发送报警间隔，如果指定时间内没有修复，则重新发送报警。
  group_by: [alertname,instance]  # 报警分组
  routes:  # 子路由的匹配设置
      - match:  # 匹配
          env: test  # 标签: 标签值  如果team: operations，则使用receiver: 'ops'也就是使用ops的报警
        receiver: 'devops'  # 发送警报的接收器名称
        continue: true
      - match_re: # 正则匹配
          auth: bj|jaic
        receiver: 'web'
        continue: true
      - match_re: # 正则匹配
          auth: bj
        receiver: 'db'
        continue: true


# 接收器指定发送人以及发送渠道
receivers:
# ops分组的定义
- name: devops
  # 钉钉配置
  webhook_configs:
  - url: http://192.168.59.134:8080/prometheusalert?type=dd&tpl=prometheus-dd&ddurl=https://oapi.dingtalk.com/robot/send?access_token=bf13fe056eaa015c12d71555374991ef8287738ec077fca5c6c41df024270360&at=18438613802
    send_resolved: true
# web
- name: web
  webhook_configs:
  - url: http://192.168.59.134:8080/prometheusalert?type=dd&tpl=prometheus-dd&ddurl=https://oapi.dingtalk.com/robot/send?access_token=bf13fe056eaa015c12d71555374991ef8287738ec077fca5c6c41df024270360&at=18538702120
# db
- name: db
  webhook_configs:
  - url: http://192.168.59.134:8080/prometheusalert?type=dd&tpl=prometheus-dd&ddurl=https://oapi.dingtalk.com/robot/send?access_token=bf13fe056eaa015c12d71555374991ef8287738ec077fca5c6c41df024270360&at=18411111111
```

> 重启一下alertmanager

通过alertmanager web-ui查看

![image-20230404213448110](https://gitee.com/root_007/md_file_image/raw/master/202304042134196.png)

查看钉钉告警信息

![image-20230404213416367](https://gitee.com/root_007/md_file_image/raw/master/202304042134421.png)

通过官方工具，路由树查看

![image-20230404214057573](https://gitee.com/root_007/md_file_image/raw/master/202304042140684.png)

##### 告警恢复

启动192.168.59.132服务器上的node_exporter

通过prometheus web-ui查看

![image-20230404214243656](https://gitee.com/root_007/md_file_image/raw/master/202304042142726.png)

钉钉消息恢复通知

![image-20230404214445150](https://gitee.com/root_007/md_file_image/raw/master/202304042144208.png)

> 因为我们没有去掉continue，所以这是一个实例的恢复，走了2个不同的告警接收器
>
> 告警恢复通知，webhook默认为true
>
> ![6bd3c7f9a5fcb6cb60875e5c4d7181df](https://gitee.com/root_007/md_file_image/raw/master/202304042146124.png)
>
> 如果不确定是否为true，可以自己设置

##### 告警恢复单独设置

![98b8635d99803623962f2e49f64ad6b1](https://gitee.com/root_007/md_file_image/raw/master/202304042146765.png)

##### 告警静默

查看当前告警

![image-20230406162300279](https://gitee.com/root_007/md_file_image/raw/master/202304061623360.png)

![image-20230406162348331](https://gitee.com/root_007/md_file_image/raw/master/202304061623413.png)

设置静默

![image-20230406162450250](https://gitee.com/root_007/md_file_image/raw/master/202304061624308.png)

![image-20230406162510548](https://gitee.com/root_007/md_file_image/raw/master/202304061625610.png)

![image-20230406162523724](https://gitee.com/root_007/md_file_image/raw/master/202304061625794.png)

> 选择需要静默的标签，本案例是有2个告警(其实是同一个告警，只是告警接收器不同)，因此选择**instance**的标签，将两个都静默

![image-20230406162801313](https://gitee.com/root_007/md_file_image/raw/master/202304061628400.png)

![image-20230406162811994](https://gitee.com/root_007/md_file_image/raw/master/202304061628067.png)

再次查看告警

![image-20230406162833866](https://gitee.com/root_007/md_file_image/raw/master/202304061628939.png)

> 此时告警为空，表示静默成功了

##### amtool使用

> 使用文档： https://github.com/prometheus/alertmanager
>
> 不做过多的实验了，需要使用可以参考**使用文档**进行操作

## grafana部署

192.168.59.132服务器上操作

```bash
[root@rs2 ~]#  wget  https://dl.grafana.com/enterprise/release/grafana-enterprise-9.4.2.linux-amd64.tar.gz
### 创建目录/data/soft
[root@rs2 ~]#  mkdir -p   /data/soft
### 将grafana解压到/data/soft
[root@rs2 ~]#   tar -xvf    grafana-enterprise-9.4.2.linux-amd64.tar.gz   -C   /data/soft
```

#### 查看配置文件

```bash
[root@rs2 grafana-9.4.2]# pwd
/data/soft/grafana-9.4.2
[root@rs2 grafana-9.4.2]# ll
total 24
drwxr-xr-x  2 root root   130 Mar  1 19:13 bin
drwxr-xr-x  3 root root   107 Apr  5 22:01 conf
drwxr-xr-x  7 root root    97 Apr  6 03:12 data
-rw-r--r--  1 root root 12155 Mar  1 19:11 LICENSE
-rw-r--r--  1 root root   105 Mar  1 19:11 NOTICE.md
drwxr-xr-x  3 root root    22 Mar  1 19:13 plugins-bundled
drwxr-xr-x 17 root root   291 Mar  1 19:13 public
-rw-r--r--  1 root root  2008 Mar  1 19:11 README.md
-rw-r--r--  1 root root     5 Mar  1 19:13 VERSION

```

conf/defaults.ini

```bash
...
[paths]
data = data
temp_data_lifetime = 24h
logs = data/log
plugins = data/plugins
provisioning = conf/provisioning
[server]
protocol = http
http_addr =
http_port = 3000
domain = localhost
enforce_domain = false

......

[database]
type = sqlite3
host = 127.0.0.1:3306
name = grafana
user = root
password =

......

[security]
disable_initial_admin_creation = false
admin_user = admin
admin_password = admin
admin_email = admin@localhost

```

> 由于配置文件太多了，列出了一下常用的配置
>
> 更多详细解读可以参考官网： https://grafana.com/docs/grafana/latest/introduction/

#### 启动grafana

后台方式启动

```
[root@rs2 bin]#   nohup ./grafana-server  >/dev/null 2>&1 &
```

#### 访问grafana

![image-20230406152903354](https://gitee.com/root_007/md_file_image/raw/master/202304061529713.png)

![image-20230406152932433](https://gitee.com/root_007/md_file_image/raw/master/202304061529521.png)

#### grafana对接prometheus

![1680746822460_B40EC36F-32C0-43bf-BD86-C5DDC0FCAB2A](https://gitee.com/root_007/md_file_image/raw/master/202304061530057.png)

![36d8df91dac39721befea9206281a6c6](https://gitee.com/root_007/md_file_image/raw/master/202304061530480.png)

![901175192c6aabaf8123ea27fa0edeeb](https://gitee.com/root_007/md_file_image/raw/master/202304061530400.png)

![image-20230406153150845](https://gitee.com/root_007/md_file_image/raw/master/202304061531902.png)

![1680746931739_B421E8A9-7D26-40f9-A0BC-79E4E621DA1B](https://gitee.com/root_007/md_file_image/raw/master/202304061532197.png)

#### 下载dashboard

> 下载地址：https://grafana.com/api/dashboards/11074/revisions/9/download
>
> 其他dashboard下载：https://grafana.com/grafana/dashboards/

#### 导入dashboard

![1680747645728_70D85789-8F06-4b36-96E8-4EBDBB44FCF3](https://gitee.com/root_007/md_file_image/raw/master/202304061534024.png)

![1680747663209_3682ED61-9F8C-4e83-92DA-1B7EC3221E5A](https://gitee.com/root_007/md_file_image/raw/master/202304061534407.png)

![1680747706561_A0220607-D60C-4ce0-9043-8D047B25C7A0](https://gitee.com/root_007/md_file_image/raw/master/202304061534073.png)

![1680747773622_7110FF72-5E1F-479b-B7E3-4F574F1A8120](https://gitee.com/root_007/md_file_image/raw/master/202304061535172.png)

#### 修改dashboard名字

![img](https://gitee.com/root_007/md_file_image/raw/master/202304061535400.png)

![8c884b090f5bf58ffae00b0e2a8db2cb](https://gitee.com/root_007/md_file_image/raw/master/202304061535363.png)

![19a3b9b720335f5a56fde0ebca6e4e1c](https://gitee.com/root_007/md_file_image/raw/master/202304061536697.png)

## prometheus常用接口

> https://note.youdao.com/s/P0VlCYal

## alertmanager常用接口

获取告警

> GET  `http://alertmanager访问而地址/api/v2/alerts`

发送告警

> POST  `http://alertmanager访问而地址/api/v2/alerts`
>
> body
>
> ```json
> [{ 
>     "labels": {
>         "alertname": "我只是一个测试",
>         "service": "别担心",
>         "severity":"Warning",
>         "instance": "IS-MBG-PRO-jenkins-01"
>     },
>     "annotations": {
>         "summary": "测试"
>     },
>     "startsAt": "2023-04-22T17:32:00.142+08:00",
> 
> }]
> ```
>
> 发送告警时，建议添加字段startAt(该字段是可选的)，需要使用上例的这种时间格式，当该告警不进行再次告警时，默认恢复时间为`resolve_timeout` 设置的时间(不设置默认为5分钟，该字段在`global`里)

发送恢复

> POST  `http://alertmanager访问而地址/api/v2/alerts`
>
> body
>
> ```json
> [{ 
>     "labels": {
>         "alertname": "我只是一个测试",
>         "service": "别担心",
>         "severity":"Warning",
>         "instance": "IS-MBG-PRO-jenkins-01"
>     },
>     "annotations": {
>         "summary": "测试"
>     },
>     "endsAt": "2023-04-22T17:33:00.142+08:00",
> 
> }]
> ```
>
> 当需要恢复告警时，需要将其endsAt字段添加上即可，这里的时间为恢复告警的时间
