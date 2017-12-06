## kubelet主要功能

```
作者：李昂 邮箱：liang@haihangyun.com
```

1. pod生命周期管理：在kubernetes中最小的管理单位是pod，而pod中可能会有多个container，他们会共享network和IPC namesapce。主要参与pod管理的有如下manager：

2. `podManager`：保存着运行在当前node的每个pod和mirrorpod的UID、fullname到pod spec的映射，还有mirrorpod UID到pod UID的映射（mirrorpod是指：那些kubelet从file、url中创建的pod称之为static pod，而为了方便查看的管理kubelet会给static pod在apiserver中创建对应的mirror pod），有了这些映射关系，kubelet在运行时就可以通过UID或者Fullname索引到任意pod，以及获取全部在本节点上运行的pod。

3. `podCache`：缓存pod的status运行时状态，其中每条缓存记录带有一个`modified`更新时间，整个缓存有一个全局时间，所有记录的更新时间必须要不早于全局时间。`podCache`提供了一个方法可以获取一条有指定时间新的记录，否则会一直阻塞，直到获取到。

4. `pleg`\(Pod Lifecircle event generator\)：主要任务是检测实际环境中container出现的变化，其每秒钟列出列出当前节点所有pod和所有container，与自己缓存中podRecord中对比，生成每个container的event，送到eventchannel，kubelet主循环syncLoop负责处理eventchannel中的事件。
5. `statusManager`：用于保存pod运行时状态，以及同步此状态到apiserver中，主要与probeManager和主循环syncloop有交互。
6. `podWorkers`：负责处理pod的各类变更事件，pod在同一时间只能被一个worker处理。

7. 主机容器监控以及垃圾回收：kubelet会内置cadvisor来收集主机和容器监控数据，以及定期执行垃圾回收释放容器和镜像占用的空间。

8. `cadvisor`：kubelet内置本地容器主机监控工具，会收集主机信息主动上报给apiserver。其他组件也会调用cadvisor接口获取容器或主机监控信息。

9. `containerGC`：根据回收策略删除`dead`容器。

10. `imageManager`：通过查询cadvisor接口获取镜像文件系统占用量，从而根据回收策略释放空间。

11. 容器健康检查：有三种方式http、tcp、exec。如果pod的spec中配置了健康监测`livenessProbe`或者`readinessProbe`，kubelet会启动独立goroutine根据spec探测容器。

12. `probeManager`：其中包含`statusManager`、`readinessManager`和`livenessManager`。`readinessManager`在探测container后会设置`statusManager`对应pod的container状态。而`livenessManager`探测后的事件会在syncloop主循环中处理。

13. 数据卷生命周期管理：kubernetes支持众多数据卷除此之外还有更加灵活的PersistentVolumeClaim和PersistentVolume。

14. `volumeManager`：同样是由多个异步循环组成，同样遵循着期望状态\(`desiredStateOfWorld`\)和实际状态\(`actualStateOfWorld`\)这种设计模式，在异步循环中使用`reconciler`将实际状态转化为pod spec中的期望状态。

15. Api接口：kubelet提供api包含node信息，运行的pod信息、获取pod的log日志以及在指定pod的容器中执行命令等。除此以外cadvisor运行在kubelet内部，其api同时也会暴露出来。



