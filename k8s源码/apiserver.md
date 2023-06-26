## apiserver启动流程分析

### CreateServerChain创建3个server

> CreateKubeAPIServer创建kubeAPIServer代表API核心服务和，包括常见的Pod、Deployment....
>
> CreateAPIExtensionsServer创建apiExtensionsServer代表API扩展服务，主要针对CRD
>
> createAggregatorServer创建aggregatorServer代表出阿里metrics的服务

### apiserver启动流程

#### 入口

`cmd\kube-apiserver\apiserver.go`

```go
func main() {
	rand.Seed(time.Now().UnixNano())

	pflag.CommandLine.SetNormalizeFunc(cliflag.WordSepNormalizeFunc)

	command := app.NewAPIServerCommand()

	logs.InitLogs()
	defer logs.FlushLogs()

	if err := command.Execute(); err != nil {
		os.Exit(1)
	}
}
```

##### NewAPIServerCommand

```go
func NewAPIServerCommand() *cobra.Command {
	s := options.NewServerRunOptions()
	cmd := &cobra.Command{
		Use: "kube-apiserver",
		Long: `The Kubernetes API server validates and configures data
for the api objects which include pods, services, replicationcontrollers, and
others. The API Server services REST operations and provides the frontend to the
cluster's shared state through which all other components interact.`,

		// stop printing usage when the command errors
		SilenceUsage: true,
		// 先执行PersistentPreRunE PersistentPreRun没有error返回 PersistentPreRunE有一个error返回
		PersistentPreRunE: func(*cobra.Command, []string) error {
			// silence client-go warnings.
			// kube-apiserver loopback clients should not log self-issued warnings.
            
            	  // 静默client-go的warning日志
			rest.SetDefaultWarningHandler(rest.NoWarnings{})
			return nil
		},
        	// 再次执行RunE，当然RunE也是一个有error返回
		RunE: func(cmd *cobra.Command, args []string) error {
            
            	   // 打印版本信息
			verflag.PrintAndExitIfRequested()
			fs := cmd.Flags()
			cliflag.PrintFlags(fs)
			// 检查不安全端口
			err := checkNonZeroInsecurePort(fs)
			if err != nil {
				return err
			}
			// set default options
            	   // 设置默认值
			completedOptions, err := Complete(s)
			if err != nil {
				return err
			}

			// validate options
                      // 参数校验
			if errs := completedOptions.Validate(); len(errs) != 0 {
				return utilerrors.NewAggregate(errs)
			}
			// 真正的Run函数，其中CreateServerChain创建了3个server
			return Run(completedOptions, genericapiserver.SetupSignalHandler())
		},
		Args: func(cmd *cobra.Command, args []string) error {
			for _, arg := range args {
				if len(arg) > 0 {
					return fmt.Errorf("%q does not take any arguments, got %q", cmd.CommandPath(), args)
				}
			}
			return nil
		},
	}

	fs := cmd.Flags()
	namedFlagSets := s.Flags()
	verflag.AddFlags(namedFlagSets.FlagSet("global"))
	globalflag.AddGlobalFlags(namedFlagSets.FlagSet("global"), cmd.Name())
	options.AddCustomGlobalFlags(namedFlagSets.FlagSet("generic"))
	for _, f := range namedFlagSets.FlagSets {
		fs.AddFlagSet(f)
	}

	usageFmt := "Usage:\n  %s\n"
	cols, _, _ := term.TerminalSize(cmd.OutOrStdout())
	cmd.SetUsageFunc(func(cmd *cobra.Command) error {
		fmt.Fprintf(cmd.OutOrStderr(), usageFmt, cmd.UseLine())
		cliflag.PrintSections(cmd.OutOrStderr(), namedFlagSets, cols)
		return nil
	})
	cmd.SetHelpFunc(func(cmd *cobra.Command, args []string) {
		fmt.Fprintf(cmd.OutOrStdout(), "%s\n\n"+usageFmt, cmd.Long, cmd.UseLine())
		cliflag.PrintSections(cmd.OutOrStdout(), namedFlagSets, cols)
	})

	return cmd
}
```

- checkNonZeroInsecurePort检查不安全端口

```go
// TODO: delete this check after insecure flags removed in v1.24
func checkNonZeroInsecurePort(fs *pflag.FlagSet) error {
	for _, name := range options.InsecurePortFlags {
		val, err := fs.GetInt(name)
		if err != nil {
			return err
		}
		if val != 0 {
			return fmt.Errorf("invalid port value %d: only zero is allowed", val)
		}
	}
	return nil
}
```

- Validate参数校验

  ```go
  func (s *ServerRunOptions) Validate() []error {
  	var errs []error
  	if s.MasterCount <= 0 {
  		errs = append(errs, fmt.Errorf("--apiserver-count should be a positive number, but value '%d' provided", s.MasterCount))
  	}
  	errs = append(errs, s.Etcd.Validate()...)
  	errs = append(errs, validateClusterIPFlags(s)...)
  	errs = append(errs, validateServiceNodePort(s)...)
  	errs = append(errs, validateAPIPriorityAndFairness(s)...)
  	errs = append(errs, s.SecureServing.Validate()...)
  	errs = append(errs, s.Authentication.Validate()...)
  	errs = append(errs, s.Authorization.Validate()...)
  	errs = append(errs, s.Audit.Validate()...)
  	errs = append(errs, s.Admission.Validate()...)
  	errs = append(errs, s.APIEnablement.Validate(legacyscheme.Scheme, apiextensionsapiserver.Scheme, aggregatorscheme.Scheme)...)
  	errs = append(errs, validateTokenRequest(s)...)
  	errs = append(errs, s.Metrics.Validate()...)
  	errs = append(errs, s.Logs.Validate()...)
  	errs = append(errs, validateAPIServerIdentity(s)...)
  
  	return errs
  }
  
  ```

  - Etcd.Validate()示例

    ```go
    func (s *EtcdOptions) Validate() []error {
    	if s == nil {
    		return nil
    	}
    
    	allErrors := []error{}
    	if len(s.StorageConfig.Transport.ServerList) == 0 {
    		// 必须指定一个etcd
    		allErrors = append(allErrors, fmt.Errorf("--etcd-servers must be specified"))
    	}
    
    	if s.StorageConfig.Type != storagebackend.StorageTypeUnset && !storageTypes.Has(s.StorageConfig.Type) {
    		allErrors = append(allErrors, fmt.Errorf("--storage-backend invalid, allowed values: %s. If not specified, it will default to 'etcd3'", strings.Join(storageTypes.List(), ", ")))
    	}
    
    	for _, override := range s.EtcdServersOverrides {
    		tokens := strings.Split(override, "#")
    		if len(tokens) != 2 {
    			allErrors = append(allErrors, fmt.Errorf("--etcd-servers-overrides invalid, must be of format: group/resource#servers, where servers are URLs, semicolon separated"))
    			continue
    		}
    
    		apiresource := strings.Split(tokens[0], "/")
    		if len(apiresource) != 2 {
    			allErrors = append(allErrors, fmt.Errorf("--etcd-servers-overrides invalid, must be of format: group/resource#servers, where servers are URLs, semicolon separated"))
    			continue
    		}
    
    	}
    
    	return allErrors
    }
    ```

- Run(completedOptions, genericapiserver.SetupSignalHandler())真正的Run函数

  ```go
  // stopCh 接受外部kill信号，正常情况下etcd是常驻进程
  func Run(completeOptions completedServerRunOptions, stopCh <-chan struct{}) error {
  	// To help debugging, immediately log version
  	klog.Infof("Version: %+v", version.Get())
  
  	server, err := CreateServerChain(completeOptions, stopCh)
  	if err != nil {
  		return err
  	}
  
  	prepared, err := server.PrepareRun()
  	if err != nil {
  		return err
  	}
  
  	return prepared.Run(stopCh)
  }
  
  ```

  - 其中CreateServerChain创建3个server

    ```go
    func CreateServerChain(completedOptions completedServerRunOptions, stopCh <-chan struct{}) (*aggregatorapiserver.APIAggregator, error) {
    	nodeTunneler, proxyTransport, err := CreateNodeDialer(completedOptions)
    	if err != nil {
    		return nil, err
    	}
    
    	kubeAPIServerConfig, serviceResolver, pluginInitializer, err := CreateKubeAPIServerConfig(completedOptions, nodeTunneler, proxyTransport)
    	if err != nil {
    		return nil, err
    	}
    
    	// If additional API servers are added, they should be gated.
    	apiExtensionsConfig, err := createAPIExtensionsConfig(*kubeAPIServerConfig.GenericConfig, kubeAPIServerConfig.ExtraConfig.VersionedInformers, pluginInitializer, completedOptions.ServerRunOptions, completedOptions.MasterCount,
    		serviceResolver, webhook.NewDefaultAuthenticationInfoResolverWrapper(proxyTransport, kubeAPIServerConfig.GenericConfig.EgressSelector, kubeAPIServerConfig.GenericConfig.LoopbackClientConfig))
    	if err != nil {
    		return nil, err
    	}
    	apiExtensionsServer, err := createAPIExtensionsServer(apiExtensionsConfig, genericapiserver.NewEmptyDelegate())
    	if err != nil {
    		return nil, err
    	}
    
    	kubeAPIServer, err := CreateKubeAPIServer(kubeAPIServerConfig, apiExtensionsServer.GenericAPIServer)
    	if err != nil {
    		return nil, err
    	}
    
    	// aggregator comes last in the chain
    	aggregatorConfig, err := createAggregatorConfig(*kubeAPIServerConfig.GenericConfig, completedOptions.ServerRunOptions, kubeAPIServerConfig.ExtraConfig.VersionedInformers, serviceResolver, proxyTransport, pluginInitializer)
    	if err != nil {
    		return nil, err
    	}
    	aggregatorServer, err := createAggregatorServer(aggregatorConfig, kubeAPIServer.GenericAPIServer, apiExtensionsServer.Informers)
    	if err != nil {
    		// we don't need special handling for innerStopCh because the aggregator server doesn't create any go routines
    		return nil, err
    	}
    
    	return aggregatorServer, nil
    }
    ```

### API核心服务的Authorication认证

#### Authentication认证的目的

> 验证你是谁，缺人"你是不是你"
>
> 包括多种方式，如：Client Certificates、Password、Plain Tokens、Bootstrap Tokens、JWT Tokens等

#### k8s使用身份认证插件

> 利用下面的策略来认证API请求的身份
>
> 1. 客户端证书
> 2. 持有者令牌(Bearer Token)
> 3. 身份认证代理(Proxy)
> 4. HTTP基本认证机制
>
> 参考文档： https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/authentication/

#### union认证的规则

> 如果某一个认证方法报错就返回，说明认证没过
>
> 如果某一个认证方法报ok，说明认证过了，直接return，无需再运行其他认证
>
> 如果所有的认证都没ok，则认证没过

#### 身份认证策略

> Kubernetes 通过身份认证插件利用客户端证书、持有者令牌（Bearer Token）或身份认证代理（Proxy） 来认证 API 请求的身份。HTTP 请求发给 API 服务器时，插件会将以下属性关联到请求本身：
>
> - 用户名：用来辩识最终用户的字符串。常见的值可以是 `kube-admin` 或 `jane@example.com`。
> - 用户 ID：用来辩识最终用户的字符串，旨在比用户名有更好的一致性和唯一性。
> - 用户组：取值为一组字符串，其中各个字符串用来标明用户是某个命名的用户逻辑集合的成员。 常见的值可能是 `system:masters` 或者 `devops-team` 等。
> - 附加字段：一组额外的键-值映射，键是字符串，值是一组字符串； 用来保存一些鉴权组件可能觉得有用的额外信息。
>
> 所有（属性）值对于身份认证系统而言都是不透明的， 只有被[鉴权组件](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/authorization/)解释过之后才有意义。
>
> 你可以同时启用多种身份认证方法，并且你通常会至少使用两种方法：
>
> - 针对服务账号使用服务账号令牌
> - 至少另外一种方法对用户的身份进行认证
>
> 当集群中启用了多个身份认证模块时，第一个成功地对请求完成身份认证的模块会直接做出评估决定。 API 服务器并不保证身份认证模块的运行顺序。
>
> 对于所有通过身份认证的用户，`system:authenticated` 组都会被添加到其组列表中。
>
> 与其它身份认证协议（LDAP、SAML、Kerberos、X509 的替代模式等等） 都可以通过使用一个[身份认证代理](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/authentication/#authenticating-proxy)或[身份认证 Webhoook](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/authentication/#webhook-token-authentication) 来实现

#### 代码解读

- 之前构建server之前生成通用配置buildGenericConfig里

  ​	`cmd\kube-apiserver\app\server.go`

  ```go
  	if lastErr = s.Authentication.ApplyTo(&genericConfig.Authentication, genericConfig.SecureServing, genericConfig.EgressSelector, genericConfig.OpenAPIConfig, clientgoExternalClient, versionedInformers); lastErr != nil {
  		return
  	}
  ```

  - ApplyTo中初始化Authentication

    ```go
    	// 初始化Authentication
    	authInfo.Authenticator, openAPIConfig.SecurityDefinitions, err = authenticatorConfig.New()
    ```

    > New代码、创建认证实例，支持多种认证方式：请求Header认证、Auth文件认证、CA证书认证、Bearer token认证

    - New代码核心变量1 `tokenAuthenticators []authenticator.Token` 其代表Bearer token认证

      ```go
      func (config Config) New() (authenticator.Request, *spec.SecurityDefinitions, error) {
      	var authenticators []authenticator.Request
      	var tokenAuthenticators []authenticator.Token
          ......
      }
      // 其authenticator.Token
      type Token interface {
      	AuthenticateToken(ctx context.Context, token string) (*Response, bool, error)
      }
      ```

    - New代码核心变量2 `authenticators []authenticator.Request`其代表用户认证

      认证方法，其是由`Request`跳转

      `AuthenticateRequest`是对应的认证方法

      ```go
      // 用户认证接口
      type Request interface {
      	AuthenticateRequest(req *http.Request) (*Response, bool, error)
      }
      ```

- 总结

  > New方法里定义了2个变量`authenticators`和`tokenAuthenticators`，用于进行身份认证，根据启用的插件进行append，例如x509：
  >
  > ```go
  > 	// 证书认证
  > 	if config.ClientCAContentProvider != nil {
  > 		certAuth := x509.NewDynamic(config.ClientCAContentProvider.VerifyOptions, x509.CommonNameUserConversion)
  > 		authenticators = append(authenticators, certAuth)
  > 	}
  > ```
  >
  > 然后进行union
  >
  > ```go
  > authenticator := union.New(authenticators...)
  > tokenAuth := tokenunion.New(tokenAuthenticators...)
  > ```

### API核心服务的Authorization鉴权

![image-20230530155034057](https://gitee.com/root_007/md_file_image/raw/master/202305301550171.png)

> - Authorization鉴权，缺人"你是否有权利做这件事"，如何判定是否有权力，通过配置策略
> - k8s使用API服务器对API请求进行鉴权
> - 它根据所有策略评估所有请求属性来决定允许或拒绝请求
> - 一个API请求的所有部分都必须被某些策略允许才能继续，这意味着默认情况下拒绝权限
> - 当系统配置了多个鉴权模块时，k8s将按照顺序使用每个模块，如果任何鉴权模块批准或拒绝请求，则立即返回该决定，并且不会与其他鉴权协商，如果所有模块对请求没意见，则拒绝该请求，被拒绝相应返回HTTP状态码403
>
> 参考文档： https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/authorization/

#### 4种鉴权模块

> - **Node** ：一个专用鉴权模式，根据调度到 kubelet 上运行的 Pod 为 kubelet 授予权限。 要了解有关使用节点鉴权模式的更多信息，请参阅[节点鉴权](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/node/)。
> - **ABAC** ：基于属性的访问控制（ABAC）定义了一种访问控制范型，通过使用将属性组合在一起的策略， 将访问权限授予用户。策略可以使用任何类型的属性（用户属性、资源属性、对象，环境属性等）。 要了解有关使用 ABAC 模式的更多信息，请参阅 [ABAC 模式](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/abac/)。
> - **RBAC**：基于角色的访问控制（RBAC） 是一种基于企业内个人用户的角色来管理对计算机或网络资源的访问的方法。 在此上下文中，权限是单个用户执行特定任务的能力， 例如查看、创建或修改文件。要了解有关使用 RBAC 模式的更多信息，请参阅RBAC 模式
>   - 被启用之后，RBAC（基于角色的访问控制）使用 `rbac.authorization.k8s.io` API 组来驱动鉴权决策，从而允许管理员通过 Kubernetes API 动态配置权限策略。
>   - 要启用 RBAC，请使用 `--authorization-mode = RBAC` 启动 API 服务器。
> - **Webhook** ： WebHook 是一个 HTTP 回调：发生某些事情时调用的 HTTP POST； 通过 HTTP POST 进行简单的事件通知。 实现 WebHook 的 Web 应用程序会在发生某些事情时将消息发布到 URL。 要了解有关使用 Webhook 模式的更多信息，请参阅 [Webhook 模式](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/webhook/)。
>
> 参考文档： https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/authorization/

#### 代码解读

- 之前构建server之前生成通用配置buildGenericConfig里

  ​	`cmd\kube-apiserver\app\server.go`

  ```go
  genericConfig.Authorization.Authorizer, genericConfig.RuleResolver, err = BuildAuthorizer(s, genericConfig.EgressSelector, versionedInformers)
  ```

  - BuildAuthorizer

    ```go
    func BuildAuthorizer(s *options.ServerRunOptions, EgressSelector *egressselector.EgressSelector, versionedInformers clientgoinformers.SharedInformerFactory) (authorizer.Authorizer, authorizer.RuleResolver, error) {
    	authorizationConfig := s.Authorization.ToAuthorizationConfig(versionedInformers)
    
    	if EgressSelector != nil {
    		egressDialer, err := EgressSelector.Lookup(egressselector.ControlPlane.AsNetworkContext())
    		if err != nil {
    			return nil, nil, err
    		}
    		authorizationConfig.CustomDial = egressDialer
    	}
    
    	return authorizationConfig.New()
    }
    ```

    > 其return authorizationConfig.New()方法
    >
    > ```go
    > func (config Config) New() (authorizer.Authorizer, authorizer.RuleResolver, error) {
    > 	if len(config.AuthorizationModes) == 0 {
    > 		return nil, nil, fmt.Errorf("at least one authorization mode must be passed")
    > 	}
    > 
    > 	var (
    > 		authorizers   []authorizer.Authorizer
    > 		ruleResolvers []authorizer.RuleResolver
    > 	)
    > 	...... // 省略其中代码，自行查看
    > }
    > ```

    - 构造函数New分析

      - 核心变量`authorizers`

        ```go
        type Authorizer interface {
        	Authorize(ctx context.Context, a Attributes) (authorized Decision, reason string, err error)
        }
        ```

        > 鉴权接口，有对应的Authorize执行鉴权操作，返回参数：
        >
        > 1. Decision：代表鉴权结果(拒绝：DecisionDeny，通过：DecisionAllow，未表态：DecisionNoOpinion)
        > 2. reason：代表拒绝的原因
        >
        > ```
        > const (
        > 	// DecisionDeny means that an authorizer decided to deny the action.
        > 	DecisionDeny Decision = iota
        > 	// DecisionAllow means that an authorizer decided to allow the action.
        > 	DecisionAllow
        > 	// DecisionNoOpionion means that an authorizer has no opinion on whether
        > 	// to allow or deny an action.
        > 	DecisionNoOpinion
        > )
        > ```
        >
        > 

      - 核心变量`RuleResolver`

        ```go
        type RuleResolver interface {
        	// RulesFor get the list of cluster wide rules, the list of rules in the specific namespace, incomplete status and errors.
        	RulesFor(user user.Info, namespace string) ([]ResourceRuleInfo, []NonResourceRuleInfo, bool, error)
        }
        ```

        > 获取rule的接口，有对应的RulesFor执行获取rule操作，返回参数：
        >
        > 1. []ResourceRuleInfo：代表资源类型rule
        > 2. []NonResourceRuleInfo：代表非资源型的如：nonResourceURLs: ["/metrics"]

      - 遍历鉴权模块，向上述切片中append

        ```go
        	// 遍历鉴权模块，向上述切片中append
        	for _, authorizationMode := range config.AuthorizationModes {
        		// Keep cases in sync with constant list in k8s.io/kubernetes/pkg/kubeapiserver/authorizer/modes/modes.go.
        		switch authorizationMode {
        		case modes.ModeNode:
        			node.RegisterMetrics()
        			graph := node.NewGraph()
        			node.AddGraphEventHandlers(
        				graph,
        				config.VersionedInformerFactory.Core().V1().Nodes(),
        				config.VersionedInformerFactory.Core().V1().Pods(),
        				config.VersionedInformerFactory.Core().V1().PersistentVolumes(),
        				config.VersionedInformerFactory.Storage().V1().VolumeAttachments(),
        			)
        			nodeAuthorizer := node.NewAuthorizer(graph, nodeidentifier.NewDefaultNodeIdentifier(), bootstrappolicy.NodeRules())
        			authorizers = append(authorizers, nodeAuthorizer)
        			ruleResolvers = append(ruleResolvers, nodeAuthorizer)
        
        		case modes.ModeAlwaysAllow:
        			alwaysAllowAuthorizer := authorizerfactory.NewAlwaysAllowAuthorizer()
        			authorizers = append(authorizers, alwaysAllowAuthorizer)
        			ruleResolvers = append(ruleResolvers, alwaysAllowAuthorizer)
        		case modes.ModeAlwaysDeny:
        			alwaysDenyAuthorizer := authorizerfactory.NewAlwaysDenyAuthorizer()
        			authorizers = append(authorizers, alwaysDenyAuthorizer)
        			ruleResolvers = append(ruleResolvers, alwaysDenyAuthorizer)
                    	
                    // 中间代码省略了 ......
                    
                    return union.New(authorizers...), union.NewRuleResolvers(ruleResolvers...), nil
        ```

        > 最后返回两个对象的**union**对象

### node类型的Authorization鉴权

> 节点鉴权是一种特殊用途的鉴权模式，专门针对kubelet发出的API请求进行鉴权
>
> 4种规则解读，如果不是node的请求则拒绝
>
> 如果nodeName没找到则拒绝
>
> 如果请求的是configmag、pod、pv、pvc、secret需要校验
>
> 如果动作是非get，拒绝
>
> 如果请求的资源和节点没关系拒绝
>
> 如果请求其他资源，需要按照定义好的rule匹配
>
> 参考文档： https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/node/

#### 概述

> 节点鉴权器允许 kubelet 执行 API 操作。包括：
>
> **读取操作：**
>
> - services
> - endpoints
> - nodes
> - pods
> - 与绑定到 kubelet 节点的 Pod 相关的 Secret、ConfigMap、PersistentVolumeClaim 和持久卷
>
> **写入操作：**
>
> - 节点和节点状态（启用 `NodeRestriction` 准入插件以限制 kubelet 只能修改自己的节点）
> - Pod 和 Pod 状态 (启用 `NodeRestriction` 准入插件以限制 kubelet 只能修改绑定到自身的 Pod)
> - 事件
>
> **身份认证与鉴权相关的操作：**
>
> - 对于基于 TLS 的启动引导过程时使用的 [certificationsigningrequests API](https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/certificate-signing-requests/) 的读/写权限
> - 为委派的身份验证/鉴权检查创建 TokenReview 和 SubjectAccessReview 的能力

#### 代码解读

`cmd\kube-apiserver\apiserver.go`(NewAPIServerCommand跳转)-->`cmd\kube-apiserver\app\server.go`(buildGenericConfig中genericConfig.Authorization.Authorizer, genericConfig.RuleResolver, err = BuildAuthorizer(s, genericConfig.EgressSelector, versionedInformers)**BuildAuthorizer**跳转)-->cmd\kube-`apiserver\app\server.go`(return authorizationConfig.New()，New()跳转)-->`pkg\kubeapiserver\authorizer\config.go`

```go
	// 遍历鉴权模块，向上述切片中append
	for _, authorizationMode := range config.AuthorizationModes {
		// Keep cases in sync with constant list in k8s.io/kubernetes/pkg/kubeapiserver/authorizer/modes/modes.go.
		switch authorizationMode {
		case modes.ModeNode:
			node.RegisterMetrics()
			graph := node.NewGraph()
			node.AddGraphEventHandlers(
				graph,
				config.VersionedInformerFactory.Core().V1().Nodes(),
				config.VersionedInformerFactory.Core().V1().Pods(),
				config.VersionedInformerFactory.Core().V1().PersistentVolumes(),
				config.VersionedInformerFactory.Storage().V1().VolumeAttachments(),
			)
			nodeAuthorizer := node.NewAuthorizer(graph, nodeidentifier.NewDefaultNodeIdentifier(), bootstrappolicy.NodeRules())
			authorizers = append(authorizers, nodeAuthorizer)
			ruleResolvers = append(ruleResolvers, nodeAuthorizer)
            ......代码省略

```

- NewAuthorizer

  ```go
  // NodeAuthorizer authorizes requests from kubelets, with the following logic:
  // 1. If a request is not from a node (NodeIdentity() returns isNode=false), reject
  // 2. If a specific node cannot be identified (NodeIdentity() returns nodeName=""), reject
  // 3. If a request is for a secret, configmap, persistent volume or persistent volume claim, reject unless the verb is get, and the requested object is related to the requesting node:
  //    node <- configmap
  //    node <- pod
  //    node <- pod <- secret
  //    node <- pod <- configmap
  //    node <- pod <- pvc
  //    node <- pod <- pvc <- pv
  //    node <- pod <- pvc <- pv <- secret
  // 4. For other resources, authorize all nodes uniformly using statically defined rules
  type NodeAuthorizer struct {
  	graph      *Graph
  	identifier nodeidentifier.NodeIdentifier
  	nodeRules  []rbacv1.PolicyRule
  
  	// allows overriding for testing
  	features featuregate.FeatureGate
  }
  
  var _ = authorizer.Authorizer(&NodeAuthorizer{})
  var _ = authorizer.RuleResolver(&NodeAuthorizer{})
  
  // NewAuthorizer returns a new node authorizer
  func NewAuthorizer(graph *Graph, identifier nodeidentifier.NodeIdentifier, rules []rbacv1.PolicyRule) *NodeAuthorizer {
  	return &NodeAuthorizer{
  		graph:      graph,
  		identifier: identifier,
  		nodeRules:  rules,
  		features:   utilfeature.DefaultFeatureGate,
  	}
  }
  ```

  > 规则解读：
  >
  > 1. If a request is not from a node (NodeIdentity() returns isNode=false), reject
  >
  >    如果请求不是来自于node，拒绝
  >
  > 2. If a specific node cannot be identified (NodeIdentity() returns nodeName=""), reject
  >
  >    如果节点没有被识别，拒绝
  >
  > 3. If a request is for a secret, configmap, persistent volume or persistent volume claim, reject unless the verb is get, and the requested object is related 
  >
  >    如果请求的是secret，configmap，pv，pvc非get请求拒绝
  >
  > 4. For other resources, authorize all nodes uniformly using statically defined rules
  >
  >    如果是其他资源，需要按照定义好的rule匹配

  - 规则3解读

    ```go
    	// subdivide access to specific resources
    	// 其他资源的访问
    	if attrs.IsResourceRequest() {
    		requestResource := schema.GroupResource{Group: attrs.GetAPIGroup(), Resource: attrs.GetResource()}
    		switch requestResource {
    		// secret资源
    		case secretResource:
    			return r.authorizeReadNamespacedObject(nodeName, secretVertexType, attrs)
    		//	configmap资源
    		case configMapResource:
    			return r.authorizeReadNamespacedObject(nodeName, configMapVertexType, attrs)
    		//	pvc资源
    		case pvcResource:
    			if r.features.Enabled(features.ExpandPersistentVolumes) {
    				if attrs.GetSubresource() == "status" {
    					return r.authorizeStatusUpdate(nodeName, pvcVertexType, attrs)
    				}
    			}
    			return r.authorizeGet(nodeName, pvcVertexType, attrs)
    		//	pv资源
    		case pvResource:
    			return r.authorizeGet(nodeName, pvVertexType, attrs)
    		//	VolumeAttachment资源
    		case vaResource:
    			return r.authorizeGet(nodeName, vaVertexType, attrs)
    		//	svc资源
    		case svcAcctResource:
    			return r.authorizeCreateToken(nodeName, serviceAccountVertexType, attrs)
    		//	lease资源 (用于控制多个节点或多个 Pod 之间的协调和同步)
    		case leaseResource:
    			return r.authorizeLease(nodeName, attrs)
    		//	csi资源 (Kubernetes CSI (Container Storage Interface) 是 Kubernetes 中一种标准化的存储插件框架，
    		//	用于扩展和管理 Kubernetes 集群中的存储资源。CSI 插件可以将外部存储系统（如云存储、网络文件系统或本地存储）
    		//	与 Kubernetes 集群集成，并作为 PVC (Persistent Volume Claim) 的后端存储提供服务。)
    		case csiNodeResource:
    			return r.authorizeCSINode(nodeName, attrs)
    		}
    
    	}
    ```

    - authorizeReadNamespacedObject验证namespace的方法

      ```go
      // authorizeReadNamespacedObject authorizes "get", "list" and "watch" requests to single objects of a
      // specified types if they are related to the specified node.
      func (r *NodeAuthorizer) authorizeReadNamespacedObject(nodeName string, startingType vertexType, attrs authorizer.Attributes) (authorizer.Decision, string, error) {
      	switch attrs.GetVerb() {
      	// get list watch方法 可以通过请求
      	case "get", "list", "watch":
      		//ok
      	//	否则验证不通过，拒绝
      	default:
      		klog.V(2).Infof("NODE DENY: '%s' %#v", nodeName, attrs)
      		return authorizer.DecisionNoOpinion, "can only read resources of this type", nil
      	}
      
      	if len(attrs.GetSubresource()) > 0 {
      		klog.V(2).Infof("NODE DENY: '%s' %#v", nodeName, attrs)
      		return authorizer.DecisionNoOpinion, "cannot read subresource", nil
      	}
      	if len(attrs.GetNamespace()) == 0 {
      		klog.V(2).Infof("NODE DENY: '%s' %#v", nodeName, attrs)
      		return authorizer.DecisionNoOpinion, "can only read namespaced object of this type", nil
      	}
      	return r.authorize(nodeName, startingType, attrs)
      }
      
      ```

      > 解读：
      >
      > DecisionNoOpinion：代表不表态，如果只有一个Authorizer，意味着拒绝
      >
      > 如果动作是变更类型的拒绝
      >
      > 如果请求保护子资源就拒绝
      >
      > 如果请求参数中没有namespace就拒绝
      >
      > 然后底层调用`authorize`

      - node底层调用**authorize**方法

        authorizeReadNamespacedObject方法的return返回方法为**authorize**

        ```go
        func (r *NodeAuthorizer) authorize(nodeName string, startingType vertexType, attrs authorizer.Attributes) (authorizer.Decision, string, error) {
        	// 资源不存在 拒绝
        	if len(attrs.GetName()) == 0 {
        		klog.V(2).Infof("NODE DENY: '%s' %#v", nodeName, attrs)
        		return authorizer.DecisionNoOpinion, "No Object name found", nil
        	}
        	// hasPathFrom代表判断资源是否和节点有关系，没关系 拒绝
        	ok, err := r.hasPathFrom(nodeName, startingType, attrs.GetNamespace(), attrs.GetName())
        	if err != nil {
        		klog.V(2).InfoS("NODE DENY", "err", err)
        		return authorizer.DecisionNoOpinion, fmt.Sprintf("no relationship found between node '%s' and this object", nodeName), nil
        	}
        	if !ok {
        		klog.V(2).Infof("NODE DENY: '%s' %#v", nodeName, attrs)
        		return authorizer.DecisionNoOpinion, fmt.Sprintf("no relationship found between node '%s' and this object", nodeName), nil
        	}
        	return authorizer.DecisionAllow, "", nil
        }
        ```

    - pvResource资源authorizeGet方法

      ```go
      // authorizeGet authorizes "get" requests to objects of the specified type if they are related to the specified node
      func (r *NodeAuthorizer) authorizeGet(nodeName string, startingType vertexType, attrs authorizer.Attributes) (authorizer.Decision, string, error) {
      	// 非get请求 拒绝
      	if attrs.GetVerb() != "get" {
      		klog.V(2).Infof("NODE DENY: '%s' %#v", nodeName, attrs)
      		return authorizer.DecisionNoOpinion, "can only get individual resources of this type", nil
      	}
      	// 如果含有subresource资源拒绝
      	if len(attrs.GetSubresource()) > 0 {
      		klog.V(2).Infof("NODE DENY: '%s' %#v", nodeName, attrs)
      		return authorizer.DecisionNoOpinion, "cannot get subresource", nil
      	}
      	// 返回调用authorize方法
      	return r.authorize(nodeName, startingType, attrs)
      }
      ```

      > 如果动作不是get就拒绝
      >
      > 如果含有subresource就拒绝
      >
      > 然后底层调用authorize方法，与上边node底层调用**authorize**方法是同一个方法哦

  - 规则4解读

    ```go
    	// Access to other resources is not subdivided, so just evaluate against the statically defined node rules
    	// 如果请求是其他资源，需要通过对应的静态规则认证
    	if rbac.RulesAllow(attrs, r.nodeRules...) {
    		return authorizer.DecisionAllow, "", nil
    	}
    ```

    - 底层调用rbac.RulesAllow

      ```go
      func RulesAllow(requestAttributes authorizer.Attributes, rules ...rbacv1.PolicyRule) bool {
      	for i := range rules {
      		if RuleAllows(requestAttributes, &rules[i]) {
      			return true
      		}
      	}
      
      	return false
      }
      
      func RuleAllows(requestAttributes authorizer.Attributes, rule *rbacv1.PolicyRule) bool {
      	if requestAttributes.IsResourceRequest() {
      		combinedResource := requestAttributes.GetResource()
      		if len(requestAttributes.GetSubresource()) > 0 {
      			combinedResource = requestAttributes.GetResource() + "/" + requestAttributes.GetSubresource()
      		}
      
      		return rbacv1helpers.VerbMatches(rule, requestAttributes.GetVerb()) &&
      			rbacv1helpers.APIGroupMatches(rule, requestAttributes.GetAPIGroup()) &&
      			rbacv1helpers.ResourceMatches(rule, combinedResource, requestAttributes.GetSubresource()) &&
      			rbacv1helpers.ResourceNameMatches(rule, requestAttributes.GetName())
      	}
      
      	return rbacv1helpers.VerbMatches(rule, requestAttributes.GetVerb()) &&
      		rbacv1helpers.NonResourceURLMatches(rule, requestAttributes.GetPath())
      }
      ```

      - VerbMatches方法

        RulesAllow 底层调用

        ```go
        // RulesAllow 底层调用
        
        func VerbMatches(rule *rbacv1.PolicyRule, requestedVerb string) bool {
        	// 请求的方法需要跟配置的方法一致，http请求方法需要跟Verbs配置的一直，当然 '*' 除外
        	for _, ruleVerb := range rule.Verbs {
        		if ruleVerb == rbacv1.VerbAll {
        			return true
        		}
        		if ruleVerb == requestedVerb {
        			return true
        		}
        	}
        
        	return false
        }
        ```

- node的rules `bootstrappolicy.NodeRules()`

  ```go
  func NodeRules() []rbacv1.PolicyRule {
  	nodePolicyRules := []rbacv1.PolicyRule{
  		// 检查 API 访问权限所需。这些创建操作是非变更性的。
  		rbacv1helpers.NewRule("create").Groups(authenticationGroup).Resources("tokenreviews").RuleOrDie(),
  		rbacv1helpers.NewRule("create").Groups(authorizationGroup).Resources("subjectaccessreviews", "localsubjectaccessreviews").RuleOrDie(),
  
  		// 需要构建服务列表以为服务填充环境变量
  		rbacv1helpers.NewRule(Read...).Groups(legacyGroup).Resources("services").RuleOrDie(),
  
  		// 节点可以注册 Node API 对象并报告状态。
  		// 使用 NodeRestriction Admission 插件将节点限制为仅能创建/更新自己的 API 对象。
  		rbacv1helpers.NewRule("create", "get", "list", "watch").Groups(legacyGroup).Resources("nodes").RuleOrDie(),
  		rbacv1helpers.NewRule("update", "patch").Groups(legacyGroup).Resources("nodes/status").RuleOrDie(),
  		rbacv1helpers.NewRule("update", "patch").Groups(legacyGroup).Resources("nodes").RuleOrDie(),
  
  		// TODO: 在 NodeRestrictions Admission 插件中将其限制为作为创建者绑定到的节点
  		rbacv1helpers.NewRule("create", "update", "patch").Groups(legacyGroup).Resources("events").RuleOrDie(),
  
  		// TODO: 一旦列表/监视授权支持字段选择器，就将其限制为已调度到绑定节点的 Pod
  		rbacv1helpers.NewRule(Read...).Groups(legacyGroup).Resources("pods").RuleOrDie(),
  
  		// 需要节点创建/删除镜像 Pod。
  		// 使用 NodeRestriction Admission 插件将节点限制为仅能创建/删除绑定到自身的镜像 Pod。
  		rbacv1helpers.NewRule("create", "delete").Groups(legacyGroup).Resources("pods").RuleOrDie(),
  		// 需要节点报告正在运行的 Pod 的状态。
  		// 使用 NodeRestriction Admission 插件将节点限制为仅能更新绑定到自身的 Pod 的状态。
  		rbacv1helpers.NewRule("update", "patch").Groups(legacyGroup).Resources("pods/status").RuleOrDie(),
  		// 需要节点创建 Pod 驱逐。
  		// 使用 NodeRestriction Admission 插件将节点限制为仅能为绑定到自身的 Pod 创建驱逐。
  		rbacv1helpers.NewRule("create").Groups(legacyGroup).Resources("pods/eviction").RuleOrDie(),
  
  		// 用于 imagepullsecrets、rbd/ceph 和密钥卷，以及 envs 中的密钥
  		// 用于配置映射卷和 envs
  		// 使用 Node 授权模式将节点限制为获取与绑定到自身的 Pod 相关联的机密/配置映射。
  		rbacv1helpers.NewRule("get", "list", "watch").Groups(legacyGroup).Resources("secrets", "configmaps").RuleOrDie(),
  		// 需要持久卷
  		// 使用 Node 授权模式将节点限制为获取与绑定到自身的 Pod 相关联的 pv/pvc 对象。
  		rbacv1helpers.NewRule("get").Groups(legacyGroup).Resources("persistentvolumeclaims", "persistentvolumes").RuleOrDie(),
  
  		// TODO: 添加到 Node 授权器并限制为节点引用的端点
  		// 需要 glusterfs 卷
  		rbacv1helpers.NewRule("get").Groups(legacyGroup).Resources("endpoints").RuleOrDie(),
  		// 用于创建节点特定客户端证书的 certificatesigningrequest，并监视其签名。这允许 kubelet 旋转自己的证书。
  		rbacv1helpers.NewRule("create", "get", "list", "watch").Groups(certificatesGroup
  ```

  ```go
  // Write and other vars are slices of the allowed verbs.
  // Label and Annotation are default maps of bootstrappolicy.
  var (
  	Write      = []string{"create", "update", "patch", "delete", "deletecollection"}
  	ReadWrite  = []string{"get", "list", "watch", "create", "update", "patch", "delete", "deletecollection"}
  	Read       = []string{"get", "list", "watch"}
  	ReadUpdate = []string{"get", "list", "watch", "update", "patch"}
  
  	Label      = map[string]string{"kubernetes.io/bootstrapping": "rbac-defaults"}
  	Annotation = map[string]string{rbacv1.AutoUpdateAnnotationKey: "true"}
  )
  ```

### rbac类型的Authorization鉴权

> rbac四种对象的关系
>
> - role、clusterrole
> - rolebinding、clusterrolebinding

#### role、clusterrole中的rules规则

> 资源对象
>
> 非资源对象
>
> apiGroups
>
> verb动作

#### rbac鉴权的代码逻辑

> - 通过informer获取clusterRoleBindings列表，根据user匹配subject，通过informer获取clusterRoleBindings的rules，遍历调用visit进行rule匹配
> - 通过informer获取RoleBindgings列表，根据user和namespace匹配subject，通过informer获取RoleBindings的rules，遍历调用visit进行rule匹配

#### rbac鉴权模型

![image-20230531150835552](https://gitee.com/root_007/md_file_image/raw/master/202305311508612.png)

> 参考文档： https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/rbac/

- RBAC简介

  > 基于角色（Role）的访问控制（RBAC）是一种基于组织中用户的角色来调节控制对计算机或网络资源的访问的方法。
  >
  > RBAC 鉴权机制使用 `rbac.authorization.k8s.io` [API 组](https://kubernetes.io/zh-cn/docs/concepts/overview/kubernetes-api/#api-groups-and-versioning)来驱动鉴权决定， 允许你通过 Kubernetes API 动态配置策略。
  >
  > 要启用 RBAC，在启动 [API 服务器](https://kubernetes.io/zh-cn/docs/concepts/overview/components/#kube-apiserver)时将 `--authorization-mode` 参数设置为一个逗号分隔的列表并确保其中包含 `RBAC`。
  >
  > ```bash
  > kube-apiserver --authorization-mode=Example,RBAC --<其他选项> --<其他选项>
  > ```

#### 代码解读

```go
	// 遍历鉴权模块，向上述切片中append
	for _, authorizationMode := range config.AuthorizationModes {
		// Keep cases in sync with constant list in k8s.io/kubernetes/pkg/kubeapiserver/authorizer/modes/modes.go.
		switch authorizationMode {
		case modes.ModeNode:
			node.RegisterMetrics()
			graph := node.NewGraph()
			node.AddGraphEventHandlers(
				graph,
				config.VersionedInformerFactory.Core().V1().Nodes(),
				config.VersionedInformerFactory.Core().V1().Pods(),
				config.VersionedInformerFactory.Core().V1().PersistentVolumes(),
				config.VersionedInformerFactory.Storage().V1().VolumeAttachments(),
			)
			nodeAuthorizer := node.NewAuthorizer(graph, nodeidentifier.NewDefaultNodeIdentifier(), bootstrappolicy.NodeRules())
			authorizers = append(authorizers, nodeAuthorizer)
			ruleResolvers = append(ruleResolvers, nodeAuthorizer)

		case modes.ModeAlwaysAllow:
			alwaysAllowAuthorizer := authorizerfactory.NewAlwaysAllowAuthorizer()
			authorizers = append(authorizers, alwaysAllowAuthorizer)
			ruleResolvers = append(ruleResolvers, alwaysAllowAuthorizer)
		case modes.ModeAlwaysDeny:
			alwaysDenyAuthorizer := authorizerfactory.NewAlwaysDenyAuthorizer()
			authorizers = append(authorizers, alwaysDenyAuthorizer)
			ruleResolvers = append(ruleResolvers, alwaysDenyAuthorizer)
		case modes.ModeABAC:
			abacAuthorizer, err := abac.NewFromFile(config.PolicyFile)
			if err != nil {
				return nil, nil, err
			}
			authorizers = append(authorizers, abacAuthorizer)
			ruleResolvers = append(ruleResolvers, abacAuthorizer)
		case modes.ModeWebhook:
			if config.WebhookRetryBackoff == nil {
				return nil, nil, errors.New("retry backoff parameters for authorization webhook has not been specified")
			}
			webhookAuthorizer, err := webhook.New(config.WebhookConfigFile,
				config.WebhookVersion,
				config.WebhookCacheAuthorizedTTL,
				config.WebhookCacheUnauthorizedTTL,
				*config.WebhookRetryBackoff,
				config.CustomDial)
			if err != nil {
				return nil, nil, err
			}
			authorizers = append(authorizers, webhookAuthorizer)
			ruleResolvers = append(ruleResolvers, webhookAuthorizer)
		case modes.ModeRBAC:
			rbacAuthorizer := rbac.New(
				&rbac.RoleGetter{Lister: config.VersionedInformerFactory.Rbac().V1().Roles().Lister()},
				&rbac.RoleBindingLister{Lister: config.VersionedInformerFactory.Rbac().V1().RoleBindings().Lister()},
				&rbac.ClusterRoleGetter{Lister: config.VersionedInformerFactory.Rbac().V1().ClusterRoles().Lister()},
				&rbac.ClusterRoleBindingLister{Lister: config.VersionedInformerFactory.Rbac().V1().ClusterRoleBindings().Lister()},
			)
			authorizers = append(authorizers, rbacAuthorizer)
			ruleResolvers = append(ruleResolvers, rbacAuthorizer)
		default:
			return nil, nil, fmt.Errorf("unknown authorization mode %s specified", authorizationMode)
		}
	}
```

> 该代码位置查找，与node类型的Authorization鉴权一致(同一个位置的代码)，这里看一下`modes.ModeRBAC`

- rbac.New传入Role、RoleBinding、ClusterRole、ClusterRoleBinding 4种对象的Getter

  ```go
  func New(roles rbacregistryvalidation.RoleGetter, roleBindings rbacregistryvalidation.RoleBindingLister,
  	clusterRoles rbacregistryvalidation.ClusterRoleGetter,
  	clusterRoleBindings rbacregistryvalidation.ClusterRoleBindingLister) *RBACAuthorizer {
  	authorizer := &RBACAuthorizer{
  		authorizationRuleResolver: rbacregistryvalidation.NewDefaultRuleResolver(
  			roles, roleBindings, clusterRoles, clusterRoleBindings,
  		),
  	}
  	return authorizer
  }
  ```

  - *RRBACAuthorizer

    Authorize解析

    ```go
    func (r *RBACAuthorizer) Authorize(ctx context.Context, requestAttributes authorizer.Attributes) (authorizer.Decision, string, error) {
    	ruleCheckingVisitor := &authorizingVisitor{requestAttributes: requestAttributes}
    
    	r.authorizationRuleResolver.VisitRulesFor(requestAttributes.GetUser(), requestAttributes.GetNamespace(), ruleCheckingVisitor.visit)
    	if ruleCheckingVisitor.allowed {
    		return authorizer.DecisionAllow, ruleCheckingVisitor.reason, nil
    	}
    		reason := ""
    	if len(ruleCheckingVisitor.errors) > 0 {
    		reason = fmt.Sprintf("RBAC: %v", utilerrors.NewAggregate(ruleCheckingVisitor.errors))
    	}
    	return authorizer.DecisionNoOpinion, reason, nil
    }
    ```

    >  其核心在于`ruleCheckingVisitor.allowed`如果为true，则通过，否则就拒绝

    - 这个allowed标志位只有在visti方法中才会被设置，条件是RuleAllows==true

      ```go
      type authorizingVisitor struct {
      	requestAttributes authorizer.Attributes
      
      	allowed bool
      	reason  string
      	errors  []error
      }
      
      func (v *authorizingVisitor) visit(source fmt.Stringer, rule *rbacv1.PolicyRule, err error) bool {
      	if rule != nil && RuleAllows(v.requestAttributes, rule) {
      		v.allowed = true
      		v.reason = fmt.Sprintf("RBAC: allowed by %s", source.String())
      		return false
      	}
      	if err != nil {
      		v.errors = append(v.errors, err)
      	}
      	return true
      }
      ```

    - VisitRulesFor调用visit方法校验每一条rule

      ```go
      	// VisitRulesFor invokes visitor() with each rule that applies to a given user in a given namespace,
      	// and each error encountered resolving those rules. Rule may be nil if err is non-nil.
      	// If visitor() returns false, visiting is short-circuited.
      	VisitRulesFor(user user.Info, namespace string, visitor func(source fmt.Stringer, rule *rbacv1.PolicyRule, err error) bool)
      ```

      - VisitRulesFor

        ![image-20230531194101752](https://gitee.com/root_007/md_file_image/raw/master/202305311941821.png)

        ```go
        func (r *DefaultRuleResolver) VisitRulesFor(user user.Info, namespace string, visitor func(source fmt.Stringer, rule *rbacv1.PolicyRule, err error) bool) {
        	// 先校验clusterRoleBinding
        	if clusterRoleBindings, err := r.clusterRoleBindingLister.ListClusterRoleBindings(); err != nil {
        		if !visitor(nil, nil, err) {
        			return
        		}
        	} else {
        		sourceDescriber := &clusterRoleBindingDescriber{}
        		// 遍历 clusterRoleBindings
        		for _, clusterRoleBinding := range clusterRoleBindings {
        			// 根据传入的user对象对比subject主体  appliesTo是对比函数
        			subjectIndex, applies := appliesTo(user, clusterRoleBinding.Subjects, "")
        			if !applies {
        				continue
        			}
        			// 根据clusterRoleBinding.RoleRef从informer获取rules
        			rules, err := r.GetRoleReferenceRules(clusterRoleBinding.RoleRef, "")
        			if err != nil {
        				if !visitor(nil, nil, err) {
        					return
        				}
        				continue
        			}
        			// 遍历rules 传入到clusterRoleBinding，调用visit进行对比
        			sourceDescriber.binding = clusterRoleBinding
        			sourceDescriber.subject = &clusterRoleBinding.Subjects[subjectIndex]
        
        			for i := range rules {
        				if !visitor(sourceDescriber, &rules[i], nil) {
        					return
        				}
        			}
        		}
        	}
        	// roleBinding
        	if len(namespace) > 0 {
        		if roleBindings, err := r.roleBindingLister.ListRoleBindings(namespace); err != nil {
        			if !visitor(nil, nil, err) {
        				return
        			}
        		} else {
        			sourceDescriber := &roleBindingDescriber{}
        			for _, roleBinding := range roleBindings {
        				subjectIndex, applies := appliesTo(user, roleBinding.Subjects, namespace)
        				if !applies {
        					continue
        				}
        				rules, err := r.GetRoleReferenceRules(roleBinding.RoleRef, namespace)
        				if err != nil {
        					if !visitor(nil, nil, err) {
        						return
        					}
        					continue
        				}
        				sourceDescriber.binding = roleBinding
        				sourceDescriber.subject = &roleBinding.Subjects[subjectIndex]
        				for i := range rules {
        					if !visitor(sourceDescriber, &rules[i], nil) {
        						return
        					}
        				}
        			}
        		}
        	}
        }
        
        ```

        appliesTo方法

        ```go
        // appliesTo returns whether any of the bindingSubjects applies to the specified subject,
        // and if true, the index of the first subject that applies
        func appliesTo(user user.Info, bindingSubjects []rbacv1.Subject, namespace string) (int, bool) {
        	for i, bindingSubject := range bindingSubjects {
        		if appliesToUser(user, bindingSubject, namespace) {
        			return i, true
        		}
        	}
        	return 0, false
        }
        
        func has(set []string, ele string) bool {
        	for _, s := range set {
        		if s == ele {
        			return true
        		}
        	}
        	return false
        }
        
        func appliesToUser(user user.Info, subject rbacv1.Subject, namespace string) bool {
        	switch subject.Kind {
        	case rbacv1.UserKind:
        		return user.GetName() == subject.Name
        
        	case rbacv1.GroupKind:
        		return has(user.GetGroups(), subject.Name)
        
        	case rbacv1.ServiceAccountKind:
        		// default the namespace to namespace we're working in if its available.  This allows rolebindings that reference
        		// SAs in th local namespace to avoid having to qualify them.
        		saNamespace := namespace
        		if len(subject.Namespace) > 0 {
        			saNamespace = subject.Namespace
        		}
        		if len(saNamespace) == 0 {
        			return false
        		}
        		// use a more efficient comparison for RBAC checking
        		return serviceaccount.MatchesUsername(saNamespace, subject.Name, user.GetName())
        	default:
        		return false
        	}
        }
        ```

### audit审计功能

> k8s审计(Auditing)功能提供了与安全相关的、按时间顺序排序的记录集，记录每个用户、使用k8s API的应用以及控制面自身引发的活动
>
> 审计记录最初产生于 [kube-apiserver](https://kubernetes.io/zh-cn/docs/reference/command-line-tools-reference/kube-apiserver/) 内部。每个请求在不同执行阶段都会生成审计事件；这些审计事件会根据特定策略 被预处理并写入后端。策略确定要记录的内容和用来存储记录的后端。 当前的后端支持日志文件和 webhook。
>
> 每个请求都可被记录其相关的**阶段（stage）**。已定义的阶段有：
>
> - `RequestReceived` - 此阶段对应审计处理器接收到请求后，并且在委托给 其余处理器之前生成的事件。
> - `ResponseStarted` - 在响应消息的头部发送后，响应消息体发送前生成的事件。 只有长时间运行的请求（例如 watch）才会生成这个阶段。
> - `ResponseComplete` - 当响应消息体完成并且没有更多数据需要传输的时候。
> - `Panic` - 当 panic 发生时生成。
>
> 审计日志记录功能会增加 API server 的内存消耗，因为需要为每个请求存储审计所需的某些上下文。 内存消耗取决于审计日志记录的配置。

- 审计功能使得集群管理员能处理以下问题

  > 发生了什么
  >
  > 什么时候发生的
  >
  > 谁触发的
  >
  > 活动发生在那些对象上
  >
  > 在哪观察到的
  >
  > 从哪触发的
  >
  > 活动的后续处理行为是什么

- 审计策略

  > None：符合这条规则的日志将不会记录
  >
  > Metadata：记录请求的元数据(请求的用户、时间戳、资源、动词等)，但是不记录请求或者响应的消息体
  >
  > Request：记录事件的元数据和请求的消息体，但是不记录响应的消息体，这不适用于非资源类型的请求
  >
  > RequestResponse：记录事件的元数据，请求和响应的消息体，这不适用于非资源类型的请求
  >
  > 参考文档： https://kubernetes.io/zh-cn/docs/tasks/debug/debug-cluster/audit/

- 以下是一个审计策略文件的示例：

  ```yaml
  apiVersion: audit.k8s.io/v1 # 这是必填项。
  kind: Policy
  # 不要在 RequestReceived 阶段为任何请求生成审计事件。
  omitStages:
    - "RequestReceived"
  rules:
    # 在日志中用 RequestResponse 级别记录 Pod 变化。
    - level: RequestResponse
      resources:
      - group: ""
        # 资源 "pods" 不匹配对任何 Pod 子资源的请求，
        # 这与 RBAC 策略一致。
        resources: ["pods"]
    # 在日志中按 Metadata 级别记录 "pods/log"、"pods/status" 请求
    - level: Metadata
      resources:
      - group: ""
        resources: ["pods/log", "pods/status"]
  
    # 不要在日志中记录对名为 "controller-leader" 的 configmap 的请求。
    - level: None
      resources:
      - group: ""
        resources: ["configmaps"]
        resourceNames: ["controller-leader"]
    # 不要在日志中记录由 "system:kube-proxy" 发出的对端点或服务的监测请求。
    - level: None
      users: ["system:kube-proxy"]
      verbs: ["watch"]
      resources:
      - group: "" # core API 组
        resources: ["endpoints", "services"]
    # 不要在日志中记录对某些非资源 URL 路径的已认证请求。
    - level: None
      userGroups: ["system:authenticated"]
      nonResourceURLs:
      - "/api*" # 通配符匹配。
      - "/version"
    # 在日志中记录 kube-system 中 configmap 变更的请求消息体。
    - level: Request
      resources:
      - group: "" # core API 组
        resources: ["configmaps"]
      # 这个规则仅适用于 "kube-system" 名字空间中的资源。
      # 空字符串 "" 可用于选择非名字空间作用域的资源。
      namespaces: ["kube-system"]
    # 在日志中用 Metadata 级别记录所有其他名字空间中的 configmap 和 secret 变更。
    - level: Metadata
      resources:
      - group: "" # core API 组
        resources: ["secrets", "configmaps"]
    # 在日志中以 Request 级别记录所有其他 core 和 extensions 组中的资源操作。
    - level: Request
      resources:
      - group: "" # core API 组
      - group: "extensions" # 不应包括在内的组版本。
    # 一个抓取所有的规则，将在日志中以 Metadata 级别记录所有其他请求。
    - level: Metadata
      # 符合此规则的 watch 等长时间运行的请求将不会
      # 在 RequestReceived 阶段生成审计事件。
      omitStages:
        - "RequestReceived"
  ```

#### 代码解读

- s.Audit.ApplyTo

  `cmd\kube-apiserver\app\server.go`

  ```go
  	lastErr = s.Audit.ApplyTo(genericConfig)
  	if lastErr != nil {
  		return
  	}
  ```

  - ApplyTo方法

    ```go
    func (o *AuditOptions) ApplyTo(
    	c *server.Config,
    ) error {
    	if o == nil {
    		return nil
    	}
    	if c == nil {
    		return fmt.Errorf("server config must be non-nil")
    	}
    
    	// 1. Build policy checker
    	// 策略检查
    	checker, err := o.newPolicyChecker()
    	if err != nil {
    		return err
    	}
    
    	// 2. Build log backend
    	// 记录到log
    	var logBackend audit.Backend
    	// 检查是否打开了log 后端
    	if w := o.LogOptions.getWriter(); w != nil {
    		if checker == nil {
    			klog.V(2).Info("No audit policy file provided, no events will be recorded for log backend")
    		} else {
    			logBackend = o.LogOptions.newBackend(w)
    		}
    	}
    
    	// 3. Build webhook backend
    	// 记录到webhook
    	var webhookBackend audit.Backend
    	// 检查是否打开了webhook 后端
    	if o.WebhookOptions.enabled() {
    		if checker == nil {
    			klog.V(2).Info("No audit policy file provided, no events will be recorded for webhook backend")
    		} else {
    			if c.EgressSelector != nil {
    				var egressDialer utilnet.DialFunc
    				egressDialer, err = c.EgressSelector.Lookup(egressselector.ControlPlane.AsNetworkContext())
    				if err != nil {
    					return err
    				}
    				webhookBackend, err = o.WebhookOptions.newUntruncatedBackend(egressDialer)
    			} else {
    				webhookBackend, err = o.WebhookOptions.newUntruncatedBackend(nil)
    			}
    			if err != nil {
    				return err
    			}
    		}
    	}
    
    	groupVersion, err := schema.ParseGroupVersion(o.WebhookOptions.GroupVersionString)
    	if err != nil {
    		return err
    	}
    
    	// 4. Apply dynamic options.
    	var dynamicBackend audit.Backend
    	// 如果存在webhook，则封装位dynamicBackend
    	if webhookBackend != nil {
    		// if only webhook is enabled wrap it in the truncate options
    		dynamicBackend = o.WebhookOptions.TruncateOptions.wrapBackend(webhookBackend, groupVersion)
    	}
    
    	// 5. Set the policy checker
    	c.AuditPolicyChecker = checker
    
    	// 6. Join the log backend with the webhooks
    	// 把logBackend和dynamicBackend做union
    	c.AuditBackend = appendBackend(logBackend, dynamicBackend)
    
    	if c.AuditBackend != nil {
    		klog.V(2).Infof("Using audit backend: %s", c.AuditBackend)
    	}
    	return nil
    }
    
    func (o *AuditOptions) newPolicyChecker() (policy.Checker, error) {
    	// 策略文件不存在
    	if o.PolicyFile == "" {
    		return nil, nil
    	}
    	// 加载策略文件
    	p, err := policy.LoadPolicyFromFile(o.PolicyFile)
    	if err != nil {
    		return nil, fmt.Errorf("loading audit policy file: %v", err)
    	}
    	return policy.NewChecker(p), nil
    }
    ```

    - appendBackend做union

      ```go
      func appendBackend(existing, newBackend audit.Backend) audit.Backend {
      	if existing == nil {
      		return newBackend
      	}
      	if newBackend == nil {
      		return existing
      	}
      	return audit.Union(existing, newBackend)
      }
      
      ```

- 最终运行的方法

  appendBackend做union中udit.Union()跳转-->返回Backend跳转

  `vendor\k8s.io\apiserver\pkg\audit\types.go`

  ```go
  
  type Sink interface {
  	// ProcessEvents handles events. Per audit ID it might be that ProcessEvents is called up to three times.
  	// Errors might be logged by the sink itself. If an error should be fatal, leading to an internal
  	// error, ProcessEvents is supposed to panic. The event must not be mutated and is reused by the caller
  	// after the call returns, i.e. the sink has to make a deepcopy to keep a copy around if necessary.
  	// Returns true on success, may return false on error.
  	ProcessEvents(events ...*auditinternal.Event) bool
  }
  
  type Backend interface {
  	Sink
  
  	// Run will initialize the backend. It must not block, but may run go routines in the background. If
  	// stopCh is closed, it is supposed to stop them. Run will be called before the first call to ProcessEvents.
  	Run(stopCh <-chan struct{}) error
  
  	// Shutdown will synchronously shut down the backend while making sure that all pending
  	// events are delivered. It can be assumed that this method is called after
  	// the stopCh channel passed to the Run method has been closed.
  	Shutdown()
  
  	// Returns the backend PluginName.
  	String() string
  }
  
  ```

  - 最终调用audit的ProcessEvents方法

    以log举例

    ![image-20230531211336771](https://gitee.com/root_007/md_file_image/raw/master/202305312113858.png)

    ```go
    func (b *backend) ProcessEvents(events ...*auditinternal.Event) bool {
    	success := true
    	for _, ev := range events {
    		success = b.logEvent(ev) && success
    	}
    	return success
    }
    
    
    func (b *backend) logEvent(ev *auditinternal.Event) bool {
    	line := ""
    	switch b.format {
    	case FormatLegacy:
    		line = audit.EventString(ev) + "\n"
    	case FormatJson:
    		bs, err := runtime.Encode(b.encoder, ev)
    		if err != nil {
    			audit.HandlePluginError(PluginName, err, ev)
    			return false
    		}
    		line = string(bs[:])
    	default:
    		audit.HandlePluginError(PluginName, fmt.Errorf("log format %q is not in list of known formats (%s)",
    			b.format, strings.Join(AllowedFormats, ",")), ev)
    		return false
    	}
    	if _, err := fmt.Fprint(b.out, line); err != nil {
    		audit.HandlePluginError(PluginName, err, ev)
    		return false
    	}
    	return true
    }
    ```

- http侧调用的handler

  ```go
  // WithAudit decorates a http.Handler with audit logging information for all the
  // requests coming to the server. Audit level is decided according to requests'
  // attributes and audit policy. Logs are emitted to the audit sink to
  // process events. If sink or audit policy is nil, no decoration takes place.
  func WithAudit(handler http.Handler, sink audit.Sink, policy policy.Checker, longRunningCheck request.LongRunningRequestCheck) http.Handler {
  	if sink == nil || policy == nil {
  		return handler
  	}
  	return http.HandlerFunc(func(w http.ResponseWriter, req *http.Request) {
  		req, ev, omitStages, err := createAuditEventAndAttachToContext(req, policy)
  		if err != nil {
  			utilruntime.HandleError(fmt.Errorf("failed to create audit event: %v", err))
  			responsewriters.InternalError(w, req, errors.New("failed to create audit event"))
  			return
  		}
  		ctx := req.Context()
  		if ev == nil || ctx == nil {
  			handler.ServeHTTP(w, req)
  			return
  		}
  
  		ev.Stage = auditinternal.StageRequestReceived
  		if processed := processAuditEvent(ctx, sink, ev, omitStages); !processed {
  			audit.ApiserverAuditDroppedCounter.WithContext(ctx).Inc()
  			responsewriters.InternalError(w, req, errors.New("failed to store audit event"))
  			return
  		}
  
  		// intercept the status code
  		var longRunningSink audit.Sink
  		if longRunningCheck != nil {
  			ri, _ := request.RequestInfoFrom(ctx)
  			if longRunningCheck(req, ri) {
  				longRunningSink = sink
  			}
  		}
  		respWriter := decorateResponseWriter(ctx, w, ev, longRunningSink, omitStages)
  
  		// send audit event when we leave this func, either via a panic or cleanly. In the case of long
  		// running requests, this will be the second audit event.
  		defer func() {
  			if r := recover(); r != nil {
  				defer panic(r)
  				ev.Stage = auditinternal.StagePanic
  				ev.ResponseStatus = &metav1.Status{
  					Code:    http.StatusInternalServerError,
  					Status:  metav1.StatusFailure,
  					Reason:  metav1.StatusReasonInternalError,
  					Message: fmt.Sprintf("APIServer panic'd: %v", r),
  				}
  				processAuditEvent(ctx, sink, ev, omitStages)
  				return
  			}
  
  			// if no StageResponseStarted event was sent b/c neither a status code nor a body was sent, fake it here
  			// But Audit-Id http header will only be sent when http.ResponseWriter.WriteHeader is called.
  			fakedSuccessStatus := &metav1.Status{
  				Code:    http.StatusOK,
  				Status:  metav1.StatusSuccess,
  				Message: "Connection closed early",
  			}
  			if ev.ResponseStatus == nil && longRunningSink != nil {
  				ev.ResponseStatus = fakedSuccessStatus
  				ev.Stage = auditinternal.StageResponseStarted
  				processAuditEvent(ctx, longRunningSink, ev, omitStages)
  			}
  
  			ev.Stage = auditinternal.StageResponseComplete
  			if ev.ResponseStatus == nil {
  				ev.ResponseStatus = fakedSuccessStatus
  			}
  			processAuditEvent(ctx, sink, ev, omitStages)
  		}()
  		handler.ServeHTTP(respWriter, req)
  	})
  }
  ```

  > 封装一个http.Handler，所有的请求都要经过这里

### admission准入控制器功能

> 准入控制插件：
>
> - 准入控制器是一段代码，他会在请求通过认证和授权之后、对象被持久化之前拦截到达API服务器的请求
> - 准入控制器可以执行**验证（Validating）** 和/或**变更（Mutating）** 操作。 变更（mutating）控制器可以根据被其接受的请求更改相关对象；验证（validating）控制器则不行。
> - 准入控制过程分为两个阶段：**第一阶段**，运行变更准入控制器。**第二阶段**，运行验证准入控制器
> - 控制器需要编译进kube-apiserver二进制文件，并且只能由集群管理员配置
> - 如果任何一个阶段的任何控制器拒绝了该请求，则整个请求将立即被拒绝，并向终端用户返回一个错误
>
> 参考文档： https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/admission-controllers/

#### k8s API请求生命周期

![image-20230601100331069](https://gitee.com/root_007/md_file_image/raw/master/202306011003166.png)

#### 为什么需要准入控制器

> - k8s的许多高级功能都要求启用一个准入控制器，以便正确的支持该特性，因此没有正确配置准入控制器的k8s API服务是不完整的，它无法支持你期望的所有特性

#### 按照是否可以修改对象分类

> - 准入控制器可以执行**验证(Validating)**和**变更(Mutating)**操作
> - 变更(mutating)控制器可以修改被其接受的对象，验证(validating)控制器则不行

#### 按照静动态分类

> - 静态的就是固定的单一功能，如：AlwaysPullImages修改每一个新创建的pod的镜像拉取策略为Always
> - 动态的如有两个特殊的控制器：MutatingAdmissionWebhook和ValidatingAdmissionWebhook
>   - 他们根据API中的配置，分别执行变更和验证准入控制webhook
>   - 相当于可以调用外部的http请求，准入控制插件

#### 代码解读

- 入口

  `cmd\kube-apiserver\app\server.go` 其是由`cmd\kube-apiserver\apiserver.go`中`NewAPIServerCommand`跳转到server.go，位置：buildGenericConfig中

  ```go
  	// 准入控制器
  	admissionConfig := &kubeapiserveradmission.Config{
  		ExternalInformers:    versionedInformers,
  		LoopbackClientConfig: genericConfig.LoopbackClientConfig,
  		CloudConfigFile:      s.CloudProvider.CloudConfigFile,
  	}
  	serviceResolver = buildServiceResolver(s.EnableAggregatorRouting, genericConfig.LoopbackClientConfig.Host, versionedInformers)
  // 初始化准入控制器
  	pluginInitializers, admissionPostStartHook, err = admissionConfig.New(proxyTransport, genericConfig.EgressSelector, serviceResolver)
  	if err != nil {
  		lastErr = fmt.Errorf("failed to create admission plugin initializer: %v", err)
  		return
  	}
  
  	err = s.Admission.ApplyTo(
  		genericConfig,
  		versionedInformers,
  		kubeClientConfig,
  		feature.DefaultFeatureGate,
  		pluginInitializers...)
  	if err != nil {
  		lastErr = fmt.Errorf("failed to initialize admission: %v", err)
  	}
  ```

  - admissionConfig.New()初始化准入控制器

    ```go
    // 设置准入控制器的插件和webhook准入的启动钩子函数
    
    func (c *Config) New(proxyTransport *http.Transport, egressSelector *egressselector.EgressSelector, serviceResolver webhook.ServiceResolver) ([]admission.PluginInitializer, genericapiserver.PostStartHookFunc, error) {
    	webhookAuthResolverWrapper := webhook.NewDefaultAuthenticationInfoResolverWrapper(proxyTransport, egressSelector, c.LoopbackClientConfig)
    	webhookPluginInitializer := webhookinit.NewPluginInitializer(webhookAuthResolverWrapper, serviceResolver)
    
    	var cloudConfig []byte
    	if c.CloudConfigFile != "" {
    		var err error
    		cloudConfig, err = ioutil.ReadFile(c.CloudConfigFile)
    		if err != nil {
    			klog.Fatalf("Error reading from cloud configuration file %s: %#v", c.CloudConfigFile, err)
    		}
    	}
    	clientset, err := kubernetes.NewForConfig(c.LoopbackClientConfig)
    	if err != nil {
    		return nil, nil, err
    	}
    
    	discoveryClient := cacheddiscovery.NewMemCacheClient(clientset.Discovery())
    	discoveryRESTMapper := restmapper.NewDeferredDiscoveryRESTMapper(discoveryClient)
    	kubePluginInitializer := NewPluginInitializer(
    		cloudConfig,
    		discoveryRESTMapper,
    		quotainstall.NewQuotaConfigurationForAdmission(),
    	)
    	// 生成一个webhook启动的钩子，每隔30s重置一下discoveryRESTMapper，重置内部缓存Discovery
    	admissionPostStartHook := func(context genericapiserver.PostStartHookContext) error {
    		discoveryRESTMapper.Reset()
    		go utilwait.Until(discoveryRESTMapper.Reset, 30*time.Second, context.StopCh)
    		return nil
    	}
    
    	return []admission.PluginInitializer{webhookPluginInitializer, kubePluginInitializer}, admissionPostStartHook, nil
    }
    ```

  - s.Admission.ApplyTo

    ```go
    func (a *AdmissionOptions) ApplyTo(
    	c *server.Config,
    	informers informers.SharedInformerFactory,
    	kubeAPIServerClientConfig *rest.Config,
    	features featuregate.FeatureGate,
    	pluginInitializers ...admission.PluginInitializer,
    ) error {
    	if a == nil {
    		return nil
    	}
    	// 准入控制器插件列表
    	if a.PluginNames != nil {
    		// pass PluginNames to generic AdmissionOptions
    		// 根据传入的控制器列表和推荐的计算开启和关闭的准入控制器插件
    		a.GenericAdmission.EnablePlugins, a.GenericAdmission.DisablePlugins = computePluginNames(a.PluginNames, a.GenericAdmission.RecommendedPluginOrder)
    	}
    
    	return a.GenericAdmission.ApplyTo(c, informers, kubeAPIServerClientConfig, features, pluginInitializers...)
    }
    ```

    > PluginNames代表 --admission-control传入的，其中`a.GenericAdmission.RecommendedPluginOrder`代表k8s官方所有，以下是所有k8s官方k8s准入控制器
    >
    > 位置： `pkg\kubeapiserver\options\plugins.go`
    >
    > ```go
    > // AllOrderedPlugins is the list of all the plugins in order.
    > var AllOrderedPlugins = []string{
    > 	admit.PluginName,                        // AlwaysAdmit
    > 	autoprovision.PluginName,                // NamespaceAutoProvision
    > 	lifecycle.PluginName,                    // NamespaceLifecycle
    > 	exists.PluginName,                       // NamespaceExists
    > 	scdeny.PluginName,                       // SecurityContextDeny
    > 	antiaffinity.PluginName,                 // LimitPodHardAntiAffinityTopology
    > 	limitranger.PluginName,                  // LimitRanger
    > 	serviceaccount.PluginName,               // ServiceAccount
    > 	noderestriction.PluginName,              // NodeRestriction
    > 	nodetaint.PluginName,                    // TaintNodesByCondition
    > 	alwayspullimages.PluginName,             // AlwaysPullImages
    > 	imagepolicy.PluginName,                  // ImagePolicyWebhook
    > 	podsecuritypolicy.PluginName,            // PodSecurityPolicy
    > 	podnodeselector.PluginName,              // PodNodeSelector
    > 	podpriority.PluginName,                  // Priority
    > 	defaulttolerationseconds.PluginName,     // DefaultTolerationSeconds
    > 	podtolerationrestriction.PluginName,     // PodTolerationRestriction
    > 	eventratelimit.PluginName,               // EventRateLimit
    > 	extendedresourcetoleration.PluginName,   // ExtendedResourceToleration
    > 	label.PluginName,                        // PersistentVolumeLabel
    > 	setdefault.PluginName,                   // DefaultStorageClass
    > 	storageobjectinuseprotection.PluginName, // StorageObjectInUseProtection
    > 	gc.PluginName,                           // OwnerReferencesPermissionEnforcement
    > 	resize.PluginName,                       // PersistentVolumeClaimResize
    > 	runtimeclass.PluginName,                 // RuntimeClass
    > 	certapproval.PluginName,                 // CertificateApproval
    > 	certsigning.PluginName,                  // CertificateSigning
    > 	certsubjectrestriction.PluginName,       // CertificateSubjectRestriction
    > 	defaultingressclass.PluginName,          // DefaultIngressClass
    > 	denyserviceexternalips.PluginName,       // DenyServiceExternalIPs
    > 
    > 	// new admission plugins should generally be inserted above here
    > 	// webhook, resourcequota, and deny plugins must go at the end
    > 
    > 	mutatingwebhook.PluginName,   // MutatingAdmissionWebhook
    > 	validatingwebhook.PluginName, // ValidatingAdmissionWebhook
    > 	resourcequota.PluginName,     // ResourceQuota
    > 	deny.PluginName,              // AlwaysDeny
    > }
    > ```
    >
    > 

    - a.GenericAdmission.ApplyTo

      ```go
      // ApplyTo adds the admission chain to the server configuration.
      // In case admission plugin names were not provided by a cluster-admin they will be prepared from the recommended/default values.
      // In addition the method lazily initializes a generic plugin that is appended to the list of pluginInitializers
      // note this method uses:
      //  genericconfig.Authorizer
      func (a *AdmissionOptions) ApplyTo(
      	c *server.Config,
      	informers informers.SharedInformerFactory,
      	kubeAPIServerClientConfig *rest.Config,
      	features featuregate.FeatureGate,
      	pluginInitializers ...admission.PluginInitializer,
      ) error {
      	if a == nil {
      		return nil
      	}
      
      	// Admission depends on CoreAPI to set SharedInformerFactory and ClientConfig.
      	if informers == nil {
      		return fmt.Errorf("admission depends on a Kubernetes core API shared informer, it cannot be nil")
      	}
      	// 根据传入关闭的、传入开启的、推荐的等插件列表计算真正要开启的列表
      	pluginNames := a.enabledPluginNames()
      	//根据配置文件读取配置 admission-control-config-file
      	pluginsConfigProvider, err := admission.ReadAdmissionConfiguration(pluginNames, a.ConfigFile, configScheme)
      	if err != nil {
      		return fmt.Errorf("failed to read plugin config: %v", err)
      	}
      	// 初始化genericInitializer
      	clientset, err := kubernetes.NewForConfig(kubeAPIServerClientConfig)
      	if err != nil {
      		return err
      	}
      	genericInitializer := initializer.New(clientset, informers, c.Authorization.Authorizer, features)
      	initializersChain := admission.PluginInitializers{}
      	pluginInitializers = append(pluginInitializers, genericInitializer)
      	initializersChain = append(initializersChain, pluginInitializers...)
      
      	admissionChain, err := a.Plugins.NewFromPlugins(pluginNames, pluginsConfigProvider, initializersChain, a.Decorators)
      	if err != nil {
      		return err
      	}
      
      	c.AdmissionControl = admissionmetrics.WithStepMetrics(admissionChain)
      	return nil
      }
      
      // 根据传入关闭的、传入开启的、推荐的等插件列表计算真正要开启的列表
      func (a *AdmissionOptions) enabledPluginNames() []string {
      	allOffPlugins := append(a.DefaultOffPlugins.List(), a.DisablePlugins...)
      	disabledPlugins := sets.NewString(allOffPlugins...)
      	enabledPlugins := sets.NewString(a.EnablePlugins...)
      	disabledPlugins = disabledPlugins.Difference(enabledPlugins)
      
      	orderedPlugins := []string{}
      	for _, plugin := range a.RecommendedPluginOrder {
      		if !disabledPlugins.Has(plugin) {
      			orderedPlugins = append(orderedPlugins, plugin)
      		}
      	}
      
      	return orderedPlugins
      }
      
      ```

      - a.Plugins.NewFromPlugins

        ```go
        // 如果不存在插件 则创建
        func (ps *Plugins) NewFromPlugins(pluginNames []string, configProvider ConfigProvider, pluginInitializer PluginInitializer, decorator Decorator) (Interface, error) {
        	handlers := []Interface{}
        	mutationPlugins := []string{}
        	validationPlugins := []string{}
        	// 遍历所有准入控制插件
        	for _, pluginName := range pluginNames {
        		pluginConfig, err := configProvider.ConfigFor(pluginName)
        		if err != nil {
        			return nil, err
        		}
        
        		plugin, err := ps.InitPlugin(pluginName, pluginConfig, pluginInitializer)
        		if err != nil {
        			return nil, err
        		}
        		if plugin != nil {
        			if decorator != nil {
        				handlers = append(handlers, decorator.Decorate(plugin, pluginName))
        			} else {
        				handlers = append(handlers, plugin)
        			}
        
        			if _, ok := plugin.(MutationInterface); ok {
        				mutationPlugins = append(mutationPlugins, pluginName)
        			}
        			if _, ok := plugin.(ValidationInterface); ok {
        				validationPlugins = append(validationPlugins, pluginName)
        			}
        		}
        	}
        	if len(mutationPlugins) != 0 {
        		klog.Infof("Loaded %d mutating admission controller(s) successfully in the following order: %s.", len(mutationPlugins), strings.Join(mutationPlugins, ","))
        	}
        	if len(validationPlugins) != 0 {
        		klog.Infof("Loaded %d validating admission controller(s) successfully in the following order: %s.", len(validationPlugins), strings.Join(validationPlugins, ","))
        	}
        	return newReinvocationHandler(chainAdmissionHandler(handlers)), nil
        }
        ```

        ps.InitPlugin

        ```go
        func (ps *Plugins) InitPlugin(name string, config io.Reader, pluginInitializer PluginInitializer) (Interface, error) {
        	if name == "" {
        		klog.Info("No admission plugin specified.")
        		return nil, nil
        	}
        	//调用getPlugin从Plugins获取plugin实例
        	plugin, found, err := ps.getPlugin(name, config)
        	if err != nil {
        		return nil, fmt.Errorf("couldn't init admission plugin %q: %v", name, err)
        	}
        	if !found {
        		return nil, fmt.Errorf("unknown admission plugin: %s", name)
        	}
        
        	pluginInitializer.Initialize(plugin)
        	// ensure that plugins have been properly initialized
        	if err := ValidateInitialization(plugin); err != nil {
        		return nil, fmt.Errorf("failed to initialize admission plugin %q: %v", name, err)
        	}
        
        	return plugin, nil
        }
        ```

        ps.getPlugin

        ```go
        func (ps *Plugins) getPlugin(name string, config io.Reader) (Interface, bool, error) {
        	ps.lock.Lock()
        	defer ps.lock.Unlock()
        	f, found := ps.registry[name]
        	if !found {
        		return nil, false, nil
        	}
        
        	config1, config2, err := splitStream(config)
        	if err != nil {
        		return nil, true, err
        	}
        	if !PluginEnabledFn(name, config1) {
        		return nil, true, nil
        	}
        
        	ret, err := f(config2)
        	return ret, true, err
        }
        ```

- 以alwayspullimages.Register(plugins)为例

  `pkg\kubeapiserver\options\plugins.go`

  ```go
  func RegisterAllAdmissionPlugins(plugins *admission.Plugins) {
  	admit.Register(plugins) // DEPRECATED as no real meaning
  	alwayspullimages.Register(plugins)
  	antiaffinity.Register(plugins)
  	defaulttolerationseconds.Register(plugins)
  	defaultingressclass.Register(plugins)
  	denyserviceexternalips.Register(plugins)
  	deny.Register(plugins) // DEPRECATED as no real meaning
  	eventratelimit.Register(plugins)
  	extendedresourcetoleration.Register(plugins)
  	gc.Register(plugins)
  	imagepolicy.Register(plugins)
  	limitranger.Register(plugins)
  	autoprovision.Register(plugins)
  	exists.Register(plugins)
  	noderestriction.Register(plugins)
  	nodetaint.Register(plugins)
  	label.Register(plugins) // DEPRECATED, future PVs should not rely on labels for zone topology
  	podnodeselector.Register(plugins)
  	podtolerationrestriction.Register(plugins)
  	runtimeclass.Register(plugins)
  	resourcequota.Register(plugins)
  	podsecuritypolicy.Register(plugins)
  	podpriority.Register(plugins)
  	scdeny.Register(plugins)
  	serviceaccount.Register(plugins)
  	setdefault.Register(plugins)
  	resize.Register(plugins)
  	storageobjectinuseprotection.Register(plugins)
  	certapproval.Register(plugins)
  	certsigning.Register(plugins)
  	certsubjectrestriction.Register(plugins)
  }
  ```

  - alwayspullimages.Register

    ```go
    // PluginName indicates name of admission plugin.
    const PluginName = "AlwaysPullImages"
    
    // Register registers a plugin
    // 工厂行数
    func Register(plugins *admission.Plugins) {
    	plugins.Register(PluginName, func(config io.Reader) (admission.Interface, error) {
    		// 返回一个AlwaysPullImages的初始化
    		return NewAlwaysPullImages(), nil
    	})
    }
    ```

    - NewAlwaysPullImages

      初始化一个AlwaysPullImages

      ```go
      func NewAlwaysPullImages() *AlwaysPullImages {
      	return &AlwaysPullImages{
      		Handler: admission.NewHandler(admission.Create, admission.Update),
      	}
      }
      ```

  - Admin

    ```go
    // Admit makes an admission decision based on the request attributes
    func (a *AlwaysPullImages) Admit(ctx context.Context, attributes admission.Attributes, o admission.ObjectInterfaces) (err error) {
    	// Ignore all calls to subresources or resources other than pods.
    	if shouldIgnore(attributes) {
    		return nil
    	}
    	// 获取pod资源
    	pod, ok := attributes.GetObject().(*api.Pod)
    	if !ok {
    		return apierrors.NewBadRequest("Resource was marked with kind Pod but was unable to be converted")
    	}
    	// 遍历pod中的字段，将其ImagePullPolicy改为Always
    	pods.VisitContainersWithPath(&pod.Spec, field.NewPath("spec"), func(c *api.Container, _ *field.Path) bool {
    		c.ImagePullPolicy = api.PullAlways
    		return true
    	})
    
    	return nil
    }
    ```

    > 1. 该准入控制器会修改每个新创建的 Pod，将其镜像拉取策略设置为 `Always`。
    > 2. 这在多租户集群中是有用的，这样用户就可以放心，他们的私有镜像只能被那些有凭证的人使用。
    > 3. 如果没有这个准入控制器，一旦镜像被拉取到节点上，任何用户的 Pod 都可以通过已了解到的镜像的名称 （假设 Pod 被调度到正确的节点上）来使用它，而不需要对镜像进行任何鉴权检查。
    > 4. 启用这个准入控制器之后，启动容器之前必须拉取镜像，这意味着需要有效的凭证

  - 校验方法

    ```go
    // Validate makes sure that all containers are set to always pull images
    // 校验方法
    func (*AlwaysPullImages) Validate(ctx context.Context, attributes admission.Attributes, o admission.ObjectInterfaces) (err error) {
    	if shouldIgnore(attributes) {
    		return nil
    	}
    
    	pod, ok := attributes.GetObject().(*api.Pod)
    	if !ok {
    		return apierrors.NewBadRequest("Resource was marked with kind Pod but was unable to be converted")
    	}
    
    	var allErrs []error
    	pods.VisitContainersWithPath(&pod.Spec, field.NewPath("spec"), func(c *api.Container, p *field.Path) bool {
    		if c.ImagePullPolicy != api.PullAlways {
    			allErrs = append(allErrs, admission.NewForbidden(attributes,
    				field.NotSupported(p.Child("imagePullPolicy"), c.ImagePullPolicy, []string{string(api.PullAlways)}),
    			))
    		}
    		return true
    	})
    	if len(allErrs) > 0 {
    		return utilerrors.NewAggregate(allErrs)
    	}
    
    	return nil
    }
    ```

    