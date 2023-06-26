## jenkins介绍

> 持续集成和持续交付
>
> 易于安装
>
> 易于配置
>
> 插件化
>
> 分布式
>
> 官网地址： https://www.jenkins.io/

## 安装前准备

- 一套k8s环境

  | 主机名      | IP             | 角色       |
  | ----------- | -------------- | ---------- |
  | k8s-master  | 192.168.70.200 | k8s master |
  | k8s-node202 | 192.168.70.202 | k8s node   |
  | k8s-node203 | 192.168.70.203 | k8s node   |
  | k8s-node201 | 192.168.70.201 | k8s node   |
  | vms_210     | 192.168.70.210 | nfs server |

## jenkins安装

本文主讲在k8s中安装以及使用jenkins

#### 使用docker安装jenkins

> 参考： https://www.jenkins.io/doc/book/installing/docker/

#### 使用k8s安装jenkins

- 创建jenkins使用的ns

  192.168.70.200服务器上

```bash
kubectl create namespace jenkins
```

- 创建sa

  192.168.70.200服务器上

```yaml
cat >>./jenkins-sa.yaml<<EOF
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: jenkins-admin
rules:
  - apiGroups: [""]
    resources: ["*"]
    verbs: ["*"]
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins-admin
  namespace: jenkins
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: jenkins-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: jenkins-admin
subjects:
- kind: ServiceAccount
  name: jenkins-admin
  namespace: jenkins

EOF
```

```bash
kubectl apply -f jenkins-sa.yaml
```

- 创建nfs服务端

  192.168.70.210服务器上

  ```bash
  ###  安装nfs
  yum install nfs-utils  -y
  ### 创建nfs目录
  mkdir  -p    /data/nfs
  echo    "/data/nfs *(rw,sync,insecure,no_subtree_check,no_root_squash)"   >>   /etc/exports
  ###  启动nfs
  systemctl start nfs-server
  ### 设置为开机启动
  systemctl enable  nfs-server
  ### 设置一下/data的权限
  chmod 777 /data -R
  ```

  > /data/nfs *(rw,sync,insecure,no_subtree_check,no_root_squash)说明：
  >
  > - /data/nfs： nfs server 目录
  > - `*` ： 表示所有的服务器都可以挂载该nfs，当然也可以指定ip地址/ip地址段
  > - rw： 读写权限
  > - sync：同步写入,即数据写入服务器后再返回响应，保证数据的可靠性和一致性
  > - insecure： 表示不进行端口校验，允许客户端使用非保留端口进行连接
  > - no_subtree_check: 禁止子树检查
  > - no_root_squash: 表示禁用 root 用户映射机制，即来自客户端的 root 用户被映射为匿名用户而没有特权

- 其他服务器上也需要安装

  ```bash
  ###  安装nfs
  yum install nfs-utils  -y
  ```

- 创建nfs client

  192.168.70.200服务器上

  ```bash
  cat >>./nfs-client-provisioner.yaml <<EOF
  kind: Deployment
  apiVersion: apps/v1
  metadata:
    name: nfs-client-provisioner
    namespace: jenkins
  spec:
    replicas: 1
    strategy:
      type: Recreate
    selector:
      matchLabels:
        app: nfs-client-provisioner
    template:
      metadata:
        labels:
          app: nfs-client-provisioner
      spec:
        serviceAccountName: nfs-client-provisioner
        containers:
          - name: nfs-client-provisioner
            image:  quay.io/external_storage/nfs-client-provisioner:latest
            volumeMounts:
              - name: nfs-client-root
                mountPath: /persistentvolumes
            env:
              - name: PROVISIONER_NAME
                value: nfs-client                  # 注意这里的值不能有下划线 _
              - name: NFS_SERVER
                value: 192.168.70.210
              - name: NFS_PATH
                value: /data/nfs
        volumes:
          - name: nfs-client-root
            nfs:
              server: 192.168.70.210
              path: /data/nfs
  
  ## 创建 RBAC 授权
  ---
  apiVersion: v1
  kind: ServiceAccount
  metadata:
    name: nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: jenkins
  ---
  kind: ClusterRole
  apiVersion: rbac.authorization.k8s.io/v1
  metadata:
    name: nfs-client-provisioner-runner
  rules:
    - apiGroups: [""]
      resources: ["persistentvolumes"]
      verbs: ["get", "list", "watch", "create", "delete"]
    - apiGroups: [""]
      resources: ["persistentvolumeclaims"]
      verbs: ["get", "list", "watch", "update"]
    - apiGroups: ["storage.k8s.io"]
      resources: ["storageclasses"]
      verbs: ["get", "list", "watch"]
    - apiGroups: [""]
      resources: ["events"]
      verbs: ["create", "update", "patch"]
  ---
  kind: ClusterRoleBinding
  apiVersion: rbac.authorization.k8s.io/v1
  metadata:
    name: run-nfs-client-provisioner
  subjects:
    - kind: ServiceAccount
      name: nfs-client-provisioner
      # replace with namespace where provisioner is deployed
      namespace: jenkins
  roleRef:
    kind: ClusterRole
    name: nfs-client-provisioner-runner
    apiGroup: rbac.authorization.k8s.io
  ---
  kind: Role
  apiVersion: rbac.authorization.k8s.io/v1
  metadata:
    name: leader-locking-nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: jenkins
  rules:
    - apiGroups: [""]
      resources: ["endpoints"]
      verbs: ["get", "list", "watch", "create", "update", "patch"]
  ---
  kind: RoleBinding
  apiVersion: rbac.authorization.k8s.io/v1
  metadata:
    name: leader-locking-nfs-client-provisioner
    # replace with namespace where provisioner is deployed
    namespace: jenkins
  subjects:
    - kind: ServiceAccount
      name: nfs-client-provisioner
      # replace with namespace where provisioner is deployed
      namespace: jenkins
  roleRef:
    kind: Role
    name: leader-locking-nfs-client-provisioner
    apiGroup: rbac.authorization.k8s.io
  EOF
  
  ```

  ```bash
  kubectl   apply   -f   nfs-client-provisioner.yaml
  ```

- 创建sc

  ```bash
  cat >>./storageclass.yaml <<EOF
  apiVersion: storage.k8s.io/v1
  kind: StorageClass
  metadata:
    name: nfs-jenkins
    namespace: jenkins
  provisioner: nfs-client
  parameters:
    archiveOnDelete: "true" # "false" 删除PVC时不会保留数据，"true"将保留PVC数据
  
  EOF
  ```

  ```
  kubectl   apply   -f   storageclass.yaml
  ```

- 创建pvc

  ```bash
  cat >>./jenkins-pvc.yaml <<EOF
  kind: PersistentVolumeClaim
  apiVersion: v1
  metadata:
    name: jenkins-pvc
    namespace: jenkins
  spec:
    accessModes:
      - ReadWriteOnce
    storageClassName: nfs-jenkins 
    resources:
      requests:
        storage: 20Gi
  EOF
  ```

  ```bash
  kubectl   apply   -f    jenkins-pvc.yaml
  ```

- 创建jenkins

  ```bash
  cat >>./jenkins-deployment.yaml  <<EOF
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: jenkins
    namespace: jenkins
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: jenkins-server
    template:
      metadata:
        labels:
          app: jenkins-server
      spec:
        securityContext:
              fsGroup: 1000
              runAsUser: 1000
        serviceAccountName: jenkins-admin
        containers:
          - name: jenkins
            image:  jenkins/jenkins:2.405
            resources:
              limits:
                memory: "2Gi"
                cpu: "1000m"
              requests:
                memory: "500Mi"
                cpu: "500m"
            ports:
              - name: httpport
                containerPort: 8080
              - name: jnlpport
                containerPort: 50000
            livenessProbe:
              httpGet:
                path: "/login"
                port: 8080
              initialDelaySeconds: 90
              periodSeconds: 10
              timeoutSeconds: 5
              failureThreshold: 5
            readinessProbe:
              httpGet:
                path: "/login"
                port: 8080
              initialDelaySeconds: 60
              periodSeconds: 10
              timeoutSeconds: 5
              failureThreshold: 3
            volumeMounts:
              - name: jenkins-data
                mountPath: /var/jenkins_home
        volumes:
          - name: jenkins-data
            persistentVolumeClaim:
                claimName: jenkins-pvc
  
  EOF
  ```

  ```bash
  kubectl   apply  -f   jenkins-deployment.yaml 
  ```

- 创建jenkins svc

  ```bash
  cat >>./jenkins-svc.yaml <<EOF
  apiVersion: v1
  kind: Service
  metadata:
    name: jenkins-service
    namespace: jenkins
    annotations:
        prometheus.io/scrape: 'true'
        prometheus.io/path:   /
        prometheus.io/port:   '8080'
  spec:
    selector:
      app: jenkins-server
    type: NodePort
    ports:
      - port: 8080
        targetPort: 8080
        nodePort: 32000
  EOF
  ```

  ```
  kubectl  apply   -f   jenkins-svc.yaml
  ```

- 兼容性问题

  > **兼容性**
  >
  > 自版本 v1.20 开始，Kubernetes 默认不再提供 metadata.selfLink 字段，然而 nfs-client-provisioner 并没有随之更新。如果您想继续使用 nfs-client-provisioner 您需要 [配置 Kubernetes ApiServer](https://kuboard.cn/install/faq/selfLink.html) 以支持 metadata.selfLink 字段。

## 访问jenkins

> 访问地址： k8sip:32000

- 访问UI展示

  ![image-20230510200522435](https://gitee.com/root_007/md_file_image/raw/master/202305102005596.png)

- 如何查找密码

  1. 在nfs server(192.168.70.210) 中，`进入挂载点/secrets`  打开文件`initialAdminPassword`
  2. 进入pod自行查看

- 安装推荐插件

  ![image-20230511104249977](https://gitee.com/root_007/md_file_image/raw/master/202305111042044.png)

- 更改插件安装源

  ![image-20230517181116566](https://gitee.com/root_007/md_file_image/raw/master/202305171811691.png)

  > https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json

- 创建管理员用户

  自行创建即可

- jenkins UI

  ![image-20230517182713331](https://gitee.com/root_007/md_file_image/raw/master/202305171827408.png)

## 插件安装

- 安装mysql插件

  ![image-20230517182746703](https://gitee.com/root_007/md_file_image/raw/master/202305171827758.png)

- 重启jenkins

  ![image-20230517183047307](https://gitee.com/root_007/md_file_image/raw/master/202305171830343.png)

- 安装kubernetes插件

  ![image-20230517183405680](https://gitee.com/root_007/md_file_image/raw/master/202305230944554.png)

- List Git Branches Parameterr插件

- 配置kubernetes

  ![image-20230521170504460](https://gitee.com/root_007/md_file_image/raw/master/202305211705568.png)

  ![image-20230521170549118](https://gitee.com/root_007/md_file_image/raw/master/202305211705186.png)

  ![image-20230521170600982](https://gitee.com/root_007/md_file_image/raw/master/202305211706019.png)

## 创建流水线任务

![image-20230521170815719](https://gitee.com/root_007/md_file_image/raw/master/202305211708782.png)

## 自定义jenkins-slave镜像

Dockerfile文件

```bash
FROM centos:7
RUN echo "Asia/shanghai" > /etc/timezone
RUN cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
COPY jenkins-agent  /usr/local/bin/jenkins-agent
RUN ln -s /usr/local/bin/jenkins-agent /usr/local/bin/jenkins-slave
COPY agent.jar /usr/share/jenkins/
RUN ln -s /usr/share/jenkins/agent.jar  /usr/share/jenkins/slave.jar
RUN chmod a+x /usr/local/bin/jenkins-agent
RUN yum -y install git  kde-l10n-Chinese glibc-common &&  yum clean all && localedef -c -f UTF-8 -i zh_CN zh_CN.utf8 && \
echo 'LANG="zh_CN.UTF-8"' > /etc/locale.conf && \
source /etc/locale.conf
RUN useradd -m -d /home/jenkins  jenkins
COPY openjdk /opt/openjdk
COPY apache-maven-3.8.1 /opt/apache-maven-3.8.1
ENV JAVA_HOME /opt/openjdk
ENV MAVEN_HOME /opt/apache-maven-3.8.1
ENV PATH "$PATH:/usr/local/bin:$JAVA_HOME/bin:$MAVEN_HOME/bin"
ENV LANG=zh_CN.UTF-8
WORKDIR /home/jenkins
USER root
ENTRYPOINT ["/usr/local/bin/jenkins-slave"]

```

> 其中 `jenkins-agent`、`agent.jar`、`openjdk`来自于`jenkins/inbound-agent:latest`镜像，当我们需要该文件时，只需要运行inbound-agent镜像，进入bash后，拷贝出来即可，当然自定义镜像我们也可以封装一些自己需要的，例如mvn、npm等，`apache-maven-3.8.1`为mvn解压后的包

## jenkins构建pipline任务

#### git仓库凭据创建

![image-20230523095857955](https://gitee.com/root_007/md_file_image/raw/master/202305230958007.png)

![image-20230523095913881](https://gitee.com/root_007/md_file_image/raw/master/202305230959930.png)

![image-20230523100016291](https://gitee.com/root_007/md_file_image/raw/master/202305231000356.png)

> 使用账号密码类型的全局凭据，其中唯一标识符

#### harbor镜像仓库凭据创建

![image-20230523100115383](https://gitee.com/root_007/md_file_image/raw/master/202305231001445.png)

#### pipline示例

```yaml
pipeline {
    parameters {
        choice(name: 'PLATFORM', choices: ["dev", "test", "prod"], description: '选择发布环境')
        listGitBranches(
          name: 'BRANCH',
          branchFilter: 'refs/heads/(.*)',
          credentialsId: 'gitee',
          defaultValue: 'master',
          description: '请选择需要构建的分支',
          type: 'PT_BRANCHORPT_TAG',
          tagFilter: '.*',
          sortMode: 'NONE',
          quickFilterEnabled: true,
          selectedValue: 'NONE',
          remoteURL: 'https://gitee.com/root_007/java-demo.git'
        )
    }
    agent {
        kubernetes {
            cloud 'kubernetes'
            slaveConnectTimeout 1200
            yaml """
apiVersion: v1
kind: Pod
metadata:
  name: jenkins-agent
  namespace: jenkins
spec:
  imagePullSecrets:
  - name: k8s-jenkins
  containers:
  - name: jnlp
    image: 192.168.10.12:9999/jenkins/jenkins-centos:v1.1
    imagePullPolicy: IfNotPresent
    tty: true
    volumeMounts:
    - name: jenkins-mvn-volume
      mountPath: /opt/apache-maven-3.8.1/conf/settings.xml
      subPath: settings.xml
    - name: docker-sock
      mountPath: /var/run/docker.sock
      readOnly: true
    - name: docker-binary
      mountPath: /usr/bin/docker
      readOnly: true
  volumes:
  - name: jenkins-mvn-volume
    configMap:
      name: jenkins-mvn
  - name: docker-sock
    hostPath:
      path: /var/run/docker.sock
  - name: docker-binary
    hostPath:
      path: /usr/bin/docker
  restartPolicy: Never
"""

        }
    }
    environment {
        MAVEN_HOME = sh(returnStdout: true,script: 'echo "/opt/apache-maven-3.8.1/conf/settings.xml"').trim() 
    }
    options {
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '3'))
        retry(2)
    }
    stages {
        stage('Git') {
            steps {
                sh 'echo "拉取gitee代码"'
                git branch: "$BRANCH",credentialsId: 'gitee', url: 'https://gitee.com/root_007/java-demo.git'
                sh 'echo "代码已经拉取，即将进入构建阶段"'
            }
        }
        stage('mvn') {
            steps {
                sh 'cd $PWD'
                sh "mvn clean install package '-Dmaven.test.skip=true'"
            }
        }
        stage('生成dockerfile文件') {
            steps {
                sh 'echo "生成启动文件"'
                sh """
cat > docker-entrypoint.sh << EOF
java -jar /data/demo-0.0.1-SNAPSHOT.jar
EOF
                """
                sh 'echo "生成dockerfile文件"'
                sh """
cat > Dockerfile << EOF
FROM java:openjdk-8u111-alpine
WORKDIR /data
COPY docker-entrypoint.sh /usr/local/bin
COPY target/demo-0.0.1-SNAPSHOT.jar /data
RUN chmod +x /usr/local/bin/docker-entrypoint.sh
ENTRYPOINT ["/usr/local/bin/docker-entrypoint.sh"]
EOF
                """
            }
            
        }
        stage('生成镜像') {
            steps {
              withCredentials([usernamePassword(credentialsId: 'harbor', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                sh "docker login 192.168.10.12:9999  -u $USERNAME -p $PASSWORD"
                sh 'docker build -t java-demo:v1.1 -f Dockerfile .'
                sh 'docker tag java-demo:v1.1  192.168.10.12:9999/jenkins/java-demo:v1.1'
                sh 'docker push 192.168.10.12:9999/jenkins/java-demo:v1.1'
                sh 'docker logout'
              }
              sh 'sleep 100s'
            }
        }
    }
}
```

> jenkins pipline 变量设置  https://www.jenkins.io/zh/doc/book/pipeline/jenkinsfile/
>
> 声明式流水线支持 [environment](https://www.jenkins.io/zh/doc/book/pipeline/jenkinsfile/#../syntax#environment) 指令，而脚本式流水线的使用者必须使用 `withEnv` 步骤。
>
> 打印当前环境中的所有变量： `sh 'printenv'`
>
> 使用 `returnStdout` 时，返回的字符串末尾会追加一个空格。可以使用 `.trim()` 将其移除
>
> 说明： **我们上边的pipline不够完整，只有拉取代码到推送镜像，只有后边的构建，可以参考下边的步骤**

#### 制作kubectl镜像

```bash
FROM alpine
WORKDIR /root
COPY kubectl /usr/local/bin/kubectl
RUN chmod +x /usr/local/bin/kubectl
ENTRYPOINT [ "/bin/sh", "-c", "while true; do sleep 60; done" ]
```

> 其中 kubectl工具，需要自己下载好，下载地址： https://dl.k8s.io/v1.20.5/kubernetes-server-linux-amd64.tar.gz，然后解压后，将`kubernetes/server/bin/kubectl`拷贝出来

#### 将kubectl镜像集成到pipline中

```groovy
pipeline {
    parameters {
        choice(name: 'PLATFORM', choices: ["dev", "test", "prod"], description: '选择发布环境')
        listGitBranches(
          name: 'BRANCH',
          branchFilter: 'refs/heads/(.*)',
          credentialsId: 'gitee',
          defaultValue: 'master',
          description: '请选择需要构建的分支',
          type: 'PT_BRANCHORPT_TAG',
          tagFilter: '.*',
          sortMode: 'NONE',
          quickFilterEnabled: true,
          selectedValue: 'NONE',
          remoteURL: 'https://gitee.com/root_007/java-demo.git'
        )
    }
    agent {
        kubernetes {
            cloud 'kubernetes'
            defaultContainer 'jnlp'
            slaveConnectTimeout 1200
            yaml """
apiVersion: v1
kind: Pod
metadata:
  name: jenkins-agent
  namespace: jenkins
spec:
  imagePullSecrets:
  - name: k8s-jenkins
  containers:
  - name: jnlp
    image: 192.168.10.12:9999/jenkins/jenkins-centos:v1.1
    imagePullPolicy: IfNotPresent
    tty: true
    volumeMounts:
    - name: jenkins-mvn-volume
      mountPath: /opt/apache-maven-3.8.1/conf/settings.xml
      subPath: settings.xml
    - name: docker-sock
      mountPath: /var/run/docker.sock
      readOnly: true
    - name: docker-binary
      mountPath: /usr/bin/docker
      readOnly: true
  - name: kubectl
    image: 192.168.10.12:9999/jenkins/jenkins-kubectl:v1.2
    imagePullPolicy: IfNotPresent
    tty: true
    volumeMounts:
    - name: jenkins-kubectl-volume
      mountPath: /root/config-dev
      subPath: config
  volumes:
  - name: jenkins-mvn-volume
    configMap:
      name: jenkins-mvn
  - name: jenkins-kubectl-volume
    configMap:
      name: jenkins-kubectl
  - name: docker-sock
    hostPath:
      path: /var/run/docker.sock
  - name: docker-binary
    hostPath:
      path: /usr/bin/docker
  restartPolicy: Never
"""

        }
    }
    environment {
        MAVEN_HOME = sh(returnStdout: true,script: 'echo "/opt/apache-maven-3.8.1/conf/settings.xml"').trim() 
    }
    options {
        timestamps()
        buildDiscarder(logRotator(numToKeepStr: '3'))
        retry(2)
    }
    stages {
        stage('Git') {
            steps {
                sh 'echo "拉取gitee代码"'
                git branch: "$BRANCH",credentialsId: 'gitee', url: 'https://gitee.com/root_007/java-demo.git'
                sh 'echo "代码已经拉取，即将进入构建阶段"'
            }
        }
        stage('mvn') {
            steps {
                sh 'cd $PWD'
                sh "mvn clean install package '-Dmaven.test.skip=true'"
            }
        }
        stage('生成dockerfile文件') {
            steps {
                sh 'echo "生成启动文件"'
                sh """
cat > docker-entrypoint.sh << EOF
java -jar /data/demo-0.0.1-SNAPSHOT.jar
EOF
                """
                sh 'echo "生成dockerfile文件"'
                sh """
cat > Dockerfile << EOF
FROM java:openjdk-8u111-alpine
WORKDIR /data
COPY docker-entrypoint.sh /usr/local/bin
COPY target/demo-0.0.1-SNAPSHOT.jar /data
RUN chmod +x /usr/local/bin/docker-entrypoint.sh
ENTRYPOINT ["/usr/local/bin/docker-entrypoint.sh"]
EOF
                """
            }
            
        }
        stage('生成镜像') {
            steps {
              withCredentials([usernamePassword(credentialsId: 'harbor', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                sh "docker login 192.168.10.12:9999  -u $USERNAME -p $PASSWORD"
                sh 'docker build -t java-demo:v1.1 -f Dockerfile .'
                sh 'docker tag java-demo:v1.1  192.168.10.12:9999/jenkins/java-demo:v1.1'
                sh 'docker push 192.168.10.12:9999/jenkins/java-demo:v1.1'
                sh 'docker logout'
              }
              sh 'sleep 100s'
            }
        }
        stage('build') {
            steps {
                container(name:'kubectl') {
                    sh 'kubectl get nodes --kubeconfig /root/config-dev'
                }
            }
        }
    }
}
```

#### 使用变量完善pipline

```yaml
def REMOTEURL = 'https://gitee.com/root_007/java-demo.git'
pipeline {
    environment {
      HARBORURL = '192.168.10.12:9999'
      JOB_PLATFORM = 'jenkins'  // job归属平台，也可以作为镜像仓库的名称空间
      JAR_ADDRESS = 'target/demo-0.0.1-SNAPSHOT.jar'
    }
    parameters {
        choice(name: 'PLATFORM', choices: ["dev", "test", "prod"], description: '选择发布环境')
        listGitBranches(
          name: 'BRANCH',
          branchFilter: 'refs/heads/(.*)',
          credentialsId: 'gitee',
          defaultValue: 'master',
          description: '请选择需要构建的分支',
          type: 'PT_BRANCHORPT_TAG',
          tagFilter: '.*',
          sortMode: 'NONE',
          quickFilterEnabled: true,
          selectedValue: 'NONE',
          remoteURL: REMOTEURL
        )
    }
    agent {
        kubernetes {
            cloud 'kubernetes'
            defaultContainer 'jnlp'
            slaveConnectTimeout 1200
            yaml """
apiVersion: v1
kind: Pod
metadata:
  name: jenkins-agent
  namespace: jenkins
spec:
  imagePullSecrets:
  - name: k8s-jenkins
  containers:
  - name: jnlp
    image: 192.168.10.12:9999/jenkins/jenkins-centos:v1.1
    imagePullPolicy: IfNotPresent
    tty: true
    volumeMounts:
    - name: jenkins-mvn-volume
      mountPath: /opt/apache-maven-3.8.1/conf/settings.xml
      subPath: settings.xml
    - name: docker-sock
      mountPath: /var/run/docker.sock
      readOnly: true
    - name: docker-binary
      mountPath: /usr/bin/docker
      readOnly: true
  - name: kubectl
    image: 192.168.10.12:9999/jenkins/jenkins-kubectl:v1.2
    imagePullPolicy: IfNotPresent
    tty: true
    volumeMounts:
    - name: jenkins-kubectl-volume
      mountPath: /root/config-dev
      subPath: config
  volumes:
  - name: jenkins-mvn-volume
    configMap:
      name: jenkins-mvn
  - name: jenkins-kubectl-volume
    configMap:
      name: jenkins-kubectl
  - name: docker-sock
    hostPath:
      path: /var/run/docker.sock
  - name: docker-binary
    hostPath:
      path: /usr/bin/docker
  restartPolicy: Never
"""

        }
    }
    options {
        timestamps() // 显示构建时间
        disableConcurrentBuilds() // 禁止单个项目并发构建
        timeout(time: 1, unit: 'HOURS') // 设置单个项目构建超时时间
        buildDiscarder(logRotator(numToKeepStr: '10')) // 设置项目构建历史保留10个
        retry(2) // 阶段失败后，重新执行该阶段次数
    }
    stages {
        stage('Git') {
            steps {
                script {
                    cleanWs()
                    checkout scmGit(branches: [[name: BRANCH]], extensions: [[$class: 'CloneOption', depth: 1,noTags: false, reference: '', shallow: true]], userRemoteConfigs: [[credentialsId: 'gitee', url: REMOTEURL]])
                    env.COMMIT_ID   = sh(script: 'git log -1 --pretty=format:%h',  returnStdout: true).trim() // 提交ID
                    env.COMMIT_USER = sh(script: 'git log -1 --pretty=format:%an', returnStdout: true).trim() // 提交者
                    env.COMMIT_TIME = sh(script: 'git log -1 --pretty=format:%ai', returnStdout: true).trim() // 提交时间
                    env.COMMIT_INFO = sh(script: 'git log -1 --pretty=format:%s',  returnStdout: true).trim() // 提交信息
                    env.IMAGE_TAG  = sh(script: "echo ${COMMIT_ID}_" + "`date '+%Y%m%d%H%M%S'`" + "_${BUILD_ID}", returnStdout: true).trim() //构建镜像版本号
                }
            }
        }
        stage('mvn') {
            steps {
                sh 'cd $PWD'
                sh "mvn clean install package '-Dmaven.test.skip=true'"
            }
        }
        stage('生成dockerfile文件') {
            steps {
                sh 'echo "生成启动文件"'
                sh """
cat > docker-entrypoint.sh << EOF
java -jar /data/demo-0.0.1-SNAPSHOT.jar
EOF
                """
                sh 'echo "生成dockerfile文件"'
                sh """
cat > Dockerfile << EOF
FROM java:openjdk-8u111-alpine
WORKDIR /data
COPY docker-entrypoint.sh /usr/local/bin
COPY ${env.JAR_ADDRESS} /data
RUN chmod +x /usr/local/bin/docker-entrypoint.sh
ENTRYPOINT ["/usr/local/bin/docker-entrypoint.sh"]
EOF
                """
            }
            
        }
        stage('生成镜像') {
            steps {
              withCredentials([usernamePassword(credentialsId: 'harbor', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                sh "docker login ${env.HARBORURL}  -u $USERNAME -p $PASSWORD"
                sh "docker build -t ${env.HARBORURL}/${env.JOB_PLATFORM}/${JOB_BASE_NAME}:$IMAGE_TAG -f Dockerfile ."
                sh "docker push ${env.HARBORURL}/${env.JOB_PLATFORM}/${JOB_BASE_NAME}:$IMAGE_TAG"
                sh 'docker logout'
              }
              sh 'sleep 100s'
            }
        }
        stage('build') {
            steps {
                container(name:'kubectl') {
                    sh 'kubectl get nodes --kubeconfig /root/config-dev'
                }
            }
        }
    }
}
```

> 在build阶段，没有写详细的步骤，可以自行完善该步骤

## 待完善问题

- 可移植性过低
- 项目java项目时，没有使用mvn缓存
- 项目构建成功与否没有响应的通知
- 使用自定义镜像过大

## 问题完善

#### 结合Database存储项目信息

- 配置Database

![image-20230524135628095](https://gitee.com/root_007/md_file_image/raw/master/202305241356228.png)

- 创建表

  ```sql
  CREATE TABLE `job_msg` (
    `job_name` varchar(255) NOT NULL COMMENT '项目名称',
    `job_platform` varchar(255) NOT NULL COMMENT '项目所属平台',
    `job_types` varchar(255) NOT NULL COMMENT '项目类型',
    `job_address` varchar(255) NOT NULL COMMENT '项目包地址',
    `job_remotrurl` varchar(255) NOT NULL COMMENT '项目git地址',
    `job_owner` varchar(255) DEFAULT NULL COMMENT '项目负责人',
    `job_jar_name` varchar(255) NOT NULL COMMENT 'jar包名称',
    PRIMARY KEY (`job_name`)
  ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
  ```

- pipline中使用该数据库

  ```yaml
  // 拉取job在数据库中的信息
  def get_mysql_msg (jobName) {
  	println "===========获取${jobName}信息============"
  	getDatabaseConnection(type: 'GLOBAL') {
  		def result = sql("SELECT *  FROM job_msg WHERE job_name=\"${jobName}\"")
  		return result
  	}
  }
  // 格式化数据信息，返回的是map信息,这个000000，是自定义了一个无用字符串，用于切割使用
  def get_job_msg (res) {
  	message = ""
  	for (i in res) {
  		if (i != "[" && i != "]") {
  			message += i
  		}
  	}
  	job_map = message.split(', ').collectEntries{it.replaceFirst(/:/, '000000').minus("]").minus("[").split('000000') as List}	
  	return job_map
  }
  
  res = get_mysql_msg("${JOB_NAME}")
  if ( res == []) {
  	throw new Exception(" ${JOB_NAME}没有找到对应的项目信息 ")
  }
  job_msg = get_job_msg(res)
  // 定义所需变量
  def job_platform = "${job_msg.job_platform}"
  def job_types = "${job_msg.job_types}"
  def job_address = "${job_msg.job_address}"
  def job_remotrurl = "${job_msg.job_remotrurl}"
  def job_owner = "${job_msg.job_owner}"
  def job_jar_name = "${job_msg.job_jar_name}"
  def REMOTEURL = "${job_remotrurl}"
  println "=================="
  println REMOTEURL
  println "=================="
  pipeline {
      environment {
        HARBORURL = '192.168.10.12:9999'
        JOB_PLATFORM = 'jenkins'  // job归属平台，也可以作为镜像仓库的名称空间
        JAR_ADDRESS = 'target/demo-0.0.1-SNAPSHOT.jar'
      }
      parameters {
          choice(name: 'PLATFORM', choices: ["dev", "test", "prod"], description: '选择发布环境')
          listGitBranches(
            name: 'BRANCH',
            branchFilter: 'refs/heads/(.*)',
            credentialsId: 'gitee',
            defaultValue: 'master',
            description: '请选择需要构建的分支',
            type: 'PT_BRANCHORPT_TAG',
            tagFilter: '.*',
            sortMode: 'NONE',
            quickFilterEnabled: true,
            selectedValue: 'NONE',
            remoteURL: REMOTEURL
          )
      }
      agent {
          kubernetes {
              cloud 'kubernetes'
              defaultContainer 'jnlp'
              slaveConnectTimeout 1200
              yaml """
  apiVersion: v1
  kind: Pod
  metadata:
    name: jenkins-agent
    namespace: jenkins
  spec:
    imagePullSecrets:
    - name: k8s-jenkins
    containers:
    - name: jnlp
      image: 192.168.10.12:9999/jenkins/jenkins-centos:v1.1
      imagePullPolicy: IfNotPresent
      tty: true
      volumeMounts:
      - name: jenkins-mvn-volume
        mountPath: /opt/apache-maven-3.8.1/conf/settings.xml
        subPath: settings.xml
      - name: docker-sock
        mountPath: /var/run/docker.sock
        readOnly: true
      - name: docker-binary
        mountPath: /usr/bin/docker
        readOnly: true
    - name: kubectl
      image: 192.168.10.12:9999/jenkins/jenkins-kubectl:v1.2
      imagePullPolicy: IfNotPresent
      tty: true
      volumeMounts:
      - name: jenkins-kubectl-volume
        mountPath: /root/config-dev
        subPath: config
    volumes:
    - name: jenkins-mvn-volume
      configMap:
        name: jenkins-mvn
    - name: jenkins-kubectl-volume
      configMap:
        name: jenkins-kubectl
    - name: docker-sock
      hostPath:
        path: /var/run/docker.sock
    - name: docker-binary
      hostPath:
        path: /usr/bin/docker
    restartPolicy: Never
  """
  
          }
      }
      options {
          timestamps() // 显示构建时间
          disableConcurrentBuilds() // 禁止单个项目并发构建
          timeout(time: 1, unit: 'HOURS') // 设置单个项目构建超时时间
          buildDiscarder(logRotator(numToKeepStr: '10')) // 设置项目构建历史保留10个
          retry(2) // 阶段失败后，重新执行该阶段次数
      }
      stages {
          stage('Git') {
              steps {
                  script {
                      cleanWs()
                      checkout scmGit(branches: [[name: BRANCH]], extensions: [[$class: 'CloneOption', depth: 1,noTags: false, reference: '', shallow: true]], userRemoteConfigs: [[credentialsId: 'gitee', url: REMOTEURL]])
                      env.COMMIT_ID   = sh(script: 'git log -1 --pretty=format:%h',  returnStdout: true).trim() // 提交ID
                      env.COMMIT_USER = sh(script: 'git log -1 --pretty=format:%an', returnStdout: true).trim() // 提交者
                      env.COMMIT_TIME = sh(script: 'git log -1 --pretty=format:%ai', returnStdout: true).trim() // 提交时间
                      env.COMMIT_INFO = sh(script: 'git log -1 --pretty=format:%s',  returnStdout: true).trim() // 提交信息
                      env.IMAGE_TAG  = sh(script: "echo ${COMMIT_ID}_" + "`date '+%Y%m%d%H%M%S'`" + "_${BUILD_ID}", returnStdout: true).trim() //构建镜像版本号
                  }
              }
          }
          stage('mvn') {
              steps {
                  sh 'echo =========='
                  sh "echo ${job_platform}"
                  sh 'cd $PWD'
                  sh "mvn clean install package '-Dmaven.test.skip=true'"
              }
          }
          stage('生成dockerfile文件') {
              steps {
                  sh 'echo "生成启动文件"'
                  sh """
  cat > docker-entrypoint.sh << EOF
  java -jar /data/demo-0.0.1-SNAPSHOT.jar
  EOF
                  """
                  sh 'echo "生成dockerfile文件"'
                  sh """
  cat > Dockerfile << EOF
  FROM java:openjdk-8u111-alpine
  WORKDIR /data
  COPY docker-entrypoint.sh /usr/local/bin
  COPY ${job_address} /data
  RUN chmod +x /usr/local/bin/docker-entrypoint.sh
  ENTRYPOINT ["/usr/local/bin/docker-entrypoint.sh"]
  EOF
                  """
              }
              
          }
          stage('生成镜像') {
              steps {
                withCredentials([usernamePassword(credentialsId: 'harbor', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                  sh "docker login ${env.HARBORURL}  -u $USERNAME -p $PASSWORD"
                  sh "docker build -t ${env.HARBORURL}/${job_platform}/${JOB_BASE_NAME}:$IMAGE_TAG -f Dockerfile ."
                  sh "docker push ${env.HARBORURL}/${job_platform}/${JOB_BASE_NAME}:$IMAGE_TAG"
                  sh 'docker logout'
                }
                sh 'sleep 100s'
              }
          }
          stage('build') {
              steps {
                  container(name:'kubectl') {
                      sh 'kubectl get nodes --kubeconfig /root/config-dev'
                  }
              }
          }
      }
  }
  ```

  #### 镜像拆分

  > 将镜像也使用变量代替，这样根据job类型使用不同的构建镜像，以达到通用性

- 单独使用mvn镜像

  mvn镜像自定义

  ```bash
  FROM alpine
  COPY apache-maven-3.8.1 /opt/apache-maven-3.8.1
  RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories
  RUN apk add --no-cache openjdk8
  RUN ln -s /opt/apache-maven-3.8.1/bin/mvn /usr/bin/mvn
  ENTRYPOINT [ "/bin/sh", "-c", "while true; do sleep 60; done" ]
  ```

  > 其中`apache-maven-3.8.1`为mvn二进制包解压后的文件，可自行下载
  >
  > 下载地址： https://repo.huaweicloud.com/apache/maven/maven-3/3.8.1/

- 单独使用jenkins-slave镜像

  ```bash
  FROM jenkins/inbound-agent:latest
  USER root
  ```

  > 因为我们需要挂载docker，所以在这里直接使用root用户，防止权限不足的问题

- 单独使用npm镜像

  npm镜像自定义

  ```bash
  FROM alpine
  RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories && \
      apk update && \
      apk add --no-cache nodejs=16.20.0-r0 && \
      apk add --no-cache npm=8.1.3-r0 && \
      npm install -g cnpm --registry=https://registry.npm.taobao.org && \ 
      npm install -g yarn --registry=https://registry.npm.taobao.org
  ENTRYPOINT [ "/bin/sh", "-c", "while true; do sleep 60; done" ]
  ```

  > 查看alpine支持的软件包 https://pkgs.alpinelinux.org/packages
  >
  > 手动指定npm缓存位置： `npm install --cache=/path/to/directory/.npm-cache`
  >
  > 其中`npm`版本为**8.1.3**，`cnpm`版本为**9.2.0**，`yarn`版本为**1.22.19**

- 单独使用kubeclt镜像

  > 该镜像继续使用前边自定义的kubeclt镜像，其地址为：`192.168.10.12:9999/jenkins/jenkins-kubectl:v1.2`

- 结合pipline使用拆分后的镜像

  ```yaml
  // 拉取job在数据库中的信息
  def get_mysql_msg (jobName) {
  	println "===========获取${jobName}信息============"
  	getDatabaseConnection(type: 'GLOBAL') {
  		def result = sql("SELECT *  FROM job_msg WHERE job_name=\"${jobName}\"")
  		return result
  	}
  }
  // 格式化数据信息，返回的是map信息,这个000000，是自定义了一个无用字符串，用于切割使用
  def get_job_msg (res) {
  	message = ""
  	for (i in res) {
  		if (i != "[" && i != "]") {
  			message += i
  		}
  	}
  	job_map = message.split(', ').collectEntries{it.replaceFirst(/:/, '000000').minus("]").minus("[").split('000000') as List}	
  	return job_map
  }
  
  res = get_mysql_msg("${JOB_NAME}")
  if ( res == []) {
  	throw new Exception(" ${JOB_NAME}没有找到对应的项目信息 ")
  }
  job_msg = get_job_msg(res)
  // 定义所需变量
  def job_platform = "${job_msg.job_platform}"
  def job_types = "${job_msg.job_types}"
  def job_address = "${job_msg.job_address}"
  def job_remotrurl = "${job_msg.job_remotrurl}"
  def job_owner = "${job_msg.job_owner}"
  def job_jar_name = "${job_msg.job_jar_name}"
  def REMOTEURL = "${job_remotrurl}"
  // 重新赋值一下变量，方便sh中引用
  env.job_types = job_types
  env.JENKINS_JNLP_IMAGE = '192.168.10.12:9999/jenkins/jenkins-agent:v1.1'
  env.JENKINS_MVN_IMAGE = '192.168.10.12:9999/jenkins/jenkins-mvn:v1.1'
  env.JENKINS_KUBECTL_IMAGE = '192.168.10.12:9999/jenkins/jenkins-kubectl:v1.2'
  env.JENKINS_NPM_IMAGE = '192.168.10.12:9999/jenkins/jenkins-npm:v1.1'
  env.JENKINS_JAR_VOLUMEMOUNTS_NAME = 'jenkins-mvn-volume'
  env.JENKINS_JAR_MOUNT_PATH = '/opt/apache-maven-3.8.1/conf/settings.xml'
  env.JENKINS_JAR_SUB_PATH = 'settings.xml'
  env.JENKINS_JAR_K8S_CM_NAME = 'jenkins-mvn'
  
  // 定义容器
  
  def myContainer() {
      if (env.job_types == 'jar') {
          return """
    - name: mvn
      image: ${JENKINS_MVN_IMAGE}
      imagePullPolicy: IfNotPresent
      tty: true
      volumeMounts:
      - name: ${JENKINS_JAR_VOLUMEMOUNTS_NAME}
        mountPath: ${JENKINS_JAR_MOUNT_PATH}
        subPath: ${JENKINS_JAR_SUB_PATH}
  """
      } else if (env.job_types == 'vue') {
          return """
    - name: npm
      image: ${JENKINS_NPM_IMAGE}
      imagePullPolicy: IfNotPresent
      tty: true
  """
      } else {
          throw new Exception("暂无该类型项目构建模板")
      }
  }
  
  // 定义容器挂载卷
  
  def myContainerVolumes() {
      if (env.job_types == 'jar') {
          return """
    - name: ${JENKINS_JAR_VOLUMEMOUNTS_NAME}
      configMap:
        name: ${JENKINS_JAR_K8S_CM_NAME}
  """
      } else if (env.job_types == 'vue') {
          return """
  """
      } else {
          throw new Exception("暂无该类型项目构建模板")
      }
  }
  
  
  
  pipeline {
      environment {
        HARBORURL = '192.168.10.12:9999'
      }
      parameters {
          choice(name: 'PLATFORM', choices: ["dev", "test", "prod"], description: '选择发布环境')
          listGitBranches(
            name: 'BRANCH',
            branchFilter: 'refs/heads/(.*)',
            credentialsId: 'gitee',
            defaultValue: 'master',
            description: '请选择需要构建的分支',
            type: 'PT_BRANCHORPT_TAG',
            tagFilter: '.*',
            sortMode: 'NONE',
            quickFilterEnabled: true,
            selectedValue: 'NONE',
            remoteURL: REMOTEURL
          )
      }
      agent {
          kubernetes {
              cloud 'kubernetes'
              defaultContainer 'jnlp'
              slaveConnectTimeout 1200
              yaml """
  apiVersion: v1
  kind: Pod
  metadata:
    name: jenkins-agent
    namespace: jenkins
  spec:
    imagePullSecrets:
    - name: k8s-jenkins
    containers:
    - name: jnlp
      image: ${JENKINS_JNLP_IMAGE}
      imagePullPolicy: IfNotPresent
      tty: true
      volumeMounts:
      - name: docker-sock
        mountPath: /var/run/docker.sock
        readOnly: true
      - name: docker-binary
        mountPath: /usr/bin/docker
        readOnly: true
  ${myContainer()}
    - name: kubectl
      image: ${JENKINS_KUBECTL_IMAGE}
      imagePullPolicy: IfNotPresent
      tty: true
      volumeMounts:
      - name: jenkins-kubectl-volume
        mountPath: /root/config-dev
        subPath: config
    volumes:
  ${myContainerVolumes()}
    - name: jenkins-kubectl-volume
      configMap:
        name: jenkins-kubectl
    - name: docker-sock
      hostPath:
        path: /var/run/docker.sock
    - name: docker-binary
      hostPath:
        path: /usr/bin/docker
    restartPolicy: Never
  """
  
          }
      }
      options {
          timestamps() // 显示构建时间
          disableConcurrentBuilds() // 禁止单个项目并发构建
          timeout(time: 1, unit: 'HOURS') // 设置单个项目构建超时时间
          buildDiscarder(logRotator(numToKeepStr: '10')) // 设置项目构建历史保留10个
          retry(2) // 阶段失败后，重新执行该阶段次数
      }
      stages {
          stage('Git') {
              steps {
                  script {
                      cleanWs()
                      checkout scmGit(branches: [[name: BRANCH]], extensions: [[$class: 'CloneOption', depth: 1,noTags: false, reference: '', shallow: true]], userRemoteConfigs: [[credentialsId: 'gitee', url: REMOTEURL]])
                      env.COMMIT_ID   = sh(script: 'git log -1 --pretty=format:%h',  returnStdout: true).trim() // 提交ID
                      env.COMMIT_USER = sh(script: 'git log -1 --pretty=format:%an', returnStdout: true).trim() // 提交者
                      env.COMMIT_TIME = sh(script: 'git log -1 --pretty=format:%ai', returnStdout: true).trim() // 提交时间
                      env.COMMIT_INFO = sh(script: 'git log -1 --pretty=format:%s',  returnStdout: true).trim() // 提交信息
                      env.IMAGE_TAG  = sh(script: "echo ${COMMIT_ID}_" + "`date '+%Y%m%d%H%M%S'`" + "_${BUILD_ID}", returnStdout: true).trim() //构建镜像版本号
                  }
              }
          }
          stage('mvn构建') {
              when {
                  expression {env.job_types == 'jar'}
              }
              steps {
                  container(name:'mvn') {
                      sh 'echo =========='
                      sh "echo ${job_platform}"
                      sh 'cd $PWD'
                      sh "mvn clean install package '-Dmaven.test.skip=true'"
                  }
              }
          }
          stage('npm构建') {
              when {
                  expression {env.job_types == 'vue'}
              }
              steps {
                  container(name:'npm') {
                      sh 'echo =========='
                      sh "echo ${job_platform}"
                      sh 'cd $PWD'
                      sh "npm install'"
                  }
              }
              
          }
          stage('生成镜像文件') {
              steps {
                  script {
                      if (env.job_types == 'jar') {
                          sh 'echo 生成jar启动文件'
                          sh """
  cat > docker-entrypoint.sh << EOF
  java -jar /data/${job_jar_name}
  EOF
                              """ 
                          sh 'echo "jar项目生成dockerfile文件"'
                          sh """
  cat > Dockerfile << EOF
  FROM java:openjdk-8u111-alpine
  WORKDIR /data
  COPY docker-entrypoint.sh /usr/local/bin
  COPY ${job_address} /data
  RUN chmod +x /usr/local/bin/docker-entrypoint.sh
  ENTRYPOINT ["/usr/local/bin/docker-entrypoint.sh"]
  EOF
                              """
                      } else if (env.job_types == 'vue') {
                          sh 'echo 生成vue启动文件'
                          sh """
  cat > docker-entrypoint.sh << EOF
  java -jar /data/${job_jar_name}
  EOF
                              """ 
                          sh 'echo "vue项目生成dockerfile文件"'
                          sh """
  cat > Dockerfile << EOF
  FROM java:openjdk-8u111-alpine
  WORKDIR /data
  COPY docker-entrypoint.sh /usr/local/bin
  COPY ${job_address} /data
  RUN chmod +x /usr/local/bin/docker-entrypoint.sh
  ENTRYPOINT ["/usr/local/bin/docker-entrypoint.sh"]
  EOF
                              """
                          
                      }
                  }
                  
              }
          }
          stage('生成镜像') {
              steps {
                withCredentials([usernamePassword(credentialsId: 'harbor', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                  sh "docker login ${env.HARBORURL}  -u $USERNAME -p $PASSWORD"
                  sh "docker build -t ${env.HARBORURL}/${job_platform}/${JOB_BASE_NAME}:$IMAGE_TAG -f Dockerfile ."
                  sh "docker push ${env.HARBORURL}/${job_platform}/${JOB_BASE_NAME}:$IMAGE_TAG"
                  sh 'docker logout'
                }
                sh 'sleep 100s'
              }
          }
          stage('build') {
              steps {
                  container(name:'kubectl') {
                      sh 'kubectl get nodes --kubeconfig /root/config-dev'
                  }
              }
          }
      }
  }
  ```

  > 其中添加了一些判断，如果类型为jar类型，那么使用mvn镜像进行构建

- 创建一个pvc供mvn镜像使用

  jenkins-mvn-pvc

  ```yaml
  kind: PersistentVolumeClaim
  apiVersion: v1
  metadata:
    name: jenkins-mvn-pvc
    namespace: jenkins
  spec:
    accessModes:
      - ReadWriteMany
    storageClassName: nfs-jenkins
    resources:
      requests:
        storage: 20Gi
  ```

  结合pipline使用该pvc

  ```yaml
  // 拉取job在数据库中的信息
  def get_mysql_msg (jobName) {
  	println "===========获取${jobName}信息============"
  	getDatabaseConnection(type: 'GLOBAL') {
  		def result = sql("SELECT *  FROM job_msg WHERE job_name=\"${jobName}\"")
  		return result
  	}
  }
  // 格式化数据信息，返回的是map信息,这个000000，是自定义了一个无用字符串，用于切割使用
  def get_job_msg (res) {
  	message = ""
  	for (i in res) {
  		if (i != "[" && i != "]") {
  			message += i
  		}
  	}
  	job_map = message.split(', ').collectEntries{it.replaceFirst(/:/, '000000').minus("]").minus("[").split('000000') as List}	
  	return job_map
  }
  
  res = get_mysql_msg("${JOB_NAME}")
  if ( res == []) {
  	throw new Exception(" ${JOB_NAME}没有找到对应的项目信息 ")
  }
  job_msg = get_job_msg(res)
  // 定义所需变量
  def job_platform = "${job_msg.job_platform}"
  def job_types = "${job_msg.job_types}"
  def job_address = "${job_msg.job_address}"
  def job_remotrurl = "${job_msg.job_remotrurl}"
  def job_owner = "${job_msg.job_owner}"
  def job_jar_name = "${job_msg.job_jar_name}"
  def REMOTEURL = "${job_remotrurl}"
  // 重新赋值一下变量，方便sh中引用
  env.job_types = job_types
  env.JENKINS_JNLP_IMAGE = '192.168.10.12:9999/jenkins/jenkins-agent:v1.1'
  env.JENKINS_MVN_IMAGE = '192.168.10.12:9999/jenkins/jenkins-mvn:v1.1'
  env.JENKINS_KUBECTL_IMAGE = '192.168.10.12:9999/jenkins/jenkins-kubectl:v1.2'
  env.JENKINS_NPM_IMAGE = '192.168.10.12:9999/jenkins/jenkins-npm:v1.1'
  env.JENKINS_JAR_VOLUMEMOUNTS_NAME = 'jenkins-mvn-volume'
  env.JENKINS_JAR_MOUNT_PATH = '/opt/apache-maven-3.8.1/conf/settings.xml'
  env.JENKINS_JAR_SUB_PATH = 'settings.xml'
  env.JENKINS_JAR_K8S_CM_NAME = 'jenkins-mvn'
  
  // 定义容器
  
  def myContainer() {
      if (env.job_types == 'jar') {
          return """
    - name: mvn
      image: ${JENKINS_MVN_IMAGE}
      imagePullPolicy: IfNotPresent
      tty: true
      volumeMounts:
      - name: ${JENKINS_JAR_VOLUMEMOUNTS_NAME}
        mountPath: ${JENKINS_JAR_MOUNT_PATH}
        subPath: ${JENKINS_JAR_SUB_PATH}
      - name: jenkins-m2-volume
        mountPath: /root/.m2 
  """
      } else if (env.job_types == 'vue') {
          return """
    - name: npm
      image: ${JENKINS_NPM_IMAGE}
      imagePullPolicy: IfNotPresent
      tty: true
  """
      } else {
          throw new Exception("暂无该类型项目构建模板")
      }
  }
  
  // 定义容器挂载卷
  
  def myContainerVolumes() {
      if (env.job_types == 'jar') {
          return """
    - name: ${JENKINS_JAR_VOLUMEMOUNTS_NAME}
      configMap:
        name: ${JENKINS_JAR_K8S_CM_NAME}
    - name: jenkins-m2-volume
      persistentVolumeClaim:
        claimName: jenkins-mvn-pvc
      
  """
      } else if (env.job_types == 'vue') {
          return """
  """
      } else {
          throw new Exception("暂无该类型项目构建模板")
      }
  }
  
  
  
  pipeline {
      environment {
        HARBORURL = '192.168.10.12:9999'
      }
      parameters {
          choice(name: 'PLATFORM', choices: ["dev", "test", "prod"], description: '选择发布环境')
          listGitBranches(
            name: 'BRANCH',
            branchFilter: 'refs/heads/(.*)',
            credentialsId: 'gitee',
            defaultValue: 'master',
            description: '请选择需要构建的分支',
            type: 'PT_BRANCHORPT_TAG',
            tagFilter: '.*',
            sortMode: 'NONE',
            quickFilterEnabled: true,
            selectedValue: 'NONE',
            remoteURL: REMOTEURL
          )
      }
      agent {
          kubernetes {
              cloud 'kubernetes'
              defaultContainer 'jnlp'
              slaveConnectTimeout 1200
              yaml """
  apiVersion: v1
  kind: Pod
  metadata:
    name: jenkins-agent
    namespace: jenkins
  spec:
    imagePullSecrets:
    - name: k8s-jenkins
    containers:
    - name: jnlp
      image: ${JENKINS_JNLP_IMAGE}
      imagePullPolicy: IfNotPresent
      tty: true
      volumeMounts:
      - name: docker-sock
        mountPath: /var/run/docker.sock
        readOnly: true
      - name: docker-binary
        mountPath: /usr/bin/docker
        readOnly: true
  ${myContainer()}
    - name: kubectl
      image: ${JENKINS_KUBECTL_IMAGE}
      imagePullPolicy: IfNotPresent
      tty: true
      volumeMounts:
      - name: jenkins-kubectl-volume
        mountPath: /root/config-dev
        subPath: config
    volumes:
  ${myContainerVolumes()}
    - name: jenkins-kubectl-volume
      configMap:
        name: jenkins-kubectl
    - name: docker-sock
      hostPath:
        path: /var/run/docker.sock
    - name: docker-binary
      hostPath:
        path: /usr/bin/docker
    restartPolicy: Never
  """
  
          }
      }
      options {
          timestamps() // 显示构建时间
          disableConcurrentBuilds() // 禁止单个项目并发构建
          timeout(time: 1, unit: 'HOURS') // 设置单个项目构建超时时间
          buildDiscarder(logRotator(numToKeepStr: '10')) // 设置项目构建历史保留10个
          retry(2) // 阶段失败后，重新执行该阶段次数
      }
      stages {
          stage('Git') {
              steps {
                  script {
                      cleanWs()
                      checkout scmGit(branches: [[name: BRANCH]], extensions: [[$class: 'CloneOption', depth: 1,noTags: false, reference: '', shallow: true]], userRemoteConfigs: [[credentialsId: 'gitee', url: REMOTEURL]])
                      env.COMMIT_ID   = sh(script: 'git log -1 --pretty=format:%h',  returnStdout: true).trim() // 提交ID
                      env.COMMIT_USER = sh(script: 'git log -1 --pretty=format:%an', returnStdout: true).trim() // 提交者
                      env.COMMIT_TIME = sh(script: 'git log -1 --pretty=format:%ai', returnStdout: true).trim() // 提交时间
                      env.COMMIT_INFO = sh(script: 'git log -1 --pretty=format:%s',  returnStdout: true).trim() // 提交信息
                      env.IMAGE_TAG  = sh(script: "echo ${COMMIT_ID}_" + "`date '+%Y%m%d%H%M%S'`" + "_${BUILD_ID}", returnStdout: true).trim() //构建镜像版本号
                  }
              }
          }
          stage('mvn构建') {
              when {
                  expression {env.job_types == 'jar'}
              }
              steps {
                  container(name:'mvn') {
                      sh 'echo =========='
                      sh "echo ${job_platform}"
                      sh 'cd $PWD'
                      sh "mvn clean install package '-Dmaven.test.skip=true'"
                  }
              }
          }
          stage('npm构建') {
              when {
                  expression {env.job_types == 'vue'}
              }
              steps {
                  container(name:'npm') {
                      sh 'echo =========='
                      sh "echo ${job_platform}"
                      sh 'cd $PWD'
                      sh "npm install'"
                  }
              }
              
          }
          stage('生成镜像文件') {
              steps {
                  script {
                      if (env.job_types == 'jar') {
                          sh 'echo 生成jar启动文件'
                          sh """
  cat > docker-entrypoint.sh << EOF
  java -jar /data/${job_jar_name}
  EOF
                              """ 
                          sh 'echo "jar项目生成dockerfile文件"'
                          sh """
  cat > Dockerfile << EOF
  FROM java:openjdk-8u111-alpine
  WORKDIR /data
  COPY docker-entrypoint.sh /usr/local/bin
  COPY ${job_address} /data
  RUN chmod +x /usr/local/bin/docker-entrypoint.sh
  ENTRYPOINT ["/usr/local/bin/docker-entrypoint.sh"]
  EOF
                              """
                      } else if (env.job_types == 'vue') {
                          sh 'echo 生成vue启动文件'
                          sh """
  cat > docker-entrypoint.sh << EOF
  java -jar /data/${job_jar_name}
  EOF
                              """ 
                          sh 'echo "vue项目生成dockerfile文件"'
                          sh """
  cat > Dockerfile << EOF
  FROM java:openjdk-8u111-alpine
  WORKDIR /data
  COPY docker-entrypoint.sh /usr/local/bin
  COPY ${job_address} /data
  RUN chmod +x /usr/local/bin/docker-entrypoint.sh
  ENTRYPOINT ["/usr/local/bin/docker-entrypoint.sh"]
  EOF
                              """
                          
                      }
                  }
                  
              }
          }
          stage('生成镜像') {
              steps {
                withCredentials([usernamePassword(credentialsId: 'harbor', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                  sh "docker login ${env.HARBORURL}  -u $USERNAME -p $PASSWORD"
                  sh "docker build -t ${env.HARBORURL}/${job_platform}/${JOB_BASE_NAME}:$IMAGE_TAG -f Dockerfile ."
                  sh "docker push ${env.HARBORURL}/${job_platform}/${JOB_BASE_NAME}:$IMAGE_TAG"
                  sh 'docker logout'
                }
              }
          }
          stage('build') {
              steps {
                  container(name:'kubectl') {
                      sh 'kubectl get nodes --kubeconfig /root/config-dev'
                  }
              }
          }
      }
  }
  ```

## jenkins共享库的使用

> 暂未补充，等有时间再补充上

## 总结

- 以上pipline只是一个简单的demo，其中还有一些地方需要自行去完善，例如：vue项目的dockerfile，执行steps失败的后发送通知等等
- 在此pipline基础上，只要是我们通过jenkins api创建项目，然后在将所需的数据插入到数据库中，就可以实现更加灵活的方式

