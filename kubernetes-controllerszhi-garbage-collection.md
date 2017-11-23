# Garbage Collection

```
苏晓林 xiaol.su@haihangyun.com
```

Kubernetes的垃圾回收的作用是删除那些失去Owner的对象。

**注意**：垃圾回收是一个测试版功能，只在Kubernetes 1.4及更高版本中默认启用。

* Owner和dependents

* 垃圾回收器删除dependents的策略
* 
# Owners和dependents

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

当该对象进入“删除中”状态时，垃圾回收器会删除该对象的所有dependents。当删除所有dependents对象后（对象的 ownerReference.blockOwnerDeletion=true），删除Owner对象。



