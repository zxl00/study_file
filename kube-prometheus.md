## kube-promethesu对k8s版本支持

------



| kube-prometheus版本 | kubenetes版本 |
| ------------------- | ------------- |
| release-0.4         | 1.16、1.17    |
| release-0.5         | 1.18          |
| release-0.6         | 1.18、1.19    |
| release-0.7         | 1.19、1.20    |
| release-0.8         | 1.20、1.21    |
| release-0.9         | 1.21、1.22    |
| release-0.10        | 1.22、1.23    |
| release-0.11        | 1.23、1.24    |
| main                | 1.24          |

> 参考连接： https://github.com/prometheus-operator/kube-prometheus#compatibility

## kube-prometheus安装

### kube-prometheus地址

```shell
https://github.com/prometheus-operator/kube-prometheus
```

### kube-prometheus组件

- prometheus-operator

- prometheus

- alertmanager

- prometheus-adapter

- node-exporter

- kube-state-metrics

- grafana

- blackbox-exporter

  > 以上组件均来源与kube-prometheus release-0.8版本

### 下载kube-prometheus

```
git clone  -b release-0.8  https://github.com/prometheus-operator/kube-prometheus.git
```

```shell
[root@k8s-master kube-prometheus-release-0.8]# ll
总用量 184
-rwxr-xr-x 1 root root   679 3月  21 2022 build.sh
-rw-r--r-- 1 root root  3039 3月  21 2022 code-of-conduct.md
-rw-r--r-- 1 root root  1422 3月  21 2022 DCO
drwxr-xr-x 2 root root  4096 3月  21 2022 docs
-rw-r--r-- 1 root root  2051 3月  21 2022 example.jsonnet
drwxr-xr-x 7 root root  4096 3月  21 2022 examples
drwxr-xr-x 3 root root    28 3月  21 2022 experimental
-rw-r--r-- 1 root root   237 3月  21 2022 go.mod
-rw-r--r-- 1 root root 59996 3月  21 2022 go.sum
drwxr-xr-x 3 root root    68 3月  21 2022 hack
drwxr-xr-x 3 root root    29 3月  21 2022 jsonnet
-rw-r--r-- 1 root root   206 3月  21 2022 jsonnetfile.json
-rw-r--r-- 1 root root  4857 3月  21 2022 jsonnetfile.lock.json
-rw-r--r-- 1 root root  4495 3月  21 2022 kustomization.yaml
-rw-r--r-- 1 root root 11325 3月  21 2022 LICENSE
-rw-r--r-- 1 root root  2153 3月  21 2022 Makefile
drwxr-xr-x 3 root root  4096 9月  19 19:39 manifests
-rw-r--r-- 1 root root   126 3月  21 2022 NOTICE
-rw-r--r-- 1 root root 38246 3月  21 2022 README.md
drwxr-xr-x 2 root root   187 3月  21 2022 scripts
-rw-r--r-- 1 root root   928 3月  21 2022 sync-to-internal-registry.jsonnet
drwxr-xr-x 3 root root    17 3月  21 2022 tests
-rwxr-xr-x 1 root root   808 3月  21 2022 test.sh

```

### 部署文件清单

```shell
[root@k8s-master kube-prometheus-release-0.8]# tree manifests/ 
manifests/
├── alertmanager-alertmanager.yaml
├── alertmanager-podDisruptionBudget.yaml
├── alertmanager-prometheusRule.yaml
├── alertmanager-secret.yaml
├── alertmanager-serviceAccount.yaml
├── alertmanager-serviceMonitor.yaml
├── alertmanager-service.yaml
├── blackbox-exporter-clusterRoleBinding.yaml
├── blackbox-exporter-clusterRole.yaml
├── blackbox-exporter-configuration.yaml
├── blackbox-exporter-deployment.yaml
├── blackbox-exporter-serviceAccount.yaml
├── blackbox-exporter-serviceMonitor.yaml
├── blackbox-exporter-service.yaml
├── grafana-dashboardDatasources.yaml
├── grafana-dashboardDefinitions.yaml
├── grafana-dashboardSources.yaml
├── grafana-deployment.yaml
├── grafana-serviceAccount.yaml
├── grafana-serviceMonitor.yaml
├── grafana-service.yaml
├── istio-servicemonitor.yaml
├── kube-prometheus-prometheusRule.yaml
├── kubernetes-prometheusRule.yaml
├── kubernetes-serviceMonitorApiserver.yaml
├── kubernetes-serviceMonitorCoreDNS.yaml
├── kubernetes-serviceMonitorKubeControllerManager.yaml
├── kubernetes-serviceMonitorKubelet.yaml
├── kubernetes-serviceMonitorKubeScheduler.yaml
├── kube-state-metrics-clusterRoleBinding.yaml
├── kube-state-metrics-clusterRole.yaml
├── kube-state-metrics-deployment.yaml
├── kube-state-metrics-prometheusRule.yaml
├── kube-state-metrics-serviceAccount.yaml
├── kube-state-metrics-serviceMonitor.yaml
├── kube-state-metrics-service.yaml
├── node-exporter-clusterRoleBinding.yaml
├── node-exporter-clusterRole.yaml
├── node-exporter-daemonset.yaml
├── node-exporter-prometheusRule.yaml
├── node-exporter-serviceAccount.yaml
├── node-exporter-serviceMonitor.yaml
├── node-exporter-service.yaml
├── prometheus-adapter-apiService.yaml
├── prometheus-adapter-clusterRoleAggregatedMetricsReader.yaml
├── prometheus-adapter-clusterRoleBindingDelegator.yaml
├── prometheus-adapter-clusterRoleBinding.yaml
├── prometheus-adapter-clusterRoleServerResources.yaml
├── prometheus-adapter-clusterRole.yaml
├── prometheus-adapter-configMap.yaml
├── prometheus-adapter-deployment.yaml
├── prometheus-adapter-podDisruptionBudget.yaml
├── prometheus-adapter-roleBindingAuthReader.yaml
├── prometheus-adapter-serviceAccount.yaml
├── prometheus-adapter-serviceMonitor.yaml
├── prometheus-adapter-service.yaml
├── prometheus-clusterRoleBinding.yaml
├── prometheus-clusterRole.yaml
├── prometheus-operator-prometheusRule.yaml
├── prometheus-operator-serviceMonitor.yaml
├── prometheus-operator.yaml
├── prometheus-podDisruptionBudget.yaml
├── prometheus-prometheusRule.yaml
├── prometheus-prometheus.yaml
├── prometheus-roleBindingConfig.yaml
├── prometheus-roleBindingSpecificNamespaces.yaml
├── prometheus-roleConfig.yaml
├── prometheus-roleSpecificNamespaces.yaml
├── prometheus-serviceAccount.yaml
├── prometheus-serviceMonitor.yaml
├── prometheus-service.yaml
└── setup
    ├── 0namespace-namespace.yaml
    ├── prometheus-operator-0alertmanagerConfigCustomResourceDefinition.yaml
    ├── prometheus-operator-0alertmanagerCustomResourceDefinition.yaml
    ├── prometheus-operator-0podmonitorCustomResourceDefinition.yaml
    ├── prometheus-operator-0probeCustomResourceDefinition.yaml
    ├── prometheus-operator-0prometheusCustomResourceDefinition.yaml
    ├── prometheus-operator-0prometheusruleCustomResourceDefinition.yaml
    ├── prometheus-operator-0servicemonitorCustomResourceDefinition.yaml
    ├── prometheus-operator-0thanosrulerCustomResourceDefinition.yaml
    ├── prometheus-operator-clusterRoleBinding.yaml
    ├── prometheus-operator-clusterRole.yaml
    ├── prometheus-operator-deployment.yaml
    ├── prometheus-operator-serviceAccount.yaml
    └── prometheus-operator-service.yaml

```

### 部署

```shell
kubectl apply -f  manifests/setup/
kubectl apply -f  manifests/
```

### 验证

```shell
[root@k8s-master kube-prometheus-release-0.8]# kubectl get all -n monitoring  
NAME                                       READY   STATUS             RESTARTS   AGE
pod/alertmanager-main-0                    2/2     Running            6          115d
pod/alertmanager-main-1                    2/2     Running            4          115d
pod/alertmanager-main-2                    2/2     Running            6          115d
pod/blackbox-exporter-55c457d5fb-swjqc     3/3     Running            6          115d
pod/grafana-9df57cdc4-wc9hn                1/1     Running            2          115d
pod/kube-state-metrics-76f6cb7996-8x7pj    2/3     ImagePullBackOff   4          115d
pod/kube-state-metrics-7749b7b647-4mzsq    2/3     ImagePullBackOff   2          10d
pod/node-exporter-9tj5z                    2/2     Running            4          113d
pod/node-exporter-hsxf7                    2/2     Running            4          115d
pod/node-exporter-q8g6m                    2/2     Running            4          115d
pod/node-exporter-zngtl                    2/2     Running            4          115d
pod/prometheus-adapter-59df95d9f5-hjb7l    1/1     Running            3          115d
pod/prometheus-adapter-59df95d9f5-kdx7n    1/1     Running            4          115d
pod/prometheus-k8s-0                       2/2     Running            5          115d
pod/prometheus-k8s-1                       2/2     Running            5          115d
pod/prometheus-operator-7775c66ccf-2r99w   2/2     Running            5          115d

NAME                            TYPE        CLUSTER-IP        EXTERNAL-IP   PORT(S)                      AGE
service/alertmanager-main       ClusterIP   100.101.252.112   <none>        9093/TCP                     115d
service/alertmanager-operated   ClusterIP   None              <none>        9093/TCP,9094/TCP,9094/UDP   115d
service/blackbox-exporter       ClusterIP   100.111.30.55     <none>        9115/TCP,19115/TCP           115d
service/grafana                 ClusterIP   100.97.190.206    <none>        3000/TCP                     115d
service/kube-state-metrics      ClusterIP   None              <none>        8443/TCP,9443/TCP            115d
service/node-exporter           ClusterIP   None              <none>        9100/TCP                     115d
service/prometheus-adapter      ClusterIP   100.109.111.30    <none>        443/TCP                      115d
service/prometheus-k8s          NodePort    100.101.111.146   <none>        9090:32101/TCP               115d
service/prometheus-operated     ClusterIP   None              <none>        9090/TCP                     115d
service/prometheus-operator     ClusterIP   None              <none>        8443/TCP                     115d

NAME                           DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
daemonset.apps/node-exporter   4         4         4       4            4           kubernetes.io/os=linux   115d

NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/blackbox-exporter     1/1     1            1           115d
deployment.apps/grafana               1/1     1            1           115d
deployment.apps/kube-state-metrics    0/1     1            0           115d
deployment.apps/prometheus-adapter    2/2     2            2           115d
deployment.apps/prometheus-operator   1/1     1            1           115d

NAME                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/blackbox-exporter-55c457d5fb     1         1         1       115d
replicaset.apps/grafana-9df57cdc4                1         1         1       115d
replicaset.apps/kube-state-metrics-76f6cb7996    1         1         0       115d
replicaset.apps/kube-state-metrics-7749b7b647    1         1         0       67d
replicaset.apps/prometheus-adapter-59df95d9f5    2         2         2       115d
replicaset.apps/prometheus-operator-7775c66ccf   1         1         1       115d

NAME                                 READY   AGE
statefulset.apps/alertmanager-main   3/3     115d
statefulset.apps/prometheus-k8s      2/2     115d

```

> 说明：关于prometheus的service type上述已经修改为NodePort类型

#### 修改grafana的service类型为NodePort

- 修改前grafana-service.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
kind: Service
metadata:
  labels:
    app.kubernetes.io/component: grafana
    app.kubernetes.io/name: grafana
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 7.5.4
  name: grafana
  namespace: monitoring
spec:
  ports:
  - name: http
    port: 3000
    targetPort: http
  selector:
    app.kubernetes.io/component: grafana
    app.kubernetes.io/name: grafana
    app.kubernetes.io/part-of: kube-prometheus
   

```

- 修改后grafana-service.yaml

```yaml
kind: Service
metadata:
  labels:
    app.kubernetes.io/component: grafana
    app.kubernetes.io/name: grafana
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 7.5.4
  name: grafana
  namespace: monitoring
spec:
  type: NodePort
  ports:
  - name: http
    port: 3000
    targetPort: http
    nodePort: 32009
  selector:
    app.kubernetes.io/component: grafana
    app.kubernetes.io/name: grafana
    app.kubernetes.io/part-of: kube-prometheus

```

- 查看修改后的效果

```shell
[root@k8s-master manifests]# kubectl get svc -n monitoring  | grep grafana 
grafana                 NodePort    100.97.190.206    <none>        3000:32009/TCP               115d
```

![image-20230112173847873](https://gitee.com/root_007/md_file_image/raw/master/202302071243791.png)

#### 持久化prometheus

-  prometheus-prometheus.yaml

```yaml
  ，，，
  serviceAccountName: prometheus-k8s
  serviceMonitorNamespaceSelector: {}
  serviceMonitorSelector: {}
  version: 2.26.0
  volumeClaimTemplates:
  - metadata:
      name: db-volume
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: mysql
      resources:
        requests:
          storage: 8Gi

```

> 具体sc的创建可以参考：https://note.youdao.com/s/T71lEiDh

------



## kube-prometheus介绍

​	![img](https://gitee.com/root_007/md_file_image/raw/master/202302071243846.png)

- Operator

  > operator在k8s中以deployment类型运行，其职责为自定义资源(CRD)，用来部署和管理prometheus server，同时监控这些CRD资源事件的变化来做出相应的处理，是整个架构中的控制中心

- Prometheus

  > 1. 该CRD声明定义了Prometheus期望在k8s集群中运行的配置，提供了配置选项来配置副本、持久化、报警等
  > 2. 对于每个Prometheus CRD资源，Operator都会以sts形式在形同的名称空间下部署对应配置，proemtheus pod的配置是通过一个包含prometheus配置的名为prometheus-name的secret对象声明挂载的
  > 3. 该CRD根据标签选择来指定部署到prometheus实例应该覆盖那些ServiceMonitors，然后Operator会根据包含的ServiceMonitor生成配置，并在包含配置的secret中进行更新

- Alertmanager

  > 1. 该CRD定义了在k8s集群中运行的Alertmanager的配置，同样提供了多种配置，包含持久化存储。
  > 2. 对于每个Alertmanager资源，Operator都会在相同的名称空间中部署一个对应配置的sts，Alertmanager pod被配置为一个包含名为alertmanager-name的secret，该secret以alertmanager.yaml为key的方式保存使用的配置文件

- ThanosRuler

  > 1. 该CRD定义了一个Thanos Ruler组件的配置，以方便在k8s集群中运行，通过Thanos Ruler，可以跨多个Proemtheus实例处理记录和报警规则
  > 2. 一个ThanosRuler实例至少需要一个queryEndpoint，它指向Thanos Queriers或prometheus实例的位置，queryEndpoints用于配置Thanos运行时的--query参数

- ServiceMonitor

  > 1. 该CRD定义了如何监控一组动态的服务，使用标签来定义那些service被选择进行监控 
  > 2. 为了让Prometheus金控k8s内的任何应用，需要存在一个Endpoints对象，Endpoints对象本质上时IP地址的列表，通常Endpoints对象是由Service对象自动填充的，Service对象通过标签选择器匹配pod，并将其添加到Endpoints对象中，一个Service可以暴露一个或多个端口，这些端口由多个Endpoints列表支持，这些端点一般情况下都是指向一个pod
  > 3. 注意：endpoints是ServiceMonitor CRD中的字段，Endpoints是k8s的一种对象

- PodMonitor

  > 1. 该CRD用于定义如何监控一组动态pod，使用标签来定义那些pod被选择进行监控。

  <!--PodMonitor和ServiceMonitor的区别是不需要对应的service-->

- Probe

  > 1. 该CRD用于定义如何监控一组Ingress和静态目标，除了target之外，Probe对象还需要一个Prober，它是监控的目标并为prometheus提供指标的服务，例如可以通过使用blackbox-exporter来提供这个服务

- PrometheusRule

  > 1. 用于配置prometheus的rule规则文件，包括recording rule和alerting，可以自动被prometheus加载

- AlertmanagerConfig

  ------

  

## kube-prometheus自定义监控

- 创建需要被监控服务的pod(deployment控制器)

  ```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: check-secret-tls
    namespace: kube-ops
    labels:
      app: check-secret-tls
      release: prod
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: check-secret-tls
        release: prod
    strategy:
      rollingUpdate:
        maxSurge: 70%
        maxUnavailable: 25%
      type: RollingUpdate
    template:
      metadata:
        labels:
          app: check-secret-tls
          release: prod
      spec:
        terminationGracePeriodSeconds: 60
        containers:
        - image: xxxxxx/kube-ops/check-secret-tls:v1.0
          imagePullPolicy: Always
          name: check-secret-tls
          readinessProbe:
            httpGet:
              port: 8090
              path: /health
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 30
            failureThreshold: 10
          livenessProbe:
            httpGet:
              port: 8090
              path: /health
            initialDelaySeconds: 330
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 3
          resources:
            requests:
              cpu: 0.5
              memory: 500Mi
            limits:
              cpu: 0.5
              memory: 500Mi
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "echo 1"]
        imagePullSecrets:
        - name: cn-beijing-ali-tope365
  
  ---
  apiVersion: v1
  kind: Service
  metadata:
    labels:
      app: check-secret-tls
      release: prod
    name: check-secret-tls
    namespace: kube-ops
  spec:
    ports:
    - name: check-secret-tls
      port: 8090
      protocol: TCP
      targetPort: 8090
    selector:
      app: check-secret-tls
      release: prod
    type: ClusterIP
  ```

- 创建ServiceMonitor

  ```yaml
  apiVersion: monitoring.coreos.com/v1
  kind: ServiceMonitor
  metadata:
    name: chekc-secret-tls
    namespace: monitoring
  spec:
    endpoints:
      - interval: 15s
        path: /metrics
        port: check-secret-tls
    namespaceSelector:
      any: true
    selector:
      matchLabels:
        app: 'check-secret-tls'
  
  ```

  > metadata.name:该ServcieMonitor的名称
  >
  > metadata.namespace:该ServiceMonitor所属的名称空间
  >
  > spec.endpoints:prometheus所采集Metrics地址配置，endpoints为一个数组，可以创建多个，但是每个endpoints包含三个字段interval、path、port
  >
  > spec.endpoints.interval:prometheus采集数据的周期，单位为秒
  >
  > spec.endpoints.path:prometheus采集数据的路径
  >
  > spec.endpoints.port:prometheus采集数据的端口，这里为port的name，主要是通过spec.selector中选择对应的svc，在选中的svc中匹配该端口
  >
  > spec.namespaceSelector:需要发现svc的范围
  >
  > spec.namespaceSelector.any:有且仅有一个值true，当该字段被设置时，表示监听所有符合selector所选择的svc
  >
  > 1. 使用matchNames时：
  >
  >    ```yaml
  >    ......
  >    namespaceSelector:
  >      matchNames:
  >      - default
  >      - kube-ops
  >      ......
  >    ```
  >
  >    matchNames数组值，表示监听的namespeces的范围，上述yaml表示监控的namespaces为default和kube-ops

------



## kube-prometheus主要yaml文件介绍

- alertmanager-prometheusRule.yaml

  ```yaml
  apiVersion: monitoring.coreos.com/v1
  kind: PrometheusRule
  metadata:
    labels:
      app.kubernetes.io/component: alert-router
      app.kubernetes.io/name: alertmanager
      app.kubernetes.io/part-of: kube-prometheus
      app.kubernetes.io/version: 0.21.0
      prometheus: k8s
      role: alert-rules
    name: alertmanager-main-rules
    namespace: monitoring
  spec:
    groups:
    - name: alertmanager.rules
      rules:
      - alert: AlertmanagerFailedReload
        annotations:
          description: Configuration has failed to load for {{ $labels.namespace }}/{{ $labels.pod}}.
          runbook_url: https://github.com/prometheus-operator/kube-prometheus/wiki/alertmanagerfailedreload
          summary: Reloading an Alertmanager configuration has failed.
        expr: |
          # Without max_over_time, failed scrapes could create false negatives, see
          # https://www.robustperception.io/alerting-on-gauges-in-prometheus-2-0 for details.
          max_over_time(alertmanager_config_last_reload_successful{job="alertmanager-main",namespace="monitoring"}[5m]) == 0
        for: 10m
        labels:
          severity: critical
  #  该yaml文件截取了一部分，剩余的其实就是其他报警项目
  # 由 -alert到severity: critical为一组报警规则，当然你也可以自己定义所需的报警规则，只需由-alert到everity: critical复制粘贴即可
  ```

  > 重要标签：
  >
  > 1. prometheus:k8s
  >
  > 2. role:alert-rules
  >
  >    这两个标签是在`prometheus-prometheus.yaml`需要使用到的，通过ruleSelector来选择对应的报警
  >
  > 所以我们要想自定义一个报警规则，只需要创建一个具有 `prometheus=k8s` 和 `role=alert-rules` 标签的 `PrometheusRule` 对象就行了

- kube-prometheus-prometheusRule.yaml

  ```yaml
  apiVersion: monitoring.coreos.com/v1
  kind: PrometheusRule
  metadata:
    labels:
      app.kubernetes.io/component: exporter
      app.kubernetes.io/name: kube-prometheus
      app.kubernetes.io/part-of: kube-prometheus
      prometheus: k8s
      role: alert-rules
    name: kube-prometheus-rules
    namespace: monitoring
  spec:
    groups:
    - name: general.rules
      rules:
      - alert: TargetDown
        annotations:
          description: '{{ printf "%.4g" $value }}% of the {{ $labels.job }}/{{ $labels.service }} targets in {{ $labels.namespace }} namespace are down.'
          runbook_url: https://github.com/prometheus-operator/kube-prometheus/wiki/targetdown
          summary: One or more targets are unreachable.
        expr: 100 * (count(up == 0) BY (job, namespace, service) / count(up) BY (job, namespace, service)) > 10
        for: 10m
        labels:
          severity: warning
      - alert: Watchdog
        annotations:
          description: |
            This is an alert meant to ensure that the entire alerting pipeline is functional.
            This alert is always firing, therefore it should always be firing in Alertmanager
            and always fire against a receiver. There are integrations with various notification
            mechanisms that send a notification when this alert is not firing. For example the
            "DeadMansSnitch" integration in PagerDuty.
          runbook_url: https://github.com/prometheus-operator/kube-prometheus/wiki/watchdog
          summary: An alert that should always be firing to certify that Alertmanager is working properly.
        expr: vector(1)
        labels:
          severity: none
    - name: node-network
      rules:
      - alert: NodeNetworkInterfaceFlapping
        annotations:
          message: Network interface "{{ $labels.device }}" changing it's up status often on node-exporter {{ $labels.namespace }}/{{ $labels.pod }}
          runbook_url: https://github.com/prometheus-operator/kube-prometheus/wiki/nodenetworkinterfaceflapping
        expr: |
          changes(node_network_up{job="node-exporter",device!~"veth.+"}[2m]) > 2
        for: 2m
        labels:
          severity: warning
    - name: kube-prometheus-node-recording.rules
      rules:
      - expr: sum(rate(node_cpu_seconds_total{mode!="idle",mode!="iowait",mode!="steal"}[3m])) BY (instance)
        record: instance:node_cpu:rate:sum
      - expr: sum(rate(node_network_receive_bytes_total[3m])) BY (instance)
        record: instance:node_network_receive_bytes:rate:sum
      - expr: sum(rate(node_network_transmit_bytes_total[3m])) BY (instance)
        record: instance:node_network_transmit_bytes:rate:sum
      - expr: sum(rate(node_cpu_seconds_total{mode!="idle",mode!="iowait",mode!="steal"}[5m])) WITHOUT (cpu, mode) / ON(instance) GROUP_LEFT() count(sum(node_cpu_seconds_total) BY (instance, cpu)) BY (instance)
        record: instance:node_cpu:ratio
      - expr: sum(rate(node_cpu_seconds_total{mode!="idle",mode!="iowait",mode!="steal"}[5m]))
        record: cluster:node_cpu:sum_rate5m
      - expr: cluster:node_cpu_seconds_total:rate5m / count(sum(node_cpu_seconds_total) BY (instance, cpu))
        record: cluster:node_cpu:ratio
    - name: kube-prometheus-general.rules
      rules:
      - expr: count without(instance, pod, node) (up == 1)
        record: count:up1
      - expr: count without(instance, pod, node) (up == 0)
        record: count:up0
  
  ```

  > 该yaml与`alertmanager-prometheusRule.yaml`是定义同一的yaml文件，由于 `kind`、`prometheus: k8s`、`role: alert-rules`不难看出

- kubernetes-prometheusRule.yaml

  > 该yaml与上述两个定义同一的yaml文件，不做过多解释

- kubernetes-serviceMonitorCoreDNS.yaml

  > 关于serviceMonitor不做过多的解释，上述已经解读过，如果不清楚可以翻看该文档的 **kube-prometheus自定义监控**

- prometheus-adapterxxx.yaml

  > 1. adapter这里简单介绍一下，有兴趣的可以自行百度查看具体用法
  > 2. prometheus采集到的metrics并不能直接给k8s用，因为两者数据格式不兼容，这时就需要一个组件(prometheus-adapter)，将prometheus采集到的数据格式转换成k8s API接口能够识别的格式。

------



## kube-prometheus报警流程介绍

![](https://gitee.com/root_007/md_file_image/raw/master/202302071243775.png)

> 东西向流量简单说明：exporter-->prometheus-->alertmanager-->报警接收渠道

- exporter：可参考 https://note.youdao.com/s/EQ3Ra7MD

- prometheus对接alertmanager

  ```yaml
  global:
    scrape_interval: 30s
    scrape_timeout: 10s
    evaluation_interval: 30s
    external_labels:
      prometheus: monitoring/k8s
      prometheus_replica: prometheus-k8s-0
  alerting:
    alert_relabel_configs:
    - separator: ;
      regex: prometheus_replica
      replacement: $1
      action: labeldrop
    alertmanagers:
    - follow_redirects: true
      scheme: http
      path_prefix: /
      timeout: 10s
      api_version: v2
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_name]
        separator: ;
        regex: alertmanager-main
        replacement: $1
        action: keep
      - source_labels: [__meta_kubernetes_endpoint_port_name]
        separator: ;
        regex: web
        replacement: $1
        action: keep
      kubernetes_sd_configs:
      - role: endpoints
        follow_redirects: true
        namespaces:
          names:
          - monitoring
  rule_files:
  - /etc/prometheus/rules/prometheus-k8s-rulefiles-0/*.yaml
  ```

  > 该文件来自于prometheus web中connfiguration

  ![image-20230114155757879](https://gitee.com/root_007/md_file_image/raw/master/202302071243405.png)

  > regex: alertmanager-main
  >
  > regex:web
  >
  > 匹配的服务为alertmanager-main，端口为web
  >
  > 查看svc为alertmanager-main
  >
  > ```yaml
  > [root@k8s-master manifests]# kubectl get svc -n monitoring  | grep alertmanager-main 
  > alertmanager-main       ClusterIP   100.101.252.112   <none>        9093/TCP                     117d
  > [root@k8s-master manifests]# kubectl describe  svc alertmanager-main   -n monitoring  
  > Name:              alertmanager-main
  > Namespace:         monitoring
  > Labels:            alertmanager=main
  >                    app.kubernetes.io/component=alert-router
  >                    app.kubernetes.io/name=alertmanager
  >                    app.kubernetes.io/part-of=kube-prometheus
  >                    app.kubernetes.io/version=0.21.0
  > Annotations:       <none>
  > Selector:          alertmanager=main,app.kubernetes.io/component=alert-router,app.kubernetes.io/name=alertmanager,app.kubernetes.io/part-of=kube-prometheus,app=alertmanager
  > Type:              ClusterIP
  > IP Families:       <none>
  > IP:                100.101.252.112
  > IPs:               100.101.252.112
  > Port:              web  9093/TCP
  > TargetPort:        web/TCP
  > Endpoints:         10.244.129.125:9093,10.244.32.198:9093,10.244.32.254:9093
  > Session Affinity:  ClientIP
  > Events:            <none>
  > 
  > ```
  >
  > 该svc正是由`alertmanager-service.yaml`生成

------



## kube-prometheus自动发现

- 配置自动发现

  ```yaml
  - job_name: "endpoints"
    kubernetes_sd_configs:
      - role: endpoints
    relabel_configs: # 指标采集之前或采集过程中去重新配置
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
        action: keep # 保留具有 prometheus.io/scrape=true 这个注解的Service
        regex: true
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels:
          [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
        action: replace
        target_label: __address__
        regex: ([^:]+)(?::\d+)?;(\d+) # RE2 正则规则，+是一次多多次，?是0次或1次，其中?:表示非匹配组(意思就是不获取匹配结果)
        replacement: $1:$2
      - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
        action: replace
        target_label: __scheme__
        regex: (https?)
      - action: labelmap
        regex: __meta_kubernetes_service_label_(.+)
        replacement: $1
      - source_labels: [__meta_kubernetes_namespace]
        action: replace
        target_label: kubernetes_namespace
      - source_labels: [__meta_kubernetes_service_name]
        action: replace
        target_label: kubernetes_service
      - source_labels: [__meta_kubernetes_pod_name]
        action: replace
        target_label: kubernetes_pod
      - source_labels: [__meta_kubernetes_node_name]
        action: replace
        target_label: kubernetes_node
  
  ```

  > 将上述文件保存为`prometheus-additional.yaml`

- 创建secret

  ```shell
  kubectl create secret generic additional-configs --from-file=prometheus-additional.yaml -n monitoring
  ```

  > 其中 `-from-file=prometheus-additional.yaml ` 该文件就是上边生成的yaml文件

- 修改prometheus-prometheus.yaml

  ```yaml
    ......
    additionalScrapeConfigs:
      name: additional-configs
      key: prometheus-additional.yaml
  
  ```

  > 在该文件末尾处添加

- 更新prometheus-prometheus.yaml

  ```shell
  kubectl apply -f prometheus-prometheus.yaml
  ```

  > 补充：如果版本过低，在`kubectl logs -f prometheus-k8s-0 prometheus -n monitoring`日志中会出现`forbidden`报错的字段，主要是因为权限设置的问题(rbac)
  >
  > prometheus关联的ServiceAccount为 `serviceAccountName: prometheus-k8s` 该ServiceAccount配置来自于`prometheus-prometheus.yaml`，然后通过`serviceAccountName: prometheus-k8s`查找发现其绑定的文件为`prometheus-clusterRole.yaml`
  >
  > prometheus-clusterRole.yaml
  >
  > ```yaml
  > apiVersion: rbac.authorization.k8s.io/v1
  > kind: ClusterRole
  > metadata:
  >   labels:
  >     app.kubernetes.io/component: prometheus
  >     app.kubernetes.io/name: prometheus
  >     app.kubernetes.io/part-of: kube-prometheus
  >     app.kubernetes.io/version: 2.26.0
  >   name: prometheus-k8s
  > rules:
  > - apiGroups:
  >   - ""
  >   resources:
  >   - nodes/metrics
  >   - services
  >   - endpoints
  >   - pods
  >   verbs:
  >   - get
  >   - list
  >   - watch
  > - nonResourceURLs:
  >   - /metrics
  >   verbs:
  >   - get
  > 
  > ```
  >
  > 其中resources中包含了services、pods的资源，不需要更改

- 通过prometheus web查看targets

  ![image-20230114173547537](https://gitee.com/root_007/md_file_image/raw/master/202302071243774.png)

  ------

  

- prometheus自动发现自定义service

  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    annotations:
      prometheus.io/scrape: "true"
    labels:
      app: check-secret-tls
      release: prod
    name: check-secret-tls
    namespace: kube-ops
  spec:
    ports:
    - name: check-secret-tls
      port: 8090
      protocol: TCP
      targetPort: 8090
    selector:
      app: check-secret-tls
      release: prod
    type: ClusterIP
  
  ```

  > 添加注解：`prometheus.io/scrape: "true"` 即可

- 验证自动发现是否生效

  ![image-20230114180454785](https://gitee.com/root_007/md_file_image/raw/master/202302071243750.png)

  > 忽略新加入的targets状态为down，这是因为我环境问题，导致程序启动失败
  >
  > 补充：关于exporter自定义开发可以参考：https://note.youdao.com/s/EQ3Ra7MD

- 关于metrics路径

  特殊项目，需要指定metrics

  ```yaml
  apiVersion: monitoring.coreos.com/v1
  kind: ServiceMonitor
  metadata:
    name: chekc-secret-tls
    namespace: monitoring
  spec:
    endpoints:
      - interval: 15s
        path: /api/metrics
        port: check-secret-tls
    namespaceSelector:
      any: true
    selector:
      matchLabels:
        app: 'check-secret-tls'
  
  ```

  > 1. 创建ServiceMonitor，通过`selector.matchLabels`标签进行匹配svc，这时匹配到的svc就不需要添加加注解：`prometheus.io/scrape: "true"`
  > 2. 其中`spec.endpoints.path` 该参数可以指定metrics的路径

- 效果展示

  ![image-20230114192948623](https://gitee.com/root_007/md_file_image/raw/master/202302071243967.png)

  > 请忽略状态为down，这是因为仅测试prometheus的功能

  如果需要使用自动发现更改metrics路径，适用于以后的所有项目

  - 更改prometheus-additional.yaml

    ```yaml
    - job_name: "endpoints"
      metrics_path: /api/v2/metrics
      kubernetes_sd_configs:
        - role: endpoints
      relabel_configs: # 指标采集之前或采集过程中去重新配置
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
          action: keep # 保留具有 prometheus.io/scrape=true 这个注解的Service
          regex: true
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
          action: replace
          target_label: __metrics_path__
          regex: (.+)
        - source_labels:
            [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
          action: replace
          target_label: __address__
          regex: ([^:]+)(?::\d+)?;(\d+) # RE2 正则规则，+是一次多多次，?是0次或1次，其中?:表示非匹配组(意思就是不获取匹配结果)
          replacement: $1:$2
        - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
          action: replace
          target_label: __scheme__
          regex: (https?)
        - action: labelmap
          regex: __meta_kubernetes_service_label_(.+)
          replacement: $1
        - source_labels: [__meta_kubernetes_namespace]
          action: replace
          target_label: kubernetes_namespace
        - source_labels: [__meta_kubernetes_service_name]
          action: replace
          target_label: kubernetes_service
        - source_labels: [__meta_kubernetes_pod_name]
          action: replace
          target_label: kubernetes_pod
        - source_labels: [__meta_kubernetes_node_name]
          action: replace
          target_label: kubernetes_node
    
    ```

    > 添加字段 `metrics_path: /api/v2/metrics`，更多字段可以参考：https://prometheus.io/docs/prometheus/latest/configuration/configuration/
    >
    > 下面步骤参考 本章节的 创建secret-->修改prometheus-prometheus.yaml-->更新prometheus-prometheus.yaml

## kube-prometheus自定义告警规则

- 在kube-prometheus主要yaml文件介绍的时候，已经说过，`prometheus-prometheusRule.yaml`这个定义的其实是报警规则，其`kind: PrometheusRule`，metadata.labels为`prometheus: k8s`、`role: alert-rules`，至于为什么添加这两个标签，这是因为`prometheus-prometheus.yaml` 这里的`spec.ruleSelector.metchLabels`进行匹配的

- 创建报警规则yaml文件

  customize_rule.yaml

  ```yaml
  apiVersion: monitoring.coreos.com/v1
  kind: PrometheusRule
  metadata:
    labels:
      prometheus: k8s
      role: alert-rules
    name: customize-rules
    namespace: monitoring
  spec:
    groups:
      - name: test
        rules:
          - alert: CustomizeRule
            annotations:
              summary: CustomizeRule summary
              description: Customize Rule use for test
            expr: |
              coredns_forward_request_duration_seconds_bucket >=3000
            for: 3m
            labels:
              severity: warning
  
  ```

  > 创建报警规则  `kubectl  apply  -f  customize_rule.yaml`
  >
  > 补充：当然也可以直接在 `prometheus-prometheusRule.yaml`继续添加报警规则

- 查看告警创建结果

  ![image-20230115140715749](https://gitee.com/root_007/md_file_image/raw/master/202302071243614.png)

  ![image-20230115140822411](https://gitee.com/root_007/md_file_image/raw/master/202302071243695.png)

  > 至此报警规则已经创建完毕，根据自己的需求创建即可

------

## kube-prometheus自定义告警渠道

- 常用告警渠道简单介绍几种

  > 1. 邮件
  > 2. webhook

- 查看alertmanager web

  ![image-20230115171119050](https://gitee.com/root_007/md_file_image/raw/master/202302071243104.png)

  > ​	该config文件，其实是`alertmanager-secret.yaml`定义的，经过base64加密，具体信息可以查看`alertmanager-main`secret

- webhook告警渠道自定义

  1. 定义告警配置

     alertmanager-config.yaml

     ```yaml
     apiVersion: monitoring.coreos.com/v1alpha1
     kind: AlertmanagerConfig
     metadata:
       name: config-example
       namespace: monitoring
       labels:
         alertmanagerConfig: example
     spec:
       route:
         groupBy: ['job']
         groupWait: 30s
         groupInterval: 5m
         repeatInterval: 12h
         receiver: 'webhook'
       receivers:
       - name: 'webhook'
         webhookConfigs:
         - url: 'http://192.168.10.70:8008/api/v1/ping'
     ```

  2. 修改alertmanager-alertmanager.yaml 

     ```yaml
       ......
       configSecret:
       alertmanagerConfigSelector: # 匹配 AlertmanagerConfig 的标签
         matchLabels:
           alertmanagerConfig: example
     ```

     > 该文件末尾添加标签选择

  3. 更新修改的文件

     ```shell
     kubectl apply -f  alertmanager-config.yaml
      kubectl apply -f alertmanager-alertmanager.yaml 
     ```

     

  4. 再次查看alertmanager web

     ![image-20230115171721311](https://gitee.com/root_007/md_file_image/raw/master/202302071244024.png)

  5. 查看url接口(/api/v1/ping)

     ![image-20230115171811951](img/image-20230115171811951.png)

     > ​	这个是我仅做测试使用的，该接口(/api/v1/ping)经过测试验证了自定义告警渠道webhook是ok的

  6. webhook告警工具推荐

     > **[PrometheusAlert](https://github.com/feiyu563/PrometheusAlert)**

  7. 更多告警渠道

     > https://prometheus.io/docs/alerting/0.21/configuration/

  8. 参考资料

     > https://prometheus.io/docs/alerting/0.21/configuration/
     >
     > https://github.com/prometheus-operator/prometheus-operator/blob/main/Documentation/user-guides/alerting.md

------

## Thanos

### 优势

- 全局视图
- 长期存储
- 兼容prometheus

### 架构图

![Thanos High Level Arch Diagram](https://gitee.com/root_007/md_file_image/raw/master/202302071244903.png)

> 1. Sidecar：连接prometheus，读取其数据进行查询或上传到云存储
> 2. Store Gateway：访问放在对象存储中的指标数据
> 3. Compact：采样压缩，清理对象存储中的数据
> 4. Ruler：根据Thanos中的数据评估记录和报警规则，进行展示、上传
> 5. Query：实现prometheus的api v1聚合
> 6. object Storage：对象存储
> 7. Frontend：查询缓存

### thanos+minio+kube-prometheus对接

​	minio需要提前部署，具体可参考：https://note.youdao.com/s/KyC0xeI3

- 修改prometheus-prometheus.yaml

  ```yaml
    thanos:
      baseImage: quay.io/thanos/thanos
      version: v0.29.0
      objectStorageConfig:
        key: thanos.yaml
        name: thanos-objstore-config
  ```

  > 在文件末尾处添加，主要作用是添加sidecar
  >
  > objectStorageConfig：这里使用secret的方式

- 创建secret供prometheus-prometheus.yaml中objectStorageConfig使用

  - thanos-config.yaml文件

    ```yaml
    type: s3
    config:
      bucket: thanos
      endpoint: 192.168.70.201:9090
      access_key: admin
      secret_key: adminadmin
      insecure: true
      signature_version2: false
      http_config:
        idle_conn_timeout: 90s
        insecure_skip_verify: true
    ```

    > bucket：存储桶
    >
    > ccess_key：登录minio的账号
    >
    > ecret_key：登录minio的密码
    >
    > insecure: true 使用http
    >
    > idle_conn_timeout：超时时间配置
    >
    > insecure_skip_verify: true 跳过TLS证书验证
    >
    > 更多配置参数可以参考： https://thanos.io/tip/thanos/storage.md/#s3

  - 创建secret

    ```shell
    kubectl -n monitoring  create secret generic thanos-objstore-config  --from-file=thanos.yaml=./thanos-config.yaml
    ```

    

- 更新prometheus-prometheus.yaml后查看pod

  ```shell
  [root@k8s-master manifests]# kubectl get pod -n monitoring  | grep prometheus-k8s
  prometheus-k8s-0                       3/3     Running            1          22h
  prometheus-k8s-1                       3/3     Running            1          22h
  ```

  > 其中3个pod已经running，3个pod中有sidecar
  >
  > ```shell
  >  [root@k8s-master manifests]# kubectl describe pod prometheus-k8s-0 -n monitoring
  >  thanos-sidecar:
  >     Container ID:  docker://b8ee9554fc2f4cb37480987a5a65da2c26c8aa16395103fed79c2ea1cdf043b9
  >     Image:         quay.io/thanos/thanos:v0.29.0
  >     Image ID:      docker-pullable://quay.io/thanos/thanos@sha256:4766a6caef0d834280fed2d8d059e922bc8781e054ca11f62de058222669d9dd
  >     Ports:         10902/TCP, 10901/TCP
  >     Host Ports:    0/TCP, 0/TCP
  >     Args:
  >       sidecar
  >       --prometheus.url=http://localhost:9090/
  >       --grpc-address=[$(POD_IP)]:10901
  >       --http-address=[$(POD_IP)]:10902
  >       --objstore.config=$(OBJSTORE_CONFIG)
  >       --tsdb.path=/prometheus
  > 
  > ```

- 下载kube-thanos需要的yaml文件

  ```shell
  git clone https://github.com/thanos-io/kube-thanos
  ```

  > 所需文件：manifests目录中

- kube-thanos清单

  ```shell
  [root@k8s-master manifests]# ll
  总用量 72
  -rw-r--r-- 1 root root 2604 1月  31 09:52 thanos-query-deployment.yaml
  -rw-r--r-- 1 root root  285 11月  3 23:20 thanos-query-serviceAccount.yaml
  -rw-r--r-- 1 root root  603 11月  3 23:20 thanos-query-serviceMonitor.yaml
  -rw-r--r-- 1 root root  539 1月  30 11:19 thanos-query-service.yaml
  -rw-r--r-- 1 root root  790 11月  3 23:20 thanos-receive-ingestor-default-service.yaml
  -rw-r--r-- 1 root root 4779 11月  3 23:20 thanos-receive-ingestor-default-statefulSet.yaml
  -rw-r--r-- 1 root root  321 11月  3 23:20 thanos-receive-ingestor-serviceAccount.yaml
  -rw-r--r-- 1 root root  729 11月  3 23:20 thanos-receive-ingestor-serviceMonitor.yaml
  -rw-r--r-- 1 root root  268 11月  3 23:20 thanos-receive-router-configmap.yaml
  -rw-r--r-- 1 root root 2676 11月  3 23:20 thanos-receive-router-deployment.yaml
  -rw-r--r-- 1 root root  308 11月  3 23:20 thanos-receive-router-serviceAccount.yaml
  -rw-r--r-- 1 root root  661 11月  3 23:20 thanos-receive-router-service.yaml
  -rw-r--r-- 1 root root  294 11月  3 23:20 thanos-store-serviceAccount.yaml
  -rw-r--r-- 1 root root  621 11月  3 23:20 thanos-store-serviceMonitor.yaml
  -rw-r--r-- 1 root root  560 11月  3 23:20 thanos-store-service.yaml
  -rw-r--r-- 1 root root 3331 1月  30 18:10 thanos-store-statefulSet.yaml
  
  ```

- 修改thanos-query-deployment.yaml

  ```yaml
        containers:
        - args:
          - query
          - --grpc-address=0.0.0.0:10901
          - --http-address=0.0.0.0:9090
          - --log.level=info
          - --log.format=logfmt
          - --query.replica-label=prometheus_replica
          - --query.replica-label=rule_replica
          - --store=dnssrv+prometheus-operated.monitoring.svc.cluster.local:10901
          - --query.auto-downsampling
  
  ```

  > 添加  `--store=dnssrv+prometheus-operated.monitoring.svc.cluster.local:10901`，这里使用跨名称空间访问prometheus的svc

- 创建thanos的名称空间

  ```shell
  kubectl create ns thanos
  ```

- 更新query相关yaml

  ```shell
  kubectl apply -f  thanos-query-deployment.yaml -f  thanos-query-serviceAccount.yaml -f  thanos-query-serviceMonitor.yaml -f thanos-query-service.yaml
  ```

- 修改query的svc类型

  thanos-query-service.yaml

  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    labels:
      app.kubernetes.io/component: query-layer
      app.kubernetes.io/instance: thanos-query
      app.kubernetes.io/name: thanos-query
      app.kubernetes.io/version: v0.29.0
    name: thanos-query
    namespace: thanos
  spec:
    type: NodePort
    ports:
    - name: grpc
      port: 10901
      targetPort: 10901
    - name: http
      port: 9090
      targetPort: 9090
    selector:
      app.kubernetes.io/component: query-layer
      app.kubernetes.io/instance: thanos-query
      app.kubernetes.io/name: thanos-query
  
  ```

- 访问query截图

  ![image-20230201102342449](https://gitee.com/root_007/md_file_image/raw/master/202302071244304.png)

- 修改thanos-store-statefulSet.yaml

  ```yaml
    volumeClaimTemplates:
     - metadata:
         labels:
           app.kubernetes.io/component: object-store-gateway
           app.kubernetes.io/instance: thanos-store
           app.kubernetes.io/name: thanos-store
         name: data
       spec:
         storageClassName: 'nfs-storage'
         accessModes:
         - ReadWriteOnce
         resources:
           requests:
             storage: 10Gi
  
  ```

  > 文件末尾处 修改处：storageClassName: 'nfs-storage'
  >
  > sc需要自行提前创建，具体可参考：https://note.youdao.com/s/IGwRd3gk
  >
  > 补充：由于该实验k8s版本为：v1.20.15，使用sc是需要修改 `/etc/kubernetes/manifests/kube-apiserver.yaml`
  >
  > 否则挂载失败，主要是k8s版本更新，去掉了一些字段
  >
  > 添加 `- --feature-gates=RemoveSelfLink=false`
  >
  > ![image-20230201103326686](img/image-20230201103326686.png)

- 更新store相关yaml

  ```shell
  kubectl apply -f thanos-store-serviceAccount.yaml -f thanos-store-serviceMonitor.yaml -f thanos-store-service.yaml -f thanos-store-statefulSet.yaml
  ```

- 修改thanos-query-deployment.yaml对接store

  ```yaml
     ......
     containers:
        - args:
          - query
          - --grpc-address=0.0.0.0:10901
          - --http-address=0.0.0.0:9090
          - --log.level=info
          - --log.format=logfmt
          - --query.replica-label=prometheus_replica
          - --query.replica-label=rule_replica
          - --store=dnssrv+prometheus-operated.monitoring.svc.cluster.local:10901
          - --store=dnssrv+_grpc._tcp.thanos-store.thanos.svc.cluster.local:10901
          - --query.auto-downsampling
          env:
          - name: HOST_IP_ADDRESS
            valueFrom:
              fieldRef:
                fieldPath: status.hostIP
          image: quay.io/thanos/thanos:v0.29.0
          imagePullPolicy: IfNotPresent
          ......
  
  ```

  > 添加 `- --store=dnssrv+_grpc._tcp.thanos-store.thanos.svc.cluster.local:10901`

- 更新thanos-query-deployment.yaml

  ```shell
  kubectl apply -f thanos-query-deployment.yaml
  ```

- 访问query截图

  ![image-20230201105147830](https://gitee.com/root_007/md_file_image/raw/master/202302071244240.png)

- 补充

  - 多集群对接，类似上边的步骤，当然使用kube-prometheus部署的prometheus，其会有external_labels` 标签为 `prometheus_replica

### thanos之query

<img src="https://gitee.com/root_007/md_file_image/raw/master/202302071244144.svg" alt="thanos-querier"  />

> ​	当使用thanos-query进行指标查询时，通过storeApi grpc进行查询的

### Query与Sidecar

![image-20230131145646042](https://gitee.com/root_007/md_file_image/raw/master/202302071244230.png)



### Sicecar上传数据到对象存储

![image-20230131153302184](https://gitee.com/root_007/md_file_image/raw/master/202302071244621.png)