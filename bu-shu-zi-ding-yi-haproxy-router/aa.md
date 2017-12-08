# 部署自定义HAProxy Router

```
苏晓林 xiaol.su@haihangyun.com                    openshift version:3.6
```

## 概览

默认的HAProxy Router已经能够满足绝大部分的用户需求。但是默认的HAProxy Router并不支持所有的HAProxy功能，一些用户可能需要用到这些功能。

有的用户可能需要用到一些application back-ends的新特性，或者修改一些当前操作。router plug-in能通过定制的方式满足所有这些需求。

Router pod使用[golang template](http://golang.org/pkg/text/template/)来创建HAProxy的配置文件。当处理template时，Router需要访问OpenShift Origin的信息，包括Router的部署配置，允许的routes集合和其它一些帮助功能。

当Router pod启动和每次reload时，都会创建一个HAProxy配置文件，然后启动HAProxy。[HAProxy configuration manual](https://cbonte.github.io/haproxy-dconv/configuration-1.5.html) 给出HAProxy的所有特性，以及如何构建一个有效的配置文件。

可以通过将[**configMap**](https://docs.openshift.org/latest/install_config/router/customized_haproxy_router.html#using-configmap-replace-template)添加到template来部署Router pod。通过这种方法部署时，需要修改Router的部署配置，将**configMap**以volume的方式挂载到Router pod中。并且将环境变量`TEMPLATE_FILE`配置为template文件在Router pod中的全路径。

或者，你也可以构建一个自定义的Router镜像，通过它来部署你的Routers。构建Routers的镜像可以不同。通过修改_**haproxy-template.config**_文件，然后重新构建Router的镜像。新的镜像会被上传到集群的镜像仓库，并且Router的部署配置中，**image:**字段更新为新的镜像名称。当集群有更改时，需要重新构建镜像并上传。

无论哪种方式，router pod的部署都是从template文件开始。

## 获得Router的template

HAProxy的template非常巨大且复杂。通常通过修改已有template的方式会更加方便和简单。可以通过以下命令在一个运行router的master节点上获得一个新的_**haproxy-config.template**_。





