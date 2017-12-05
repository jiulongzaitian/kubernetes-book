## kube-scheduler源码分析

```
                                作者：李昂  邮箱：liang@haihangyun.com
```

## kube-scheduler作用

Kube-scheduler作为组件运行在`master`节点，主要任务是把从kube-apiserver中获取的未被调度的`pod`调度到最适合的`node`上。Kube-scheduler以插件形式运行，用户如果觉得官方的不满足需求还可以自己实现调度插件。目前Kubernetes同时支持多个kube-scheduler同时运行，以调度器名字区分，每个`pod`的spec文件中可以指定调度自己的kube-scheduler的`schedulerName`。

Kube-scheduler主要调度框架其实就是输入`pod`，经过一系列Predicate和Priority调度算法后排序每个`node`的得分最后输出得分最高的`node`。

