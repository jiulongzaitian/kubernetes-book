# ConfigMap介绍



## ConfigMap的使命

在许多应用程序的实际配置中，常常需要通过命令行参数和环境变量进行混合的配置。复杂的配置会严重增加容器的部署和更新的难度、降低容器的可移植性。ConfigMap的出现就是为了解决该问题，通过ConfigMap将配置与镜像内容进行分离，镜像内容中不再包含需要动态变化的配置，全部放在ConfigMap中。从而使容器镜像具备了更好的可移植性，部署和更新也变得更容易。ConfigMap能够存储细粒度的信息（如单个属性）和粗粒度的信息（如整个配置文件或JSON对象），在容器运行时通过ConfigMap API资源将配置注入到容器中。

## ConfigMap概述

ConfigMap API资源中能够保存Pod中能够使用的key-value配置，也能够存储系统组件（如Controller）的配置数据。ConfigMap在一定程度上与Secrets有些类似，但是ConfigMap在设计能够更方便的支持不包含敏感信息的字符串。

ConfigMap示例：

```
kind: ConfigMap
apiVersion: v1
metadata:
  creationTimestamp: 2016-02-18T19:14:38Z
  name: example-config
  namespace: default
data:
  example.property.1: hello
  example.property.2: world
  example.property.file: |-
    property.1=value-1
    property.2=value-2
    property.3=value-3

```

在该示例中，data字段包含了细粒度的配置信息example.property.1和example.property.2，也包括粗粒度的信息example.property.file。

ConfigMap在Pod中的使用支持多种方法，如：

1.设置环境变量的值

2.设置容器中的命令行参数

3.设置volume中的配置文件

