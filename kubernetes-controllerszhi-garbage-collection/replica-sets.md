# Replica Sets

```
苏晓林 xiaol.su@haihangyun.com
```

ReplicaSet是下一代的Replication Controller。两者之间的唯一区别在于对selector的支持。ReplicaSet支持集合类型的selector，而Replication Controller只支持等式类型的selector。

## 如何使用ReplicaSet {#how-to-use-a-replicaset}

基本上所有支持Replication Controllers的[`kubectl`](https://kubernetes.io/docs/user-guide/kubectl/)命令都支持ReplicaSet（[`rolling-update`](https://kubernetes.io/docs/user-guide/kubectl/v1.8/#rolling-update)命令除外）。当你准备使用滚动升级功能时，最好使用Deployments。并且，由于[`rolling-update`](https://kubernetes.io/docs/user-guide/kubectl/v1.8/#rolling-update)命令是固定式的，而Deployments属于声明式，因此强烈建议使用[`rollout`](https://kubernetes.io/docs/user-guide/kubectl/v1.8/#rollout)命令使用Deployments。

虽然ReplicaSet能够独立使用，但是在实际使用过程中，通常通过Deployments进行管理。当通过Deployments管理ReplicaSet时，你不必关心ReplicaSet，Deployments会自动对其进行管理。

## 何时使用ReplicaSet

ReplicaSet的作用是确保指定数量的pods在运行。而Deployment作为一个上层概念，通过管理ReplicaSet来来完成pod的更新和其它一些特性。强烈建议通过Deployments来管理ReplicaSets，而不要直接操作ReplicaSets，除非需要进行自定义更新流程或根本不需要更行。

因此实际上你可能永远也不用直接操作ReplicaSet，通过Deployments来管理并定义你的应用程序。

#### ReplicaSet例子

frontend.yaml

```
apiVersion: apps/v1beta2 # for versions before 1.8.0 use apps/v1beta1
kind: ReplicaSet
metadata:
  name: frontend
  labels:
    app: guestbook
    tier: frontend
spec:
  # this replicas value is default
  # modify it according to your case
  replicas: 3
  selector:
    matchLabels:
      tier: frontend
    matchExpressions:
      - {key: tier, operator: In, values: [frontend]}
  template:
    metadata:
      labels:
        app: guestbook
        tier: frontend
    spec:
      containers:
      - name: php-redis
        image: gcr.io/google_samples/gb-frontend:v3
        resources:
          requests:
            cpu: 100m
            memory: 100Mi
        env:
        - name: GET_HOSTS_FROM
          value: dns
          # If your cluster config does not include a dns service, then to
          # instead access environment variables to find service host
          # info, comment out the 'value: dns' line above, and uncomment the
          # line below.
          # value: env
        ports:
        - containerPort: 80
```

创建并查看该ReplicaSet及其管理的pods

```
$ kubectl create -f frontend.yaml
replicaset "frontend" created
$ kubectl describe rs/frontend
Name:		frontend
Namespace:	default
Selector:	tier=frontend,tier in (frontend)
Labels:		app=guestbook
		tier=frontend
Annotations:	<none>
Replicas:	3 current / 3 desired
Pods Status:	3 Running / 0 Waiting / 0 Succeeded / 0 Failed
Pod Template:
  Labels:       app=guestbook
                tier=frontend
  Containers:
   php-redis:
    Image:      gcr.io/google_samples/gb-frontend:v3
    Port:       80/TCP
    Requests:
      cpu:      100m
      memory:   100Mi
    Environment:
      GET_HOSTS_FROM:   dns
    Mounts:             <none>
  Volumes:              <none>
Events:
  FirstSeen    LastSeen    Count    From                SubobjectPath    Type        Reason            Message
  ---------    --------    -----    ----                -------------    --------    ------            -------
  1m           1m          1        {replicaset-controller }             Normal      SuccessfulCreate  Created pod: frontend-qhloh
  1m           1m          1        {replicaset-controller }             Normal      SuccessfulCreate  Created pod: frontend-dnjpy
  1m           1m          1        {replicaset-controller }             Normal      SuccessfulCreate  Created pod: frontend-9si5l
$ kubectl get pods
NAME             READY     STATUS    RESTARTS   AGE
frontend-9si5l   1/1       Running   0          1m
frontend-dnjpy   1/1       Running   0          1m
frontend-qhloh   1/1       Running   0          1m
```

## 定义ReplicaSet

与其它的Kubernetes API一样，ReplicaSet也需要`apiVersion，kind`和`metadata`字段。关于通用字段信息可查看[对象管理](https://kubernetes.io/docs/tutorials/object-management-kubectl/object-management/)

ReplicaSet的[`.spec`](https://git.k8s.io/community/contributors/devel/api-conventions.md#spec-and-status)信息：

##### Pod Template

`.spec.template`字段是`.spec`的唯一必需字段。该字段与pod的定义模版几乎完全一致，但是没有`apiVersion`和`kind`字段，并且必须指定标签和重启策略。

标签必须不能与其它controllers的标签重叠，标签的详细信息可以查看[pod selector](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/#pod-selector)。

对于重启策略，`.spec.template.spec.restartPolicy`字段唯一允许的值为`Always`，默认即为此值。

##### Pod Selector

`.spec.selector`字段是一个标签selector，ReplicaSet会管理所有标签与该selector匹配的pods。无论pod是否由其创建、删除。因此可以在不影响运行中pod的情况下，更改该pod的ReplicaSet。

对于`.spec.template.metadata.labels`字段不匹配`.spec.selector`的pod，将会被拒绝。

在Kubernetes 1.8中，`apps/v1beta2`已经作为默认版本，`extensions/v1beta1`版本将会被移除。在`apps/v1beta2`版本中，`.spec.selector`和`.metadata.labels`字段将不会默认填充`.spec.template.metadata.labels`字段的值，必须明确指定。注意，在`apps/v1beta2`版本中，`.spec.selector`字段在创建后将不允许改变。并且不要以任何其它的方式创建带有能够匹配该selector标签的pods，因为该ReplicaSet会认为这些pods是自己创建的。Kubernetes不会阻止你这么做，所以你必须自己规避这种情况。

当多个controllers的selectors重叠时，你需要自己管理删除操作。

##### ReplicaSet标签

ReplicaSet拥有自己的标签字段（`.metadata.labels`）。通常该字段的值与`.spec.template.metadata.labels`的值一致。但并不是强制性的，且`.metadata.labels`的值并不影响ReplicaSet的行为。

##### Replicas

通过设置`.spec.replicas`字段来控制pods的数量。未指定时，默认值为1。

## ReplicaSet操作

##### 删除ReplicaSet及其pods

可以通过命令[`kubectl delete`](https://kubernetes.io/docs/user-guide/kubectl/v1.8/#delete)ReplicaSet及其pods。Kubectl会首先将ReplicaSet的Replicas设置为0，然后等待所有pods删除以后，最后删除该ReplicaSet。如果这个kubectl命令被中断，它可以被重新启动。

当使用REST API或go客户端删除ReplicaSet及其pods时，你必须自己完成Kubectl完成的三个步骤（1.设置ReplicaSet为0  2.等待所有pods删除 3.删除ReplicaSet）。

##### 只删除ReplicaSet

通过[`kubectl delete`](https://kubernetes.io/docs/user-guide/kubectl/v1.8/#delete)的`--cascade=false`选项可以实现只删除ReplicaSet，而不影响其拥有的pods。

当使用REST API或go客户端时，只需删除该ReplicaSet即可。

当删除ReplicaSet后，你可以新建一个`.spec.selector`字段值相同的ReplicaSet来替代它。先创建的ReplicaSet会接管之前的pods。这种方式不会对之前已经存在的pod有任何影响。当你需要升级pod时，请使用[滚动升级](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/#rolling-updates)。

##### pod脱离

pod可通过更改自己的标签，来脱离ReplicaSet的控制。当你需要调试或数据恢复时，可能会这么做。通过这种方式移除的pod，ReplicaSet会自动创建新的pod来替代原来的pod。

##### 弹性伸缩

通过设置`.spec.replicas`字段可以控制该ReplicaSet下pods的数量。ReplicaSet controller会确保该ReplicaSet下有指定数量的可用的pods。

##### 作为Horizontal Pod Autoscaler的目标

可以把ReplicaSet作为[Horizontal Pod Autoscalers \(HPA\)](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)的目标，来实现自动扩容／缩容。下面是一个HPA的例子。

```
//hpa-rs.yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: frontend-scaler
spec:
  scaleTargetRef:
    kind: ReplicaSet
    name: frontend
  minReplicas: 3
  maxReplicas: 10
  targetCPUUtilizationPercentage: 50
```

创建该HPA

```
kubectl create -f hpa-rs.yaml
```

也可以通过`kubectl autoscale`命令完成上述操作

```
kubectl autoscale rs frontend
```

## ReplicaSet的可替代者

##### Deployment \(推荐\)

Deployment是一个更高层的API对象，可以像`kubectl rolling-update`命令一样，升级其下的ReplicaSets及pods。目前官方推荐使用Deployment来完成滚动升级功能。因为Deploymen是声明式、服务端的，并且有一些附加特性。想查看更多通过Deployment部署无状态应用的信息，请点击[Run a Stateless Application Using a Deployment](https://kubernetes.io/docs/tasks/run-application/run-stateless-application-deployment/)。



