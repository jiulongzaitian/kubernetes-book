### 主循环

```
作者：李昂 邮箱：liang@haihangyun.com
```

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

dispatchWork首先会判断pod中所有container是否全部处于非运行状态（`kl.podIsTerminated(pod)`），如果是则会调用`statusManager.TerminatePod(pod)`更新apiserver中全部container状态，当pod资源清理完成可以被删除，就使用kubeclient调用apiserver的Delete来正式删除这个pod，这一块是pod被delete时的逻辑。我们继续看pod add时的下面代码，`kl.podWorkers.UpdatePod`这里会调用podWorker创建一个异步的worker完成pod的变更。`podWorker.UpdatePod`如下：

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
* 调用kl.canRunPod\(pod\)判断pod是否可以运行在当前节点，如果不行，将原因更新到pod状态，并同步到statusManager。
* 当pod不应该运行时调用kl.killPod删除pod
* 当pod时static时，为其创建mirror pod。
* 为pod在默认的/var/lib/kubelet目录中创建对应的目录。
* 等待pod spec中volume都被attach/mount
* 从apiserver中获取pull secrets（镜像仓库认证相关信息）
* 最后调用containerRuntime.SyncPod\(...\)完成pod在node上同步（为什么说是SyncPod同步而不是创建，因为每次都要通过pod的spec和statuc状态判断哪些container需要删除哪些需要创建）
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

plegCh接收pleg\(pod lifecircle event generator\)产生的事件，pleg调用relist每秒比较自己缓存中的所有pod的container状态和当前在node上运行的全部pod的container，如果pod中的container或pod本身container发生变化则会产生一条事件，从而触发pod同步。如果事件是`pleg.ContainerDied`，就会在apiserver中删除这个pod下的container。

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

#### kl.livenessManager.Updates\(\)

当pod spec中配置了livenessProbe，kubelet的probeManager会开启一个独立的goroutine对配置探测点\(http、tcp、exec\)进行探测，probeManager中包含livenessManager\(表示container是否存活\)和readinessManager\(表示container是否准备好接受访问或者服务\)，因此当livenessManager探测到某个container`update.Result == proberesults.Failure`时，主循环会监听这个事件从而触发pod同步。

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

housekeepingCh顾名思义，就是每隔`housekeepingPeriod`\(hardcode为2秒钟\)做一下清理工作，主要是以podmanager为准，清理podworker、probemanager、statusManage、podmanager中的孤儿mirrorpod、cgroupQOS目录，以removeOrphanedPodStatuses为准清理、rumtimecacahe、poddir

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

