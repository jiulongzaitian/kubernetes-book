# Garbage Collection

```
苏晓林 xiaol.su@haihangyun.com
```

Kubernetes的垃圾回收的作用是删除那些失去Owner的对象。

**注意**：垃圾回收是一个测试版功能，只在Kubernetes 1.4及更高版本中默认启用。

* Owner和dependents

* 垃圾回收器删除dependents的策略

* # Owners和dependents

Kubernetes中，一些对象是另一些对象的Owner，如：ReplicaSet是一组Pod的Owner。属于该Owner的对象称为该Owner的dependent。每个dependent对象都拥有`metadata.ownerReferences`字段来指向其Owner。

一些情况下，Kubernetes会自动设置`ownerReference`字段的值。如：当创建ReplicaSet时，Kubernetes会自动该ReplicaSet所有Pod的`ownerReference`字段指向该ReplicaSet。在Kubernetes 1.8版本，由ReplicationController, ReplicaSet, StatefulSet, DaemonSet, Deployment, Job and CronJob创建或管理的对象都会自动设置`ownerReference`字段。Kubernetes也支持手动设置`ownerReference`字段来指定Owner和dependent的关系。

下面示例中，展示了一个拥有3个Pods的ReplicaSet的配置

```
//my-repset.yaml
apiVersion: extensions/v1beta1
kind: ReplicaSet
metadata:
  name: my-repset
spec:
  replicas: 3
  selector:
    matchLabels:
      pod-is-for: garbage-collection-example
  template:
    metadata:
      labels:
        pod-is-for: garbage-collection-example
    spec:
      containers:
      - name: nginx
        image: nginx
```

创建该ReplicaSet，并查看Pod的`metadata.ownerReferences`字段

```
kubectl create -f https://k8s.io/docs/concepts/controllers/my-repset.yaml
kubectl get pods --output=yaml
```

下面输出结果中可以看到，该Pod的Owner类型为ReplicaSet，名称为my-repset

```
apiVersion: v1
kind: Pod
metadata:
  ...
  ownerReferences:
  - apiVersion: extensions/v1beta1
    controller: true
    blockOwnerDeletion: true
    kind: ReplicaSet
    name: my-repset
    uid: d9607e19-f88f-11e6-a518-42010a800195
  ...
```

# 垃圾回收器删除dependents的方式

当删除某个对象时，可以设置是否自动删除该对象的dependents。自动删除dependents也称为级联删除。Kubernetes中有两种级联删除的模式：background和foreground。

非自动删除的dependents对象将变为孤儿对象。

## Foreground级联删除

在foreground级联删除模式中，root对象会首先进入“删除中”状态，在该状态中：

* 该对象在REST API中仍然可见
* 对象的`deletionTimestamp`字段为已设置
* 对象的`metadata.finalizers`字段将包含值“foregroundDeletion”

当该对象进入“删除中”状态时，垃圾回收器会删除该对象的所有dependents。当删除所有dependents对象后（对象的 ownerReference.blockOwnerDeletion=true），删除Owner对象。

注意，在 “foreground 删除” 模式下，Dependent 只有通过 ownerReference.blockOwnerDeletion 才能阻止删除 Owner 对象。在 Kubernetes 1.7 版本中增加了 admission controller，基于 Owner 对象上的删除权限来控制用户去设置 `blockOwnerDeletion` 的值为 true，未授权的 Dependent 不能够延迟 Owner 对象的删除。

如果一个对象的`ownerReferences` 字段被一个 Controller（例如 Deployment 或 ReplicaSet）设置，blockOwnerDeletion 会被自动设置，没必要手动修改这个字段。

## Background级联删除

在 background 级联删除 模式下，Kubernetes 会立即删除 Owner 对象，然后垃圾收集器会在后台删除dependents。

## 设置级联删除策略

通过为 Owner 对象设置` deleteOptions.propagationPolicy `字段，可以控制级联删除策略。支持的值为：“orphan”、“Foreground” 或 “Background”。

对很多 Controller 资源，包括 ReplicationController、ReplicaSet、StatefulSet、DaemonSet 和 Deployment，默认的垃圾收集策略是 `orphan`。因此，除非指定其它的垃圾收集策略，否则所有 dependent 对象使用的都是 orphan 策略。

下面是一个在后台删除 dependent 对象的例子：

```
kubectl proxy --port=8080
curl -X DELETE localhost:8080/apis/extensions/v1beta1/namespaces/default/replicasets/my-repset \
-d '{"kind":"DeleteOptions","apiVersion":"v1","propagationPolicy":"Background"}' \
-H "Content-Type: application/json"
```

下面是一个在前台删除dependent 对象的例子：

```
kubectl proxy --port=8080
curl -X DELETE localhost:8080/apis/extensions/v1beta1/namespaces/default/replicasets/my-repset \
-d '{"kind":"DeleteOptions","apiVersion":"v1","propagationPolicy":"Foreground"}' \
-H "Content-Type: application/json"
```

下面是一个孤儿 Dependent 的例子：

```
kubectl proxy --port=8080
curl -X DELETE localhost:8080/apis/extensions/v1beta1/namespaces/default/replicasets/my-repset \
-d '{"kind":"DeleteOptions","apiVersion":"v1","propagationPolicy":"Orphan"}' \
-H "Content-Type: application/json"
```

kubectl 也支持级联删除。 通过设置 --cascade 为 true，可以使用 kubectl 自动删除 dependent 对象。设置 --cascade 为 false，会使 dependent 对象成为孤儿 dependent 对象。--cascade 的默认值是 true。

下面是一个例子，使一个ReplicaSet的dependent成为孤儿 dependent：

```
kubectl delete replicaset my-repset --cascade=false
```

## 关于Deployments的注意事项

当使用级联方式删除Deployments时，必须使用字段`propagationPolicy: Foreground`不仅要删除ReplicaSets，还要删除它们的Pods。如果为设置`propagationPolicy`字段，只会删除ReplicaSets，Pods将变为孤儿对象。更多信息请点击[kubeadm/\#149](https://github.com/kubernetes/kubeadm/issues/149#issuecomment-284766613)。

  


