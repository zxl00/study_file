## API核心server启动流程分析

> - 通用的GenericApiServerNew函数
> - apiserver核心服务的初始化
> - 最终的apiserver启动流程

### 通用的GenericApiServerNew函数

> - 之前分析了使用buildGenericConfig构建api核心服务的配置
>
> - 然后回到CerateServerChain函数中`cmd\kube-apiserver\app\server.go`，可以根据apiserver.go函数跳转NewAPIServerCommand-->return Run(completedOptions, genericapiserver.SetupSignalHandler())-->CreateServerChain
>
>   - 发现会调用三个server的create函数，传入对应的配置初始化
>
>     - apiExtensionsServer  API扩展，主要是针对CRD
>     - kubeAPIServer  API核心服务，包括常见的pod、deployment、service
>     - aggregatorConfig  API聚合服务，主要针对metrics
>
>   - 代码如下：
>
>     ```go
>     	// API扩展，主要是针对CRD
>     	apiExtensionsServer, err := createAPIExtensionsServer(apiExtensionsConfig, genericapiserver.NewEmptyDelegate())
>     		// API核心服务，包括常见的pod、deployment、service
>     	kubeAPIServer, err := CreateKubeAPIServer(kubeAPIServerConfig, apiExtensionsServer.GenericAPIServer)
>     		// API聚合服务，主要针对metrics
>     	aggregatorConfig, err := createAggregatorConfig(*kubeAPIServerConfig.GenericConfig, completedOptions.ServerRunOptions, kubeAPIServerConfig.ExtraConfig.VersionedInformers, serviceResolver, proxyTransport, pluginInitializer)
>     	    
>     ```

- CreateServerChain中的三个server的create都会调用completedConfig的New

  kubeAPIServer  

  ```go
  
  kubeAPIServer, err := CreateKubeAPIServer(kubeAPIServerConfig, apiExtensionsServer.GenericAPIServer)
  ```

  - CreateKubeAPIServer

    ```go
    func CreateKubeAPIServer(kubeAPIServerConfig *controlplane.Config, delegateAPIServer genericapiserver.DelegationTarget) (*controlplane.Instance, error) {
    	kubeAPIServer, err := kubeAPIServerConfig.Complete().New(delegateAPIServer)
    	if err != nil {
    		return nil, err
    	}
    
    	return kubeAPIServer, nil
    }
    ```

    - Complete().New(delegateAPIServer)

      ```go
      func (c completedConfig) New(delegationTarget genericapiserver.DelegationTarget) (*Instance, error) {
      	if reflect.DeepEqual(c.ExtraConfig.KubeletClientConfig, kubeletclient.KubeletClientConfig{}) {
      		return nil, fmt.Errorf("Master.New() called with empty config.KubeletClientConfig")
      	}
      	// 初始化通用server
      	s, err := c.GenericConfig.New("kube-apiserver", delegationTarget)
      ```
      
      - c.GenericConfig.New("kube-apiserver", delegationTarget)
      
        ```go
        func (c completedConfig) New(name string, delegationTarget DelegationTarget) (*GenericAPIServer, error) {
        ''''''
        	// 添加钩子函数
        	for k, v := range delegationTarget.PostStartHooks() {
        		s.postStartHooks[k] = v
        	}
        
        	for k, v := range delegationTarget.PreShutdownHooks() {
        		s.preShutdownHooks[k] = v
        	}
        	
        		// 添加健康检查
        	for _, delegateCheck := range delegationTarget.HealthzChecks() {
        		skip := false
        		for _, existingCheck := range c.HealthzChecks {
        			if existingCheck.Name() == delegateCheck.Name() {
        				skip = true
        				break
        			}
        		}
        		if skip {
        			continue
        		}
        		s.AddHealthChecks(delegateCheck)
        	}
        		// 初始化api路由的installAPI
        	installAPI(s, c.Config)
        ''''''
        }
        ```
      
      - installAPI(s,c.Config)
      
        ```go
        func installAPI(s *GenericAPIServer, c *Config) {
        	if c.EnableIndex {
        		routes.Index{}.Install(s.listedPathProvider, s.Handler.NonGoRestfulMux)
        	}
        	// pprof
        	if c.EnableProfiling {
        		routes.Profiling{}.Install(s.Handler.NonGoRestfulMux)
        		if c.EnableContentionProfiling {
        			goruntime.SetBlockProfileRate(1)
        		}
        		// so far, only logging related endpoints are considered valid to add for these debug flags.
        		routes.DebugFlags{}.Install(s.Handler.NonGoRestfulMux, "v", routes.StringFlagPutHandler(logs.GlogSetter))
        	}
        	// metrics
        	if c.EnableMetrics {
        		if c.EnableProfiling {
        			routes.MetricsWithReset{}.Install(s.Handler.NonGoRestfulMux)
        		} else {
        			routes.DefaultMetrics{}.Install(s.Handler.NonGoRestfulMux)
        		}
        	}
        	// 版本信息
        	routes.Version{Version: c.Version}.Install(s.Handler.GoRestfulContainer)
        	// 服务发现
        	if c.EnableDiscovery {
        		s.Handler.GoRestfulContainer.Add(s.DiscoveryGroupManager.WebService())
        	}
        	if c.FlowControl != nil && feature.DefaultFeatureGate.Enabled(features.APIPriorityAndFairness) {
        		c.FlowControl.Install(s.Handler.NonGoRestfulMux)
        	}
        }
        ```
      
        

  apiExtensionsServer  

  ```go
  genericServer, err := c.GenericConfig.New("apiextensions-apiserver", delegationTarget)
  ```

  aggregatorConfig  

  ```go
  genericServer, err := c.GenericConfig.New("kube-aggregator", delegationTarget)
  ```

- completedConfig的New生成通用的GenericAPIServer

  `vendor\k8s.io\apiserver\pkg\server\config.go`

  ```
  
  ```

  