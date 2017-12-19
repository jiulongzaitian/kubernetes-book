# kubelet Run 方法解读

```
作者：张杰  j.zhang8@haihangyun.com
```

读此文章之前，建议先看 [kubelet 核心组件](/yuan-ma-fen-xi/scheduler/kubelethe-xin-zu-jian.md) 和 [kubelet  核心manager](/yuan-ma-fen-xi/scheduler/kubelet-he-xin-manager.md)

# 主流程图解

![](/assets/1.png)

# sourcesReady

在看kubelet代码时候，会经常看到 sourcesReady 的字段，或者 AllReady 的方法, 可以查看：[ kubelet sourceReady](/yuan-ma-fen-xi/scheduler/kubelet-sourceready.md) 一章



pkg/kubelet/kubelet.go

Kubelet ： Run\(\) -&gt; kl.initializeModules\(\) -&gt; [kl.imageManager.Start\(\)](/yuan-ma-fen-xi/scheduler/kubelet-image-la-ji-hui-shou.md) 



# ... 未完待续



