# kubelet SourceReady

在看kubelet代码时候，会经常看到 sourcesReady 的字段，或者 AllReady 的方法，sourcesReady 主要是用来判断3个source\([PodConfig 分析](/yuan-ma-fen-xi/scheduler/podconfig-fen-xi.md)\) 是否已经ready，具体实现方式如下：

pkg/kubelet/kubelet.go

```golang
NewMainKubelet（）方法初始化

    klet := &Kubelet{
        hostname:        hostname,
        nodeName:        nodeName,
        kubeClient:      kubeDeps.KubeClient,
        heartbeatClient: kubeDeps.HeartbeatClient,
        rootDirectory:   rootDirectory,
        resyncInterval:  kubeCfg.SyncFrequency.Duration,
        // podConfig 是 makePodSourceConfig 初始化的
        sourcesReady:                   config.NewSourcesReady(kubeDeps.PodConfig.SeenAllSources),
    ...
    }
```

sourceReady 会使用 PodConfig 的 SeenAllSources\(\)方法, 具体实现逻辑会 判断传入的参数 seenSources\(一个map\) 里是否已经包含了 PodConfig 里已经包含的所有source，上文介绍到 加入一个pod 来源时候，podconfig 就会在sources 里记录这个来源。同样在Podconfig 的pods字段里（实际上是一个storage）也有source记录，只有 seenSources 和 storage 都包含 podconfig 的所有source 时候，才代表了ready 状态。

```golang
func (c *PodConfig) SeenAllSources(seenSources sets.String) bool {
    if c.pods == nil {
        return false
    }
    glog.V(6).Infof("Looking for %v, have seen %v", c.sources.List(), seenSources)
    return seenSources.HasAll(c.sources.List()...) && c.pods.seenSources(c.sources.List()...)
}
```

那sourceReady 的source 是什么时候add 的，跟下面的主函数 syncLoop有关，syncLoop 是在Run 方法中被调用的，继而 syncLoop调用 syncLoopIteration方法， 在此方法中会调用  kl.sourcesReady.AddSource\(u.Source\)。

```golang
func (kl *Kubelet) syncLoopIteration(configCh <-chan kubetypes.PodUpdate, handler SyncHandler,
    syncCh <-chan time.Time, housekeepingCh <-chan time.Time, plegCh <-chan *pleg.PodLifecycleEvent) bool {
    select {
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
...
        if u.Op != kubetypes.RESTORE {
            // If the update type is RESTORE, it means that the update is from
            // the pod checkpoints and may be incomplete. Do not mark the
            // source as ready.

            // Mark the source ready after receiving at least one update from the
            // source. Once all the sources are marked ready, various cleanup
            // routines will start reclaiming resources. It is important that this
            // takes place only after kubelet calls the update handler to process
            // the update to ensure the internal pod cache is up-to-date.
            kl.sourcesReady.AddSource(u.Source)
        }
...
```

 

# 



