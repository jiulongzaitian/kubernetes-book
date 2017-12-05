## kubelet介绍
```
作者：李昂 邮箱：liang@haihangyun.com
```

kubelet是在每个node节点上的一个类agent组件，他的主要作用简单来说就是监听三个pod源过来的pod事件运行或变更对应的pod从而达到期望状态。这三个pod源分别是apiserver、pod manifest文件和pod manifest url，这就意味着kubelet除了可以工作在标准的kubernetes集群模式下，还可以单独运行`Standalone`模式，直接作为一个符合kubernetes pod spec标准的容器执行代理，脱离其他kubernetes组件而整合到其他需要容器特性的系统中去。

虽然大体上看kubelet流程并不复杂，但是其接触到的外部系统是最多了包括容器运行时（docker、rkt等）、网络插件（calico、flannel）、以及数据卷插件（nfs、rbd等），而且kubernetes中基本都是遵循的declarative programming哲学使得所有代码的目标都是使系统的实际状态逐渐reconcile（调和）到期望状态。因此其中的各类manager尤其之多，管理各类资源达到期望状态。

## kubelet主要功能

1. pod生命周期管理：在kubernetes中最小的管理单位是pod，而pod中可能会有多个container，他们会共享network和IPC namesapce。主要参与pod管理的有如下manager：

* `podManager`：保存着运行在当前node的每个pod和mirrorpod的UID、fullname到pod spec的映射，还有mirrorpod UID到pod UID的映射（mirrorpod是指：那些kubelet从file、url中创建的pod称之为static pod，而为了方便查看的管理kubelet会给static pod在apiserver中创建对应的mirror pod），有了这些映射关系，kubelet在运行时就可以通过UID或者Fullname索引到任意pod，以及获取全部在本节点上运行的pod。
* `podCache`：缓存pod的status运行时状态，其中每条缓存记录带有一个`modified`更新时间，整个缓存有一个全局时间，所有记录的更新时间必须要不早于全局时间。`podCache`提供了一个方法可以获取一条有指定时间新的记录，否则会一直阻塞，直到获取到。
* `pleg`(Pod Lifecircle event generator)：主要任务是检测实际环境中container出现的变化，其每秒钟列出列出当前节点所有pod和所有container，与自己缓存中podRecord中对比，生成每个container的event，送到eventchannel，kubelet主循环syncLoop负责处理eventchannel中的事件。
* `statusManager`：用于保存pod运行时状态，以及同步此状态到apiserver中，主要与probeManager和主循环syncloop有交互。
* `podWorkers`：负责处理pod的各类变更事件，pod在同一时间只能被一个worker处理。

2. 主机容器监控以及垃圾回收：kubelet会内置cadvisor来收集主机和容器监控数据，以及定期执行垃圾回收释放容器和镜像占用的空间。

* `cadvisor`：kubelet内置本地容器主机监控工具，会收集主机信息主动上报给apiserver。其他组件也会调用cadvisor接口获取容器或主机监控信息。
* `containerGC`：根据回收策略删除`dead`容器。
* `imageManager`：通过查询cadvisor接口获取镜像文件系统占用量，从而根据回收策略释放空间。

3. 容器健康检查：有三种方式http、tcp、exec。如果pod的spec中配置了健康监测`livenessProbe`或者`readinessProbe`，kubelet会启动独立goroutine根据spec探测容器。

* `probeManager`：其中包含`statusManager`、`readinessManager`和`livenessManager`。`readinessManager`在探测container后会设置`statusManager`对应pod的container状态。而`livenessManager`探测后的事件会在syncloop主循环中处理。

4. 数据卷生命周期管理：kubernetes支持众多数据卷除此之外还有更加灵活的PersistentVolumeClaim和PersistentVolume。

* `volumeManager`：同样是由多个异步循环组成，同样遵循着期望状态(`desiredStateOfWorld`)和实际状态(`actualStateOfWorld`)这种设计模式，在异步循环中使用`reconciler`将实际状态转化为pod spec中的期望状态。

5. Api接口：kubelet提供api包含node信息，运行的pod信息、获取pod的log日志以及在指定pod的容器中执行命令等。除此以外cadvisor运行在kubelet内部，其api同时也会暴露出来。

## kubelet源码分析
kubelet代码有些函数比较长，为了便于阅读在下面的本文只列出核心代码，会着重分析几个比较重要的manager以及各个manager和主循环syncloop之前的关系，希望能捋出一个kubelet大致的脉络，知其然也要知其所以然。（本文基于v1.9.0-alpha.0.572+9636522137039，其他版本会略有出入）

### 主启动流程
其实启动流程方面，kuberntes家族组件基本都是大同小异，基本都遵循以下路程：1.构建配置结构体；2.载入默认配置；3.解析加载命令行配置到配置结构体；4.通过配置结构体构建实际核心结构体；5.开始运行主循环。
kubelet这里多了一个ExperimentalDockershim，这是为以后全面使用CRI做测试用的，我们暂时不用管它。
```golang
func main() {
	s := options.NewKubeletServer()
	s.AddFlags(pflag.CommandLine)

	flag.InitFlags()
	logs.InitLogs()
	defer logs.FlushLogs()

	verflag.PrintAndExitIfRequested()

	if s.ExperimentalDockershim {
		if err := app.RunDockershim(&s.KubeletConfiguration, &s.ContainerRuntimeOptions); err != nil {
			fmt.Fprintf(os.Stderr, "error: %v\n", err)
			os.Exit(1)
		}
	}

	if err := app.Run(s, nil); err != nil {
		fmt.Fprintf(os.Stderr, "error: %v\n", err)
		os.Exit(1)
	}
}
```
之后就会来到app.Run函数，参数是我们上面构建的options.KubeletServer配置结构体和kubelet.Dependencies，这里第二个参数传入的是nil，因此之后会构造它。这个app.Run只是个外壳，内部实际调用了run函数。

```golang
// Run runs the specified KubeletServer with the given Dependencies.  This should never exit.
// The kubeDeps argument may be nil - if so, it is initialized from the settings on KubeletServer.
// Otherwise, the caller is assumed to have set up the Dependencies object and a default one will
// not be generated.
func Run(s *options.KubeletServer, kubeDeps *kubelet.Dependencies) error {
	if err := run(s, kubeDeps); err != nil {
		return fmt.Errorf("failed to run Kubelet: %v", err)
	}
	return nil
}

func run(s *options.KubeletServer, kubeDeps *kubelet.Dependencies) (err error) {

	standaloneMode := true
	......
	// Set feature gates based on the value in KubeletConfiguration
	err = utilfeature.DefaultFeatureGate.Set(s.KubeletConfiguration.FeatureGates)
	if err != nil {
		return err
	}
	// Register current configuration with /configz endpoint
	cfgz, cfgzErr := initConfigz(&s.KubeletConfiguration)
	if utilfeature.DefaultFeatureGate.Enabled(features.DynamicKubeletConfig) {
    ...... 
	// if in standalone mode, indicate as much by setting all clients to nil
	if standaloneMode {
		kubeDeps.KubeClient = nil
		kubeDeps.ExternalKubeClient = nil
		kubeDeps.EventClient = nil
		glog.Warningf("standalone mode, no API client")
	} else if kubeDeps.KubeClient == nil || kubeDeps.ExternalKubeClient == nil || kubeDeps.EventClient == nil {
		// initialize clients if not standalone mode and any of the clients are not provided
		var kubeClient clientset.Interface
		var eventClient v1core.EventsGetter
		var externalKubeClient clientgoclientset.Interface

		clientConfig, err := CreateAPIServerClientConfig(s)

		var clientCertificateManager certificate.Manager
		if err == nil {
            ......
		} else {
            ......
		}
		kubeDeps.KubeClient = kubeClient
		kubeDeps.ExternalKubeClient = externalKubeClient
		if eventClient != nil {
			kubeDeps.EventClient = eventClient
		}
	}

	if kubeDeps.Auth == nil {
		auth, err := BuildAuth(nodeName, kubeDeps.ExternalKubeClient, s.KubeletConfiguration)
		if err != nil {
			return err
		}
		kubeDeps.Auth = auth
	}

	if kubeDeps.CAdvisorInterface == nil {
		kubeDeps.CAdvisorInterface, err = cadvisor.New(s.Address, uint(s.CAdvisorPort), s.ContainerRuntime, s.RootDirectory)
		if err != nil {
			return err
		}
	}

	// Setup event recorder if required.
	makeEventRecorder(&s.KubeletConfiguration, kubeDeps, nodeName)

	if kubeDeps.ContainerManager == nil {
		if s.CgroupsPerQOS && s.CgroupRoot == "" {
			glog.Infof("--cgroups-per-qos enabled, but --cgroup-root was not specified.  defaulting to /")
			s.CgroupRoot = "/"
		}
            ......
		if err != nil {
			return err
		}
	}

	if err := checkPermissions(); err != nil {
		glog.Error(err)
	}

	utilruntime.ReallyCrash = s.ReallyCrashForTesting

	rand.Seed(time.Now().UTC().UnixNano())

	// TODO(vmarmol): Do this through container config.
	oomAdjuster := kubeDeps.OOMAdjuster
	if err := oomAdjuster.ApplyOOMScoreAdj(0, int(s.OOMScoreAdj)); err != nil {
		glog.Warning(err)
	}

	if err := RunKubelet(&s.KubeletFlags, &s.KubeletConfiguration, kubeDeps, s.RunOnce); err != nil {
		return err
	}
    ......
	if s.RunOnce {
		return nil
	}

	<-done
	return nil
}
```
run函数的主要目的就是为kubelet.Dependencies做初始化，但是初始化之前需要通过命令行是否传过来KubeConfig判断kubelet运行在standalone模式还是标准集群模式，如果是是合法的KubeConfig，那么就是非standalone模式，如果是standalone模式kubelet.Dependencies中很多client可以直接置为nil。kubelet.Dependencies实际上是一类kubelet运行时需要外部依赖的抽象，它包含了各类Client（dockerclient、kubeclient等）以及网络插件和数据卷插件等，这么做还有另一个好处是可以通过参数传进来一些Dependencies来帮助测试。最后如果是非standalone模式，那么将构建正式的Dependencies里的各类对象。这里特别说明的是Dependencies.ContainerManager是管理kubelet进程docker进程以及相关cgroup和oom_adj_score设置的，它和kubelet管理的pod上运行的container是没有关系的。之后在RunKubelet中就开始创建真正的kubelet核心结构体以及运行主循环。

```golang
func RunKubelet(kubeFlags *options.KubeletFlags, kubeCfg *componentconfig.KubeletConfiguration, kubeDeps *kubelet.Dependencies, runOnce bool) error {
	hostname := nodeutil.GetHostname(kubeFlags.HostnameOverride)
	// Query the cloud provider for our node name, default to hostname if kcfg.Cloud == nil
	nodeName, err := getNodeName(kubeDeps.Cloud, hostname)
	if err != nil {
		return err
	}
	// Setup event recorder if required.
	makeEventRecorder(kubeCfg, kubeDeps, nodeName)

	hostNetworkSources, err := kubetypes.GetValidatedSources(kubeCfg.HostNetworkSources)
	if err != nil {
		return err
	}

	hostPIDSources, err := kubetypes.GetValidatedSources(kubeCfg.HostPIDSources)
	if err != nil {
		return err
	}

	hostIPCSources, err := kubetypes.GetValidatedSources(kubeCfg.HostIPCSources)
	if err != nil {
		return err
	}

	privilegedSources := capabilities.PrivilegedSources{
		HostNetworkSources: hostNetworkSources,
		HostPIDSources:     hostPIDSources,
		HostIPCSources:     hostIPCSources,
	}
	capabilities.Setup(kubeCfg.AllowPrivileged, privilegedSources, 0)

	credentialprovider.SetPreferredDockercfgPath(kubeCfg.RootDirectory)
	glog.V(2).Infof("Using root directory: %v", kubeCfg.RootDirectory)

	builder := kubeDeps.Builder
	if builder == nil {
		builder = CreateAndInitKubelet
	}
	if kubeDeps.OSInterface == nil {
		kubeDeps.OSInterface = kubecontainer.RealOS{}
	}
	k, err := builder(kubeCfg, kubeDeps, &kubeFlags.ContainerRuntimeOptions, kubeFlags.HostnameOverride, kubeFlags.NodeIP, kubeFlags.ProviderID)
	if err != nil {
		return fmt.Errorf("failed to create kubelet: %v", err)
	}

	// NewMainKubelet should have set up a pod source config if one didn't exist
	// when the builder was run. This is just a precaution.
	if kubeDeps.PodConfig == nil {
		return fmt.Errorf("failed to create kubelet, pod source config was nil")
	}
	podCfg := kubeDeps.PodConfig

	rlimit.RlimitNumFiles(uint64(kubeCfg.MaxOpenFiles))

	// process pods and exit.
	if runOnce {
		if _, err := k.RunOnce(podCfg.Updates()); err != nil {
			return fmt.Errorf("runonce failed: %v", err)
		}
		glog.Infof("Started kubelet %s as runonce", version.Get().String())
	} else {
		startKubelet(k, podCfg, kubeCfg, kubeDeps)
		glog.Infof("Started kubelet %s", version.Get().String())
	}
	return nil
}

func CreateAndInitKubelet(kubeCfg *componentconfig.KubeletConfiguration, kubeDeps *kubelet.Dependencies, crOptions *options.ContainerRuntimeOptions, hostnameOverride, nodeIP, providerID string) (k kubelet.Bootstrap, err error) {
	// TODO: block until all sources have delivered at least one update to the channel, or break the sync loop
	// up into "per source" synchronizations

	k, err = kubelet.NewMainKubelet(kubeCfg, kubeDeps, crOptions, hostnameOverride, nodeIP, providerID)
	if err != nil {
		return nil, err
	}

	k.BirthCry()

	k.StartGarbageCollection()

	return k, nil
}

func startKubelet(k kubelet.Bootstrap, podCfg *config.PodConfig, kubeCfg *componentconfig.KubeletConfiguration, kubeDeps *kubelet.Dependencies) {
	// start the kubelet
	go wait.Until(func() { k.Run(podCfg.Updates()) }, 0, wait.NeverStop)

	// start the kubelet server
	if kubeCfg.EnableServer {
		go wait.Until(func() {
			k.ListenAndServe(net.ParseIP(kubeCfg.Address), uint(kubeCfg.Port), kubeDeps.TLSOptions, kubeDeps.Auth, kubeCfg.EnableDebuggingHandlers, kubeCfg.EnableContentionProfiling)
		}, 0, wait.NeverStop)
	}
	if kubeCfg.ReadOnlyPort > 0 {
		go wait.Until(func() {
			k.ListenAndServeReadOnly(net.ParseIP(kubeCfg.Address), uint(kubeCfg.ReadOnlyPort))
		}, 0, wait.NeverStop)
	}
}
```
RunKubelet根据穿过来的配置设置capabilities，如果kubeDeps中没有外部注入builder，那么会用默认的CreateAndInitKubelet函数创建kubelet核心对象。CreateAndInitKubelet调用kubelet.NewMainKubelet创建kubelet对象k，k.BirthCry()调用eventrecorder写入一条"kubelet start"事件，而StartGarbageCollection()会启动containerGC和imagemanager的GarbageCollect()方法开启容器和镜像的垃圾回收。startKubelet会正式启动kubelet，并且如果配置了EnableServer和ReadOnlyPort还会监听端口开启Api。这里k.Run(podCfg.Updates())传入的podCfg.Updates()返回一个channel，会收到了apiserver、file、url中的pod更新事件。到这里其实主启动流程就算执行完了，前面启动流程的部分逻辑比较简单，本文只是大致分析到。而后面的NewMainKubelet和k.Run(podCfg.Updates())才包含了kubelet核心逻辑。

### 核心代码分析
由于NewMainKubelet函数十分长，我们会删除一些非核心代码，保证可读性。
```golang
// NewMainKubelet instantiates a new Kubelet object along with all the required internal modules.
// No initialization of Kubelet and its modules should happen here.
func NewMainKubelet(kubeCfg *componentconfig.KubeletConfiguration, kubeDeps *Dependencies, crOptions *options.ContainerRuntimeOptions, hostnameOverride, nodeIP, providerID string) (*Kubelet, error) {
    ......
	hostname := nodeutil.GetHostname(hostnameOverride)
	// Query the cloud provider for our node name, default to hostname
    nodeName := types.NodeName(hostname)
	// kubeDeps.PodConfig 其中包含了三种source，file，url和apiserver，PodConfig会给三个source分别提供一个channel
	// PodConfig 混合三个source的channel到一个，从而在kubelet主循环syncloop中监听这个混合后的channel获得pod事件
	if kubeDeps.PodConfig == nil {
		var err error
		kubeDeps.PodConfig, err = makePodSourceConfig(kubeCfg, kubeDeps, nodeName)
		if err != nil {
			return nil, err
		}
	}
    ......
	klet := &Kubelet{
		hostname:                       hostname,
        nodeName:                       nodeName,
        ......
	}
    ......
	imageBackOff := flowcontrol.NewBackOff(backOffPeriod, MaxContainerBackOff)
	klet.livenessManager = proberesults.NewManager()
	klet.podCache = kubecontainer.NewCache()
	// podManager is also responsible for keeping secretManager and configMapManager contents up-to-date.
	klet.podManager = kubepod.NewBasicPodManager(kubepod.NewBasicMirrorClient(klet.kubeClient), secretManager, configMapManager)
    ......
	// rktnetes cannot be run with CRI.
	if kubeCfg.ContainerRuntime != kubetypes.RktContainerRuntime {
		// kubelet defers to the runtime shim to setup networking. Setting
		// this to nil will prevent it from trying to invoke the plugin.
		// It's easier to always probe and initialize plugins till cri
		// becomes the default.
		klet.networkPlugin = nil

		switch kubeCfg.ContainerRuntime {
		case kubetypes.DockerContainerRuntime:
			// Create and start the CRI shim running as a grpc server.
			streamingConfig := getStreamingConfig(kubeCfg, kubeDeps)
			ds, err := dockershim.NewDockerService(kubeDeps.DockerClient, kubeCfg.SeccompProfileRoot, crOptions.PodSandboxImage,
				streamingConfig, &pluginSettings, kubeCfg.RuntimeCgroups, kubeCfg.CgroupDriver, crOptions.DockerExecHandlerName,
				crOptions.DockershimRootDirectory, crOptions.DockerDisableSharedPID)
			if err != nil {
				return nil, err
			}
			if err := ds.Start(); err != nil {
				return nil, err
			}
			// For now, the CRI shim redirects the streaming requests to the
			// kubelet, which handles the requests using DockerService..
			klet.criHandler = ds
			// The unix socket for kubelet <-> dockershim communication.
			glog.V(5).Infof("RemoteRuntimeEndpoint: %q, RemoteImageEndpoint: %q",
				kubeCfg.RemoteRuntimeEndpoint,
				kubeCfg.RemoteImageEndpoint)
			glog.V(2).Infof("Starting the GRPC server for the docker CRI shim.")
			// 使用docker的httpapi注册到了grpcserver中！而最后实际的调用就是
			// kuberuntime --> runtimeservice ---> remotegrpc ---> dockerhttpapi
			server := dockerremote.NewDockerServer(kubeCfg.RemoteRuntimeEndpoint, ds)
			// 开启uxixsocket监听，注册grpcserver各种method
			if err := server.Start(); err != nil {
				return nil, err
			}
			// Create dockerLegacyService when the logging driver is not supported.
			supported, err := dockershim.IsCRISupportedLogDriver(kubeDeps.DockerClient)
			if err != nil {
				return nil, err
			}
			if !supported {
				klet.dockerLegacyService = dockershim.NewDockerLegacyService(kubeDeps.DockerClient)
			}
		case kubetypes.RemoteContainerRuntime:
			// No-op.
			break
		default:
			return nil, fmt.Errorf("unsupported CRI runtime: %q", kubeCfg.ContainerRuntime)
		}
		// dockershim.sock 由kubelet所创建
		runtimeService, imageService, err := getRuntimeAndImageServices(kubeCfg)
		if err != nil {
			return nil, err
		}
		runtime, err := kuberuntime.NewKubeGenericRuntimeManager(
			kubecontainer.FilterEventRecorder(kubeDeps.Recorder),
			klet.livenessManager,
			......
		)
		if err != nil {
			return nil, err
		}
		klet.containerRuntime = runtime
		klet.runner = runtime
	} else {
		// rkt uses the legacy, non-CRI, integration. Configure it the old way.
        // TODO: Include hairpin mode settings in rkt?
        ......
	}

	// TODO: Factor out "StatsProvider" from Kubelet so we don't have a cyclic dependency
	klet.resourceAnalyzer = stats.NewResourceAnalyzer(klet, kubeCfg.VolumeStatsAggPeriod.Duration, klet.containerRuntime)
	klet.pleg = pleg.NewGenericPLEG(klet.containerRuntime, plegChannelCapacity, plegRelistPeriod, klet.podCache, clock.RealClock{})
	// 保存着运行时错误
	klet.runtimeState = newRuntimeState(maxWaitForContainerRuntime)
	klet.runtimeState.addHealthCheck("PLEG", klet.pleg.Healthy)
	klet.updatePodCIDR(kubeCfg.PodCIDR)
	// setup containerGC
	containerGC, err := kubecontainer.NewContainerGC(klet.containerRuntime, containerGCPolicy, klet.sourcesReady)
	if err != nil {
		return nil, err
	}
	klet.containerGC = containerGC
	klet.containerDeletor = newPodContainerDeletor(klet.containerRuntime, integer.IntMax(containerGCPolicy.MaxPerPodContainer, minDeadContainerInPod))
	// setup imageManager
	imageManager, err := images.NewImageGCManager(klet.containerRuntime, kubeDeps.CAdvisorInterface, kubeDeps.Recorder, nodeRef, imageGCPolicy)
	if err != nil {
		return nil, fmt.Errorf("failed to initialize image manager: %v", err)
	}
	klet.imageManager = imageManager
    klet.statusManager = status.NewManager(klet.kubeClient, klet.podManager, klet)
    ......
	// probeManager里包含的statusManager用于每次探测前获得最新的containerID，
	// 包含的livenessManager，是进行liveness探测时会把结果通知到syncloopIteration
	klet.probeManager = prober.NewManager(
		klet.statusManager,
		klet.livenessManager,
		klet.runner,
		containerRefManager,
		kubeDeps.Recorder)

	klet.volumePluginMgr, err =
		NewInitializedVolumePluginMgr(klet, secretManager, configMapManager, kubeDeps.VolumePlugins)
	if err != nil {
		return nil, err
	}
	// If the experimentalMounterPathFlag is set, we do not want to
	// check node capabilities since the mount path is not the default
	if len(kubeCfg.ExperimentalMounterPath) != 0 {
		kubeCfg.ExperimentalCheckNodeCapabilitiesBeforeMount = false
	}
	// setup volumeManager
	klet.volumeManager = volumemanager.NewVolumeManager(
		kubeCfg.EnableControllerAttachDetach,
        ......
		kubeCfg.KeepTerminatedPodVolumes)
	// runtimecache就是自带一个2s缓存时间戳的containerRuntime.GetPods
	runtimeCache, err := kubecontainer.NewRuntimeCache(klet.containerRuntime)
	if err != nil {
		return nil, err
	}
	klet.runtimeCache = runtimeCache
	klet.reasonCache = NewReasonCache()
	klet.workQueue = queue.NewBasicWorkQueue(klet.clock)
	klet.podWorkers = newPodWorkers(klet.syncPod, kubeDeps.Recorder, klet.workQueue, klet.resyncInterval, backOffPeriod, klet.podCache)

	klet.backOff = flowcontrol.NewBackOff(backOffPeriod, MaxContainerBackOff)
	klet.podKillingCh = make(chan *kubecontainer.PodPair, podKillingChannelCapacity)
	klet.setNodeStatusFuncs = klet.defaultNodeStatusFuncs()
	// setup eviction manager
	evictionManager, evictionAdmitHandler := eviction.NewManager(klet.resourceAnalyzer, evictionConfig, killPodNow(klet.podWorkers, kubeDeps.Recorder), klet.imageManager, klet.containerGC, kubeDeps.Recorder, nodeRef, klet.clock)
	// evictionManager == evictionAdmitHandler
	klet.evictionManager = evictionManager
	klet.admitHandlers.AddPodAdmitHandler(evictionAdmitHandler)
    ......
	// enable active deadline handler
	activeDeadlineHandler, err := newActiveDeadlineHandler(klet.statusManager, kubeDeps.Recorder, klet.clock)
	if err != nil {
		return nil, err
	}
	// 在getpodtosync中调用，和workqueue中的pod一起再次被sync
	klet.AddPodSyncLoopHandler(activeDeadlineHandler)
	// 在syncpod中generteapistatus中调用，查看pod是否应该被evict
	klet.AddPodSyncHandler(activeDeadlineHandler)
	criticalPodAdmissionHandler := preemption.NewCriticalPodAdmissionHandler(klet.GetActivePods, killPodNow(klet.podWorkers, kubeDeps.Recorder), kubeDeps.Recorder)
	// 这个generalPredicate本来在scheduler中已经完成，但是如果是standalone模式，还需要在本地做一次，
	// 防止冲突
	// criticalPodAdmissionHandler 对predicate因资源不够而调度失败的使用kill一些pod去释放一些资源
	klet.admitHandlers.AddPodAdmitHandler(lifecycle.NewPredicateAdmitHandler(klet.getNodeAnyWay, criticalPodAdmissionHandler))
	// apply functional Option's
	for _, opt := range kubeDeps.Options {
		opt(klet)
	}
	klet.appArmorValidator = apparmor.NewValidator(kubeCfg.ContainerRuntime)
	// 根据pod anotation和node上的profile判断是否可以admit
	klet.softAdmitHandlers.AddPodAdmitHandler(lifecycle.NewAppArmorAdmitHandler(klet.appArmorValidator))
	// 检查docker版本
	klet.softAdmitHandlers.AddPodAdmitHandler(lifecycle.NewNoNewPrivsAdmitHandler(klet.containerRuntime))
	// 只有docker支持gpu加速
	if utilfeature.DefaultFeatureGate.Enabled(features.Accelerators) {
		if kubeCfg.ContainerRuntime == kubetypes.DockerContainerRuntime {
			if klet.gpuManager, err = nvidia.NewNvidiaGPUManager(klet, kubeDeps.DockerClient); err != nil {
				return nil, err
			}
		} else {
			glog.Errorf("Accelerators feature is supported with docker runtime only. Disabling this feature internally.")
		}
	}
	// Set GPU manager to a stub implementation if it is not enabled or cannot be supported.
	if klet.gpuManager == nil {
		klet.gpuManager = gpu.NewGPUManagerStub()
	}
	// Finally, put the most recent version of the config on the Kubelet, so
	// people can see how it was configured.
	klet.kubeletConfiguration = *kubeCfg
	return klet, nil
}
```
NewMainKubelet函数非常之长，但函数里做的基本都是构建Kubelet对象运行时需要的各类组件包含如manager、cache缓存等。本文这里按照在NewMainKubelet出现的前后顺序着重对比较各个重要组件做详细分析。
* PodConfig：PodConfig其中包含了三种source，file，url和apiserver，其中PodConfig会给三个source各提供一个channel，PodConfig混合三个source的channel到一个update channel中。在apiserver source中会有本地的缓存保存着三种source的各个podspec，每次有pod的变化都会将新的变化后的全量pod列表和老的缓存做比较（根据podspec的deepequal），更新缓存然后发送对应需要处理的事件给updates channel，同时kubelet运行syncloop后会监听这个channel的事件，从而及时处理。
* podCache：缓存pod的status运行时状态，其中每条缓存记录带有一个`modified`更新时间，整个缓存有一个全局时间，所有记录的更新时间必须要不早于全局时间。podCache提供了一个方法可以获取一条有指定时间新的记录，如果当前没有比这个事件新的记录会一直阻塞，直到获取到。
* podManager：保存着运行在当前node的每个pod和mirrorpod的UID、fullname到pod spec的映射，还有mirrorpod UID到pod UID的映射（mirrorpod是指：那些kubelet从file、url中创建的pod称之为static pod，而为了方便查看的管理kubelet会给static pod在apiserver中创建对应的mirror pod），有了这些映射关系，kubelet在运行时就可以通过UID或者Fullname索引到任意pod，以及获取全部在本节点上运行的pod。同时podManager发现新的static pod时会通过kubeclient在apiserver上创建对应的mirrorpod，并记录对应关系。
* containerRuntime：kubelet运行时接口的抽象，它操作的时pod级别的对象如`GetPods(...)`、`KillPod(...)`、`GetPodStatus(...)`等等，containerRuntime由[CRI(Container Runtime Interface)](http://blog.kubernetes.io/2016/12/container-runtime-interface-cri-in-kubernetes.html)实现，CRI重新定义了Kubelet Container Runtime API，将原来完全面向Pod级别的API拆分成面向Sandbox和Container的API，并分离镜像管理和容器引擎到不同的服务。正是CRI抽象了具体容器运行时这一层，因此CRI可以由docker以及正处在孵化中的rkt和kubernetes官方CRI实现[cri-o](http://cri-o.io/) 等。我们以docker实现为例，分析containerRuntime创建的过程，首先调用`dockershim.NewDockerService(...)`创建了实现了CRI的DockerService，DockerService通过参数dockerclient向docker daemon发送请求从而完成容器的创建、删除和查询等操作。之后调用`dockerremote.NewDockerServer(...)`创建了CRI的grpc server，这里传入了刚才创建的DockerService作为CRI的实现注册到grpc server中。这里还会传入一个`kubeCfg.RemoteRuntimeEndpoint`，他的值通过命令行传递过来，默认值是`unix:///var/run/dockershim.sock`，因此grpc server会创建这个创建并监听这个unix domain sock文件，处理来自grpc客户端发送到`dockershim.sock`的请求。grpc的客户端恰恰在下面`getRuntimeAndImageServices(...)`调用后获得`runtimeService, imageService`。最后通过调用`NewKubeGenericRuntimeManager(...)`并传入刚才的runtimeService和imageService最终创建了containerRuntime对象。因此最终的调用栈是这样的：containerRuntime-->remoteRuntimeService(grpc客户端)-->dockershim.dock-->DockerService(CRI grpc服务端)-->dockerclient-->docker daemon。之所以如此设计主要还是因为需要兼容目前最稳定的容器运行时docker，未来cri-o正式登场的时候或许会变得美好很多。
* pleg(pod lifecircle event generator)：检测实际环境中container出现的变化，其每秒钟列出列出当前节点所有pod和所有container，与自己缓存中podRecord中对比，生成每个container的event，送到eventchannel，kubelet主循环syncLoop负责处理eventchannel中的事件。（如果一个container在1秒钟的周期内完成了创建终止并删除，那么pleg会丢失掉这个container的所有event）
* runtimeState：保存kubelet运行时错误，如networkError、runtimeError等，kubelet其他各组件检测到runtimeState包含相关错误时可能会跳过此次操作。
* containerGC：根据回收策略每分钟定时清理容器释放空间。而`containerGC.GarbageCollect()`是由`containerRuntime.GarbageCollect(...)`负责实现，
先会删除最老的一批dead container，之后会找出那些没有具体contianer的sandbox删除。
* imageManager：负责定时回收容器镜像以释放空间。优先删除很久没有使用的镜像，直到空间处于回收策略的阈值内。
* statusManager：负责管理pod运行时状态，同时会向apiserver同步更新pod状态。为了防止同一时间大量pod状态变更同时向apiserver中写入会给apiserver造成过大的压力，statusManager为每个pod状态还设置了一个`version`字段，每次状态有变化`version`都会自增1，而statusManager会在闲时调用`syncBatch()`状态`version`大于上次更新时的状态同步到apiserver中。此外statusManager还会提供更新和查询单个pod状态的接口`GetPodStatus(...)`和`SetPodStatus(...)`以及终止pod的接口`TerminatePod(...)`（相当于更新apiserver中pod的状态）。
* probeManager：其中包含`readinessManager`（容器是否时ready状态检测）和`livenessManager`（容器是否存活检测）。每个container都会使用单独的goroutine探测http、tcp、exec中的一种。`readinessManager`在探测container后会调用`statusManager`的`SetPodStatus(...)`设置apiserver中对应pod的container状态。而`livenessManager`在探测后发生的事件会在syncloop主循环中监听并处理。
* volumeManager：用于管理pod的数据卷，使kubelet按照每个pod的spec期望挂载数据卷到容器内。其中包含两个缓存`desiredStateOfWorld`（pod的期望状态），`actualStateOfWorld`（pod目前在node上的实际挂载状态），以及一个`reconciler`使得实际状态向期望状态转变。volumeManager运行时首先会`desiredStateOfWorldPopulator.Run(...)`，用于定时查询当前podmanager中的pod，将期望挂载的volume写入到`desiredStateOfWorld`，且将`desiredStateOfWorld`中存在而podmanager中不存在的pod删除。之后运行`reconciler.Run(...)`这个循环，`reconciler`的作用有四个：1.`desiredStateOfWorld`期望中没出现但是`actualStateOfWorld`实际中出现的数据卷卸载；2.`desiredStateOfWorld`期望中出现而`actualStateOfWorld`实际中没出现的数据卷挂载到对应目录；3.确保实际情况中没有挂载的数据卷从node上detach掉。（这里每个数据卷会有mount和attach操作，从笔者理解来看mount是将一个具体文件系统或目录挂到容器中，attach则是将一些分布式存储或网络存储作为一个设备接到当前node上）。4.扫描/var/lib/kubelet目录，将目录中的绝对实际volume同步到`actualStateOfWorld`。volumeManager使用了多个纯异步循环实现数据卷的正确挂载和卸载，而且再次践行了kubernetes使整个系统按照用户期望的状态操作这种设计哲学。
* podWorkers：处理pod变更事件的实际执行者。对每个pod都会使用单独的goroutine处理以pod.UID区分，处理完后还会将pod放入到workQueue中等待下次同步。这里有一个特殊设计，当某个pod的worker正在处理时这个pod又来了一个变更事件，那么会将这个事件放到`lastUndeliveredWorkUpdate`等待worker处理完后再次处理这个事件，但`lastUndeliveredWorkUpdate`并不是一个等待队列而这是一个`map[types.UID]UpdatePodOptions{}`map，因此当worker正在处理时来了多个同一个pod的事件，最后只会处理最后一个最新的事件，这是kuberntes声明式api的好处，最后一次事件就是用户最终期望的pod状态。
* evictionManager：通过一个定时循环检查node是否已经超过threshold，以内存为第一优先级驱逐一个pod缓解node压力。evictionManager首先会解析传过来的threshold配置（e.g. memory.available<1Gi，默认值是"memory.available<100Mi,nodefs.available<10%,nodefs.inodesFree<5%"），通过cadvisor检查node当前状态memory和fs，当超过threshold时会以内存优先根据pod的QOS，每次循环驱逐一个pod，直到处于threshold正常状态。
* admitHandlers：当处理有新增pod事件时判断pod是否可以在node上与其他pod共存，包含：
	1. evictionAdmitHandler：当node处于MemoryPressure时，防止QOS为BestEffort的pod运行在node上，因为BestEffort优先级最低，即使调度上来也会马上被驱逐，导致系统不稳定。其实这个判断在kube-scheduler上调度的时候就已经做过了，在kubelet上还要判断一次主要是kubelet运行在standalone模式，没有scheduler做调度，只能在node上做判断。
	2. runtimeSupport：检查容器运行时是否支持sysctl，当是docker时需版本大于1.12
	3. safeWhitelist：sysctl安全白名单，如果在pod的annotation中security.alpha.kubernetes.io/sysctls的值需在白名单列表里。
	4. unsafeWhitelist：不安全白名单包含了上面的safeWhitelist和命令行传过来的用户指定sysctl设置项。
	5. PredicateAdmitHandler：调用scheduler的`GeneralPredicates(...)`检查pod是否和已运行的其他pod有冲突如hostPort端口映射冲突和资源请求超出等。
* softAdmitHandlers：每次syncPod时判断pod是否可以在node上运行。包含：
	1. AppArmorAdmitHandler：根据node上apparmor的profile配置和pod的annotation key判断是否可以被admit。
	2. NoNewPrivsAdmitHandler：当pod中container设置了no_new_privs（docker中的一个安全选项，阻止一个进程或其子进程获得新的privilege）时，检查容器运行时是否支持，需要docker版本大于1.23。
* gpuManager：目前只有docker运行时支持的NvidiaGPU管理。扫描/dev目录发现nvidia gpu设备，当需要时分配给容器。
到这里kubelet中各个组件全部初始化完毕，整个kubelet对象也就创建出来了。之后让我们回到`k.Run(podCfg.Updates())`看看真正的kubelet主循环。

### 主循环
Run函数中除了`kl.syncLoop(updates, kl)`其他部分都是将kubelet各组件启动起来，各组件的功能和运行原理根据之前的介绍和上面的注释，想必应该比较清晰，下面我们重点看kubelet核心循环`kl.syncLoop(updates, kl)`。

```
// Run starts the kubelet reacting to config updates
func (kl *Kubelet) Run(updates <-chan kubetypes.PodUpdate) {
	if kl.logServer == nil {
		kl.logServer = http.StripPrefix("/logs/", http.FileServer(http.Dir("/var/log/")))
	}
	if kl.kubeClient == nil {
		glog.Warning("No api server defined - no node status update will be sent.")
	}

	if err := kl.initializeModules(); err != nil {
		kl.recorder.Eventf(kl.nodeRef, v1.EventTypeWarning, events.KubeletSetupFailed, err.Error())
		glog.Error(err)
		kl.runtimeState.setInitError(err)
	}

	// Start volume manager
	// 包括vm.desiredStateOfWorldPopulator.Run:主要是根据每个pod的spec来填充desiredStateOfWorld
	// vm.reconciler.Run(stopCh)根据desiredStateOfWorld和actualStateOfWorld来mount和unmount volume
	// 以及定时从/var/lib/kubelet中通过目录名syncState状态到actualStateOfWorld和desiredStateOfWorld
	go kl.volumeManager.Run(kl.sourcesReady, wait.NeverStop)

	if kl.kubeClient != nil {
		// Start syncing node status immediately, this may set up things the runtime needs to run.
		// syncNodeStatus分为registerWithAPIServer和updateNodeStatus
		// registerWithAPIServer：只在第一次启动的时候根据配置项向apiserver注册当前node：
		// 包括调用cadvisor注册资源信息，runtime注册images和调用volumemanager注册volume
		// updateNodeStatus定期调用updateNodeStatus更新上面的状态
		go wait.Until(kl.syncNodeStatus, kl.nodeStatusUpdateFrequency, wait.NeverStop)
	}
	// 未来使用cri后将从runtimeUp中获取网络状态而不是networkplugin
	go wait.Until(kl.syncNetworkStatus, 30*time.Second, wait.NeverStop)
	// updateRuntimeUp会调用docker_service检查nodeready和networkready状态
	// 然后根据状态更新kl.runtimestate
	// 同时会启动cadviosr和evictManager
	// evictmanager: 通过cadvisor获取资源用量，然后根据threshold提供
	// nodecondition（memorypressure或者diskpressure），每次选择根据qos等因素进行排序
	// 最后evict占用最多的一个pod
	go wait.Until(kl.updateRuntimeUp, 5*time.Second, wait.NeverStop)

	// Start loop to sync iptables util rules
	// 确保 规则能应用到filter表和nat表
	if kl.makeIPTablesUtilChains {
		go wait.Until(kl.syncNetworkUtil, 1*time.Minute, wait.NeverStop)
	}

	// Start a goroutine responsible for killing pods (that are not properly
	// handled by pod workers).
	// kill掉从kl.podKillingCh管道中传过来的pod
	// 通过kubeGenericRuntimeManager.KillPod --->
	// kubeGenericRuntimeManager.RemoteRuntimeService.StopPodSandbox、StopContainer
	// kl.podKillingCh是deletePod函数在主循环中把将要删除的pod塞进来
	go wait.Until(kl.podKiller, 1*time.Second, wait.NeverStop)

	// Start gorouting responsible for checking limits in resolv.conf
	// 只是通过eventRecorder通知事件，并未做实际动作
	if kl.resolverConfig != "" {
		go wait.Until(func() { kl.checkLimitsForResolvConf() }, 30*time.Second, wait.NeverStop)
	}

	// Start component sync loops.
	// 主要用于同步apiserver中pod状态和kubelet中管理的pod状态
	// 其中分为syncPod（处理单个podstatus）和syncBatch（处理一批），这里做的优化是
	// 当同一时间需要更新的pod超过channel buffer时（1000），就不再往channel中放
	// 防止阻塞，而是不予处理（只是对本地status缓存version+1），通过定时的syncBatch找出status version比apiserver里version
	// 大的status从而更新apiserver
	// PodManager是pod和mirrorpod的本地缓存，可以得到pod和mirrorpod的关系
	kl.statusManager.Start()
	// probeManager对每个pod的每个container中的readiness和liveness都会启动一个
	// probe worker（单独goroutine），每次最多retry三次，只要成功就不再retry。
	// livenessManager被kubelet runtime和syncIteration共用
	// probemanager只处理readinessManager发生的update，而
	// livenessManager发生的update在syncIteration接收并处理
	kl.probeManager.Start()

	// Start the pod lifecycle event generator.
	// podManager.GetPods得到的是Pod是apiserver types里的Pod标准结构
	// pleg=pod lifecycle event generator, 每一秒钟列出列出当前节点所有pod和所有container，
	// 与自己缓存中podRecord中对比，生成每个container的event，送到eventchannel，
	// syncLoop负责处理eventchannel中的事件
	// 如果pod（pause）这个container挂掉了同样会生成两个container-died removed事件
	kl.pleg.Start()
	// podManager就是desired state即apiserver中的状态
	kl.syncLoop(updates, kl)
}
```


下图展示了kubelet中各个组件和主循环之间的关系：
![image](http://on-img.com/chart_image/5a0d48a0e4b0d53d97999f2a.png)

```golang
func (kl *Kubelet) syncLoop(updates <-chan kubetypes.PodUpdate, handler SyncHandler) {
	glog.Info("Starting kubelet main sync loop.")
	// The resyncTicker wakes up kubelet to checks if there are any pod workers
	// that need to be sync'd. A one-second period is sufficient because the
	// sync interval is defaulted to 10s.
	syncTicker := time.NewTicker(time.Second)
	defer syncTicker.Stop()
	housekeepingTicker := time.NewTicker(housekeepingPeriod)
	defer housekeepingTicker.Stop()
	plegCh := kl.pleg.Watch()
	for {
		if rs := kl.runtimeState.runtimeErrors(); len(rs) != 0 {
			// runtimeState中有任何错误sleep 5秒后跳过本次同步
			glog.Infof("skipping pod synchronization - %v", rs)
			time.Sleep(5 * time.Second)
			continue
		}

		kl.syncLoopMonitor.Store(kl.clock.Now())
		if !kl.syncLoopIteration(updates, handler, syncTicker.C, housekeepingTicker.C, plegCh) {
			break
		}
		kl.syncLoopMonitor.Store(kl.clock.Now())
	}
}
```
`syncLoop`处理来自updates这个管道里过来的Pod更新事件，之前分析过PodConfig中这个update管道是混合了来自apiserver、file、url三个源的pod更新事件，`syncTicker`每隔一秒钟将kubelet管理的pod的期望状态落实；`housekeepingTicker`主要用于每隔两秒钟做一些清理工作。`plegCh`是接收来自pleg（Pod lifecircle event generator）的事件，如果runtimeState中包含了任何运行时错误，将会休眠5秒跳过此次循环。最后由`syncLoopIteration`负责处理各类管道中的事件。

```golang
func (kl *Kubelet) syncLoopIteration(configCh <-chan kubetypes.PodUpdate, handler SyncHandler,
	syncCh <-chan time.Time, housekeepingCh <-chan time.Time, plegCh <-chan *pleg.PodLifecycleEvent) bool {
	select {
	// 从source中传过来事件，判断事件类型然后dispathwork
	case u, open := <-configCh:
		// Update from a config source; dispatch it to the right handler
		// callback.
		if !open {
			glog.Errorf("Update channel is closed. Exiting the sync loop.")
			return false
		}

		switch u.Op {
		case kubetypes.ADD:
			glog.V(2).Infof("SyncLoop (ADD, %q): %q", u.Source, format.Pods(u.Pods))
			// After restarting, kubelet will get all existing pods through
			// ADD as if they are new pods. These pods will then go through the
			// admission process and *may* be rejected. This can be resolved
			// once we have checkpointing.
			handler.HandlePodAdditions(u.Pods)
		case kubetypes.UPDATE:
			glog.V(2).Infof("SyncLoop (UPDATE, %q): %q", u.Source, format.PodsWithDeletiontimestamps(u.Pods))
			handler.HandlePodUpdates(u.Pods)
		case kubetypes.REMOVE:
			glog.V(2).Infof("SyncLoop (REMOVE, %q): %q", u.Source, format.Pods(u.Pods))
			handler.HandlePodRemoves(u.Pods)
		case kubetypes.RECONCILE:
			glog.V(4).Infof("SyncLoop (RECONCILE, %q): %q", u.Source, format.Pods(u.Pods))
			handler.HandlePodReconcile(u.Pods)
		case kubetypes.DELETE:
			glog.V(2).Infof("SyncLoop (DELETE, %q): %q", u.Source, format.Pods(u.Pods))
			// DELETE is treated as a UPDATE because of graceful deletion.
			handler.HandlePodUpdates(u.Pods)
		case kubetypes.SET:
			// TODO: Do we want to support this?
			glog.Errorf("Kubelet does not support snapshot update")
		}

		// Mark the source ready after receiving at least one update from the
		// source. Once all the sources are marked ready, various cleanup
		// routines will start reclaiming resources. It is important that this
		// takes place only after kubelet calls the update handler to process
		// the update to ensure the internal pod cache is up-to-date.
		kl.sourcesReady.AddSource(u.Source)
		// 从pleg的每次relist的event传过来
	case e := <-plegCh:
		if isSyncPodWorthy(e) {
			// PLEG event for a pod; sync it.
			if pod, ok := kl.podManager.GetPodByUID(e.ID); ok {
				glog.V(2).Infof("SyncLoop (PLEG): %q, event: %#v", format.Pod(pod), e)
				handler.HandlePodSyncs([]*v1.Pod{pod})
			} else {
				// If the pod no longer exists, ignore the event.
				glog.V(4).Infof("SyncLoop (PLEG): ignore irrelevant event: %#v", e)
			}
		}

		if e.Type == pleg.ContainerDied {
			if containerID, ok := e.Data.(string); ok {
				kl.cleanUpContainersInPod(e.ID, containerID)
			}
		}
	case <-syncCh:
		// 当每秒钟的syncTicker到时，也会从getPodsToSync会从workqueue取出pod做重新同步
		// workqueue中的pod是每次sync后会再把pod放到queue中等待下次同步
		// Sync pods waiting for sync
		podsToSync := kl.getPodsToSync()
		if len(podsToSync) == 0 {
			break
		}
		glog.V(4).Infof("SyncLoop (SYNC): %d pods; %s", len(podsToSync), format.Pods(podsToSync))
		kl.HandlePodSyncs(podsToSync)
	case update := <-kl.livenessManager.Updates():
		if update.Result == proberesults.Failure {
			// The liveness manager detected a failure; sync the pod.
			// 从 probemanager中传过来的最新事件，如果有container spec中有livenessprobe，
			// 那么当运行的时候会有事件过来
			// We should not use the pod from livenessManager, because it is never updated after
			// initialization.
			pod, ok := kl.podManager.GetPodByUID(update.PodUID)
			if !ok {
				// If the pod no longer exists, ignore the update.
				glog.V(4).Infof("SyncLoop (container unhealthy): ignore irrelevant update: %#v", update)
				break
			}
			glog.V(1).Infof("SyncLoop (container unhealthy): %q", format.Pod(pod))
			handler.HandlePodSyncs([]*v1.Pod{pod})
		}
	case <-housekeepingCh:
		// 以podmanager为准，清理podworker、probemanager、statusManage、podmanager中的孤儿mirrorpod、cgroupQOS目录
		// 以removeOrphanedPodStatuses为准清理、rumtimecacahe、poddir
		// 最后清理backoff
		if !kl.sourcesReady.AllReady() {
			// If the sources aren't ready or volume manager has not yet synced the states,
			// skip housekeeping, as we may accidentally delete pods from unready sources.
			glog.V(4).Infof("SyncLoop (housekeeping, skipped): sources aren't ready yet.")
		} else {
			glog.V(4).Infof("SyncLoop (housekeeping)")
			if err := handler.HandlePodCleanups(); err != nil {
				glog.Errorf("Failed cleaning pods: %v", err)
			}
		}
	}
	return true
}
```
这里syncLoopIteration同时监听了多个管道，我们逐个分析各个管道的处理方式：

#### configCh
configCh中的事件全部是来自于外部apiserver、file、url，此外从configCh中接收到的事件`u`还会附带有事件源source的信息，可以标记这个事件是从哪个source过来的，当从某个事件源接收到任意一个事件时，就会调用`kl.sourcesReady.AddSource(u.Source)`认为这个源已经ready准备好，当所有source都ready时，kubelet就可以开启housekeeping资源清理和回收工作。

这里我们以Pod的添加事件`kubetypes.ADD`为例，看一下在`handler.HandlePodAdditions(u.Pods)`具体一个pod时如何被创建出来的。

```golang
// HandlePodAdditions is the callback in SyncHandler for pods being added from
// a config source.
func (kl *Kubelet) HandlePodAdditions(pods []*v1.Pod) {
	start := kl.clock.Now()
	sort.Sort(sliceutils.PodsByCreationTime(pods))
	// 对每个pod，先加到podmanager中，然后检查是否admit，之后dispatchwork
	// 最后给probemanager加上此pod（如果spec中有probe的需要）
	for _, pod := range pods {
		existingPods := kl.podManager.GetPods()
		// Always add the pod to the pod manager. Kubelet relies on the pod
		// manager as the source of truth for the desired state. If a pod does
		// not exist in the pod manager, it means that it has been deleted in
		// the apiserver and no action (other than cleanup) is required.
		kl.podManager.AddPod(pod)
		// 如果是mirror就相当于原始的staticpod发生了UPDATE事件
		if kubepod.IsMirrorPod(pod) {
			kl.handleMirrorPod(pod, start)
			continue
		}

		if !kl.podIsTerminated(pod) {
			// Only go through the admission process if the pod is not
			// terminated.

			// We failed pods that we rejected, so activePods include all admitted
			// pods that are alive.
			activePods := kl.filterOutTerminatedPods(existingPods)

			// Check if we can admit the pod; if not, reject it.
			if ok, reason, message := kl.canAdmitPod(activePods, pod); !ok {
				kl.rejectPod(pod, reason, message)
				continue
			}
		}
		mirrorPod, _ := kl.podManager.GetMirrorPodByPod(pod)
		kl.dispatchWork(pod, kubetypes.SyncPodCreate, mirrorPod, start)
		kl.probeManager.AddPod(pod)
	}
}

```
HandlePodAdditions首先对传过来的Pod更新事件列表按创建时间排序，这是为了当kubelet重启时，之前运行的所有pod会以ADD事件传过来，为了使每次都是相同的结果因此按照创建时间进行排序。之后遍历pod列表对每个pod进行单独处理，从podManager中获取现在所有`existingPods`，然后将当前pod添加到podManager，podManager对应着apiserver中的pod，因此当有pod Add事件过来时，说明apiserver中已经有了对应的Pod，此时podManager会直接将Pod添加进来，保持对apiserver的同步。之后判断pod是否为mirror pod，因为mirror pod是file或url在apiserver上的映射，如果是mirror pod发生了ADD\UPDATE\DELETE事件都会认为是原来的static pod发生的事件从而进行处理。在`dispatchWork`之前，我们需要获得`existingPods`中处于运行状态的pod`activePods`，以及使用之前我们介绍过的admitHandler判断此pod是否和`activePods`以及当前节点有冲突（如果不能admit，则会直接`rejectPod`，`rejectPod`只是会想apiserver写入失败事件，并不会删除pod），以及从尝试从podmanager中获取当前pod的mirrorpod。现在让我们看看`dispatchWork`都做了什么：

```golang
// dispatchWork starts the asynchronous sync of the pod in a pod worker.
// If the pod is terminated, dispatchWork
func (kl *Kubelet) dispatchWork(pod *v1.Pod, syncType kubetypes.SyncPodType, mirrorPod *v1.Pod, start time.Time) {
	if kl.podIsTerminated(pod) {
		if pod.DeletionTimestamp != nil {
			// If the pod is in a terminated state, there is no pod worker to
			// handle the work item. Check if the DeletionTimestamp has been
			// set, and force a status update to trigger a pod deletion request
			// to the apiserver.
			kl.statusManager.TerminatePod(pod)
		}
		return
	}
	// Run the sync in an async worker.
	kl.podWorkers.UpdatePod(&UpdatePodOptions{
		Pod:        pod,
		MirrorPod:  mirrorPod,
		UpdateType: syncType,
		OnCompleteFunc: func(err error) {
			if err != nil {
				metrics.PodWorkerLatency.WithLabelValues(syncType.String()).Observe(metrics.SinceInMicroseconds(start))
			}
		},
	})
	// Note the number of containers for new pods.
	if syncType == kubetypes.SyncPodCreate {
		metrics.ContainersPerPodCount.Observe(float64(len(pod.Spec.Containers)))
	}
}
```
dispatchWork首先会判断pod中所有container是否全部处于非运行状态（`kl.podIsTerminated(pod) `），如果是则会调用`statusManager.TerminatePod(pod)`更新apiserver中全部container状态，当pod资源清理完成可以被删除，就使用kubeclient调用apiserver的Delete来正式删除这个pod，这一块是pod被delete时的逻辑。我们继续看pod add时的下面代码，`kl.podWorkers.UpdatePod`这里会调用podWorker创建一个异步的worker完成pod的变更。`podWorker.UpdatePod`如下：

```golang
// Apply the new setting to the specified pod.
// If the options provide an OnCompleteFunc, the function is invoked if the update is accepted.
// Update requests are ignored if a kill pod request is pending.
func (p *podWorkers) UpdatePod(options *UpdatePodOptions) {
	pod := options.Pod
	uid := pod.UID
	var podUpdates chan UpdatePodOptions
	var exists bool

	p.podLock.Lock()
	defer p.podLock.Unlock()
	if podUpdates, exists = p.podUpdates[uid]; !exists {
		// We need to have a buffer here, because checkForUpdates() method that
		// puts an update into channel is called from the same goroutine where
		// the channel is consumed. However, it is guaranteed that in such case
		// the channel is empty, so buffer of size 1 is enough.
		podUpdates = make(chan UpdatePodOptions, 1)
		p.podUpdates[uid] = podUpdates

		// Creating a new pod worker either means this is a new pod, or that the
		// kubelet just restarted. In either case the kubelet is willing to believe
		// the status of the pod for the first pod worker sync. See corresponding
		// comment in syncPod.
		go func() {
			defer runtime.HandleCrash()
			p.managePodLoop(podUpdates)
		}()
	}
	if !p.isWorking[pod.UID] {
		p.isWorking[pod.UID] = true
		podUpdates <- *options
	} else {
		// if a request to kill a pod is pending, we do not let anything overwrite that request.
		// 当podworker正在工作的时候，如果lastUndeliveredWorkUpdate未找到此pod或者这个pod不是SyncPodKill
		// 会在lastUndeliveredWorkUpdate打个标记，待p.managePodLoop(podUpdates)处理完当前，之后处理
		// lastUndeliveredWorkUpdate的内容
		// lastUndeliveredWork 当处理和等待map中都满了，仍然有pod event过来时
		// 会直接覆盖lastUndeliveredWorkUpdate这个map，只会找最近的更新不会频繁更新
		// 在删除pod的过程中，killpod是同步函数会一直在这working，直到sandbox被kill掉，
		// 而每次pleg会发生事件，走到这里但是不会进入managePodLoop，只会更新在lastUndeliveredWorkUpdate
		// 直到sandbox删除，才会working空闲
		update, found := p.lastUndeliveredWorkUpdate[pod.UID]
		if !found || update.UpdateType != kubetypes.SyncPodKill {
			p.lastUndeliveredWorkUpdate[pod.UID] = *options
		}
	}
}

```
在`UpdatePod`中设计颇为巧妙，为每个pod创建一个使用channel实现的等待队列`podUpdates`，缓冲只有1是因为某个pod在同一时间只能处理一个更新事件，而新来的更新事件会检查podWorker的对应pod的worker是否在工作`p.isWorking[pod.UID]`，如果为true就把更新事件放到lastUndeliveredWorkUpdate这个缓冲map中，等待pod worker处理完当前事件后处理此事件。这里如果pod worker正在工作时又来了多条pod更新事件，那么只有最新的事件会在lastUndeliveredWorkUpdate中，前面的都会被覆盖，正式由于每次的事件变更都是声明式（如把某个资源增加到100，而不是加10减10），所以结合这种lastUndeliveredWorkUpdate缓存可以减少很多中间的变更操作。

podWoker具体的异步处理过程是在`managePodLoop`完成，`managePodLoop`接收参数为`podUpdates`任务队列。

```golang
func (p *podWorkers) managePodLoop(podUpdates <-chan UpdatePodOptions) {
	var lastSyncTime time.Time
	for update := range podUpdates {
		err := func() error {
			podUID := update.Pod.UID
			// This is a blocking call that would return only if the cache
			// has an entry for the pod that is newer than minRuntimeCache
			// Time. This ensures the worker doesn't start syncing until
			// after the cache is at least newer than the finished time of
			// the previous sync.
			status, err := p.podCache.GetNewerThan(podUID, lastSyncTime)
			if err != nil {
				return err
			}
			err = p.syncPodFn(syncPodOptions{
				mirrorPod:      update.MirrorPod,
				pod:            update.Pod,
				podStatus:      status,
				killPodOptions: update.KillPodOptions,
				updateType:     update.UpdateType,
			})
			lastSyncTime = time.Now()
			return err
		}()
		// notify the call-back function if the operation succeeded or not
		if update.OnCompleteFunc != nil {
			update.OnCompleteFunc(err)
		}
		if err != nil {
			glog.Errorf("Error syncing pod %s (%q), skipping: %v", update.Pod.UID, format.Pod(update.Pod), err)
			// if we failed sync, we throw more specific events for why it happened.
			// as a result, i question the value of this event.
			// TODO: determine if we can remove this in a future release.
			// do not include descriptive text that can vary on why it failed so in a pathological
			// scenario, kubelet does not create enough discrete events that miss default aggregation
			// window.
			p.recorder.Eventf(update.Pod, v1.EventTypeWarning, events.FailedSync, "Error syncing pod")
		}
		p.wrapUp(update.Pod.UID, err)
	}
}

func (p *podWorkers) wrapUp(uid types.UID, syncErr error) {
	// Requeue the last update if the last sync returned error.
	switch {
	case syncErr == nil:
		// No error; requeue at the regular resync interval.
		p.workQueue.Enqueue(uid, wait.Jitter(p.resyncInterval, workerResyncIntervalJitterFactor))
	default:
		// Error occurred during the sync; back off and then retry.
		p.workQueue.Enqueue(uid, wait.Jitter(p.backOffPeriod, workerBackOffPeriodJitterFactor))
	}
	p.checkForUpdates(uid)
}

func (p *podWorkers) checkForUpdates(uid types.UID) {
	p.podLock.Lock()
	defer p.podLock.Unlock()
	if workUpdate, exists := p.lastUndeliveredWorkUpdate[uid]; exists {
		p.podUpdates[uid] <- workUpdate
		delete(p.lastUndeliveredWorkUpdate, uid)
	} else {
		p.isWorking[uid] = false
	}
}
```
managePodLoop监听`podUpdates`管道获取pod变更事件，首先获得从podCache中阻塞获取一个比上次同步时间`lastSyncTime`更新的pod状态`status`，然后调用`syncPodFn`用于真正处理pod的各类变更事件。处理完后记录最新的完成时间`lastSyncTime`，在这个事件最后调用`p.wrapUp(...)`将pod放入kubelet的workQuque中等待下次同步，而`checkForUpdates`会检查在`lastUndeliveredWorkUpdate`在刚才处理的时候是否有新的更新事件，如果有将其放入到podUpdates中由managePodLoop处理，如果没有则标记当前worker是空闲的`p.isWorking[uid] = false`。

```golang
func (kl *Kubelet) syncPod(o syncPodOptions) error {
	// pull out the required options
	pod := o.pod
	mirrorPod := o.mirrorPod
	podStatus := o.podStatus
	updateType := o.updateType

	// if we want to kill a pod, do it now!
	if updateType == kubetypes.SyncPodKill {
		killPodOptions := o.killPodOptions
		if killPodOptions == nil || killPodOptions.PodStatusFunc == nil {
			return fmt.Errorf("kill pod options are required if update type is kill")
		}
		apiPodStatus := killPodOptions.PodStatusFunc(pod, podStatus)
		kl.statusManager.SetPodStatus(pod, apiPodStatus)
		// we kill the pod with the specified grace period since this is a termination
		if err := kl.killPod(pod, nil, podStatus, killPodOptions.PodTerminationGracePeriodSecondsOverride); err != nil {
			// there was an error killing the pod, so we return that error directly
			utilruntime.HandleError(err)
			return err
		}
		return nil
	}
    ......
    
	// Generate final API pod status with pod and status manager status
	apiPodStatus := kl.generateAPIPodStatus(pod, podStatus)
	// The pod IP may be changed in generateAPIPodStatus if the pod is using host network. (See #24576)
	// TODO(random-liu): After writing pod spec into container labels, check whether pod is using host network, and
	// set pod IP to hostIP directly in runtime.GetPodStatus
	podStatus.IP = apiPodStatus.PodIP

	// Record the time it takes for the pod to become running.
	existingStatus, ok := kl.statusManager.GetPodStatus(pod.UID)
	// 当statusmanager里状态为pending 传过来的status位runing，记录pod从pending到running的时间
	if !ok || existingStatus.Phase == v1.PodPending && apiPodStatus.Phase == v1.PodRunning &&
		!firstSeenTime.IsZero() {
		metrics.PodStartLatency.Observe(metrics.SinceInMicroseconds(firstSeenTime))
	}
	// 通过检查softAdmitHandlers和capbilities判断pod是否可以运行在本节点
	runnable := kl.canRunPod(pod)
	......
    
	// Kill pod if it should not be running
	// 这里会处理第一次从apiserver收过来的pod.DeletionTimestamp != nil，调用kubectl delete pod时，
	// 会给pod的metedata加上 DeletionTimestamp和deletionGracePeriodSeconds
	// 而在dispachwork中会把这个时间当作Update事件处理，此时所有pod下所有container都在running
	if !runnable.Admit || pod.DeletionTimestamp != nil || apiPodStatus.Phase == v1.PodFailed {
		var syncErr error
		if err := kl.killPod(pod, nil, podStatus, nil); err != nil {
			syncErr = fmt.Errorf("error killing pod: %v", err)
			utilruntime.HandleError(syncErr)
		} else {
			if !runnable.Admit {
				// There was no error killing the pod, but the pod cannot be run.
				// Return an error to signal that the sync loop should back off.
				syncErr = fmt.Errorf("pod cannot be run: %s", runnable.Message)
			}
		}
		return syncErr
	}
    ......
    // Create Cgroups for the pod and apply resource parameters
	// to them if cgroups-per-qos flag is enabled.
	// 这里特别处理了当kubelet重新启动，且开启了cgroups-per-qos这个flag选项
	// 每个种类的qos 分别用一种cgroup
	pcm := kl.containerManager.NewPodContainerManager()
	// If pod has already been terminated then we need not create
	// or update the pod's cgroup
	if !kl.podIsTerminated(pod) {
		// When the kubelet is restarted with the cgroups-per-qos
		// flag enabled, all the pod's running containers
		// should be killed intermittently and brought back up
		// under the qos cgroup hierarchy.
		// Check if this is the pod's first sync
		firstSync := true
		for _, containerStatus := range apiPodStatus.ContainerStatuses {
			if containerStatus.State.Running != nil {
				firstSync = false
				break
			}
		}
		// Don't kill containers in pod if pod's cgroups already
		// exists or the pod is running for the first time
		// 第一次运行的pod或已经被qoscgroup控制的pod 就不需要被kill从而更新qoscgroup了
		podKilled := false
		if !pcm.Exists(pod) && !firstSync {
			if err := kl.killPod(pod, nil, podStatus, nil); err == nil {
				podKilled = true
			}
		}
		// pod被kill 且 他的RestartPolicy== Never，那么就不用更新此pod的qoscgroup了
		if !(podKilled && pod.Spec.RestartPolicy == v1.RestartPolicyNever) {
			if !pcm.Exists(pod) {
				if err := kl.containerManager.UpdateQOSCgroups(); err != nil {
					glog.V(2).Infof("Failed to update QoS cgroups while syncing pod: %v", err)
				}
				if err := pcm.EnsureExists(pod); err != nil {
					return fmt.Errorf("failed to ensure that the pod: %v cgroups exist and are correctly applied: %v", pod.UID, err)
				}
			}
		}
	}

	// Create Mirror Pod for Static Pod if it doesn't already exist
	if kubepod.IsStaticPod(pod) {
	   ......
	}
	// Make data directories for the pod
	// 再/var/lib/kubelet下给pod创建目录
	if err := kl.makePodDataDirs(pod); err != nil {
		glog.Errorf("Unable to make pod data directories for pod %q: %v", format.Pod(pod), err)
		return err
	}
	// Wait for volumes to attach/mount
	// 此函数是阻塞方法 直到所有volume都被mount上才会返回
	// 除非超时会返回错误
	if err := kl.volumeManager.WaitForAttachAndMount(pod); err != nil {
		kl.recorder.Eventf(pod, v1.EventTypeWarning, events.FailedMountVolume, "Unable to mount volumes for pod %q: %v", format.Pod(pod), err)
		glog.Errorf("Unable to mount volumes for pod %q: %v; skipping pod", format.Pod(pod), err)
		return err
	}
	// Fetch the pull secrets for the pod
	pullSecrets := kl.getPullSecretsForPod(pod)
	
	// Call the container runtime's SyncPod callback
	result := kl.containerRuntime.SyncPod(pod, apiPodStatus, podStatus, pullSecrets, kl.backOff)
	kl.reasonCache.Update(pod.UID, result)
	if err := result.Error(); err != nil {
		return err
	}
	return nil
}

```
managePodLoop中的`syncPodFn`实际上是`syncPod`，这个函数比较长，本文删减了部分非核心代码。现在这个`syncPod`通过传过来的syncPodOptions需要处理各类pod的更新事件。具体逻辑遵循以下流程
* 如果pod正在被创建，需要记录pod worker的从开始到现在的处理时间
* 调用`generateAPIPodStatus`生成一个给pod生成一个v1.PodStatus
* 当statusmanager里状态为pending 传过来的status位runing，记录pod从pending到running的时间
* 调用kl.canRunPod(pod)判断pod是否可以运行在当前节点，如果不行，将原因更新到pod状态，并同步到statusManager。
* 当pod不应该运行时调用kl.killPod删除pod
* 当pod时static时，为其创建mirror pod。
* 为pod在默认的/var/lib/kubelet目录中创建对应的目录。
* 等待pod spec中volume都被attach/mount
* 从apiserver中获取pull secrets（镜像仓库认证相关信息）
* 最后调用containerRuntime.SyncPod(...)完成pod在node上同步（为什么说是SyncPod同步而不是创建，因为每次都要通过pod的spec和statuc状态判断哪些container需要删除哪些需要创建）
至此一个pod的主流程就完成了，由于containerRuntime.SyncPod同样是异步的，因此并不需要等待所有pod的所有init container和container都创建完成才会返回，下次这个pod会在kubelet workQueue和pleg中得到再次同步，直到达到用户期望的状态。

#### plegCh

```golang
	case e := <-plegCh:
		if isSyncPodWorthy(e) {
			// PLEG event for a pod; sync it.
			if pod, ok := kl.podManager.GetPodByUID(e.ID); ok {
				glog.V(2).Infof("SyncLoop (PLEG): %q, event: %#v", format.Pod(pod), e)
				handler.HandlePodSyncs([]*v1.Pod{pod})
			} else {
				// If the pod no longer exists, ignore the event.
				glog.V(4).Infof("SyncLoop (PLEG): ignore irrelevant event: %#v", e)
			}
		}

		if e.Type == pleg.ContainerDied {
			if containerID, ok := e.Data.(string); ok {
				kl.cleanUpContainersInPod(e.ID, containerID)
			}
		}
		
func (kl *Kubelet) HandlePodSyncs(pods []*v1.Pod) {
	start := kl.clock.Now()
	for _, pod := range pods {
		mirrorPod, _ := kl.podManager.GetMirrorPodByPod(pod)
		kl.dispatchWork(pod, kubetypes.SyncPodSync, mirrorPod, start)
	}
}
```
plegCh接收pleg(pod lifecircle event generator)产生的事件，pleg调用relist每秒比较自己缓存中的所有pod的container状态和当前在node上运行的全部pod的container，如果pod中的container或pod本身container发生变化则会产生一条事件，从而触发pod同步。如果事件是`pleg.ContainerDied`，就会在apiserver中删除这个pod下的container。

#### syncCh
syncCh是golang的time.Ticker，每隔1秒钟会调用`getPodsToSync`取得所有要需要同步的pod，`getPodsToSync`主要是获取workQueue中的pod进行同步，`PodSyncLoopHandlers`目前还未见使用。那么这个workQueue是什么时候入队的呢，我们之前在podWorker的`wrapUp`会在每个pod同步完成后，再次放入到workQueue中等待`resyncInterval`间隔后同步，如果本次pod同步失败的话会有在`backOffPeriod`10s后再入队等待同步。
```golang
	case <-syncCh:
		// 当每秒钟的syncTicker到时，也会从getPodsToSync会从workqueue取出pod做重新同步
		// workqueue中的pod是每次sync后会再把pod放到queue中等待下次同步
		// Sync pods waiting for sync
		podsToSync := kl.getPodsToSync()
		if len(podsToSync) == 0 {
			break
		}
		glog.V(4).Infof("SyncLoop (SYNC): %d pods; %s", len(podsToSync), format.Pods(podsToSync))
		kl.HandlePodSyncs(podsToSync)

func (kl *Kubelet) getPodsToSync() []*v1.Pod {
	allPods := kl.podManager.GetPods()
	podUIDs := kl.workQueue.GetWork()
	podUIDSet := sets.NewString()
	for _, podUID := range podUIDs {
		podUIDSet.Insert(string(podUID))
	}
	var podsToSync []*v1.Pod
	for _, pod := range allPods {
		if podUIDSet.Has(string(pod.UID)) {
			// The work of the pod is ready
			podsToSync = append(podsToSync, pod)
			continue
		}
		for _, podSyncLoopHandler := range kl.PodSyncLoopHandlers {
			if podSyncLoopHandler.ShouldSync(pod) {
				podsToSync = append(podsToSync, pod)
				break
			}
		}
	}
	return podsToSync
}
```

#### kl.livenessManager.Updates()
当pod spec中配置了livenessProbe，kubelet的probeManager会开启一个独立的goroutine对配置探测点(http、tcp、exec)进行探测，probeManager中包含livenessManager(表示container是否存活)和readinessManager(表示container是否准备好接受访问或者服务)，因此当livenessManager探测到某个container`update.Result == proberesults.Failure`时，主循环会监听这个事件从而触发pod同步。
```golang
	case update := <-kl.livenessManager.Updates():
		if update.Result == proberesults.Failure {
			// The liveness manager detected a failure; sync the pod.
			// We should not use the pod from livenessManager, because it is never updated after
			// initialization.
			pod, ok := kl.podManager.GetPodByUID(update.PodUID)
			if !ok {
				// If the pod no longer exists, ignore the update.
				glog.V(4).Infof("SyncLoop (container unhealthy): ignore irrelevant update: %#v", update)
				break
			}
			glog.V(1).Infof("SyncLoop (container unhealthy): %q", format.Pod(pod))
			handler.HandlePodSyncs([]*v1.Pod{pod})
		}
```

#### housekeepingCh
housekeepingCh顾名思义，就是每隔`housekeepingPeriod`(hardcode为2秒钟)做一下清理工作，主要是以podmanager为准，清理podworker、probemanager、statusManage、podmanager中的孤儿mirrorpod、cgroupQOS目录，以removeOrphanedPodStatuses为准清理、rumtimecacahe、poddir

```golang
	case <-housekeepingCh:
		if !kl.sourcesReady.AllReady() {
			// If the sources aren't ready or volume manager has not yet synced the states,
			// skip housekeeping, as we may accidentally delete pods from unready sources.
			glog.V(4).Infof("SyncLoop (housekeeping, skipped): sources aren't ready yet.")
		} else {
			glog.V(4).Infof("SyncLoop (housekeeping)")
			if err := handler.HandlePodCleanups(); err != nil {
				glog.Errorf("Failed cleaning pods: %v", err)
			}
		}
```

最后我们注意到整个同步主循环syncLoop是在单个goroutine进行的，因此对任何channel的处理都不能是阻塞的，都应该是异步进行的，这样才能保证不会影响到其他channel的处理。

## 总结

kubelet可以说是一个典型的由事件驱动的异步架构。通过kubelet我们再次审视了kubernetes的设计理念，声明式设计保证kubelet总是去让系统向用户所期待的状态所演进。最后我们总结一下kubernetes三个重要的特性：
* 声明式(Declarative)：总是定义用户设定期望的状态。
* 水平触发(Level-triggered)：当事件触发后会产生通知，而在之后任意时间都会检测到这个被触发的事件。
* 异步(asynchronous)：由多个异步循环控制整个系统，使系统状态总是像用户期待的转变。