# Replica Sets

```
苏晓林 xiaol.su@haihangyun.com
```

ReplicaSet是下一代的Replication Controller。两者之间的唯一区别在于对selector的支持。ReplicaSet支持集合类型的selector，而Replication Controller只支持等式类型的selector。

## 如何使用ReplicaSet {#how-to-use-a-replicaset}

基本上所有支持Replication Controllers的[`kubectl`](https://kubernetes.io/docs/user-guide/kubectl/)命令都支持ReplicaSet（[`rolling-update`](https://kubernetes.io/docs/user-guide/kubectl/v1.8/#rolling-update)命令除外）。当你准备使用滚动升级功能时，最好使用Deployments。并且，由于[`rolling-update`](https://kubernetes.io/docs/user-guide/kubectl/v1.8/#rolling-update)命令是固定式的，而Deployments属于声明式，因此强烈建议使用[`rollout`](https://kubernetes.io/docs/user-guide/kubectl/v1.8/#rollout)命令使用Deployments。

虽然ReplicaSet能够独立使用，但是在实际使用过程中，通常通过Deployments进行管理。当通过Deployments管理ReplicaSet时，你不必关心ReplicaSet，Deployments会自动对其进行管理。

## 何时使用ReplicaSet

ReplicaSet的作用是确保指定数量的pods在运行。而Deployment



