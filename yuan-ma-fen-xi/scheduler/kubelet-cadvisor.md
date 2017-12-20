# kubelet cadvisor

```
作者 张杰  j.zhang8@haihangyun.com
```

---



cAdvisor是谷歌开源的一个容器监控工具，该工具提供了webUI和REST API两种方式来展示数据，从而可以帮助管理者了解主机以及容器的资源使用情况和性能数据。

cAdvisor集成到了kubelet组件内，因此可以在kube集群中每个启动了kubelet的节点使用cAdvisor来查看该节点的运行数据



## **源码分析**

cmd/kubelet/app/server.go run() 方法里，有对cadivisor 的初始化

``` golang
	if kubeDeps.CAdvisorInterface == nil {
		imageFsInfoProvider := cadvisor.NewImageFsInfoProvider(s.ContainerRuntime, s.RemoteRuntimeEndpoint)
		// RootDirectory  : Directory path for managing kubelet files (volume mounts,etc). (default "/var/lib/kubelet")
		kubeDeps.CAdvisorInterface, err = cadvisor.New(s.Address, uint(s.CAdvisorPort), imageFsInfoProvider, s.RootDirectory, cadvisor.UsingLegacyCadvisorStats(s.ContainerRuntime, s.RemoteRuntimeEndpoint))
		if err != nil {
			return err
		}
	}

```
cadvisor.New 方法是在 pkg/kubelet/cadvisor/cadvisor_linux.go (如果是linux 环境， 在这里 k8s 用了 go语言的build 约束 方式，在文件的开头有：// +build cgo,linux 方式。 支持不同平台的build )

``` golang
func New(address string, port uint, imageFsInfoProvider ImageFsInfoProvider, rootPath string, usingLegacyStats bool) (Interface, error) {
	sysFs := sysfs.NewRealSysFs()

...
	// Create and start the cAdvisor container manager.
	m, err := manager.New(memory.New(statsCacheDuration, nil), sysFs, maxHousekeepingInterval, allowDynamicHousekeeping, ignoreMetrics, http.DefaultClient)
	if err != nil {
		return nil, err
	}

...

	err = cadvisorClient.exportHTTP(address, port)
	if err != nil {
		return nil, err
	}
	return cadvisorClient, nil
}

```
New 方法主要有两个核心方法，manager.New 和 cadvisorClient.exportHTTP ，分别是创建cadvisor 的manager 和 httpserver

manager.New () 方法在 vendor/github.com/google/cadvisor/manager/manager.go 定义
主要是生成一个manage 对象。
``` golang
	newManager := &manager{
		containers:               make(map[namespacedContainerName]*containerData),
		quitChannels:             make([]chan error, 0, 2),
		memoryCache:              memoryCache,
		fsInfo:                   fsInfo,
		cadvisorContainer:        selfContainer,
		inHostNamespace:          inHostNamespace,
		startupTime:              time.Now(),
		maxHousekeepingInterval:  maxHousekeepingInterval,
		allowDynamicHousekeeping: allowDynamicHousekeeping,
		ignoreMetrics:            ignoreMetricsSet,
		containerWatchers:        []watcher.ContainerWatcher{},
		eventsChannel:            eventsChannel,
		collectorHttpClient:      collectorHttpClient,
		nvidiaManager:            &accelerators.NvidiaManager{},
	}

```
其中containers 是本机container 的缓存，起数据来源是manger 的start 方法去更新的，后续我们会讲。
cadvisorClient.exportHTTP() 方法是起httpserver ，里面的核心方法是 cadvisorhttp.RegisterHandlers（）
用来注册不同的url 和function
``` golang
func (cc *cadvisorClient) exportHTTP(address string, port uint) error {
	// Register the handlers regardless as this registers the prometheus
	// collector properly.
	mux := http.NewServeMux()
        // 核心方法
	err := cadvisorhttp.RegisterHandlers(mux, cc, "", "", "", "")
	if err != nil {
		return err
	}

	cadvisorhttp.RegisterPrometheusHandler(mux, cc, "/metrics", containerLabels)

	// Only start the http server if port > 0
	if port > 0 {
		serv := &http.Server{
			Addr:    net.JoinHostPort(address, strconv.Itoa(int(port))),
			Handler: mux,
		}

		// TODO(vmarmol): Remove this when the cAdvisor port is once again free.
		// If export failed, retry in the background until we are able to bind.
		// This allows an existing cAdvisor to be killed before this one registers.
		go func() {
			defer runtime.HandleCrash()

			err := serv.ListenAndServe()
			for err != nil {
				glog.Infof("Failed to register cAdvisor on port %d, retrying. Error: %v", port, err)
				time.Sleep(time.Minute)
				err = serv.ListenAndServe()
			}
		}()
	}

	return nil
}

```

vendor/github.com/google/cadvisor/http/handlers.go
cadvisorhttp.RegisterHandlers()

``` golang
func RegisterHandlers(mux httpmux.Mux, containerManager manager.Manager, httpAuthFile, httpAuthRealm, httpDigestFile, httpDigestRealm string) error {
	// Basic health handler.
	// 注册healthz handler
	if err := healthz.RegisterHandler(mux); err != nil {
		return fmt.Errorf("failed to register healthz handler: %s", err)
	}

	// Validation/Debug handler.
	// 注册 /validate/ url 的方法
	mux.HandleFunc(validate.ValidatePage, func(w http.ResponseWriter, r *http.Request) {
		err := validate.HandleRequest(w, containerManager)
		if err != nil {
			http.Error(w, err.Error(), http.StatusInternalServerError)
		}
	})

	// Register API handler.
	// 注册API的方法  /api/ 
	if err := api.RegisterHandlers(mux, containerManager); err != nil {
		return fmt.Errorf("failed to register API handlers: %s", err)
	}

	// Redirect / to containers page.
	mux.Handle("/", http.RedirectHandler(pages.ContainersPage, http.StatusTemporaryRedirect))

	var authenticated bool

...
	return nil
}
```
以 api.RegisterHandlers(） 方法为例 /vendor/github.com/google/cadvisor/api/handler.go
最终 调用 handleRequest（） 方法
``` golang 
func RegisterHandlers(mux httpmux.Mux, m manager.Manager) error {
	apiVersions := getApiVersions()
	// 支持不同的version
	supportedApiVersions := make(map[string]ApiVersion, len(apiVersions))
	for _, v := range apiVersions {
		supportedApiVersions[v.Version()] = v
	}

	mux.HandleFunc(apiResource, func(w http.ResponseWriter, r *http.Request) {
		err := handleRequest(supportedApiVersions, m, w, r)
		if err != nil {
			http.Error(w, err.Error(), 500)
		}
	})
	return nil
}
```
handlerRequest() 方法
``` golang 
func handleRequest(supportedApiVersions map[string]ApiVersion, m manager.Manager, w http.ResponseWriter, r *http.Request) error {
	start := time.Now()
...

	const apiPrefix = "/api"
	if !strings.HasPrefix(request, apiPrefix) {
		return fmt.Errorf("incomplete API request %q", request)
	}
...

	version := requestElements[apiVersion]
	requestType := requestElements[apiRequestType]
	requestArgs := strings.Split(requestElements[apiRequestArgs], "/")

...
	return versionHandler.HandleRequest(requestType, requestArgs, m, w, r)

}

```
最终调用 versionHandler.HandleRequest（） 我们以version1_1 和 containers types 为例

vendor/github.com/google/cadvisor/api/versions.go
``` golang
func (self *version1_1) HandleRequest(requestType string, request []string, m manager.Manager, w http.ResponseWriter, r *http.Request) error {
	switch requestType {
	case subcontainersApi:
		containerName := getContainerName(request)
		glog.V(4).Infof("Api - Subcontainers(%s)", containerName)

		// Get the query request.
		query, err := getContainerInfoRequest(r.Body)
		if err != nil {
			return err
		}

		// Get the subcontainers.
		containers, err := m.SubcontainersInfo(containerName, query)
		if err != nil {
			return fmt.Errorf("failed to get subcontainers for container %q with error: %s", containerName, err)
		}

		// Only output the containers as JSON.
		err = writeResult(containers, w)
		if err != nil {
			return err
		}
		return nil
	default:
		return self.baseVersion.HandleRequest(requestType, request, m, w, r)
	}
}

```
如果RequestType 是container, 转到 default 的 self.baseVersion.HandleRequest 方法
``` golang 
func (self *version1_0) HandleRequest(requestType string, request []string, m manager.Manager, w http.ResponseWriter, r *http.Request) error {
	switch requestType {
...
	case containersApi:
		containerName := getContainerName(request)
		glog.V(4).Infof("Api - Container(%s)", containerName)

		// Get the query request.
		query, err := getContainerInfoRequest(r.Body)
		if err != nil {
			return err
		}

		// Get the container.
		cont, err := m.GetContainerInfo(containerName, query)
		if err != nil {
			return fmt.Errorf("failed to get container %q with error: %s", containerName, err)
		}

		// Only output the container as JSON.
		err = writeResult(cont, w)
		if err != nil {
			return err
		}
...
	}
	return nil
}
```
最终会调用 m.GetContainerInfo（） 方法，我们一步不追：
vendor/github.com/google/cadvisor/manager/manager.go
``` golang 
func (self *manager) GetContainerInfo(containerName string, query *info.ContainerInfoRequest) (*info.ContainerInfo, error) {
	cont, err := self.getContainerData(containerName)
	if err != nil {
		return nil, err
	}
	return self.containerDataToContainerInfo(cont, query)
}

```
核心方法： getContainerData（） 这个方法的核心逻辑就是从 containers map 里拿到对应container 的数据，返回，然后让http w 写会，response 给client 端。 这样就实现了 api:  curl http://kubeletip:4194/api/v1.1/containers  的api 请求的逻辑
``` golang
func (self *manager) getContainerData(containerName string) (*containerData, error) {
	var cont *containerData
	var ok bool
	func() {
		self.containersLock.RLock()
		defer self.containersLock.RUnlock()

		// Ensure we have the container.
		cont, ok = self.containers[namespacedContainerName{
			Name: containerName,
		}]
	}()
	if !ok {
		return nil, fmt.Errorf("unknown container %q", containerName)
	}
	return cont, nil
}

```
那么上文也说了 ，containers 数据是哪里来的

代码追踪：
pkg/kubelet/kubelet.go NewMainKubelet（） 方法,构造kubelet  对象时候 ,会将 kubeDeps.CAdvisorInterface 赋值给Kubelet 的 cadvisor 变量
``` golang 
klet := &Kubelet{
		hostname:        hostname,
...
		cadvisor:                       kubeDeps.CAdvisorInterface,
...
		keepTerminatedPodVolumes:                keepTerminatedPodVolumes,
	}
```
在 最终的kubelet的 Run 方法里会调用  updateRuntimeUp（） -> initializeRuntimeDependentModules() -> kl.cadvisor.Start()
``` golang 
// Run starts the kubelet reacting to config updates
func (kl *Kubelet) Run(updates <-chan kubetypes.PodUpdate) {
	if kl.logServer == nil {
		kl.logServer = http.StripPrefix("/logs/", http.FileServer(http.Dir("/var/log/")))
	}
	if kl.kubeClient == nil {
		glog.Warning("No api server defined - no node status update will be sent.")
	}

...
	go wait.Until(kl.updateRuntimeUp, 5*time.Second, wait.NeverStop)
...
	kl.pleg.Start()
	kl.syncLoop(updates, kl)
}

```
我们追踪 kl.cadvisor.Start() 方法 （如果是linux 环境 pkg/kubelet/cadvisor/cadvisor_linux.go）

``` golang 
func (cc *cadvisorClient) Start() error {
	return cc.Manager.Start()
}

```

继续追 cc.Manager.Start() 又到了manager vendor/github.com/google/cadvisor/manager/manager.go

``` golang 

// Start the container manager.
func (self *manager) Start() error {
	err := docker.Register(self, self.fsInfo, self.ignoreMetrics)
	if err != nil {
		glog.Warningf("Docker container factory registration failed: %v.", err)
	}
...
	err = containerd.Register(self, self.fsInfo, self.ignoreMetrics)
	if err != nil {
		glog.Warningf("Registration of the containerd container factory failed: %v", err)
	}
...
	go self.globalHousekeeping(quitGlobalHousekeeping)

	return nil
}

```
函数中几个Register 方法，会将对应的Factory 注册到 全局变量 factories 中，k8s 在一些注册方法中，经常喜欢使用全局变量，这些风格可以在scheduler ，controller-manager 中经常看到，并且会加上类似  的sync.RWMutex锁

追踪核心方法： self.globalHousekeeping（） 这个一个协程,会调用核心方法  detectSubcontainers（）

``` golang
func (self *manager) globalHousekeeping(quit chan error) {
	// Long housekeeping is either 100ms or half of the housekeeping interval.
...
	ticker := time.Tick(*globalHousekeepingInterval)
	for {
		select {
		case t := <-ticker:
			start := time.Now()

			// Check for new containers.
			err := self.detectSubcontainers("/")
			if err != nil {
				glog.Errorf("Failed to detect containers: %s", err)
			}

			// Log if housekeeping took too long.
			duration := time.Since(start)
			if duration >= longHousekeeping {
				glog.V(3).Infof("Global Housekeeping(%d) took %s", t.Unix(), duration)
			}
...		}
	}
}

```
detectSubcontainers（），此方法主要调用两个方法 createContainer和destroyContainer

``` golang 


```// Detect the existing subcontainers and reflect the setup here.
func (m *manager) detectSubcontainers(containerName string) error {
	added, removed, err := m.getContainersDiff(containerName)
	if err != nil {
		return err
	}

	// Add the new containers.
	for _, cont := range added {
		err = m.createContainer(cont.Name, watcher.Raw)
		if err != nil {
			glog.Errorf("Failed to create existing container: %s: %s", cont.Name, err)
		}
	}

	// Remove the old containers.
	for _, cont := range removed {
		err = m.destroyContainer(cont.Name)
		if err != nil {
			glog.Errorf("Failed to destroy existing container: %s: %s", cont.Name, err)
		}
	}

	return nil
}
```
以createContainer 为例 调用了 createContainerLocked（） 最终处理了add 和 destroy container
而 containers 的更新 是在 watchForNewContainers 实现的

cadvisor 支持web 访问，
```
http://IP:4194/

```


