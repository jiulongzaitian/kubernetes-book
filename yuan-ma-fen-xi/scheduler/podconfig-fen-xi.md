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

# PodConfig 初始化

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

pkg/kubelet/config/config.go   NewPodConfig\(\)

```golang
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
先创建了一个updates 对象，这是一个有50个缓存的chan，具体为什么是50个可以仔细研究一下，通过代码我们可以看到updates 是storage 和podConfig里的一个对象， 而storage 是mux 里的一个对象，因此updates 贯穿storage，podConfig，mux，中，最终通过mux 的操作，将merge 的数据放到updates里，供podconfig 所用。 通过研究 PodUpdate 结构体我们可以看到
pkg/kubelet/types/pod_update.go

``` golang

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
updates: 存的是updates 对象
再看 mux 对象： config.NewMux(storage),
```golang
func NewMux(merger Merger) *Mux {
	mux := &Mux{
		sources: make(map[string]chan interface{}),
		merger:  merger,
	}
	return mux
}
```
此方法将外面创建的storage 设置为merger 字段，代表将要进行merger， sources 字段是一个map，key 是来源， value 是一个chan interface{}， 它实际上是一个PodUpdate 对象
我们以 makePodSourceConfig\(\) 方法的 NewSourceFile 进行分析跟踪
```golang
  config.NewSourceFile(kubeCfg.PodManifestPath, nodeName, kubeCfg.FileCheckFrequency.Duration, cfg.Channel(kubetypes.FileSource))
```
最后一个参数 调用podconfig 的Channel 方法
```golang
func (c *PodConfig) Channel(source string) chan<- interface{} {
	c.sourcesLock.Lock()
	defer c.sourcesLock.Unlock()
	c.sources.Insert(source)
	return c.mux.Channel(source)
}
```
这个Channel 会返回一个只写的chan，  作为NewSourceFile 的最后一个参数， Channel方法里面又调用c.mux.Channel 方法
```golang
func (m *Mux) Channel(source string) chan interface{} {
	if len(source) == 0 {
		panic("Channel given an empty name")
	}
	m.sourceLock.Lock()
	defer m.sourceLock.Unlock()
	channel, exists := m.sources[source]
	if exists {
		return channel
	}
	newChannel := make(chan interface{})
	m.sources[source] = newChannel
	go wait.Until(func() { m.listen(source, newChannel) }, 0, wait.NeverStop)
	return newChannel
}

// listenChannel 循环遍历可读 channel
func (m *Mux) listen(source string, listenChannel <-chan interface{}) {
	for update := range listenChannel {
		m.merger.Merge(source, update)
	}
}

```

在mux 的channel 里 我们发现如果没有source 的key，则从新创建一个新的chan interface{}: newChannel
之后不断调用listen方法，
listen 则会遍历一下newChannel， 然后调用m.merger.Merge 方法进行最终的合并，稍后我们会分析Merge 方法。
根据这个分析，我们可以初步猜测，NewSourceFile（） NewSourceURL（） NewSourceApiserver（）这三种方法实际上是生产者，生产出来的pod 数据，会最终写到 Channel 返回的 chan<- interface{} chan 中，这样整个产生pod 和 合并pod 的流程就已经很清楚了，三种方法不断将监控到的pod 写入到 chan interface{} 中，这个chan 是分source 的，最终会调用m.merger.Merge方法进行最终的合并。


# POD 是如何获取的
我们来分析  chan<- interface{}  到底是什么

以 NewSourceFile(）方法为例  pkg/kubelet/config/file.go

```golang
func NewSourceFile(path string, nodeName types.NodeName, period time.Duration, updates chan<- interface{}) {
	// "golang.org/x/exp/inotify" requires a path without trailing "/"
	path1 := strings.TrimRight(path, string(os.PathSeparator))

	config := newSourceFile(path1, nodeName, period, updates)
	glog.V(1).Infof("Watching path %q period %v", path1, period)
	go wait.Forever(config.run, period)
	glog.V(1).Info("after wait.Forever(config.run, period) ...")
}

func newSourceFile(path string, nodeName types.NodeName, period time.Duration, updates chan<- interface{}) *sourceFile {
	send := func(objs []interface{}) {
		var pods []*v1.Pod
		for _, o := range objs {
			pods = append(pods, o.(*v1.Pod))
		}
		updates <- kubetypes.PodUpdate{Pods: pods, Op: kubetypes.SET, Source: kubetypes.FileSource}
	}
	store := cache.NewUndeltaStore(send, cache.MetaNamespaceKeyFunc)
	return &sourceFile{
		path:           path,
		nodeName:       nodeName,
		store:          store,
		fileKeyMapping: map[string]string{},
		updates:        updates,
	}
}

```

此时 chan<- interface{} 在 NewSourceFile 方法里  是 updates 参数


现在我们暂且不分析pod 是具体怎么来的，我们先分析pod 获得后是如何merger 的。在下面分析过程中，我们要时刻关注updates 这个对象。updates 最终， 在 newSourceFile ，updates 存放于 send 这个闭包，这样我们基本上已经知道updates 是什么了，
```
updates <- kubetypes.PodUpdate{Pods: pods, Op: kubetypes.SET, Source: kubetypes.FileSource}
```
就是一个 PodUpdate 对象，存放了Pods,操作是SET，还有来源，通过pods 我们可以看到，pods 是一个全量数据。
那为什么要用全量数据， 我们看一下send 是怎么调用的。看一下 NewUndeltaStore 方法：
/vendor/k8s.io/client-go/tools/cache/undelta_store.go
```
// NewUndeltaStore returns an UndeltaStore implemented with a Store.
func NewUndeltaStore(pushFunc func([]interface{}), keyFunc KeyFunc) *UndeltaStore {
	return &UndeltaStore{
		Store:    NewStore(keyFunc),
		PushFunc: pushFunc,
	}
}

func (u *UndeltaStore) Add(obj interface{}) error {
	if err := u.Store.Add(obj); err != nil {
		return err
	}
	u.PushFunc(u.Store.List())
	return nil
}
```
我们可以简单看一下Add 方法，每次往UndeltaStore 写完数据时候，都会最终调用PushFunc 这个方法，就是send 方法，而u.Store.List() 则是一个添加完新数据后的全量方法，那么问题来了，在哪里开始调用这个Add 方法呢：
我们回头来看  NewSourceFile() 方法， 方法里调用了 `go wait.Forever(config.run, period)`

run 方法的实现: 它又调用了watch 方法
```golang
func (s *sourceFile) run() {
	//张杰 在linux 环境下，watch 会执行本package 里 file_linux.go 文件的watch 方法
	if err := s.watch(); err != nil {
		glog.Errorf("Unable to read manifest path %s: %v -------", s.path, err)
	}
}

```
如果你是在linux 环境里，watch 实现是在
pkg/kubelet/config/file_linux.go
```golang

func (s *sourceFile) watch() error 

通过分析watch，可以追到processEvent 方法,在 processEvent 方法里，你可以清晰的看到：
s.store.Add(pod) 方法， 具体实现暂不分析

```

这就是updates 的来源

# POD 是如何合并的
分析 m.merger.Merge 方法：pkg/kubelet/config/config.go 
```
func (s *podStorage) Merge(source string, change interface{}) error {
	s.updateLock.Lock()
	defer s.updateLock.Unlock()

	seenBefore := s.sourcesSeen.Has(source)
	adds, updates, deletes, removes, reconciles, restores := s.merge(source, change)
	firstSet := !seenBefore && s.sourcesSeen.Has(source)

	// deliver update notifications
	switch s.mode {
	// 默认用的这个模式
...
}

```
此方法有个重要的实现  s.merge（），最后将经过merge 的数据 扔到 storage 中的updates 中，这样podConfig 的updates 也有数据了，因为用的是相同的chan

merge 实现 具体看注释
```golang
func (s *podStorage) merge(source string, change interface{}) (adds, updates, deletes, removes, reconciles, restores *kubetypes.PodUpdate) {
	s.podLock.Lock()
	defer s.podLock.Unlock()

	// 初始化不同的分类
	addPods := []*v1.Pod{}
	updatePods := []*v1.Pod{}
	deletePods := []*v1.Pod{}
	removePods := []*v1.Pod{}
	reconcilePods := []*v1.Pod{}
	restorePods := []*v1.Pod{}

	// 获得storage 中source 来源的老的数据
	pods := s.pods[source]
	if pods == nil {
		pods = make(map[types.UID]*v1.Pod)
	}

	// updatePodFunc is the local function which updates the pod cache *oldPods* with new pods *newPods*.
	// After updated, new pod will be stored in the pod cache *pods*.
	// Notice that *pods* and *oldPods* could be the same cache.
	// 对于SET 操作来说 newPods 就是新channnel 里的 pods
	// oldPods 是当前缓存里的pods
	// pods 刚开始是空的

	updatePodsFunc := func(newPods []*v1.Pod, oldPods, pods map[types.UID]*v1.Pod) {
		// 因为对于SET 操作 newpods 里的是全量的pod， 需要过滤重名的
		//[]*v1.Pod
		filtered := filterInvalidPods(newPods, source, s.recorder)
		//过滤完后
		for _, ref := range filtered {
			// Annotate the pod with the source before any comparison.
			// 这个方式挺厉害 Annotations 并没有重新发布到etcd中
			if ref.Annotations == nil {
				ref.Annotations = make(map[string]string)
			}
			ref.Annotations[kubetypes.ConfigSourceAnnotationKey] = source
			if existing, found := oldPods[ref.UID]; found {
				// old 里面如果有 新pod 里的索引，则天机到pods里
				pods[ref.UID] = existing

				// 开始比较老的 和 新的的差异 重要方法 
				needUpdate, needReconcile, needGracefulDelete := checkAndUpdatePod(existing, ref)
				if needUpdate {
					updatePods = append(updatePods, existing)
				} else if needReconcile {
					reconcilePods = append(reconcilePods, existing)
				} else if needGracefulDelete {
					deletePods = append(deletePods, existing)
				}
				continue
			}
			// 没有找到  才设置 FirstSeenTime
			recordFirstSeenTime(ref)
			pods[ref.UID] = ref
			addPods = append(addPods, ref)
		}
	}

	update := change.(kubetypes.PodUpdate)
	switch update.Op {
	case kubetypes.ADD, kubetypes.UPDATE, kubetypes.DELETE:
		if update.Op == kubetypes.ADD {
			glog.V(4).Infof("Adding new pods from source %s : %v", source, update.Pods)
		} else if update.Op == kubetypes.DELETE {
			glog.V(4).Infof("Graceful deleting pods from source %s : %v", source, update.Pods)
		} else {
			glog.V(4).Infof("Updating pods from source %s : %v", source, update.Pods)
		}
		updatePodsFunc(update.Pods, pods, pods)

	case kubetypes.REMOVE:
		glog.V(4).Infof("Removing pods from source %s : %v", source, update.Pods)
		for _, value := range update.Pods {
			if existing, found := pods[value.UID]; found {
				// this is a delete
				delete(pods, value.UID)
				removePods = append(removePods, existing)
				continue
			}
			// this is a no-op
		}

	case kubetypes.SET:
		// 对于三种来源来说，都是SET 方法，可以重点看看
		glog.V(4).Infof("Setting pods for source %s", source)
		s.markSourceSet(source)
		// Clear the old map entries by just creating a new map
		oldPods := pods
		pods = make(map[types.UID]*v1.Pod)
		updatePodsFunc(update.Pods, oldPods, pods)
		for uid, existing := range oldPods {
			if _, found := pods[uid]; !found {
				// this is a delete
				removePods = append(removePods, existing)
			}
		}
	case kubetypes.RESTORE:
		glog.V(4).Infof("Restoring pods for source %s", source)

	default:
		glog.Warningf("Received invalid update type: %v", update)

	}

	// 更新
	s.pods[source] = pods

	adds = &kubetypes.PodUpdate{Op: kubetypes.ADD, Pods: copyPods(addPods), Source: source}
	updates = &kubetypes.PodUpdate{Op: kubetypes.UPDATE, Pods: copyPods(updatePods), Source: source}
	deletes = &kubetypes.PodUpdate{Op: kubetypes.DELETE, Pods: copyPods(deletePods), Source: source}
	removes = &kubetypes.PodUpdate{Op: kubetypes.REMOVE, Pods: copyPods(removePods), Source: source}
	reconciles = &kubetypes.PodUpdate{Op: kubetypes.RECONCILE, Pods: copyPods(reconcilePods), Source: source}
	restores = &kubetypes.PodUpdate{Op: kubetypes.RESTORE, Pods: copyPods(restorePods), Source: source}

	return adds, updates, deletes, removes, reconciles, restores
}

```
通过merger 后，数据就会放到podconfig 对象的updates 字段中


kubernetes 用来大量生产者 消费者分离的机制，并大量运用channel 做关联
 









