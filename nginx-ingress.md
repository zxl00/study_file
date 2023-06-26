#### nginx-ingress版本对应关系

```
https://github.com/kubernetes/ingress-nginx
```

| Ingress-NGINX version | k8s supported version        | Alpine Version | Nginx Version |
| --------------------- | ---------------------------- | -------------- | :-----------: |
| v1.5.2                | 1.26, 1.25, 1.24, 1.23       | 3.17.2         |    1.21.6     |
| v1.5.1                | 1.25, 1.24, 1.23             | 3.16.2         |    1.21.6     |
| v1.4.0                | 1.25, 1.24, 1.23, 1.22       | 3.16.2         |   1.19.10†    |
| v1.3.1                | 1.24, 1.23, 1.22, 1.21, 1.20 | 3.16.2         |   1.19.10†    |
| v1.3.0                | 1.24, 1.23, 1.22, 1.21, 1.20 | 3.16.0         |   1.19.10†    |
| v1.2.1                | 1.23, 1.22, 1.21, 1.20, 1.19 | 3.14.6         |   1.19.10†    |
| v1.1.3                | 1.23, 1.22, 1.21, 1.20, 1.19 | 3.14.4         |   1.19.10†    |
| v1.1.2                | 1.23, 1.22, 1.21, 1.20, 1.19 | 3.14.2         |    1.19.9†    |
| v1.1.1                | 1.23, 1.22, 1.21, 1.20, 1.19 | 3.14.2         |    1.19.9†    |
| v1.1.0                | 1.22, 1.21, 1.20, 1.19       | 3.14.2         |    1.19.9†    |
| v1.0.5                | 1.22, 1.21, 1.20, 1.19       | 3.14.2         |    1.19.9†    |
| v1.0.4                | 1.22, 1.21, 1.20, 1.19       | 3.14.2         |    1.19.9†    |
| v1.0.3                | 1.22, 1.21, 1.20, 1.19       | 3.14.2         |    1.19.9†    |
| v1.0.2                | 1.22, 1.21, 1.20, 1.19       | 3.14.2         |    1.19.9†    |
| v1.0.1                | 1.22, 1.21, 1.20, 1.19       | 3.14.2         |    1.19.9†    |
| v1.0.0                | 1.22, 1.21, 1.20, 1.19       | 3.13.5         |    1.20.1     |

#### nginx-ingress安装

- 根据对应版本下载yaml文件

  ![image-20230103101532914](E:\md文档\img\image-20230103101532914.png)

- 第二种方式下载yaml文件

  ```
  https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.5.1/deploy/static/provider/cloud/deploy.yaml
  ```

  > 根据自己对应的版本下载所需的yaml文件

  #### 安装

  ```
  kubectl  apply  -f  nginx-ingress.yaml
  ```

  #### 验证

  ```
  [root@k8s-master ~]# kubectl get all -n ingress-nginx  
  NAME                                            READY   STATUS      RESTARTS   AGE
  pod/ingress-nginx-admission-create-b66mg        0/1     Completed   0          6m58s
  pod/ingress-nginx-admission-patch-v4zmt         0/1     Completed   0          6m58s
  pod/ingress-nginx-controller-689fdc88ff-kwf8t   1/1     Running     0          6m58s
  
  NAME                                         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
  service/ingress-nginx-controller             NodePort    100.106.84.14   <none>        80:32286/TCP,443:31029/TCP   6m58s
  service/ingress-nginx-controller-admission   ClusterIP   100.99.76.91    <none>        443/TCP                      6m58s
  
  NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
  deployment.apps/ingress-nginx-controller   1/1     1            1           6m58s
  
  NAME                                                  DESIRED   CURRENT   READY   AGE
  replicaset.apps/ingress-nginx-controller-689fdc88ff   1         1         1       6m58s
  
  NAME                                       COMPLETIONS   DURATION   AGE
  job.batch/ingress-nginx-admission-create   1/1           2s         6m58s
  job.batch/ingress-nginx-admission-patch    1/1           3m54s      6m58s
  
  ```

#### 所需镜像下载问题

> 可以访问docker hub下载所需镜像
>
> ![image-20230103145440057](E:\md文档\img\image-20230103145440057.png)

> https://hub.docker.com/repository/docker/jaic01/nginx-ingress/tags?page=1&ordering=last_updated



#### nginx-ingress常用注解

```
nginx.ingress.kubernetes.io/ssl-redirect
```

> 取值： true、false
>
> 当为true时：启动TLS，http请求会重定向到https

- TLS配置

  ```
  kubectl create secret tls www-example-com --key tls.key --cert tls.crt -n default
  ```

  > www-exapmle-com：secret名称
  >
  > tls.key：证书key
  >
  > tls.crt：证书crt

- ingress创建

  ```
  apiVersion: extensions/v1beta1
  kind: Ingress
  metadata:
    name: math-manager
    namespace: default
  spec:
    rules:
    - host: math-manager.xx.com
      http:
        paths:
        - backend:
            serviceName: math-manager
            servicePort: 8090
          path: /
          pathType: ImplementationSpecific
    tls:
    - hosts:
      - math-manager.xx.com
      secretName: www-example-com
  
  ```

```
nginx.ingress.kubernetes.io/whitelist-source-range
```

> 取值：IP范围，多个使用`,`隔开

- 配置IP白名单范围

  ```
  apiVersion: extensions/v1beta1
  kind: Ingress
  metadata:
    name: demo-ingress
    namespace: default
    annotations:
      kubernetes.io/ingress.class: "nginx" # 绑定ingress-class
      nginx.ingress.kubernetes.io/whitelist-source-range: 10.0.0.0/24,172.10.0.1
  spec:
    rules:
    - host: math-manager.xx.com
      http:
        paths:
        - path: /
          backend:
            serviceName: math-manager
            servicePort: 8090
  ```

  > 注意：该IP白名单只是限制了该域名下的请求，如果想要限制所有的，需要添加到configmap中

```
nginx.ingress.kubernetes.io/server-snippet
```

> 取值：符合nginx中server配置即可

- 举例说明，限制指定接口IP访问

  ```
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    annotations:
      nginx.ingress.kubernetes.io/server-snippet: |
  	set $test '';
  	if ($request_uri ~* (/english-parent-web/call/callback|/english-parent-web/call/send)){ 
  	  set $test 1;
  	  }
  	if ( $remote_addr !~* 123.149.112.156 ) {
  	  set $test "${test}2";
  	  }
  	if( $test = 12 ) {
  	  return 403;
  	  }
      nginx.ingress.kubernetes.io/ssl-redirect: "false"
    name: qa-gateway-web
    namespace: english-qa
  spec:
    rules:
    - host: math-manager.xx.com
      http:
        paths:
        - path: /
          backend:
            serviceName: math-manager
            servicePort: 8090
    tls:
    - hosts:
      - qa-gateway.xx.com
      secretName: secretname
  status:
    loadBalancer:
      ingress:
      - ip: 1.1.1.1
  ```

  > /english-parent-web/call/callback|/english-parent-web/call/send：正则代表接口
  >
  > $remote_addr !~* 123.149.112.156 ： 访问的IP不等于123.149.112.156，后续判断返回403

```
nginx-ingress跨域配置
```

```
    nginx.ingress.kubernetes.io/cors-allow-headers: DNT,X-Mx-ReqToken,Keep-Alive,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Authorization,X-CustomHeader,verifyCode,lang,timeZoneId,coutry_code,request-origion
    nginx.ingress.kubernetes.io/cors-allow-methods: PUT, GET, POST, OPTIONS,DELETE
    nginx.ingress.kubernetes.io/cors-allow-origin: '*'
    nginx.ingress.kubernetes.io/enable-cors: "true"
```

> 配置在注解里，这里就不做过多的解释了

```
nginx.ingress.kubernetes.io/proxy-body-size
```

> 取值：1024m ....大小
>
> 上传文件大小限制

- 配置上传文件大小为1024m

  ```
  nginx.ingress.kubernetes.io/proxy-body-size: 1024M
  ```

  > 注意：这个需要配置在注解里
  >
  > **建议添加这三个注解配合使用**
  >
  >     nginx.ingress.kubernetes.io/proxy-connect-timeout: "600"
  >     nginx.ingress.kubernetes.io/proxy-read-timeout: "600"
  >     nginx.ingress.kubernetes.io/proxy-send-timeout: "600"

```
nginx.ingress.kubernetes.io/canary
```

> 取值：true
>
> 基于权重的发布方案

- 基于请求流量进行灰度发布

  ```
  apiVersion: networking.k8s.io/v1beta1
  kind: Ingress
  metadata:
    name: wx  # Ingress 的名字，仅用于标识
    namespace: general-components
    annotations:
      nginx.ingress.kubernetes.io/canary: "true"
      nginx.ingress.kubernetes.io/canary-weight: "50"
  spec:
    rules:                      # Ingress 中定义 L7 路由规则
    - host: qa-wx-server.xx.com   # 根据 virtual hostname 进行路由（请使用您自己的域名）
      http:
        paths:                  # 按路径进行路由
        - path: /
          backend:
            serviceName: wx  # 指定后端的 Service 为之前创建的 nginx-service
            servicePort: 8090
  
  apiVersion: networking.k8s.io/v1beta1
  kind: Ingress
  metadata:
    name: wechat  # Ingress 的名字，仅用于标识
    namespace: general-components
  spec:
    rules:                      # Ingress 中定义 L7 路由规则
    - host: qa-wx-server.xx.com   # 根据 virtual hostname 进行路由（请使用您自己的域名）
      http:
        paths:                  # 按路径进行路由
        - path: /
          backend:
            serviceName: wechat  # 指定后端的 Service 为之前创建的 nginx-service
            servicePort: 8090
  
  ```

  > nginx.ingress.kubernetes.io/canary-weight: "50"： 表示50%的流量分发到serviceName: wx
  >
  > 注意：这两个需要配合使用才可以

  >     nginx.ingress.kubernetes.io/canary: "true"
  >     nginx.ingress.kubernetes.io/canary-weight: "50"
  > 更多发布方案，请自行百度解决

