## kube-scheduler源码分析
```
                                作者：李昂  邮箱：liang@haihangyun.com
```

## kube-scheduler作用

Kube-scheduler作为组件运行在`master`节点，主要任务是把从kube-apiserver中获取的未被调度的`pod`调度到最适合的`node`上。Kube-scheduler以插件形式运行，用户如果觉得官方的不满足需求还可以自己实现调度插件。目前Kubernetes同时支持多个kube-scheduler同时运行，以调度器名字区分，每个`pod`的spec文件中可以指定调度自己的kube-scheduler的`schedulerName `。

Kube-scheduler主要调度框架其实就是输入`pod`，经过一系列Predicate和Priority调度算法后排序每个`node`的得分最后输出得分最高的`node`。

## 代码简析
> 注：本文基于kubernetes版本为`GitVersion:"v1.9.0-alpha.0.572+9636522137039d", GitCommit:"9636522137039d48555d2441725133d69805e010"`。截止到本文完成时，kube-scheduler出了全新的Pod优先级和抢占机制，主流程可能会和本文略有出入。 
### 主启动流程

程序入口在plugin/cmd/kube-scheduler/scheduler.go:

```golang
func main() {
    s := options.NewSchedulerServer()
    s.AddFlags(pflag.CommandLine)

    flag.InitFlags()
    logs.InitLogs()
    defer logs.FlushLogs()

    verflag.PrintAndExitIfRequested()

    if err := app.Run(s); err != nil {
        glog.Fatalf("scheduler app failed to run: %v", err)
    }
}
```

`NewSchedulerServer()`返回一个可以解析命令行配置的对象，运行命令时传递的参数都会加载到对应的字段中，如果命令行参数中没有指定则会使用默认值。同时由于kube-apiserver里对象是分Group、Version、Kind，类似`core v1 pod`，`batch v2alpha1 CronJob`, `extensions v1beta1 DaemonSet`等，所以`NewSchedulerServer()`还需要将不同`Version`的参数转换组合起来成完整的对象。

得到到完整的`SchedulerServer`对象后，就可以直接进入`app.Run(s)`，此函数会一直运行直到发生`panic`或者进程被kill掉。

```golang
func Run(s *options.SchedulerServer) error {
	kubecli, err := createClient(s)
	if err != nil {
		return fmt.Errorf("unable to create kube client: %v", err)
	}

	recorder := createRecorder(kubecli, s)

	informerFactory := informers.NewSharedInformerFactory(kubecli, 0)
	// cache only non-terminal pods
	podInformer := factory.NewPodInformer(kubecli, 0)

	sched, err := CreateScheduler(
		s,
		kubecli,
		//这种链式调用方式区分不同的Group，Version，Kind和实际的k8s对象，非常清晰。CORE()调用core包生成一个
        //新的对象group对象，其中包含informerFactory，之后V1()再次生成version对象
		//同样包含informerFactory，最后version直接生成&nodeInformer{factory: v.SharedInformerFactory}对象。
		//也就是整个nodeinformer的创建是，先通过&nodeInformer{factory: v.SharedInformerFactory}直接生成，再通过
		//nodeInformer的Informer()函数f.factory.InformerFor(&core_v1.Node{}, defaultNodeInformer)生成。
		informerFactory.Core().V1().Nodes(),
		podInformer,
		informerFactory.Core().V1().PersistentVolumes(),
		informerFactory.Core().V1().PersistentVolumeClaims(),
		informerFactory.Core().V1().ReplicationControllers(),
		informerFactory.Extensions().V1beta1().ReplicaSets(),
		informerFactory.Apps().V1beta1().StatefulSets(),
		informerFactory.Core().V1().Services(),
		recorder,
	)
    //开启了golang内置的性能测量和用于Prometheus抓取数据的url
	go startHTTP(s)

	stop := make(chan struct{})
	defer close(stop)
	go podInformer.Informer().Run(stop)
	informerFactory.Start(stop)
	// Waiting for all cache to sync before scheduling.
	informerFactory.WaitForCacheSync(stop)
	controller.WaitForCacheSync("scheduler", stop, podInformer.Informer().HasSynced)

	run := func(_ <-chan struct{}) {
		sched.Run()
		select {}
	}

	if !s.LeaderElection.LeaderElect {
		run(nil)
		panic("unreachable")
	}
	......
	
	leaderelection.RunOrDie(leaderelection.LeaderElectionConfig{
		Lock:          rl,
		LeaseDuration: s.LeaderElection.LeaseDuration.Duration,
		RenewDeadline: s.LeaderElection.RenewDeadline.Duration,
		RetryPeriod:   s.LeaderElection.RetryPeriod.Duration,
		Callbacks: leaderelection.LeaderCallbacks{
			OnStartedLeading: run,
			OnStoppedLeading: func() {
				glog.Fatalf("lost master")
			},
		},
	})

	panic("unreachable")


```

`app.Run()`首先创建了一个Clientset对象kubecli，关于client-go这个包是从原来kube-apiserver中抽离出来的项目，通过高级的抽象和封装（informer和listener）用于对apiserver中的对象进行高效的增删改查以及watch，具体的使用方式可以见[这篇文章](https://www.kubernetes.org.cn/1309.html)。`recorder`是用来向kube-apiserver写入调度过程中发生的`Event`。然后通过`informerFactory`创建了各种类型的`informer`，用于在调度过程中对kube-apiserver各种资源的获取，由于`podInformer`只需要关注非terminal状态的`pod`所以没有选择`informerFactory`通用的方式创建。而后`CreateScheduler()`会创建真正用于运行调度的`Scheduler`对象。得到已经初始化完成的`sched`后启动各个`informer`，等待`informer`将handler将本地缓存的全部处理完毕后，就正式启动了kube-scheduler主循环`sched.Run()`。

`leaderelection`是k8s中一个通用的选举框架，用于kube-controller-manager和kube-scheduler这种只能同时存在一个实例的进程实现高可用。`leaderelection`相当于借助etcd这类高可用一致性系统实现自身的主从选举，原理还是比较简单的，而如果原生实现一套高可用架构的话势必会需要paxos或者raft这类选举算法的支持。在client-go的相关文章会详细介绍`leaderelection`模块。

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

### 调度流程

调度过程主要分为三步：

* `schedule()`：执行`predicate`调度算法和`priority`调度算法，对选出的`node`按得分大小排序，返回得分最高的`nodeName`。
* `assume()`：如果调度成功则将`pod`写入本地缓存`schedulerCache`，同时对本地缓冲中`node`资源占用进行更新，以便下次调度时以此最新的缓存作为参考。
* `bind()`：异步将`pod`的Binding信息写入到kube-apiserver中。一次调度完成。
![scheduler-flowchart](http://ovfdoadvt.bkt.clouddn.com/%E4%B8%BB%E6%B5%81%E7%A8%8B.png)

```golang
// Run begins watching and scheduling. It waits for cache to be synced, then starts a goroutine and returns immediately.
func (sched *Scheduler) Run() {
	if !sched.config.WaitForCacheSync() {
		return
	}

	go wait.Until(sched.scheduleOne, 0, sched.config.StopEverything)
}
......
// scheduleOne does the entire scheduling workflow for a single pod.  It is serialized on the scheduling algorithm's host fitting.
func (sched *Scheduler) scheduleOne() {
	// NextPod()如果podQueue为空则会一直阻塞直到有pod可以pop
	pod := sched.config.NextPod()
	if pod.DeletionTimestamp != nil {
		sched.config.Recorder.Eventf(pod, v1.EventTypeWarning, "FailedScheduling", "skip schedule deleting pod: %v/%v", pod.Namespace, pod.Name)
		glog.V(3).Infof("Skip schedule deleting pod: %v/%v", pod.Namespace, pod.Name)
		return
	}

	glog.V(3).Infof("Attempting to schedule pod: %v/%v", pod.Namespace, pod.Name)

	// Synchronously attempt to find a fit for the pod.
	start := time.Now()
	suggestedHost, err := sched.schedule(pod)
	metrics.SchedulingAlgorithmLatency.Observe(metrics.SinceInMicroseconds(start))
	if err != nil {
		return
	}
	// 调度算法执行完就马上向cache中写入pod，这样的好处在于当调度大量pod的时候不用等待pod已经在node上运行，下次调度算法就可以算上之前这个pod。
	// Tell the cache to assume that a pod now is running on a given node, even though it hasn't been bound yet.
	// This allows us to keep scheduling without waiting on binding to occur.
	err = sched.assume(pod, suggestedHost)
	if err != nil {
		return
	}

	// bind the pod to its host asynchronously (we can do this b/c of the assumption step above).
	go func() {
		err := sched.bind(pod, &v1.Binding{
			ObjectMeta: metav1.ObjectMeta{Namespace: pod.Namespace, Name: pod.Name, UID: pod.UID},
			Target: v1.ObjectReference{
				Kind: "Node",
				Name: suggestedHost,
			},
		})
		metrics.E2eSchedulingLatency.Observe(metrics.SinceInMicroseconds(start))
		if err != nil {
			glog.Errorf("Internal error binding pod: (%v)", err)
		}
	}()
}
```
#### 1. schedule()
```golang
// schedule implements the scheduling algorithm and returns the suggested host.
func (sched *Scheduler) schedule(pod *v1.Pod) (string, error) {
	host, err := sched.config.Algorithm.Schedule(pod, sched.config.NodeLister)
	if err != nil {
		glog.V(1).Infof("Failed to schedule pod: %v/%v", pod.Namespace, pod.Name)
		copied, cerr := api.Scheme.Copy(pod)
		if cerr != nil {
			runtime.HandleError(err)
			return "", err
		}
		pod = copied.(*v1.Pod)
		sched.config.Error(pod, err)
		sched.config.Recorder.Eventf(pod, v1.EventTypeWarning, "FailedScheduling", "%v", err)
		sched.config.PodConditionUpdater.Update(pod, &v1.PodCondition{
			Type:    v1.PodScheduled,
			Status:  v1.ConditionFalse,
			Reason:  v1.PodReasonUnschedulable,
			Message: err.Error(),
		})
		return "", err
	}
	return host, err
}
```
`schedule`通过调用`genericScheduler`（实现了`ScheduleAlgorithm`接口）的`Schedule`方法，完成调度流程，最终的返回是一个string类型`nodeName`。当调度过程发生错误，先使用`api.Scheme.Copy(pod)`复制一份`pod`对象，然后调用`sched.config.Error`进行错误处理，实际调用的是`MakeDefaultErrorFunc`。之后向kube-apiserver记录`Event`以及更新`pod`状态就完成了`schedule()`的错误处理部分。

```golang
func (factory *ConfigFactory) MakeDefaultErrorFunc(backoff *util.PodBackoff, podQueue *cache.FIFO) func(pod *v1.Pod, err error) {
	return func(pod *v1.Pod, err error) {
		if err == core.ErrNoNodesAvailable {
			glog.V(4).Infof("Unable to schedule %v %v: no nodes are registered to the cluster; waiting", pod.Namespace, pod.Name)
		} else {
			if _, ok := err.(*core.FitError); ok {
				glog.V(4).Infof("Unable to schedule %v %v: no fit: %v; waiting", pod.Namespace, pod.Name, err)
			} else {
				glog.Errorf("Error scheduling %v %v: %v; retrying", pod.Namespace, pod.Name, err)
			}
		}
		backoff.Gc()
		// Retry asynchronously.
		// Note that this is extremely rudimentary and we need a more real error handling path.
		go func() {
			defer runtime.HandleCrash()
			podID := types.NamespacedName{
				Namespace: pod.Namespace,
				Name:      pod.Name,
			}

			entry := backoff.GetEntry(podID)
			if !entry.TryWait(backoff.MaxDuration()) {
				glog.Warningf("Request for pod %v already in flight, abandoning", podID)
				return
			}
			// Get the pod again; it may have changed/been scheduled already.
			getBackoff := initialGetBackoff
			for {
				pod, err := factory.client.Core().Pods(podID.Namespace).Get(podID.Name, metav1.GetOptions{})
				if err == nil {
					if len(pod.Spec.NodeName) == 0 {
						podQueue.AddIfNotPresent(pod)
					}
					break
				}
				if errors.IsNotFound(err) {
					glog.Warningf("A pod %v no longer exists", podID)
					return
				}
				glog.Errorf("Error getting pod %v for retry: %v; retrying...", podID, err)
				if getBackoff = getBackoff * 2; getBackoff > maximalGetBackoff {
					getBackoff = maximalGetBackoff
				}
				time.Sleep(getBackoff)
			}
		}()
	}
}
```
在`sched.config.Error`中首先判断错误种类打印日志。podbackoff中保存了每个`pod`对应的重新加入队列的间隔，每次加入间隔时间会加倍直到最大间隔。`backoff.Gc()`清理backoff中已经超过最大存活时间的条目。`entry.TryWait(backoff.MaxDuration())`会首先获取锁然后将上次的间隔时间翻倍后Sleep新的间隔，如果获取锁失败说明此`pod`已经在其他协程中Sleep则直接返回。之后重新从kube-apiserver中获取`pod`，当`pod`还没有被调度时（`len(pod.Spec.NodeName) == 0`）就可以把pod重新放入到未调度队列`podQueue`。如果从kube-apiserver获取失败，等待一段时间重新获取直到成功。

#### 2. assume()
```golang
// assume signals to the cache that a pod is already in the cache, so that binding can be asnychronous.
func (sched *Scheduler) assume(pod *v1.Pod, host string) error {
	//......
	assumed := *pod
	assumed.Spec.NodeName = host
	if err := sched.config.SchedulerCache.AssumePod(&assumed); err != nil {
		glog.Errorf("scheduler cache AssumePod failed: %v", err)
		//......
		return err
	}
	// Optimistically assume that the binding will succeed, so we need to invalidate affected
	// predicates in equivalence cache.
	// If the binding fails, these invalidated item will not break anything.
	if sched.config.Ecache != nil {
		sched.config.Ecache.InvalidateCachedPredicateItemForPodAdd(pod, host)
	}
	return nil
}
```
`assume()`主要做了两个事情，首先调用`SchedulerCache`的`AssumePod()`，将`pod`的最新调度结果写入`SchedulerCache`，并更新schedulerCache中对应`node`的资源占用和`hostPort`端口占用。
然后使`equivalencePodCache`中的`GeneralPredicates`算法对应的缓存失效。因为`GeneralPredicates`算法包含了对`node`资源和端口占用的检查，当有新的`pod`调度在`node`上这两项资源都会得到更新，所以原始缓存必须要失效。

#### 3. bind()
```golang
// bind binds a pod to a given node defined in a binding object.  We expect this to run asynchronously, so we
// handle binding metrics internally.
func (sched *Scheduler) bind(assumed *v1.Pod, b *v1.Binding) error {
	bindingStart := time.Now()
	// If binding succeeded then PodScheduled condition will be updated in apiserver so that
	// it's atomic with setting host.
	err := sched.config.Binder.Bind(b)
	if err != nil {
		glog.V(1).Infof("Failed to bind pod: %v/%v", assumed.Namespace, assumed.Name)
		//如果binding失败，则从cache中删除之前assume的pod
		if err := sched.config.SchedulerCache.ForgetPod(assumed); err != nil {
			return fmt.Errorf("scheduler cache ForgetPod failed: %v", err)
		}
		//ConfigFactory.MakeDefaultErrorFunc(podBackoff, f.podQueue)
		sched.config.Error(assumed, err)
		sched.config.Recorder.Eventf(assumed, v1.EventTypeWarning, "FailedScheduling", "Binding rejected: %v", err)
		sched.config.PodConditionUpdater.Update(assumed, &v1.PodCondition{
			Type:   v1.PodScheduled,
			Status: v1.ConditionFalse,
			Reason: "BindingRejected",
		})
		return err
	}

	//给pod加上deadline可以被删除
	if err := sched.config.SchedulerCache.FinishBinding(assumed); err != nil {
		return fmt.Errorf("scheduler cache FinishBinding failed: %v", err)
	}

	metrics.BindingLatency.Observe(metrics.SinceInMicroseconds(bindingStart))
	sched.config.Recorder.Eventf(assumed, v1.EventTypeNormal, "Scheduled", "Successfully assigned %v to %v", assumed.Name, b.Target.Name)
	return nil
}
```
`bind()`是异步进行的，即`assume()`调用完成就会进入下一轮新的调度循环。`sched.config.Binder.Bind(b)`将`Binding`这个对象写入到kube-apiserver中。如果写入发生错误，首先`SchedulerCache.ForgetPod(assumed)`将之前`assume()`加入到缓存中的pod删除，清除对应`node`资源占用。`sched.config.Error(assumed, err)`跟`schedule()`的错误处理相同，将失败的`pod`等一个间隔时间后重新放入未调度`pod`队列中。生成`Event`事件和更新`pod`状态与之前类似。而`sched.config.Binder.Bind(b)`成功写入kube-apiserver后，会调用`SchedulerCache.FinishBinding(assumed)`，其主要作用是给`SchedulerCache`中的那些通过assume的`pod`加上过期时间ttl，从而可以定时清除他们释放内存占用，而之后调度的时候也并不会完全依赖这些`SchedulerCache`中assume的`pod`，因为当`pod`成功在`node`上创建后，`Scheduler.Config`中的`NodeLister`会定时同步全部`node`节点状态和资源占用，这些信息才是最准确的。至此，一个完整的调度循环就结束了。

### 调度算法
kube-scheduler的调度分为predicate和priority两类算法，predicate用于将不符合`pod`调度条件的`node`剔除出待调度`node`列表；而priority从剩下的`node`中应用多个算法将每个`node`按0-10评分。这样得分最高的`node`具有最高优先级。

```golang
// Schedule tries to schedule the given pod to one of node in the node list.
// If it succeeds, it will return the name of the node.
// If it fails, it will return a Fiterror error with reasons.
func (g *genericScheduler) Schedule(pod *v1.Pod, nodeLister algorithm.NodeLister) (string, error) {
	trace := utiltrace.New(fmt.Sprintf("Scheduling %s/%s", pod.Namespace, pod.Name))
	defer trace.LogIfLong(100 * time.Millisecond)

	nodes, err := nodeLister.List()
	if err != nil {
		return "", err
	}
	if len(nodes) == 0 {
		return "", ErrNoNodesAvailable
	}

	// Used for all fit and priority funcs.
	// 每次调度之前更新到一个最新的node信息的map，每个nodeInfo都会有generation属性
	// 每次nodeInfo有更新的话generation会加1
	err = g.cache.UpdateNodeNameToInfoMap(g.cachedNodeInfoMap)
	if err != nil {
		return "", err
	}

	trace.Step("Computing predicates")
	// 调度前需要两部分node，一个是从apiserver里监听到的最新的nodes
	// 另一部分是schedulercache找到的已经assumepod了的那些node g.cachedNodeInfoMap
	filteredNodes, failedPredicateMap, err := findNodesThatFit(pod, g.cachedNodeInfoMap, nodes, g.predicates, g.extenders, g.predicateMetaProducer, g.equivalenceCache)
	if err != nil {
		return "", err
	}

	if len(filteredNodes) == 0 {
		return "", &FitError{
			Pod:              pod,
			FailedPredicates: failedPredicateMap,
		}
	}

	trace.Step("Prioritizing")
	metaPrioritiesInterface := g.priorityMetaProducer(pod, g.cachedNodeInfoMap)
	priorityList, err := PrioritizeNodes(pod, g.cachedNodeInfoMap, metaPrioritiesInterface, g.prioritizers, filteredNodes, g.extenders)
	if err != nil {
		return "", err
	}

	trace.Step("Selecting host")
	return g.selectHost(priorityList)
}
```
`Schedule`大体是比较清晰的，`findNodesThatFit`使用注册的所有predicate算法进行过滤，留下适合待调度`pod`的`node`列表以及不适合调度`node`原因。`PrioritizeNodes`使用注册的priority算法对之前过滤后的每个`node`进行打分，最终`g.selectHost`选出得分最高的`node`（如果多个node得分相同且都是最高，为了调度更加平均`g.selectHost`进行了优化）。

#### Predicate
```golang
// Filters the nodes to find the ones that fit based on the given predicate functions
// Each node is passed through the predicate functions to determine if it is a fit
func findNodesThatFit(
	pod *v1.Pod,
	nodeNameToInfo map[string]*schedulercache.NodeInfo,
	nodes []*v1.Node,
	predicateFuncs map[string]algorithm.FitPredicate,
	extenders []algorithm.SchedulerExtender,
	metadataProducer algorithm.MetadataProducer,
	ecache *EquivalenceCache,
) ([]*v1.Node, FailedPredicateMap, error) {
	var filtered []*v1.Node
	failedPredicateMap := FailedPredicateMap{}

	if len(predicateFuncs) == 0 {
		filtered = nodes
	} else {
		// Create filtered list with enough space to avoid growing it
		// and allow assigning.
		filtered = make([]*v1.Node, len(nodes))
		errs := errors.MessageCountMap{}
		var predicateResultLock sync.Mutex
		var filteredLen int32

		// We can use the same metadata producer for all nodes.
		// pkg/scheduler/algorithm/predicates/metadata.go:GetMetadata()
		meta := metadataProducer(pod, nodeNameToInfo)
		checkNode := func(i int) {
			nodeName := nodes[i].Name
			fits, failedPredicates, err := podFitsOnNode(pod, meta, nodeNameToInfo[nodeName], predicateFuncs, ecache)
			if err != nil {
				predicateResultLock.Lock()
				errs[err.Error()]++
				predicateResultLock.Unlock()
				return
			}
			if fits {
				filtered[atomic.AddInt32(&filteredLen, 1)-1] = nodes[i]
			} else {
				predicateResultLock.Lock()
				failedPredicateMap[nodeName] = failedPredicates
				predicateResultLock.Unlock()
			}
		}
		workqueue.Parallelize(16, len(nodes), checkNode)
		filtered = filtered[:filteredLen]
		if len(errs) > 0 {
			return []*v1.Node{}, FailedPredicateMap{}, errors.CreateAggregateFromMessageCountMap(errs)
		}
	}
	......
	return filtered, failedPredicateMap, nil
}

......

func Parallelize(workers, pieces int, doWorkPiece DoWorkPieceFunc) {
	toProcess := make(chan int, pieces)
	for i := 0; i < pieces; i++ {
		toProcess <- i
	}
	close(toProcess)

	if pieces < workers {
		workers = pieces
	}

	wg := sync.WaitGroup{}
	wg.Add(workers)
	for i := 0; i < workers; i++ {
		go func() {
			defer utilruntime.HandleCrash()
			defer wg.Done()
			for piece := range toProcess {
				doWorkPiece(piece)
			}
		}()
	}
	wg.Wait()
}

```
`findNodesThatFit`首先初始化了过滤的`node`列表以及并发所需要的锁，`meta`是提前准备一些通用的调度元数据。在对每个`node`进行判断时，使用的一个生产者消费者模型的并发框架`workqueue.Parallelize`，这里并发goroutine数hardcode为16，对每个`node[i]`调用`checkNode(i)`检验是否符合要求。这里使用了`atomic.AddInt32`来保证在`filterd`中增加`node`的原子性。

```golang
// Checks whether node with a given name and NodeInfo satisfies all predicateFuncs.
func podFitsOnNode(pod *v1.Pod, meta interface{}, info *schedulercache.NodeInfo, predicateFuncs map[string]algorithm.FitPredicate,
	ecache *EquivalenceCache) (bool, []algorithm.PredicateFailureReason, error) {
	var (
		equivalenceHash  uint64
		failedPredicates []algorithm.PredicateFailureReason
		eCacheAvailable  bool
		invalid          bool
		fit              bool
		reasons          []algorithm.PredicateFailureReason
		err              error
	)
	if ecache != nil {
		// getHashEquivalencePod will return immediately if no equivalence pod found
		// 先用plugin/pkg/scheduler/algorithmprovider/defaults/defaults.go#GetEquivalencePod()取出pod的OwnerReferences，
		// 再计算hash值，判断每个pod是否是相同的controller创建出来这，这样如果hash值相同，那就直接用EquivalenceCache缓存中的结果，
		// 这种EquivalenceCache只用在predicate中，因为相同controller的pod必然适用于之前pod的node，但是priority就不一定适用了
		equivalenceHash = ecache.getHashEquivalencePod(pod)
		eCacheAvailable = (equivalenceHash != 0)
	}
	for predicateKey, predicate := range predicateFuncs {
		// If equivalenceCache is available
		if eCacheAvailable {
			// PredicateWithECache will returns it's cached predicate results
			fit, reasons, invalid = ecache.PredicateWithECache(pod, info.Node().GetName(), predicateKey, equivalenceHash)
		}
		if !eCacheAvailable || invalid {
			// we need to execute predicate functions since equivalence cache does not work
			fit, reasons, err = predicate(pod, meta, info)
			if err != nil {
				return false, []algorithm.PredicateFailureReason{}, err
			}
			if eCacheAvailable {
				// update equivalence cache with newly computed fit & reasons
				// TODO(resouer) should we do this in another thread? any race?
				ecache.UpdateCachedPredicateItem(pod, info.Node().GetName(), predicateKey, fit, reasons, equivalenceHash)
			}
		}
		if !fit {
			// eCache is available and valid, and predicates result is unfit, record the fail reasons
			failedPredicates = append(failedPredicates, reasons...)
		}
	}
	return len(failedPredicates) == 0, failedPredicates, nil
}
```

`podFitsOnNode`在顺序调用每个predicate算法之前，会首先查看`EquivalenceCache`是否有记录，其作用是相同controller生成的`pod`只会用第一个`pod`的结果，从而优化了调度执行时间。
如果用户没有指定`AlgorithmProvider`使用的默认的`DefaultProvider`，每个predicate算法都会传入三个参数：

1. pod：待调度的`pod`
1. meta：调度需要的元数据包含：`pod`的[QoS](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/resource-qos.md)等级是否为BestEffort（关于k8s的QoS可以先查看官方文档的说明）；`pod`的各种资源请求量；`pod`的HostPort（与宿主机端口映射占用的端口）；待调度`pod`与所有`node`上运行的`pod`的AntiAffinity匹配（关于[NodeAffinity](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/nodeaffinity.md)和[PodAffinity](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/podaffinity.md)的相关概念和设计可以参考官方文档），这个涉及到podAffinity的symmetry对称性问题（假设Pod A中有对Pod B的AntiAffinity，如果Pod A先运行在某个`node`上，在调度Pod B时即使Pod B的Spec中没有写对Pod A的AntiAffinity，由于Pod A的AntiAffinity，也是不能调度在运行Pod A的那个`node`上的）。
1. nodeInfo：待检查的`node`详细信息

在plugin/pkg/scheduler/algorithmprovider/defaults/defaults.go中`DefaultProvider`提供以下算法：

* `NoVolumeZoneConflict`：如果pod使用的pvc的pv里声明了区域，则优先选择标签也是声明区域的`node`。
* `MaxEBSVolumeCount`：防止新`pod`的EBS加上原有`node`上的EBS超过最大限额。
* `MaxGCEPDVolumeCount`：防止新`pod`的GCEPDVolume加上原有`node`上的GCEPDVolume超过最大最大限额
* `MaxAzureDiskVolumeCount`：防止新`pod`的AzureDiskVolume加上原有`node`上的AzureDiskVolume超过最大限额
* `MatchInterPodAffinity`：首先使用meta中的`matchingAntiAffinityTerms`（包含全部运行`pod`的`anitAffinityTerm`）检查待调度`pod`和当前`node`是否有冲突，之后再检查待调度pod的Affinity和AntiAffinity与当前`node`的关系。
* `NoDiskConflict`：检查`pod`请求的`volume`是否就绪和冲突。如果主机上已经挂载了某个卷，则使用相同卷的`pod`不能调度到这个主机上。k8s使用的`volume`类型不同，过滤逻辑也不同。比如不同云主机的`volume`使用限制不同：GCE允许多个pods使用同时使用`volume`，前提是它们是只读的；AWS不允许pods使用同一个volume；Ceph RBD不允许pods共享同一个monitor。
* `GeneralPredicates`：包含以下几种通用调度算法
	1. `PodFitsResources`：检查`node`上资源是否满足待调度`pod`的请求。
	1. `PodFitsHost`：检查`node`的`NodeName`是否是pod.Spec中指定的。
	1. `PodFitsHostPorts`：检查`pod`申请的端口映射的主机端口是否被`node`上已经运行的`pod`占用。
	1. `PodMatchNodeSelector`：检查`node`上标签是否满足`pod`的`nodeSelector`和`NodeAffinity`，这两项需要同时满足。
* `PodToleratesNodeTaints`：检查`pod`上的`toleration`能否适配`node`上`taint`（[toleration和taint文档](https://kubernetes.io/docs/concepts/configuration/assign-pod-node/#taints-and-tolerations-beta-feature)）。
* `CheckNodeMemoryPressure`：如果`pod`的QoS级别为BestEffort，当`node`处在MemoryPressureCondition时，不允许调度。（BestEffort级别的`pod`的oom_score分数会很高，是omm killer首要的kill对象，因此内存在有压力状态即使`pod`调度过去也会被马上杀掉）。
* `CheckNodeDiskPressure`：如果`node`的处于DiskPressureCondition状态，则不允许任何`pod`调度在上。
* `NoVolumeNodeConflict`：检查`node`是否符合`pod`的Spec中的PersistentVolume里的NodeAffinity。用于[PersistentLocalVolumesz](https://github.com/kubernetes-incubator/external-storage/tree/master/local-volume)，目前是alpha feature。

上述算法中比较复杂的是`MatchInterPodAffinity`，在k8s的`PodAffinity`中加入了一个TopologyKey概念，和其他的predicate算法不同的是一般算法只关注当前`node`自身的状态（即拓扑域只到节点），而`PodAffinity`还需要全部运行`pod`信息从而找出符合Affinity条件的`pod`，再比较符合条件的`pod`所在的`node`和当前`node`是否在同一个拓扑域（即TopologyKey的键值都相同）。

```golang
func (c *PodAffinityChecker) InterPodAffinityMatches(pod *v1.Pod, meta interface{}, nodeInfo *schedulercache.NodeInfo) (bool, []algorithm.PredicateFailureReason, error) {
	node := nodeInfo.Node()
	if node == nil {
		return false, nil, fmt.Errorf("node not found")
	}

	//如果node与meta里AntiAffinityterm的node拓扑域相同，则返回false，说明有对称AntiAffinity出现
	if !c.satisfiesExistingPodsAntiAffinity(pod, meta, node) {
		return false, []algorithm.PredicateFailureReason{ErrPodAffinityNotMatch}, nil
	}

	// Now check if <pod> requirements will be satisfied on this node.
	affinity := pod.Spec.Affinity
	if affinity == nil || (affinity.PodAffinity == nil && affinity.PodAntiAffinity == nil) {
		return true, nil, nil
	}

	// 检查node是否满足pod的affinity和aitiaffinity
	if !c.satisfiesPodsAffinityAntiAffinity(pod, node, affinity) {
		return false, []algorithm.PredicateFailureReason{ErrPodAffinityNotMatch}, nil
	}

	if glog.V(10) {
		// We explicitly don't do glog.V(10).Infof() to avoid computing all the parameters if this is
		// not logged. There is visible performance gain from it.
		glog.Infof("Schedule Pod %+v on Node %+v is allowed, pod (anti)affinity constraints satisfied",
			podName(pod), node.Name)
	}
	return true, nil, nil
}
```
![region](http://ovfdoadvt.bkt.clouddn.com/region.png)

上图有两个region：north和south，每个region分别有node1、node2和node3、node4。region信息也会体现在node的标签label中作为调度时的TopologyKey使用。

`InterPodAffinityMatches`首先会检查当前`node`是否满足已经存在的`pod`的AntiAffinity，前面介绍meta的时候分析过`pod`的AntiAffinity是具有对称性的。之后才会根据待调度`pod`的Affinity和AntiAffinity检查全部`node`。

1. `satisfiesExistingPodsAntiAffinity`首先会计算出所有已经存在的`pod`上的AntiAffinity和待调度`pod`的匹配条目，对每条匹配项，如果当前`node`和已存在`pod`的`node`TopologyKey键值相同，则当前`node`不能被调度。假设node2上运行着Pod A，其对Pod B具有AntiAffinity，拓扑域是region，当我们要调度Pod B时，首先会算出一个匹配条目{Pod A AntiAffinity，node2}，之后对每个`node`计算可否调度，而node1，node2由于和条目中的node2在拓扑域下（`node`上存在key为region且value为south的标签）则无法调度，node3、node4由于拓扑域不同则可以调度。
1. `satisfiesPodsAffinityAntiAffinity`：判断待调度`pod`的Affinity和AntiAffinity时仍然需要全部运行pod的信息，`anyPodMatchesPodAffinityTerm`在使用`PodAffinity`找出所有已存在`pod`中满足`PodAffinity`条件的`pod`，之后如果有任何一个符合条件的`pod`的拓扑域与当前`node`相同，则返回true。假设Pod A对Pod B有`PodAffinity`，拓扑域为region，Pod B已经运行在node1上，当调度Pod A时，找到符合条件的Pod B并记录Pod B所在的node1，在检查每个node时，node1、node2因为和之前记录的node2拓扑域相同（key为region，value为north）则返回true，node3、node4则返回false。之后还特别处理了`PodAffinity`匹配到`pod`自己本身的label的情况，这种情况会发生在希望一个controller下的`pod`全部调度在同一个节点上。AntiAffinity与Affinity正好相反，如果`anyPodMatchesPodAffinityTerm`匹配则直接返回false。
```golang
// Checks if scheduling the pod onto this node would break any rules of this pod.
func (c *PodAffinityChecker) satisfiesPodsAffinityAntiAffinity(pod *v1.Pod, node *v1.Node, affinity *v1.Affinity) bool {
	//列出集群中所有pod
	allPods, err := c.podLister.List(labels.Everything())
	if err != nil {
		return false
	}
	// Check all affinity terms.
	for _, term := range getPodAffinityTerms(affinity.PodAffinity) {
		termMatches, matchingPodExists, err := c.anyPodMatchesPodAffinityTerm(pod, allPods, node, &term)
		if err != nil {
			glog.Errorf("Cannot schedule pod %+v onto node %v, because of PodAffinityTerm %v, err: %v",
				podName(pod), node.Name, term, err)
			return false
		}
		if !termMatches {
			// If the requirement matches a pod's own labels are namespace, and there are
			// no other such pods, then disregard the requirement. This is necessary to
			// not block forever because the first pod of the collection can't be scheduled.
			if matchingPodExists {
				//没有node合适，但是却有匹配的pod，说明此node确实不符合拓扑域匹配，属于正常情况
				glog.V(10).Infof("Cannot schedule pod %+v onto node %v, because of PodAffinityTerm %v",
					podName(pod), node.Name, term)
				return false
			}
			// 当此node不能满足调度需求时，需要考虑当pod的affinityterm满足自己的label的情况，这种需求是
			// 当一个rc或者deployment想把管理的三个pod都调度到一个拓扑域上的时候，第一个pod总会发生阻塞，
			// 所以现在当没有node合适且还没有matchingpod时，发现匹配自己的label就无视了term默认当node合适。
			namespaces := priorityutil.GetNamespacesFromPodAffinityTerm(pod, &term)
			selector, err := metav1.LabelSelectorAsSelector(term.LabelSelector)
			if err != nil {
				glog.Errorf("Cannot parse selector on term %v for pod %v. Details %v",
					term, podName(pod), err)
				return false
			}
			match := priorityutil.PodMatchesTermsNamespaceAndSelector(pod, namespaces, selector)
			if !match {
				glog.V(10).Infof("Cannot schedule pod %+v onto node %v, because of PodAffinityTerm %v",
					podName(pod), node.Name, term)
				return false
			}
		}
	}
	// Check all anti-affinity terms.
	for _, term := range getPodAntiAffinityTerms(affinity.PodAntiAffinity) {
		termMatches, _, err := c.anyPodMatchesPodAffinityTerm(pod, allPods, node, &term)
		if err != nil || termMatches {
			glog.V(10).Infof("Cannot schedule pod %+v onto node %v, because of PodAntiAffinityTerm %v, err: %v",
				podName(pod), node.Name, term, err)
			return false
		}
	}
	if glog.V(10) {
		// We explicitly don't do glog.V(10).Infof() to avoid computing all the parameters if this is
		// not logged. There is visible performance gain from it.
		glog.Infof("Schedule Pod %+v on Node %+v is allowed, pod affinity/anti-affinity constraints satisfied.",
			podName(pod), node.Name)
	}
	return true
}
```

#### Priority
![node-score](http://ovfdoadvt.bkt.clouddn.com/prirority%E8%A1%A8%E6%A0%BC.png)
priority会分配一个二维切片，每个算法为一行，每个`node`为一列，以此才临时存储每个priority算法对每个`node`的打分，最后再使用每个算法的Weight权重乘以每个`node`分数，累加起来得到每个`node`最后的总分数。priority算法目前分为两类：

1. 第一类是早期版本使用的priorityConfig.Function，需要传入待调度`pod`、`nodeNameToInfo`、`node`列表，使用多个goroutine并行处理每个priorityConfig.Function。这些Function类型的算法都被标记了DEPRECATED即未来都会重构为第二类。
1. 第二类使用的priorityConfig.Map和priorityConfig.Reduce，每次priorityConfig.Map只处理一个`node`，使用workqueue.Parallelize()生产者消费者的并行方式，最后由priorityConfig.Reduce把所有`node`的得分映射到0-10这个分数段。

```golang
// Prioritizes the nodes by running the individual priority functions in parallel.
// Each priority function is expected to set a score of 0-10
// 0 is the lowest priority score (least preferred node) and 10 is the highest
// Each priority function can also have its own weight
// The node scores returned by the priority function are multiplied by the weights to get weighted scores
// All scores are finally combined (added) to get the total weighted scores of all nodes
func PrioritizeNodes(
	pod *v1.Pod,
	nodeNameToInfo map[string]*schedulercache.NodeInfo,
	meta interface{},
	priorityConfigs []algorithm.PriorityConfig,
	nodes []*v1.Node,
	extenders []algorithm.SchedulerExtender,
) (schedulerapi.HostPriorityList, error) {
	// If no priority configs are provided, then the EqualPriority function is applied
	// This is required to generate the priority list in the required format
	// 给予所有node全部是1的score
	if len(priorityConfigs) == 0 && len(extenders) == 0 {
		result := make(schedulerapi.HostPriorityList, 0, len(nodes))
		for i := range nodes {
			hostPriority, err := EqualPriorityMap(pod, meta, nodeNameToInfo[nodes[i].Name])
			if err != nil {
				return nil, err
			}
			result = append(result, hostPriority)
		}
		return result, nil
	}

	var (
		mu   = sync.Mutex{}
		wg   = sync.WaitGroup{}
		errs []error
	)
	appendError := func(err error) {
		mu.Lock()
		defer mu.Unlock()
		errs = append(errs, err)
	}
	// HostPriority的二维slice
	results := make([]schedulerapi.HostPriorityList, 0, len(priorityConfigs))

	// 每个priorityConfigs都会生成一个HostPriorityList(长度都为node个数)，然后对所有的HostPriorityList进行统计
	for range priorityConfigs {
		results = append(results, nil)
	}
	for i, priorityConfig := range priorityConfigs {
		if priorityConfig.Function != nil {
			// DEPRECATED
			wg.Add(1)
			go func(index int, config algorithm.PriorityConfig) {
				defer wg.Done()
				var err error
				results[index], err = config.Function(pod, nodeNameToInfo, nodes)
				if err != nil {
					appendError(err)
				}
			}(i, priorityConfig)
		} else {
			// 由于map使用的是对每个node进行并发的方式，所以要提前初始化每个schedulerapi.HostPriorityList
			results[i] = make(schedulerapi.HostPriorityList, len(nodes))
		}
	}
	processNode := func(index int) {
		nodeInfo := nodeNameToInfo[nodes[index].Name]
		var err error
		for i := range priorityConfigs {
			if priorityConfigs[i].Function != nil {
				continue
			}
			//顺序不会改变， priorityConfigs 和 nodes都是有序的
			results[i][index], err = priorityConfigs[i].Map(pod, meta, nodeInfo)
			if err != nil {
				appendError(err)
				return
			}
		}
	}
	workqueue.Parallelize(16, len(nodes), processNode)
	for i, priorityConfig := range priorityConfigs {
		if priorityConfig.Reduce == nil {
			continue
		}
		wg.Add(1)
		go func(index int, config algorithm.PriorityConfig) {
			defer wg.Done()
			if err := config.Reduce(pod, meta, nodeNameToInfo, results[index]); err != nil {
				appendError(err)
			}
		}(i, priorityConfig)
	}
	// Wait for all computations to be finished.
	wg.Wait()
	if len(errs) != 0 {
		return schedulerapi.HostPriorityList{}, errors.NewAggregate(errs)
	}

	// Summarize all scores.
	result := make(schedulerapi.HostPriorityList, 0, len(nodes))

	for i := range nodes {
		result = append(result, schedulerapi.HostPriority{Host: nodes[i].Name, Score: 0})
		for j := range priorityConfigs {
			result[i].Score += results[j][i].Score * priorityConfigs[j].Weight
		}
	}
	......
	return result, nil

```
默认的使用`DefaultProvider`注册了下列priority算法：

* `SelectorSpreadPriority`：尽量把同一个`Service`、`ReplicationController`、`ReplicaSet`、`StatefulSet`下的 pod 分配到不同的节点，权重为1。具体算法为对每个node计算匹配待调度pod的svc或者rc或者rs的selector的pod数量，然后根据公式`10*(max-current)/max`，max为所有`node`中最大的匹配`pod`个数，匹配的越少得分越高。再根据zone标签计算每个zone上匹配pod的数量，依然是`10*(zonemax-currentzone)/zonemax`最后得分是`(1-zoneWeighting)*nodescore + zoneWeighting*zonescore`。默认权重1。
* `InterPodAffinityPriority`：根据PodAffinity和PodAntiAffinity中的`preferredDuringSchedulingIgnoredDuringExecution`来判断每个`node`的分数，同时由于上文提到的对称性原理，还需要考察已经运行`pod`对待调度`pod`的亲和性和反亲和性，权重为1。
* `LeastRequestedPriority`：`node`上可分配资源（包括cpu和memory）越多得分越高，权重为1。
* `BalancedResourceAllocation`：资源平衡分配。把`pod`的请求的资源加到`node`上后计算cpu和memory资源利用率的差，差值越小说明资源越平衡，因此`node`得分越高。默认权重1。（这个算法参考了这篇paper：An Energy Efficient Virtual Machine Placement Algorithm with Balanced Resource Utilization）
* `NodePreferAvoidPodsPriority`：有些`node`上会有AvoidPods的annotation，来禁止rc或者rs这种controller的pod调度在上面，默认权重是 10000，即一旦该函数的结果不为 0，就由它决定排序结果。
* `NodeAffinityPriority`：使用MapReduce方式，先使用Map计算每个node的初始得分（满足NodeAffinity的PreferredDuringSchedulingIgnoredDuringExecution的node会累加PreferredDuringScheduling中的weight值），然后通过Reduce把得分映射到0-10的分数段。权重是1。
* `TaintTolerationPriority`：统计每个`node`上待调度`pod`不能容忍的taints个数，数量越多最后得分越低。权重是1。

对于Priority算法，InterPodAffinityPriority是实现较为复杂的一个，我们着重分析

```golang
// CalculateInterPodAffinityPriority compute a sum by iterating through the elements of weightedPodAffinityTerm and adding
// "weight" to the sum if the corresponding PodAffinityTerm is satisfied for
// that node; the node(s) with the highest sum are the most preferred.
// Symmetry need to be considered for preferredDuringSchedulingIgnoredDuringExecution from podAffinity & podAntiAffinity,
// symmetry need to be considered for hard requirements from podAffinity
func (ipa *InterPodAffinity) CalculateInterPodAffinityPriority(pod *v1.Pod, nodeNameToInfo map[string]*schedulercache.NodeInfo, nodes []*v1.Node) (schedulerapi.HostPriorityList, error) {
	affinity := pod.Spec.Affinity
	hasAffinityConstraints := affinity != nil && affinity.PodAffinity != nil
	hasAntiAffinityConstraints := affinity != nil && affinity.PodAntiAffinity != nil

	allNodeNames := make([]string, 0, len(nodeNameToInfo))
	for name := range nodeNameToInfo {
		allNodeNames = append(allNodeNames, name)
	}

	// convert the topology key based weights to the node name based weights
	var maxCount float64
	var minCount float64
	// priorityMap stores the mapping from node name to so-far computed score of
	// the node.
	pm := newPodAffinityPriorityMap(nodes)

	processPod := func(existingPod *v1.Pod) error {
		existingPodNode, err := ipa.info.GetNodeInfo(existingPod.Spec.NodeName)
		if err != nil {
			return err
		}
		existingPodAffinity := existingPod.Spec.Affinity
		existingHasAffinityConstraints := existingPodAffinity != nil && existingPodAffinity.PodAffinity != nil
		existingHasAntiAffinityConstraints := existingPodAffinity != nil && existingPodAffinity.PodAntiAffinity != nil

		if hasAffinityConstraints {
			// For every soft pod affinity term of <pod>, if <existingPod> matches the term,
			// increment <pm.counts> for every node in the cluster with the same <term.TopologyKey>
			// value as that of <existingPods>`s node by the term`s weight.
			// 如果待调度的pod的podaffinity匹配了当前运行的pod，则给和匹配pod的node的所有相同拓扑域的node加一份
			// 这里1份是要 1*term.Weight
			terms := affinity.PodAffinity.PreferredDuringSchedulingIgnoredDuringExecution
			pm.processTerms(terms, pod, existingPod, existingPodNode, 1)
		}
		if hasAntiAffinityConstraints {
			// For every soft pod anti-affinity term of <pod>, if <existingPod> matches the term,
			// decrement <pm.counts> for every node in the cluster with the same <term.TopologyKey>
			// value as that of <existingPod>`s node by the term`s weight.
			// 同理如果待调度的pod的podantiaffinity匹配了当前运行的pod，则给和匹配pod的node的所有相同拓扑域的node减一份
			terms := affinity.PodAntiAffinity.PreferredDuringSchedulingIgnoredDuringExecution
			pm.processTerms(terms, pod, existingPod, existingPodNode, -1)
		}

		if existingHasAffinityConstraints {
			// For every hard pod affinity term of <existingPod>, if <pod> matches the term,
			// increment <pm.counts> for every node in the cluster with the same <term.TopologyKey>
			// value as that of <existingPod>'s node by the constant <ipa.hardPodAffinityWeight>

			//However, although RequiredDuringScheduling affinity is not symmetric, there is an implicit PreferredDuringScheduling
			//affinity rule corresponding to every RequiredDuringScheduling affinity rule: if the pods of S1 have a
			//RequiredDuringScheduling affinity rule "run me on nodes that are running pods from S2" then it is not required
			//that there be S1 pods on a node in order to schedule a S2 pod onto that node, but it would be better if there are.
			//根据设计文档pod的Required Affinity是没有对称性的，但是会转化为Prefer Affinity：如果待调度的pod满足existingPod的Required selector
			//那么可以给和此node相同拓扑域的node加hardPodAffinityWeight份
			// 这个操作相当于每次调度遍历所有node的所有pod来检查PodAffinity.RequiredDuringSchedulingIgnoredDuringExecution

			if ipa.hardPodAffinityWeight > 0 {
				terms := existingPodAffinity.PodAffinity.RequiredDuringSchedulingIgnoredDuringExecution
				// TODO: Uncomment this block when implement RequiredDuringSchedulingRequiredDuringExecution.
				//if len(existingPodAffinity.PodAffinity.RequiredDuringSchedulingRequiredDuringExecution) != 0 {
				//	terms = append(terms, existingPodAffinity.PodAffinity.RequiredDuringSchedulingRequiredDuringExecution...)
				//}
				// hardPodAffinityWeight = HardPodAffinitySymmetricWeight = 1
				for _, term := range terms {
					pm.processTerm(&term, existingPod, pod, existingPodNode, float64(ipa.hardPodAffinityWeight))
				}
			}
			// For every soft pod affinity term of <existingPod>, if <pod> matches the term,
			// increment <pm.counts> for every node in the cluster with the same <term.TopologyKey>
			// value as that of <existingPod>'s node by the term's weight.
			//同样的待调度pod满足existingPod上的PreferredDuringSchedulingIgnoredDuringExecution同样会加1份
			terms := existingPodAffinity.PodAffinity.PreferredDuringSchedulingIgnoredDuringExecution
			pm.processTerms(terms, existingPod, pod, existingPodNode, 1)
		}
		if existingHasAntiAffinityConstraints {
			// For every soft pod anti-affinity term of <existingPod>, if <pod> matches the term,
			// decrement <pm.counts> for every node in the cluster with the same <term.TopologyKey>
			// value as that of <existingPod>'s node by the term's weight.
			// 待调度pod满足existingPod的PodAntiAffinity.PreferredDuringSchedulingIgnoredDuringExecution
			// 同样会给相同拓扑域的node减1份
			terms := existingPodAffinity.PodAntiAffinity.PreferredDuringSchedulingIgnoredDuringExecution
			pm.processTerms(terms, existingPod, pod, existingPodNode, -1)
		}
		//Tips: exsitingPod的检查全部做完了其中三项在上面做了Affinity的Require和Prefer，AntiAffinity的Prefer
		// 而 AntiAffinity的Require检查由于有对称性约束已经在predicate里做了（meta）
		return nil
	}
	processNode := func(i int) {
		nodeInfo := nodeNameToInfo[allNodeNames[i]]
		// 如果待调度pod上有Affinity或AntiAffinity项，则需要考量每个node的所有pod
		// 如果没有这两项，那只需要考虑每个node上有Affinity或AntiAffinity的pod，来验证
		// 待调度pod就可以了
		if hasAffinityConstraints || hasAntiAffinityConstraints {
			// We need to process all the nodes.
			for _, existingPod := range nodeInfo.Pods() {
				if err := processPod(existingPod); err != nil {
					pm.setError(err)
				}
			}
		} else {
			// The pod doesn't have any constraints - we need to check only existing
			// ones that have some.
			for _, existingPod := range nodeInfo.PodsWithAffinity() {
				if err := processPod(existingPod); err != nil {
					pm.setError(err)
				}
			}
		}
	}
	workqueue.Parallelize(16, len(allNodeNames), processNode)
	if pm.firstError != nil {
		return nil, pm.firstError
	}

	// 找出得分中的最大值和最小值
	for _, node := range nodes {
		if pm.counts[node.Name] > maxCount {
			maxCount = pm.counts[node.Name]
		}
		if pm.counts[node.Name] < minCount {
			minCount = pm.counts[node.Name]
		}
	}

	// calculate final priority score for each node
	result := make(schedulerapi.HostPriorityList, 0, len(nodes))
	for _, node := range nodes {
		fScore := float64(0)
		// maxCount == minCount == 0 只会发生都等于0的情况，即全都没有使用affinity机制，这样node得分都为0
		if (maxCount - minCount) > 0 {
			fScore = float64(schedulerapi.MaxPriority) * ((pm.counts[node.Name] - minCount) / (maxCount - minCount))
		}
		result = append(result, schedulerapi.HostPriority{Host: node.Name, Score: int(fScore)})
		if glog.V(10) {
			// We explicitly don't do glog.V(10).Infof() to avoid computing all the parameters if this is
			// not logged. There is visible performance gain from it.
			glog.V(10).Infof("%v -> %v: InterPodAffinityPriority, Score: (%d)", pod.Name, node.Name, int(fScore))
		}
	}
	return result, nil
}
```
在PodAffinity中一般都会进行双向检查，即待调度的`pod`的Affinity检查已存在`pod`，以及已存在`pod`的Affinity检查待调度`pod`。`CalculateInterPodAffinityPriority`首先创建了一个`podAffinityPriorityMap`对象pm，其中保存了`node`列表，`node`初始得分列表，和默认的失败域（`node`标签上的nodename，zone，region）。之后定义了两个函数`processNode`和`processPod`分别处理每个`node`和每个`node`上的每个正在运行的`pod`。其中对于`processNode`使用了之前常见的workqueue.Parallelize这种并行执行方式。在`processPod`中首先检查了待调度`pod`的Affinity和AntiAffinity的`PreferredDuringSchedulingIgnoredDuringExecution`能否匹配当前检查已存在的pod，如果匹配成功则会给已运行`pod`所在`node`及相同拓扑域的所有`node`加上（AntiAffinity对应减去）`1*Weight`得分。而对于已存在`pod`检查待调度`pod`除了常规的`PreferredDuringSchedulingIgnoredDuringExecution`外，还特别检查了Affinity的`RequiredDuringSchedulingIgnoredDuringExecution`，Require应该都是出现在Predicate算法中，而在这Priority出现原因通过官方设计文档解读，还是由于类似的对称性，这里特意给了这个Require一个特殊的参数`hardPodAffinityWeight`，这个参数是由DefaultProvider提供的（默认值是1），因此已存在的`pod`的`RequiredDuringSchedulingIgnoredDuringExecution`如果匹配到待调度`pod`，与其运行的`node`具有相同拓扑域的全部`node`都会增加`hardPodAffinityWeight*Weight`得分。最后得到全部`node`得分后在将映射到0-10段。

#### 选择最佳node
经过Predicate和Priority算过的过滤后，我们会得到一个具有附带得分的`node`列表，我们只需选择最高得分的`node`即可。但是，如果是这样的方式，我们仍然会面临调度不均匀的情况，假设所有`node`都是普通没有任何标记的`node`，现在我们需要连续创建一些不归任何controller控制的`pod`，且没有任何资源请求。现在就会出现所有pod调度在同一个节点的情况，因为所有`node`得分相同，排序后最高的都是同一个。而在genericScheduler.selectHost()中引入了`lastNodeIndex`，每次调度都会自增1，而最后返回的是`(lastNodeIndex)%(具有相同最高分node的个数)`，这样就会Round-Robin所有的相同最高得分的node，达到调度均衡。

```golang
func (g *genericScheduler) selectHost(priorityList schedulerapi.HostPriorityList) (string, error) {
	if len(priorityList) == 0 {
		return "", fmt.Errorf("empty priorityList")
	}

	sort.Sort(sort.Reverse(priorityList))
	maxScore := priorityList[0].Score
	firstAfterMaxScore := sort.Search(len(priorityList), func(i int) bool { return priorityList[i].Score < maxScore })

	g.lastNodeIndexLock.Lock()
	ix := int(g.lastNodeIndex % uint64(firstAfterMaxScore))
	g.lastNodeIndex++
	g.lastNodeIndexLock.Unlock()

	return priorityList[ix].Host, nil
}
```

## 总结
Kubernetes的kube-scheduler是以plugin形式开发出来的，方便用户开发自己的调度器。
其核心调度过程其实是对kube-apiserver的`pod`对象的CRUD：

* 客户端通过kubectl或者restapi获取kube-apiserver中NodeName为空的`pod`。
* 调度器通过一系列调度算法计算出最适合`pod`的`node`。
* 最后使用kubectl或者restapi向kube-apiserver中写入`pod`的Binding信息（包含之前计算出来的最适合`node`的NodeName）。

实现自己的调度器并不困难，Kubernetes官方也提供了一个基于shell脚本的[随机调度器](http://blog.kubernetes.io/2017/03/advanced-scheduling-in-kubernetes.html)，

本文只是分析了kube-scheduler的执行流程和重点的调度算法，而Kubernetes作为**Production-Grade**生产级容器调度管理平台，其代码在细节处理以及可靠性上是非常高的。而Kubernetes也成为当前十分活跃的社区，其中也有大量高手贡献代码，由此看来开源项目和社区形成的非常好的良性循环成就kubernetes为容器平台真正的领跑者。


## 参考文章

* [使用 client-go 控制原生及拓展的 Kubernetes API](https://www.kubernetes.org.cn/1309.html)
* [Advanced Scheduling in Kubernetes](http://blog.kubernetes.io/2017/03/advanced-scheduling-in-kubernetes.html)
