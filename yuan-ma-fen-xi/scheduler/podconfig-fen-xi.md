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
 研究代码时候需要重点关注updates 对象 和 merge 过程


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
先创建了一个updates 对象，这是一个有50个缓存的chan，具体为什么是50个可以仔细研究一下，通过代码我们可以看到updates 是storage 和podConfig里的一个对象， 而storage 是mux 里的一个对象，因此updates 贯穿storage，podConfig，mux，中，最终通过mux 的操作，将merge 的数据放到updates里，供podconfig 所用。 通过研究 PodUpdate 结构体我们可以看到，
```golang
pkg/kubelet/types/pod_update.go
type PodUpdate struct {
	Pods   []*v1.Pod
	Op     PodOperation
	Source string
}
``` 
每个PodUpdate 主要是存的 相同来源:source 下，相同操作：OP 下的一堆POD 列表。
再看newPodStorage 方法：
```golang
pkg/kubelet/config/config.go
func newPodStorage(updates chan<- kubetypes.PodUpdate, mode PodConfigNotificationMode, recorder record.EventRecorder) *podStorage {
	return &podStorage{
		pods:        make(map[string]map[types.UID]*v1.Pod),
		mode:        mode,
		updates:     updates,
		sourcesSeen: sets.String{},
		recorder:    recorder,
	}
}
```
storage 里pods 字段是一个二重map，第一层代表的是来源source，第二层代表的是pod 的uuid，value是pod 值



