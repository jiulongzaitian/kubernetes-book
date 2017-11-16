## Pod优先级和抢占


作者：李昂 邮箱：liang@haihangyun.com
---
#### 目前版本状态：1.8v1alpha1
Pod在kubernetes1.8及其以后的版本是可以有优先级配置的。优先级表示当前Pod相对于其他Pod的重要性。当不能调度Pod时（如资源不足pod处于pending状态），scheduler会试图抢占（驱逐）较低优先级的Pod，以便调度高优先级的Pod。在未来的Kubernetes版本中优先级也将影响节点上的资源不足时pod的驱逐顺序。


### 如何使用Pod优先级和抢占
---
在kubernetes1.8中使用pod优先级和抢占请遵循以下步骤：
1. 开启这个特性
2. 添加一个或多个`PriorityClasses`对象。
3. 创建Pod并将PriorityClassName设置为第二步添加的PriorityClasses。当然我们一般不会直接创建pod，而是将PriorityClassName添加到集合对象的Pod模板中（如`Deployment`或`ReplicaController`）。

关于上面的步骤下面的章节会提供更多的信息。

### 开启优先级和抢占
---
Pod优先级和抢占在kubernetes1.8中默认被禁用。如果需要启用这个特性，需要在kube-apiserver中和kube-scheduler的启动命令行中开启：

```
--feature-gates=PodPriority=true
```
还需要kube-apiserver开启`scheduling.k8s.io/v1alpha1`的API和`Priority admission controller`。

```
--runtime-config=scheduling.k8s.io/v1alpha1=true --admission-control=...,Priority
```
上述配置开启之后，就可以创建`PriorityClasses `这个对象，然后正常创建Pod并设置之前`PriorityClasses`的`PriorityClassName`。

如果您试用过，然后决定禁用它时，则必须删除PodPriority命令行设置或将其设置为false，然后重新启动apiserver和scheduler。禁用功能后，现有的Pod将保留其优先级字段，但抢占用能会被禁用，在处理时也会忽略优先级字段，并且不能在新Pod中设置`PriorityClassName`。

### PriorityClass
---
`PriorityClass`在kubernetes中不会属于任何namespace，它定义了从优先级类名到优先级整数值的映射。优先级类名在`PriorityClass`spec文件的metadata中`name`字段。优先级数值定义在`value`字段。数值越高优先级越高。

`PriorityClass`对象可以是小于或等于10亿的任何32位整数值。较大的数字被保留给一般不会被抢占或驱逐的关键的系统级Pod。集群管理员应为每个他们想要的映射创建一个`PriorityClass`对象。

`PriorityClass`还有两个可选字段：`globalDefault`和`description`。 前者表示这个`PriorityClass`的值应该用于没有`PriorityClassName`的Pod。 在整个系统中只能有唯一的`PriorityClass`的`globalDefault`设置为true。如果没有设置`globalDefault`的`PriorityClass`，则没有`PriorityClassName`的Pod的优先级为零。

`description`字段是一个任意的字符串。这是为了告诉群集的用户什么时候应该使用这个`PriorityClass`。
> 注1：如果升级现有群集并启用此功能，则现有Pod的优先级将被视为零。
> 注2：添加一个`globalDefault`设置为true的`PriorityClass`不会改变现有Pod的优先级。这种`PriorityClass`的值仅用于添加了`PriorityClass`之后创建的Pod。
> 注3：如果删除`PriorityClass`，则使用已删除优先级类名称的现有Pod保持不变，但无法创建更多使用已删除的PriorityClass名称的Pod。

##### PriorityClass例子

```
apiVersion: scheduling.k8s.io/v1alpha1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "This priority class should be used for XYZ service pods only."
```

### Pod优先级
---

在拥有一个或多个`PriorityClass`之后，可以创建Pod，在其spec文件中指定其中一个`PriorityClass`名称。优先准入控制器(priority admission controller)使用`priorityClassName`字段并将数值赋予pod优先级值。如果没有找到对应的优先级，Pod将被拒绝。

以下YAML文件是使用前面示例中创建的`PriorityClass`的Pod配置的示例。优先准入控制器检查spec并将Pod的优先级解析为1000000。

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  priorityClassName: high-priority
```

### 抢占
---
当Pod被创建时，会进入队列并等待被调度。Scheduler从队列中选取一个Pod并尝试在一个Node上进行调度。如果没有发现满足Pod的spec所有要求的Node，则为处于pending状态的Pod触发抢占机制。我们先把pending状态的Pod称之为P。抢占机制是：试图找到一个Node，删除在上面运行的一个或多个低于P优先级的Pod，从而使P可以调度在该节点。如果找到这样的Node，则从节点中删除一个或多个较低优先级的Pod。在这些Pod删除后，P就可以调度在该Node。

**抢占的局限性(alpha版本)**

#### 抢占Pod饥饿
当Pod被抢占时，Pod会有一个graceful termination period，这段时间用于结束Pod的工作并退出。如果超过这个时间Pod还没有退出就会被强制kill掉。但是graceful termination period会在scheduler抢占Pod的时间与可以在Node上调度pending状态Pod的时间之间造成一个间隔。同时，scheduler会继续调度其他pending状态的Pod。当被抢占的Pod退出或处于终止状态时，scheduler会在待处理队列中继续调度Pod，当一些Pod调度到刚才的Node上且早于高优先级的Pod。在这种情况下，虽然刚才被强占的Pod都退出了，但是Pod P却不再适合Node N了。因此，scheduler必须抢占节点N上的其他Pod或者其他节点，以便P可以被调度。这种场景可能会再次被重复进行第二轮以及之后的抢先，Pod P可能一段时间内都不会被调度成功了。这种情况可能会在多种群集中造成一些问题，但在具有较高的Pod创建速率的集群中尤其突出。

上述问题将在beta版中解决。解决方案可点击此处[Preemption mechanics](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/scheduling/pod-preemption.md#preemption-mechanics)。

#### 无法支持PodDisruptionBudget

Pod Disruption Budget(PDB) 设置应用Pod处于运行状态最低个数，也可以设置应用Pod处于运行状态的最低百分比，这样可以保证在主动销毁应用Pod的时候，不会一次性销毁太多的应用Pod，从而保证业务不中断或业务SLA不降级。然而alpha版本的Pod优先级抢占并不会考虑PDB，当需要在加点上抢占Pod的时候。Kubernetes计划在beta版中添加对PDB支持，但即使在beta版中，考虑PDB也是尽力而为。Scheduler会试图找到那些不会违反PDB的Pod进行抢占，但是如果没有找到这样的Pod，那么抢占仍然会发生，而届时尽管会违反PDB，但是优先权较低的Pod还是将会被移除。

#### 与低优先级Pod的亲和关系(Inter-Pod affinity)

在Kubernetes1.8版本中，一个Node决定是否被抢占只需考虑目前这一个问题“当所有比待调度Pod优先级低的Pod被移除后，这个待调度Pod是否就可以调度在该Node上了?”

> 注：抢占并不一定会删除所有较低优先级的Pod。如果删除一部分低优先级Pod就能调度，则只有一部分较低优先级的Pod被删除。即便如此，上述问题的答案肯定是肯定的。如果答案是否定的，则节点不被认为是抢占。

如果一个待调度Pod与节点上的一个或多个较低优先级的Pod具有亲和性，则在没有这些较低优先级的Pod的情况下，不能满足Pod间的亲和性规则。在这种情况下，scheduler不会抢占节点上的任何Pod，而是寻找另一个节点。Scheduler会找到合适的节点或者也可能找不到。因此无法保证待调度的Pod都可以被成功调度。

Kubernetes可能会在将来的版本中解决这个问题，但还没有一个明确的计划。这个问题不会阻止Beta或GA版本的发布。部分原因是找到满足所有Pod间关联规则的较低优先级Pod的集合在计算上是昂贵的，并且为抢占机制增加了相当复杂的逻辑。此外，即使抢占保持较低优先级的Pod以满足Pod间的亲和性，较低优先级的Pod也可能被其他Pod被抢占，这样一来即使使用复杂逻辑实现了满足Pod亲和的抢占机制也就没有什么用了。

我们针对此问题推荐的解决方案是仅针对相同或更高优先级的Pod创建Pod间关联。

#### 跨节点抢占

假设Node N正在被抢占机制参考，以便可以在N上调度pending状态Pod P。有一种情况，只有在另一个节点上的Pod被抢占时P才可能在N上变得可行。我们看这个例子：
1. 当Pod P正在被计算是否可以调度在在Node N上。
2. Pod Q在另一个Node上运行，这个Node与Node N处于一个区域（zone）。
3. Pod P与Pod Q具有反亲和属性（anit-affinity）。
4. Pod P与当前区域内其他Pod没有反亲和的情况。
5. 为了调度Pod P到Node N上，Pod Q应该被抢占。但是scheduler不能执行跨节点抢占，因此最终计算结果是Pod P无法调度在Node N上。

如果将Pod Q从其Node上移除，则反亲和规则将会消失，这样的话Pod P可能在节点N上被调度。

如果Kubernetes找到一个性能合理的算法，可能会考虑在未来的版本中添加跨节点抢占。目前它不能承诺任何事情，并且跨节点抢占不会阻碍Beta或GA版本的发布。