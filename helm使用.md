#### Helm基础使用

##### helm与k8s版本对应

| Helm版本 | k8s版本       |
| -------- | ------------- |
| 3.9.x    | 1.24.x-1.21.x |
| 3.8.x    | 1.23.x-1.20.x |
| 3.7.x    | 1.22.x-1.19.x |
| 3.6.x    | 1.21.x-1.18.x |
| 3.5.x    | 1.20.x-1.17.x |
| 3.4.x    | 1.19.x-1.16.x |
| 3.3.x    | 1.18.x-1.15.x |

##### helm使用前置条件

- 一个健康的k8s集群
- 所需的helm版本

##### helm安装

- 手动安装

```shell
# helm地址： https://github.com/helm/helm
tar -zxvf helm-v3.8.0-linux-amd64.tar.gz
mv linux-amd64/helm /usr/local/bin/helm
```

- 脚本安装

```shell
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```

- 查看安装后的版本信息

```shell
helm version
version.BuildInfo{Version:"v3.8.0", GitCommit:"d14138609b01886f544b2025f5000351c9eb092e", GitTreeState:"clean", GoVersion:"go1.17.5"}
```

##### 关键字解释

- Chart：代表Helm包，它包含在k8s集群内部运行应用程序、工具或服务所需资源定义。
- Repository：仓库，是用来存放和共享charts的地方
- Releast：运行在k8s集群中的chart实例，一个chart通常可以在同一个集群中安装多次，每次安装都会创建一个新的release。

##### helm官方charts地址

```shell
https://artifacthub.io/
```

- 这里可以搜索你所需要的chart包

##### 使用命令行搜索需要的chart包

```shell
helm search hub loki
```

##### helm常用命令

- helm search hub  xxx   表示：从默认的公共chart中搜索chart包
- helm repo add brigade https://brigadecore.github.io/charts   表示：添加一个charg库，名称未brigade，地址为：https://brigadecore.github.io/charts 
- helm search repo brigade  表示：从添加的库中搜索
- helm install xxx  brigade/*  表示：`使用自定义的库安装包*,`名称为xxx
- helm status  xxx  表示：查看xxx安装的状态
- helm uninstall  xxx  表示：卸载xxx包

##### 自定义chart仓库

```shell
# 创建chart，chart文件存在与当前目录中，名称为test-chart
helm create test-chart

# chart中文件
[root@k8s-master test-chart]# ll
总用量 8
drwxr-xr-x 2 root root    6 12月 12 13:30 charts # 包含chart依赖的其他chart
-rw-r--r-- 1 root root 1146 12月 12 13:30 Chart.yaml # 包含了chart信息的yaml文件
drwxr-xr-x 3 root root  162 12月 12 13:30 templates # 模板目录，当和values结合时，可生产有效的k8s manifest文件
-rw-r--r-- 1 root root 1877 12月 12 13:30 values.yaml # chart默认的配置值
```

- 打包chart

  ```shell
  helm package test-chart
  Successfully packaged chart and saved it to: /root/secret/test-chart-0.1.0.tgz
  [root@k8s-master secret]# ll
  总用量 16
  drwxr-xr-x 4 root root   93 12月 12 13:30 test-chart
  -rw-r--r-- 1 root root 3758 12月 12 13:33 test-chart-0.1.0.tgz
  
  ```

- 使用已经打包好的chart安装

  ```shell
  [root@k8s-master secret]# helm install test-chart ./test-chart-0.1.0.tgz
  ```

- 文件说明

  Chart.yaml

  ```shell
  [root@k8s-master test-chart]# cat Chart.yaml  
  apiVersion: v2 # chart版本 必须
  name: test-chart # chart名称 必须
  description: A Helm chart for Kubernetes # 项目描述，可选
  
  # A chart can be either an 'application' or a 'library' chart.
  #
  # Application charts are a collection of templates that can be packaged into versioned archives
  # to be deployed.
  #
  # Library charts provide useful utilities or functions for the chart developer. They're included as
  # a dependency of application charts to inject those utilities and functions into the rendering
  # pipeline. Library charts do not define any templates and therefore cannot be deployed.
  type: application # chart类型 可选
  
  # This is the chart version. This version number should be incremented each time you make changes
  # to the chart and its templates, including the app version.
  # Versions are expected to follow Semantic Versioning (https://semver.org/)
  version: 0.1.0 # 语义化版本 必须
  
  # This is the version number of the application being deployed. This version number should be
  # incremented each time you make changes to the application. Versions are not expected to
  # follow Semantic Versioning. They should reflect the version the application is using.
  # It is recommended to use it with quotes.
  appVersion: "1.16.0" # 包含的应用版本（可选）。不需要是语义化，建议使用引号
  
  ```

- 安装chart到k8s中

  ```shell
  [root@k8s-master secret]# ll
  drwxr-xr-x 4 root root   93 12月 12 13:45 test-chart
  [root@k8s-master secret]# helm install demo test-chart --namespace nginx-sample-env-dev 
  NAME: demo
  LAST DEPLOYED: Mon Dec 12 13:53:39 2022
  NAMESPACE: nginx-sample-env-dev
  STATUS: deployed
  REVISION: 1
  NOTES:
  1. Get the application URL by running these commands:
    export POD_NAME=$(kubectl get pods --namespace nginx-sample-env-dev -l "app.kubernetes.io/name=test-chart,app.kubernetes.io/instance=demo" -o jsonpath="{.items[0].metadata.name}")
    export CONTAINER_PORT=$(kubectl get pod --namespace nginx-sample-env-dev $POD_NAME -o jsonpath="{.spec.containers[0].ports[0].containerPort}")
    echo "Visit http://127.0.0.1:8080 to use your application"
    kubectl --namespace nginx-sample-env-dev port-forward $POD_NAME 8080:$CONTAINER_PORT
  [root@k8s-master secret]# kubectl get pod -n nginx-sample-env-dev 
  NAME                               READY   STATUS             RESTARTS   AGE
  demo-test-chart-7cd96f47b7-df9td   0/1     Running            0          20s
  
  ```

  - install时，需要在chart所在的目录下进行

