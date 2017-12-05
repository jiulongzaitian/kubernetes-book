### Scheduler初始化
```golang
func CreateScheduler(
	s *options.SchedulerServer,
	kubecli clientset.Interface,
	nodeInformer coreinformers.NodeInformer,
	podInformer coreinformers.PodInformer,
	pvInformer coreinformers.PersistentVolumeInformer,
	pvcInformer coreinformers.PersistentVolumeClaimInformer,
	replicationControllerInformer coreinformers.ReplicationControllerInformer,
	replicaSetInformer extensionsinformers.ReplicaSetInformer,
	statefulSetInformer appsinformers.StatefulSetInformer,
	serviceInformer coreinformers.ServiceInformer,
	recorder record.EventRecorder,
) (*scheduler.Scheduler, error) {
	// 初始化各种Informer，构造ConfigFactory(configurator的实现)
	configurator := factory.NewConfigFactory(
		s.SchedulerName,
		kubecli,
		nodeInformer,
		podInformer,
		pvInformer,
		pvcInformer,
		replicationControllerInformer,
		replicaSetInformer,
		statefulSetInformer,
		serviceInformer,
		s.HardPodAffinitySymmetricWeight,
		utilfeature.DefaultFeatureGate.Enabled(features.EnableEquivalenceClassCache),
	)
	// schedulerConfigurator 覆盖了Create()方法，原本默认走的是ConfigFactory的Create
	// 现在会检测是否有policyconfigfile。
	// Rebuild the configurator with a default Create(...) method.
	configurator = &schedulerConfigurator{
		configurator,
		s.PolicyConfigFile,
		s.AlgorithmProvider,
		s.PolicyConfigMapName,
		s.PolicyConfigMapNamespace,
		s.UseLegacyPolicyConfig,
	}

	return scheduler.NewFromConfigurator(configurator, func(cfg *scheduler.Config) {
		cfg.Recorder = recorder
	})
}
```
`factory.NewConfigFactory`创建了`ConfigFactory`对象(实现了Configurator接口)，这个对象里面主要各种informer的初始化，用来从kube-apiserver中同步各种资源的内容，用于后面调度算法。需要特别关注的是`ConfigFactory`两个重要的结构体成员：`PodQueue` 和 `schedulerCache`，在`ConfigFactory`的`podInformer`的中注册了两套处理函数用于监听kube-apiserver中的最新的`pod`信息，分别用于把未调度的`pod`放入`PodQueue`中以及把已经调度过的`pod`放入`schedulerCache`中作为调度的参考。`hardPodAffinitySymmetricWeight`和`enableEquivalenceClassCache`这两个参数用于调度后面算法中，此处先按下不表。

```golang
func NewConfigFactory(
	schedulerName string,
	client clientset.Interface,
	nodeInformer coreinformers.NodeInformer,
	podInformer coreinformers.PodInformer,
	pvInformer coreinformers.PersistentVolumeInformer,
	pvcInformer coreinformers.PersistentVolumeClaimInformer,
	replicationControllerInformer coreinformers.ReplicationControllerInformer,
	replicaSetInformer extensionsinformers.ReplicaSetInformer,
	statefulSetInformer appsinformers.StatefulSetInformer,
	serviceInformer coreinformers.ServiceInformer,
	hardPodAffinitySymmetricWeight int,
	enableEquivalenceClassCache bool,
) scheduler.Configurator {
	stopEverything := make(chan struct{})
	schedulerCache := schedulercache.New(30*time.Second, stopEverything)

	c := &ConfigFactory{
		client:                         client,
		podLister:                      schedulerCache,
		podQueue:                       cache.NewFIFO(cache.MetaNamespaceKeyFunc),
		pVLister:                       pvInformer.Lister(),
		pVCLister:                      pvcInformer.Lister(),
		serviceLister:                  serviceInformer.Lister(),
		controllerLister:               replicationControllerInformer.Lister(),
		replicaSetLister:               replicaSetInformer.Lister(),
		statefulSetLister:              statefulSetInformer.Lister(),
		schedulerCache:                 schedulerCache,
		StopEverything:                 stopEverything,
		schedulerName:                  schedulerName,
		hardPodAffinitySymmetricWeight: hardPodAffinitySymmetricWeight,
		enableEquivalenceClassCache:    enableEquivalenceClassCache,
	}

	c.scheduledPodsHasSynced = podInformer.Informer().HasSynced
	// scheduled pod cache
	podInformer.Informer().AddEventHandler(
		cache.FilteringResourceEventHandler{
			FilterFunc: func(obj interface{}) bool {
				switch t := obj.(type) {
				case *v1.Pod:
					return assignedNonTerminatedPod(t)
				default:
					runtime.HandleError(fmt.Errorf("unable to handle object in %T: %T", c, obj))
					return false
				}
			},
			Handler: cache.ResourceEventHandlerFuncs{
				AddFunc:    c.addPodToCache,
				UpdateFunc: c.updatePodInCache,
				DeleteFunc: c.deletePodFromCache,
			},
		},
	)
	// unscheduled pod queue
	podInformer.Informer().AddEventHandler(
		cache.FilteringResourceEventHandler{
			FilterFunc: func(obj interface{}) bool {
				switch t := obj.(type) {
				case *v1.Pod:
					return unassignedNonTerminatedPod(t)
				default:
					runtime.HandleError(fmt.Errorf("unable to handle object in %T: %T", c, obj))
					return false
				}
			},
			Handler: cache.ResourceEventHandlerFuncs{
				AddFunc: func(obj interface{}) {
					if err := c.podQueue.Add(obj); err != nil {
						runtime.HandleError(fmt.Errorf("unable to queue %T: %v", obj, err))
					}
				},
				UpdateFunc: func(oldObj, newObj interface{}) {
					if err := c.podQueue.Update(newObj); err != nil {
						runtime.HandleError(fmt.Errorf("unable to update %T: %v", newObj, err))
					}
				},
				DeleteFunc: func(obj interface{}) {
					if err := c.podQueue.Delete(obj); err != nil {
						runtime.HandleError(fmt.Errorf("unable to dequeue %T: %v", obj, err))
					}
				},
			},
		},
	)
	......
```
回到`CreateScheduler()`中，构造`schedulerConfigurator`对象时使用了之前生成的`configurator`对象，在`schedulerConfigurator`中主要包含调度算法配置相关的字段，因为k8s支持使用默认提供的调度算法，同时也支持读取用户配置文件中指定的调度算法和自定义调度算法。最后调用`scheduler.NewFromConfigurator`函数传入之前创建的`configurator`和一个简单的赋值函数后就可以生成我们调度需要的`Scheduler`对象。

```golang
// NewFromConfigurator returns a new scheduler that is created entirely by the Configurator.  Assumes Create() is implemented.
// Supports intermediate Config mutation for now if you provide modifier functions which will run after Config is created.
func NewFromConfigurator(c Configurator, modifiers ...func(c *Config)) (*Scheduler, error) {
	cfg, err := c.Create()
	if err != nil {
		return nil, err
	}
	// Mutate it if any functions were provided, changes might be required for certain types of tests (i.e. change the recorder).
	for _, modifier := range modifiers {
		modifier(cfg)
	}
	// From this point on the config is immutable to the outside.
	s := &Scheduler{
		config: cfg,
	}
	metrics.Register()
	return s, nil
}

```
`NewFromConfigurator`调用了之前传过来的`configurator`的Create()方法生成`Scheduler`的`config`，之后用我们之前传过来的匿名函数`func(cfg *scheduler.Config) {cfg.Recorder = recorder}`给`cfg`的`Recorder`赋值，最后返回
`Scheduler`的对象。

```golang
// Create implements the interface for the Configurator, hence it is exported
// even though the struct is not.
func (sc schedulerConfigurator) Create() (*scheduler.Config, error) {
	policy, err := sc.getSchedulerPolicyConfig()
	if err != nil {
		return nil, err
	}
	// If no policy is found, create scheduler from algorithm provider.
	if policy == nil {
		if sc.Configurator != nil {
			return sc.Configurator.CreateFromProvider(sc.algorithmProvider)
		}
		return nil, fmt.Errorf("Configurator was nil")
	}

	return sc.CreateFromConfig(*policy)
}
```
`Create()`方法会检查用户是否自定义调度算法的配置文件已经相关的配置项（schedulerConfigurator中的相关项是由SchedulerServer中解析用户命令行参数构造的），如果没有定义则使用默认的`algorithmProvider`创建`scheduler.Config`。

```golang
// Creates a scheduler from the name of a registered algorithm provider.
func (f *ConfigFactory) CreateFromProvider(providerName string) (*scheduler.Config, error) {
	glog.V(2).Infof("Creating scheduler from algorithm provider '%v'", providerName)
	provider, err := GetAlgorithmProvider(providerName)
	if err != nil {
		return nil, err
	}

	return f.CreateFromKeys(provider.FitPredicateKeys, provider.PriorityFunctionKeys, []algorithm.SchedulerExtender{})
}
```
`CreateFromProvider`中首先根据`providerName`获取`provider`，`providerName`也是通过命令行传入到`schedulerConfigurator`中，默认值是"DefaultProvider"，然后`GetAlgorithmProvider`使用之前已经注册完成的`algorithmProviderMap`返回`provider`实例。实际上`algorithmProviderMap`在`plugin/cmd/kube-scheduler/app/server.go`中`import _ "k8s.io/kubernetes/plugin/pkg/scheduler/algorithmprovider"`里就已经注册完成了，这里使用的golang的package的init()函数特性，实际最后完成注册逻辑是在`plugin/pkg/scheduler/algorithmprovider/defaults/defaults.go`文件中。之后分析每个调度算法时会重点关注`defaults.go`这个文件。回到之前的流程，得到`provider`后就直接调用`f.CreateFromKeys`返回`scheduler.Config`。

```golang
// Creates a scheduler from a set of registered fit predicate keys and priority keys.
func (f *ConfigFactory) CreateFromKeys(predicateKeys, priorityKeys sets.String, extenders []algorithm.SchedulerExtender) (*scheduler.Config, error) {
	glog.V(2).Infof("Creating scheduler with fit predicates '%v' and priority functions '%v", predicateKeys, priorityKeys)

	if f.GetHardPodAffinitySymmetricWeight() < 1 || f.GetHardPodAffinitySymmetricWeight() > 100 {
		return nil, fmt.Errorf("invalid hardPodAffinitySymmetricWeight: %d, must be in the range 1-100", f.GetHardPodAffinitySymmetricWeight())
	}
	//用字符串set生成具体调度算法
	predicateFuncs, err := f.GetPredicates(predicateKeys)
	if err != nil {
		return nil, err
	}

	priorityConfigs, err := f.GetPriorityFunctionConfigs(priorityKeys)
	if err != nil {
		return nil, err
	}
	// priorityMetaProducer和predicateMetaProducer都是在default里注册工厂方法，然后调用GetPriorityMetadataProducer()
	// 传入f.getPluginArgs()由得到的PluginFactoryArgs生成PriorityMetadata()函数，最后生成metadata
	priorityMetaProducer, err := f.GetPriorityMetadataProducer()
	if err != nil {
		return nil, err
	}

	predicateMetaProducer, err := f.GetPredicateMetadataProducer()
	if err != nil {
		return nil, err
	}

	// Init equivalence class cache
	if f.enableEquivalenceClassCache && getEquivalencePodFunc != nil {
		f.equivalencePodCache = core.NewEquivalenceCache(getEquivalencePodFunc)
		glog.Info("Created equivalence class cache")
	}
	algo := core.NewGenericScheduler(f.schedulerCache, f.equivalencePodCache, predicateFuncs, predicateMetaProducer, priorityConfigs, priorityMetaProducer, extenders)

	podBackoff := util.CreateDefaultPodBackoff()
	return &scheduler.Config{
		SchedulerCache: f.schedulerCache,
		Ecache:         f.equivalencePodCache,
		// The scheduler only needs to consider schedulable nodes.
		NodeLister:          &nodePredicateLister{f.nodeLister},
		Algorithm:           algo,
		Binder:              f.getBinder(extenders),
		PodConditionUpdater: &podConditionUpdater{f.client},
		WaitForCacheSync: func() bool {
			return cache.WaitForCacheSync(f.StopEverything, f.scheduledPodsHasSynced)
		},
		NextPod: func() *v1.Pod {
			return f.getNextPod()
		},
		Error:          f.MakeDefaultErrorFunc(podBackoff, f.podQueue),
		StopEverything: f.StopEverything,
	}, nil
}
```
`CreateFromKeys()`首先验证了`hardPodAffinitySymmetricWeight`参数是否在1-100范围内，此参数用在`priority`的`CalculateInterPodAffinityPriority`算法中，用于选出在相同拓扑域下和`pod`亲和度最大的`node`。`f.GetPredicates(predicateKeys)`通过`predicate`算法的name集合和在`default.go`中注册的每个`predicate`算法的工厂算法来生成名字-算法一一对应的`map[string]algorithm.FitPredicate`。`predicateMetaProducer`是为所有调度算法提前计算一些公用的数据例如pod的资源请求量，pod使用的hostPort等等。`equivalencePodCache`用于优化`predicate`算法，其缓存了由同一个controller（如ReplicaController或ReplicaSet）复制出来的多个pod不用重复应用所有`predicate`算法而是使用第一个pod调度结果就可以了，目前`equivalencePodCache`只应用到`predicate`算法中。`NewGenericScheduler()`生成了具体用于执行调度算法的实例`genericScheduler`。`podBackoff`用于pod调度失败后控制重新调度的间隔，每次默认间隔都会加倍直到增加到间隔最大值。`scheduler.Config`中其它关键字段含义如下

* `Binder`用于将调度完成的`pod`和`node`生一个`Binding`写入到kube-apiserver中，这个过程称之为Bind。
* `PodConditionUpdater`把pod最新状态写会到kube-apiserver中。
* `WaitForCacheSync`等待`podinformer`中`fifo`缓存队列中的全部被取走，`podinformer`就绪。
* `NextPod`是从pod的未调度队中取出一个最新的`pod`。
* `Error`是处理pod调度失败的函数，间隔podBackoff时间后会重新将`pod`加入到未调度队列中。

至此，调度需要的所有组件已经构建或者初始化完成，之后的`sched.Run()`就会进入打到核心的调度循环。
