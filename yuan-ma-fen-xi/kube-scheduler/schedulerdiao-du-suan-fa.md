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
