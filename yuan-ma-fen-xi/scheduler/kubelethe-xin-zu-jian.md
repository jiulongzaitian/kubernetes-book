### 核心组件分析
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
