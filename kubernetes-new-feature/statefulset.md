### StatefulSets

```
作者：李昂 邮箱：liang@haihangyun.com
```
---
StatefulSet是用于管理有状态应用程序的工作负载API对象。
> 注：StatefulSets在1.9版本中已经是stable稳定版。

管理一组Pod的部署和扩展，并为这些Pod的顺序和唯一性提供保证。

像Deployment一样，StatefulSet管理基于相同容器spec的Pod。与Deployment不同是，StatefulSet为每个Pod维护一个粘性标识。这些Pod是从相同的spec创建的，但是不可互换：每个Pod都有一个持久的标识符，在任何重调度中都会一直保持。

StatefulSet与任何其他Controller一样运行。您可以在一个StatefulSet对象中定义所需的状态，并且StatefulSet控制器会进行必要的更新以从当前状态达到期望状态。

### 使用StatefulSets

---
StatefulSets对于需要以下一项或多项的应用程序非常有用。

- 稳定，唯一的网络标识符。 
- 稳定，持久的存储。 
- 有序，优雅的部署和缩放。 
- 有序，优雅的删除和终止。 
- 有序，自动滚动更新。

在上文中，稳定性与Pod（重新）调度中的持久性是同义的。如果应用程序不需要任何稳定的标识符或有序的部署，删除或扩展，则应该使用提供一组无状态副本的控制器部署应用程序。诸如Deployment或ReplicaSet等控制器可能更适合您的无状态需求。

### 限制

---
- StatefulSet是1.9之前的beta版资源，在1.5之前的任何Kubernetes版本中都不可用。 
- 与所有的alpha/beta资源一样，您可以通过传递给apiserver的--runtime-config选项禁用StatefulSet。 
- 给定Pod的存储必须由PersistentVolume Provisioner根据请求的存储级别进行设置，或者由管理员预先设置。 
- 删除和/或缩放StatefulSet不会删除与StatefulSet关联的卷。这样做是为了确保数据安全，这通常比自动清除所有相关的StatefulSet资源更有价值。 
- StatefulSets目前需要一个Headless Service来负责Pods的网络身份。用户负责创建此服务。

### 组成

---
下面的例子演示了一个StatefulSet的组件。 
- 名为nginx的Headless Service用于控制网络域。 
- StatefulSet（名为web）具有一个Spec，它指示nginx容器的3个副本将在唯一的Pod中启动。 
- volumeClaimTemplates将使用由PersistentVolume Provisioner置备的PersistentVolume来提供稳定的存储。
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  selector:
    matchLabels:
      app: nginx # has to match .spec.template.metadata.labels
  serviceName: "nginx"
  replicas: 3 # by default is 1
  template:
    metadata:
      labels:
        app: nginx # has to match .spec.selector.matchLabels
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: nginx
        image: gcr.io/google_containers/nginx-slim:0.8
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: my-storage-class
      resources:
        requests:
          storage: 1Gi
```

### Pod Selector

---
您必须将StatefulSet的spec.selector字段设置为匹配其.spec.template.metadata.labels的标签。在Kubernetes 1.8之前，spec.selector字段在省略时是默认的。在1.8及更高版本中，无法指定匹配的Pod选择器将在创建StatefulSet期间导致验证错误。

### Pod标识

---
StatefulSet Pod有一个唯一的标识，由一个序号，一个稳定的网络标识和稳定的存储组成。无论哪个节点（重新）安排在身上，身份都粘在Pod上。

#### 序数索引

对于一个有N个副本的StatefulSet，StatefulSet中的每个Pod将被分配一个整数序列，范围是[0，N），这个序号在Set上是唯一的。

#### 稳定的网络ID

StatefulSet中的每个Pod从StatefulSet的名称和Pod的序号中派生出主机名。构造主机名的模式是$(statefulset name) - $(ordinal)。上面的例子将创建三个名为web-0，web-1，web-2的Pod。 StatefulSet可以使用无头服务来控制其Pod的域。由此服务管理的域采用以下形式：$(service name)。$(namespace).svc.cluster.local，其中“cluster.local”是集群域。当创建每个Pod时，它将获得匹配的DNS子域，格式为：$
(podname)。$(管理服务域)，其中管理服务由StatefulSet上的serviceName字段定义。

Cluster Domain	 | Service (ns/name) | StatefulSet (ns/name) | StatefulSet Domain | Pod DNS | Pod Hostname
----|------|---- | ----|------|----
cluster.local| default/nginx| default/web| nginx.default.svc.cluster.local	 | web-{0..N-1}.nginx.default.svc.cluster.local| web-{0..N-1}
cluster.local| foo/nginx	  | foo/web	 | nginx.foo.svc.cluster.local	 | web-{0..N-1}.nginx.foo.svc.cluster.local	  | web-{0..N-1}
kube.local	 | foo/nginx	  | foo/web	 | nginx.foo.svc.kube.local	 | web-{0..N-1}.nginx.foo.svc.kube.local	  | web-{0..N-1}

请注意，除非另行配置，否则群集域将设置为cluster.local。

#### 持久存储

Kubernetes为每个VolumeClaimTemplate创建一个PersistentVolume。在上面的nginx示例中，每个Pod将收到一个具有my-storage-class的StorageClass的PersistentVolume和一个1 Gb的配置存储。如果没有指定StorageClass，则将使用默认的StorageClass。当一个Pod被重新调度到一个节点上时，它的volumeMounts将挂载与其PersistentVolume声明相关联的PersistentVolumes。请注意，删除Pod或StatefulSet时，与Pods的PersistentVolume声明关联的PersistentVolume不会被删除。这必须手动完成。

#### Pod名称标签

当StatefulSet控制器创建一个Pod时，它添加一个标签statefulset.kubernetes.io/pod-name，它被设置为Pod的名字。此标签允许您将服务附加到StatefulSet中的特定Pod。

### 部署和缩放保证

---
- 对于具有N个副本的StatefulSet，当部署Pod时，按顺序从{0..N-1}创建它们。 
- 当Pod被删除时，它们以相反的顺序从{N-1..0}终止。 
- 在对Pod进行缩放操作之前，其所有前面的Pod都必须是Running和Ready。
- 在Pod终止之前，所有的后面的Pod必须完全关闭。

StatefulSet不应该指定pod.Spec.TerminationGracePeriodSeconds为0.这种做法是不安全的，强烈地阻止。有关详细说明，请参阅强制删除StatefulSet Pod。

当创建上面的nginx示例时，将按照web-0，web-1，web-2的顺序部署三个Pod。在web-0运行并准备就绪之前web-1将不会部署，并且在web-1正在运行和准备就绪之前web-2将不会部署。如果web-0失败，在web-1运行并准备好之后，但在web-2启动之前，web-2将不会启动，直到web-0成功重新启动并成为Running and Ready。

如果用户修改StatefulSet来扩展已部署的示例，以使replicas= 1，则web-2将首先被终止。在web-2完全关闭和删除之前，web-1将不会被终止。如果web-0在web-2终止并完全关闭之后，但在web-1终止之前失败，那么web-1将不会被终止，直到web-0正在运行并就绪。

#### Pod管理策略

在Kubernetes 1.7和更高版本中，StatefulSet允许您通过其.spec.podManagementPolicy字段保留其唯一性和身份保证，从而放松顺序的保障。

##### OrderedReady Pod管理
OrderedReady pod管理是StatefulSets的默认设置。它实现了上面描述的行为。

##### 并行Pod管理
并行pod管理告诉StatefulSet控制器并行地启动或终止所有的Pod，并且在启动或终止另一个Pod之前不等待Pods变为Running和Ready或完全终止。

### 更新策略

---
在Kubernetes 1.7及更高版本中，StatefulSet的.spec.updateStrategy字段允许您为StatefulSet中的容器配置和禁用容器，标签，资源请求/限制和注释的自动滚动更新。

#### On Delete
OnDelete更新策略实现了传统（1.6和更早版本）的行为。这是spec.updateStrategy未指定时的默认策略。当StatefulSet的.spec.updateStrategy.type设置为OnDelete时，StatefulSet控制器将不会自动更新StatefulSet中的Pod。用户必须手动删除Pod以使控制器创建反映对StatefulSet的.spec.template所做修改的新Pod。

#### Rolling Updates
RollingUpdate更新策略为StatefulSet中的Pod实现自动滚动更新。当StatefulSet的.spec.updateStrategy.type设置为RollingUpdate时，StatefulSet控制器将删除并重新创建StatefulSet中的每个Pod。它将按照Pod终止（从最大的序数到最小）的顺序进行，每次更新一个Pod。在更新其前任之前，它将等待更新的Pod正在运行并准备就绪。

##### Partitions
RollingUpdate更新策略可以通过指定.spec.updateStrategy.rollingUpdate.partition进行分区。如果指定了分区，当StatefulSet的.spec.template被更新时，所有具有大于或等于该分区的序号的Pod将被更新。所有具有小于分区的序号的Pod不会被更新，即使它们被删除，它们也会在以前的版本中被重新创建。如果StatefulSet的.spec.updateStrategy.rollingUpdate.partition大于它的.spec.replicas，那么它的.spec.template的更新将不会传播到它的Pod。在大多数情况下，你不需要使用分区，但是如果你想要更新，推出一个金丝雀，或者分阶段推出，它们就很有用。
