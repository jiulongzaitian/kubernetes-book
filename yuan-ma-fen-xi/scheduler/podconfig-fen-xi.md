# PodConfig 分析

```
作者：张杰 邮箱：j.zhang8@haihangyun.com
```

---

kubelet 可以支持三类Pod的创建方式：

* From File    配置文件
* From Http    HTTP
* From ApiServer    APiserver

前两种方式可以称之为Static Pod

配置文件就是放在特定目录下的标准的JSON或YAML格式的pod定义文件。用`kubelet --pod-manifest-path=<the directory>`来启动kubelet进程，kubelet将会周期扫描这个目录，根据这个目录下出现或消失的YAML/JSON文件来创建或删除静态pod。

通过HTTP创建静态Pods Kubelet周期地从–manifest-url=参数指定的地址下载文件，并且把它翻译成JSON/YAML格式的pod定义。此后的操作方式与--pod-manifest-path=相同，kubelet会不时地重新下载该文件，当文件变化时对应地终止或启动静态pod

具体对应的代码为：

```golang
pkg/kubelet/kubelet.go

//NewMainKubelet() 方法 调用的 makePodSourceConfig() 方法

//代码段：

        kubeDeps.PodConfig, err = makePodSourceConfig(kubeCfg, kubeDeps, nodeName, bootstrapCheckpointPath)
  
```

具体看 pkg/kubelet/kubelet.go  makePodSourceConfig\(\)

```golang
#核心方法

        // source of all configuration
        // 创建一个PodConfig 对象，最终这个podConfig 会汇总三种pod 来源
    cfg := config.NewPodConfig(config.PodConfigNotificationIncremental, kubeDeps.Recorder)
    #1
    config.NewSourceFile(kubeCfg.PodManifestPath, nodeName, kubeCfg.FileCheckFrequency.Duration, cfg.Channel(kubetypes.FileSource))
    #2
    config.NewSourceURL(kubeCfg.ManifestURL, manifestURLHeader, nodeName, kubeCfg.HTTPCheckFrequency.Duration, cfg.Channel(kubetypes.HTTPSource))

    #3
    updatechannel := cfg.Channel(kubetypes.ApiserverSource)
    config.NewSourceApiserver(kubeDeps.KubeClient, nodeName, updatechannel)    
```
三种pod 来源方式分别是 NewSourceFile NewSourceURL NewSourceApiserver 方式获得的，注意每个方法的最后一个参数: cfg.Channel  这个方法会最终调用merge 来合并数据， 合并完的数据 最终都会放到 podstorage 的updates里，而updates 又贯穿到podconfig 中，所以最终数据全部到了podconfig 的updates中。 

podConfig struct 分析

pkg/kubelet/config/config.go   NewPodConfig\(\)

```golang
type PodConfig struct {
    pods *podStorage
    mux  *config.Mux

    // the channel of denormalized changes passed to listeners
    updates chan kubetypes.PodUpdate

    // contains the list of all configured sources
    sourcesLock       sync.Mutex
    sources           sets.String
    checkpointManager checkpoint.Manager
}


```

pkg/kubelet/config/config.go   NewPodConfig\(\)

```
// NewPodConfig creates an object that can merge many configuration sources into a stream
// of normalized updates to a pod configuration.
func NewPodConfig(mode PodConfigNotificationMode, recorder record.EventRecorder) *PodConfig {
    updates := make(chan kubetypes.PodUpdate, 50)
    storage := newPodStorage(updates, mode, recorder)
    // storage 里的updates 和 config 里的updates 是共用的
    podConfig := &PodConfig{
        pods:    storage,
        mux:     config.NewMux(storage),
        updates: updates,
        sources: sets.String{},
    }
    return podConfig
}
```



