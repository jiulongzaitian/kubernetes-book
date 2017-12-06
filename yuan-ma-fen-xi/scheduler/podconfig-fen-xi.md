# PodConfig 分析

```
作者：张杰 邮箱：j.zhang8@haihangyun.com
```

---

kubelet 可以支持三类Pod的创建方式：

* From File    配置文件
* From Http    HTTP
* From ApiServer    APiserver



前两种方式可以称之为Static Pod

配置文件就是放在特定目录下的标准的JSON或YAML格式的pod定义文件。用`kubelet --pod-manifest-path=<the directory>`来启动kubelet进程，kubelet将会周期扫描这个目录，根据这个目录下出现或消失的YAML/JSON文件来创建或删除静态pod。

通过HTTP创建静态Pods Kubelet周期地从–manifest-url=参数指定的地址下载文件，并且把它翻译成JSON/YAML格式的pod定义。此后的操作方式与--pod-manifest-path=相同，kubelet会不时地重新下载该文件，当文件变化时对应地终止或启动静态pod

具体对应的代码为：

```golang
pkg/kubelet/kubelet.go


```



