## 总结
Kubernetes的kube-scheduler是以plugin形式开发出来的，方便用户开发自己的调度器。
其核心调度过程其实是对kube-apiserver的`pod`对象的CRUD：

* 客户端通过kubectl或者restapi获取kube-apiserver中NodeName为空的`pod`。
* 调度器通过一系列调度算法计算出最适合`pod`的`node`。
* 最后使用kubectl或者restapi向kube-apiserver中写入`pod`的Binding信息（包含之前计算出来的最适合`node`的NodeName）。

实现自己的调度器并不困难，Kubernetes官方也提供了一个基于shell脚本的[随机调度器](http://blog.kubernetes.io/2017/03/advanced-scheduling-in-kubernetes.html)，

本文只是分析了kube-scheduler的执行流程和重点的调度算法，而Kubernetes作为**Production-Grade**生产级容器调度管理平台，其代码在细节处理以及可靠性上是非常高的。而Kubernetes也成为当前十分活跃的社区，其中也有大量高手贡献代码，由此看来开源项目和社区形成的非常好的良性循环成就kubernetes为容器平台真正的领跑者。


## 参考文章

* [使用 client-go 控制原生及拓展的 Kubernetes API](https://www.kubernetes.org.cn/1309.html)
* [Advanced Scheduling in Kubernetes](http://blog.kubernetes.io/2017/03/advanced-scheduling-in-kubernetes.html)
