## APIserver总结

> - apiserver启动三个server
> - 三个APIServer底层均依赖通用的GenericServer，使用go-restful对外提供RESTful风格的API服务
> - kube-apiserver对请求进行Authentication、Authorization和Admission三层验证
> - 验证完成后，请求会根据路由规则，触发到对应资源的handler，主要包括数据的预处理和保存
> - kube-apiserver的底层存储为etcd v3，它被抽象为一种RESTStorage，使请求和存储一一对应

### apiserver中的三个server

> - apiExtensionsServer API扩展服务，主要针对CRD
> - kubeAPIServer API核心服务，包括常见的Pod/Deployment/Service
> - apiExtensionsServer API聚合服务，主要针对metrics

### Admission准入控制

![image-20230601100331069](https://gitee.com/root_007/md_file_image/raw/master/202306141559212.png)

## kube-apiserver createPod数据的保存

### 什么是Scheme

> - k8s系统拥有众多资源，每一种资源就是一个资源类型
> - 这些资源类型需要统一的注册、存储、查询、管理等机制
> - 目前k8s系统中的所有资源类型都已注册到Scheme资源列表中，其是一个内存型的资源注册表，拥有如下特点：
>   - 支持注册多种资源类型，包括内部版本和外部版本
>   - 支持多种版本转换机制
>   - 支持不同资源的序列化/反序列化机制