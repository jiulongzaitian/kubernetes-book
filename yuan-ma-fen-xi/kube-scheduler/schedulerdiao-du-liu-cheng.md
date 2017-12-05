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
