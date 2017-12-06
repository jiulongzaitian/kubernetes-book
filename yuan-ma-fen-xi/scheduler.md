## kubelet介绍

```
作者：李昂 邮箱：liang@haihangyun.com
```

kubelet是在每个node节点上的一个类agent组件，他的主要作用简单来说就是监听三个pod源过来的pod事件运行或变更对应的pod从而达到期望状态。这三个pod源分别是apiserver、pod manifest文件和pod manifest url，这就意味着kubelet除了可以工作在标准的kubernetes集群模式下，还可以单独运行`Standalone`模式，直接作为一个符合kubernetes pod spec标准的容器执行代理，脱离其他kubernetes组件而整合到其他需要容器特性的系统中去。

虽然大体上看kubelet流程并不复杂，但是其接触到的外部系统是最多了包括容器运行时（docker、rkt等）、网络插件（calico、flannel）、以及数据卷插件（nfs、rbd等），而且kubernetes中基本都是遵循的declarative programming哲学使得所有代码的目标都是使系统的实际状态逐渐reconcile（调和）到期望状态。因此其中的各类manager尤其之多，管理各类资源达到期望状态。

