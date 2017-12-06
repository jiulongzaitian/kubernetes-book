## kubelet源码分析

```
作者：李昂 邮箱：liang@haihangyun.com
```

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

run函数的主要目的就是为kubelet.Dependencies做初始化，但是初始化之前需要通过命令行是否传过来KubeConfig判断kubelet运行在standalone模式还是标准集群模式，如果是是合法的KubeConfig，那么就是非standalone模式，如果是standalone模式kubelet.Dependencies中很多client可以直接置为nil。kubelet.Dependencies实际上是一类kubelet运行时需要外部依赖的抽象，它包含了各类Client（dockerclient、kubeclient等）以及网络插件和数据卷插件等，这么做还有另一个好处是可以通过参数传进来一些Dependencies来帮助测试。最后如果是非standalone模式，那么将构建正式的Dependencies里的各类对象。这里特别说明的是Dependencies.ContainerManager是管理kubelet进程docker进程以及相关cgroup和oom\_adj\_score设置的，它和kubelet管理的pod上运行的container是没有关系的。之后在RunKubelet中就开始创建真正的kubelet核心结构体以及运行主循环。

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

RunKubelet根据穿过来的配置设置capabilities，如果kubeDeps中没有外部注入builder，那么会用默认的CreateAndInitKubelet函数创建kubelet核心对象。CreateAndInitKubelet调用kubelet.NewMainKubelet创建kubelet对象k，k.BirthCry\(\)调用eventrecorder写入一条"kubelet start"事件，而StartGarbageCollection\(\)会启动containerGC和imagemanager的GarbageCollect\(\)方法开启容器和镜像的垃圾回收。startKubelet会正式启动kubelet，并且如果配置了EnableServer和ReadOnlyPort还会监听端口开启Api。这里k.Run\(podCfg.Updates\(\)\)传入的podCfg.Updates\(\)返回一个channel，会收到了apiserver、file、url中的pod更新事件。到这里其实主启动流程就算执行完了，前面启动流程的部分逻辑比较简单，本文只是大致分析到。而后面的NewMainKubelet和k.Run\(podCfg.Updates\(\)\)才包含了kubelet核心逻辑。

