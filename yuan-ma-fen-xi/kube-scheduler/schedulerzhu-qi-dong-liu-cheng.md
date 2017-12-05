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
