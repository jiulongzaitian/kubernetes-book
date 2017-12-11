

# kubelet CRI

```
作者 张杰  j.zhang8@haihangyun.com
```



每种容器运行时各有所长，许多用户都希望Kubernetes支持更多的运行时,Kubelet与容器运行时通信（或者是CRI插件填充了容器运行时）时，Kubelet就像是客户端，而CRI插件就像对应的服务器。它们之间可以通过Unix 套接字或者gRPC框架进行通信。

![](/assets/import.png)

protocol buffers API包含了两个gRPC服务：ImageService和RuntimeService。ImageService提供了从镜像仓库拉取、查看、和移除镜像的RPC。RuntimeSerivce包含了Pods和容器生命周期管理的RPC，以及跟容器交互的调用\(exec/attach/port-forward\)。一个单块的容器运行时能够管理镜像和容器（例如：Docker和Rkt），并且通过同一个套接字同时提供这两种服务。这个套接字可以在Kubelet里通过标识–container-runtime-endpoint和–image-service-endpoint进行设置。



## 源码分析

kubelet 的CRI 是用proto buffer  和 GRPC 实现的

目录:pkg/kubelet/apis/cri/v1alpha1/runtime

```
api.pb.go  api.proto  BUILD  constants.go
```

其中有一个api.proto 的描述文件，用来生成api.pb.go 代码，具体可以查看[golang grpc 初体验](http://www.jianshu.com/p/774b38306c30)，

